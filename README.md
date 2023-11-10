# Neilyst
基于Python和ccxt的全市场量化回测框架

## 1. 简介
这将是我本人的量化回测框架，主要功能将包括数据获取，数据落盘，数据清洗，时间序列聚合，指标计算，数据可视化，策略评估等。至于策略的运行引擎部分，实现有些过于复杂，暂时可以手动，以后如果有机会再慢慢实现。

## 2. 文件层级

Neilyst/
|-- neilyst/

|   |-- BaseNeilyst         # 定义属性

|   |-- DataManager
|   |   |-- fetcher     # 数据获取功能
|   |   |-- cleaner     # 数据清洗功能
|   |   |-- aggregator  # 时间序列聚合功能
|
|   |-- Analytics
|   |   |-- indicators  # 指标计算功能
|   |   |-- visualizer  # 数据可视化功能
|
|   |-- Strategy
|   |   |-- evaluator   # 策略评估功能
|   |   |-- example_strategy  # 示例策略
|
|-- tests/                 # 单元测试和其他测试
|   |-- data_manager/
|   |-- analytics/
|   |-- strategy/
|
|-- config/
|   |-- settings.py        # 配置文件，例如API密钥，数据库配置等
|
|-- setup.py               # Python包的安装和配置脚本
|-- README.md

## 3. User Dairy

对于数据获取，我希望能够简单的设定币种，现货还是合约，时间起点和终点，时间周期，数据类型这几个变量。就可以拉下来数据，自动整理为dataframe格式，并可以通过show方法展示部分数据，以及可以通过save方法落盘。落盘的地址默认为当前目录下，也可以手动在config文件中设置。交易所的名称和其他弱相关变量，最好也能默认指定而不是每次都需要手动输入。对于已经落盘的数据，应当可以通过load方法直接读取，并输出相关信息，例如开始和结束的时间，时间周期，币种，数据类型等等。这个load应该需要更加智能一些，例如如果只键入币种的话应该默认读取所有该币种的数据，如果时间周期一致的话。

在进行数据分析时，我希望建立一个对象就行。例如btc=neilyst(), 然后使用btc.fetch()进行数据获取。获取之后可以使用btc.indicators().MA()计算其指标。这样的话是否应该重新建立一个基类，然后之前的fetch类和之后所有类都应该是这个基类的子类。

那么我在此应该重新规划一下我的基类。我这个基类的主要作用应该是定义属性，然后使用其他的方法再获取这些属性。那么其主要属性应该有ohlcv_data, 用来保存原始数据。exchange_name，保存交易所名字，这个交易所名字可以不写在config里面，而是初始化时去定义。可以在注释中注明一些常用的交易所关键字。proxy和timeout可以继续写在config里面，因为这些设置写好了就不用改动了。还应该加上一个dataframe，indicators，用来保存各个指标的值。策略在编写开始时，可以生成一个data变量，这个变量相当于是ohlcv_data和indicators的拼接，可以方便策略编写时候调用各个数据。但是这个变量可以不保存在属性当中以节省内存，但是应该给出对应的方法来进行拼接，然后直接返回dataframe。

所以对应的策略编写过程其实就是一个对data迭代的过程，如果满足策略的信号，则应该有对应的方法来记录当前的时间戳，即data这个dataframe的index，并且记录当前的开仓方向，仓位，剩余现金。注意，每次开仓时，应保持一个不变，总资产不变。总资产=剩余现金+仓位*开仓时的价格。开多时，剩余现金减少，用以买仓位，仓位为正。平仓时卖掉仓位，将卖掉所得的钱数加回到剩余现金上。开空时，剩余现金增加，因为视为提前卖掉了仓位，此时仓位为负。但相加起来的总资产依然不变。平仓时，视为将之前提前卖掉的仓位买回来，此时剩余现金相对于开空仓后应该是减少的，并且把仓位平掉。所以开仓方向也可以不用去记录，可以用仓位的正负来直接表示。但是在持仓时的总资产依然是一个值得记录的变量，可以直观的看出该策略运行时的总资产变化情况。

在策略运行结束时，应当自动统计胜率，盈亏比，最大回撤，收益率。。。等等评价指标。还应该将每个交易的的点位，收益等等可视化。目前有一个基础的可视化方案，第一张图记录标的价格曲线，用灰色表示不开仓，红色表示开空，绿色表示开多。并且在同一张图上记录收益曲线，即总资产曲线。第二张图可视化仓位。最好能够在图上表明相关图例。这是默认方案。用户还应该能够自主画出单独的收益曲线，价格曲线等等。

对于整个策略的运行引擎的实现，首先应该设定一些参数，例如手续费，滑点比率，初始资金，作为初始参数。这些参数其实可以写在config中，因为不太改变。另外还需要初始化一些回测中可能使用到的参数，当前仓位，总资产，交易记录，仓位历史记录。

其中，对于交易记录这个参数，应该是一个dataframe，index是时间，至少有以下几列，symbol，方向(开多，开空，平多，平空，止盈，止损)，价格（即完成交易的价格），数量，已实现盈亏。

对于仓位历史记录，每开一笔仓，则会有一个未完成的仓位历史记录，直到该仓位全部平仓，本条历史记录才完成。这个参数也应该是一个dataframe，其index是时间，列名有symbol，方向，开仓时间，开仓价格，开仓数量，平仓时间，平仓均价，平仓盈亏。

所以，在数据迭代时，根据策略的条件，触发的操作应该有开多（long），开空（short），平仓（close），止盈(take_profit), 止损（stop_loss）。这五种操作应该对应着strategy类的五个方法，每种方法触发时，应该在self.data(蜡烛图数据和指标数据拼起来的大dataframe)，的对应行中标记出开平仓方向，仓位数量。并且更新交易纪律和仓位历史记录。

平仓时更新总资产数量。

每次迭代一组新的数据时，应该先检查当前仓位，如果有，看看是否平仓。所以data这个数据总表，应该还包含当前仓位，总资产，余额这三个列，来辅助策略信号生成。

2024.11.2：
还应该考虑全市场的回测思路，本框架还应该提供一些全市场回测的工具。

2024.11.6：
策略运行时应设定手续费率和滑点比率

2024.11.8:
平均持仓时间，交易次数，平均开仓间隔

添加一个持仓量的图

2024.11.9:
添加一个开仓曲线和各个指标的图
策略运行中，要能够监控之前的成交价，用来监控浮盈浮亏，以控制止盈止损