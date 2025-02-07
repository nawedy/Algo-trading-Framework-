.. _`Scanners`:

Scanners
========

As explained in the `Concepts` section, scanners filter stocks of interest from the stock universe.
When the `trader` application starts, it reads the `tradeplan.toml` file and looks at the **[[scanners]]** section to identify the classes of scanners to be instantiated and executed. They are executed by the order of appearance in the `tradeplan` file. The framework comes with a generic, momentum scanner, but it is advised to be used as a reference for designing your own scanners.

Some configuration parameters are generic to all scanners, and some may be specific for your own scanner.

No Liability Disclaimer
-----------------------
Any example, or sample is provided for training purposes only.
You may choose to use it, modify it or completely
disregard it - `LiuAlgoTrader` and its authors bare
no responsibility to any possible loses from using
a scanner, strategy, miner, or any other part of `LiuAlgoTrader` (on the other hand, they also won't
share your profits, if there are any).

Developing a Scanner
--------------------

When developing your scanner, you first need to import base-class `Scanner`:

.. code-block:: python

   import alpaca_trade_api as tradeapi
   from liualgotrader.scanners.base import Scanner

Creating a scanner involves:

1. Declare a class object that is inherited from base-class `Scanner`,
2. Overwrite both the `__init__()` and `run()` methods. `__init__()` is called during initialization, and is passed the configuration parameters from the `tradeplan.toml` file. The `run()` is called when the framework is ready to execute the scanner.

.. _base:
    https://github.com/amor71/LiuAlgoTrader/blob/master/liualgotrader/scanners/base.py

The `__init__()` function
*************************

A Generic implementation for the `__init__()` function:

.. code-block:: python

    def __init__(
        self,
        name: str,
        data_loader: DataLoader,
        recurrence: Optional[timedelta],
        target_strategy_name: Optional[str],
        data_source: object = None,
    )


The run() function
******************

The `run()` method is at the heart of the Scanner. The skeleton implementation:

.. code-block:: python

    async def run(self, back_time: datetime = None) -> List[str]:


The run() function returns a list of stock symbols which the framework needs to track. back_time is used by the `backtester` to simulate past time-stamps, allowing developer to test various scenarios before deployment.

.. code-block:: python

    self.data_loader

Holds the generic interface for querying the stock universe. 

Using DataLoader() class
************************

The DataLoader() class is core for writing scanners (as well as strategies). The class wraps on top of `pandas.DataFrame()` class which supports automatically downloading data.

For example, to load recent `ohlc` data for Apple Inc all you can do:

.. code-block:: python

    apple_ohlc = self.data_loader['AAPL'][-1] 

Notes:

1. Data will be loaded based on the time-scale (min/hour/day) configured by the LiuAlgoTrader framework. During trading you can expect per-minute time-scale, during back-testing you can select your desired time-frame,
2. Data is loaded from the data-providers configured in the `DATA_CONNECTOR` environment variable, the framework canonizes column names and guarantees to include at least open-high-low-close data, as well as volume. Different data providers may include additional details (such as VWAP, average, count and more),
3. Calling `self.data_loader.symbol_data` grants direct access to the `DataFrame` class,
4. Calling `self.data_loader.data_api` grants access to the instance of the `DataAPI` class, that provides an abstraction for the different data-providers. Using this class, the scanner can get a list of available / tradable symbols. For more details, check the implementation of the momenutum_ scanner.

.. _momenutum:
    https://github.com/amor71/LiuAlgoTrader/blob/master/liualgotrader/scanners/momentum.py#L82




Behind the scenes
*****************
As a developer, I hate 'magic' that happens without my understanding, hence it's important for me to detail the inner workings of the framework. All scanners run inside a dedicated process. When the process is executed, it reads the configuration files specific to the scanners, and creates an `asyncio` task per scanner. In fact, it's up to the scanners to make sure they play along nicely and not starve each other with overly long calculations.

The scanner task is to execute the `run()` method to determine a list of filtered stocks and transmits them over a Queue to the `producer` process. The task would then `sleep()` for the duration of the `recurrence` parameter (or just run once, if that parameter is not present).

Once the producer receives a list of selected symbols, it will register them for events on the `Polygon.io` data-stream, and persist them in database with the timestamp of receiving the selected stocks. This data are then used by the `backtester` application to replicate the real conditions presented to a strategy.

Example
*******

An example of *my_scanner.py*:

.. literalinclude:: ../../examples/scanners/my_scanner.py
  :language: python
  :linenos:


Configuring a custom scanner in the *tradeplan* TOML file is as easy as:

.. code-block:: none

    [scanners]
        [scanners.MyScanner]
            filename = "my_scanner.py"

            my_arg1 = 30000
            my_arg2 = 3.5

Upon execution, **trader** application will look for *my_scanner.py* and
instantiate the `MyScaner` class, and then call it with the arguments defined
in the `tradeplan` configuration file, while adding the trade-api object.

Advanced
********

Scanners can "direct" symbol picks to a specific strategy. To direct symbol picks, include `target_strategy_name` in the configuration file. That parameter, if present, will be passed along from the scanner process to the producer process, and later to the consumer process to make sure only the relevant strategy receives the pick.

Normally, the producer subscribes to all Polygon.io available events per picked stock. However, to improve performance, if the strategies do not make use of per-second events, or quotes or trades, it's recommended to select only the relevant events in the configuration file:

.. code-block:: bash

    events = ["second", "minute", "trade", "quote"]

Lastly, if configuration parameter `test_scanners` is set to `true`. The `trader` application will only execute the scanners without running strategies for debugging purpose.



Momentum Scanner
----------------
Scanners are defined in the *tradeplan.toml* TOML
configuration file under the section **[[scanners]]**. Note that
several Scanners may run, at different schedules and
identify new stocks that needs to be fed into the Strategies
pipeline.

The **[scanners.momentum]** momentum scanner has several
properties:

+------------------+-----------------------------------------------+
| Name             | Description                                   |
+------------------+-----------------------------------------------+
| provider         | stock data provider: *polygon*, *finnhub*     |
+------------------+------------------------+----------------------+
| min_volume       | scan for tickers with minimal volume since    |
|                  | day start                                     |
+------------------+-----------------------------------------------+
| min_gap          | tickers gaping up in percentage (e.g. 3.5)    |
+------------------+-----------------------------------------------+
| min_last_dv      | minimum last day dollar x volume              |
+------------------+-----------------------------------------------+
| min_share_price  | minimum stock price                           |
+------------------+-----------------------------------------------+
| max_share_price  | maximum stock price                           |
+------------------+-----------------------------------------------+
| from_market_open | number of minutes to wait, since market open  |
|                  | before scanning for relevant stocks           |
+------------------+-----------------------------------------------+
| max_symbols      | maximum number of symbols to scan for         |
+------------------+-----------------------------------------------+
| recurrence       | frequency of re-running scanner, if not       |
|                  | present, run only once                        |
+------------------+-----------------------------------------------+

**Notes**

- Unless specified, the trader applications scans & trades a limited number of stocks, however that limitation can be overwritten by **LIU_MAX_SYMBOLS** env variable.