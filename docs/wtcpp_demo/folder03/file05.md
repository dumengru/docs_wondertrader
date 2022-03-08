# HFT仿真进阶3: 策略回调

source: `{{ page.path }}`

承接上文 "HFT仿真进阶2: 策略初始化"

## 解决目标

本文主要解决的问题是仿真时的策略运行逻辑, 包括如何回调各类策略接口

## 仿真运行流程

### 结构梳理

涉及HFT策略仿真的文件主要有以下几个

ParserCTP | 行情解析器 | 解析不同行情接口
TraderCTP | 交易解析器 | 解析不同交易接口


"WtCore"文件夹

ParserAdapter | 行情适配器 | 适配行情接口
TraderAdapter | 交易适配器 | 适配交易接口
WtHftEngine | HFT引擎 | 承上: 继承主引擎并实现行情响应和订单等重要接口, 启下: 调用HFT步进器执行系列动作
WtHftTicker | 步进器 | 主要处理Tick响应
HftStraBaseCtx | HFT环境管理器 | 为HFT策略提供接口及各类环境
HftStraContext | HFT上下文管理器 | 承上: 继承环境管理器并实现策略相关接口, 启下: 回调策略接口
HftStrategyMgr | 加载策略dll文件
WtEngine | 主引擎 | 继承 `WtPortContext` 为策略组合提供环境支持, 继承 `IParserStub` 触发Tick事件 
WtDtMgr | 数据管理器 | 为策略提供数据支持

"WtDataStorage"文件夹

WtDataReader 数据读取器 | 数据读取, 涉及回调on_bar事件


### Tick数据传递流程

顺序执行用 "->"
在一个方法中并列执行的用 "=" 

1. 市场推送一笔Tick数据 -> 
2. `ParserCTP::OnRtnDepthMarketData`: 获取行情数据转为自定义结构体数据, 向后传递 -> 
3. `ParserAdapter::handleQuote`: 交易所过滤检查, 交易时间检查, 交易合约检查, 将原生品种码转为标准码, 向后传递 ->
4. `WtHftEngine::handle_push_quote`: 向后传递->
5. `WtHftRtTicker::on_tick`: 行情时间检查, 触发 `trigger_price` ->
6. `WtHftRtTicker::trigger_price`: 包含主力合约处理, 向后传递 ->
7. `WtHftEngine::on_tick`: 同时向后传递三次 ->
8. `WtEngine::on_tick`: 检查信号触发并保存数据, 停止传递 =
9. `WtDtMgr::handle_push_quote`: 保存数据, 停止传递 =
10. `HftStraContext::on_tick`: 检查是否订阅, 向后传递 ->
11. `WtHftStraDemo::on_tick`: 若订阅, 执行策略 `on_tick` =
12. `HftStraBaseCtx::on_tick`: 保存"userdata"文件夹数据, 停止传递

在第5步 `WtHftRtTicker::on_tick` 中

1. 这里的时间校验迫使我在7*24小时仿真环境中修改了action_time等字段
2. 这里也可能触发Bar线闭合事件(非正常闭合)

### on_bar闭合流程

`WtHftRtTicker::run()` 中线程死循环执行中...
当出现新的一分钟, 正常会顺序执行三句代码

```cpp
_store->onMinuteEnd(_date, thisMin);
_engine->on_minute_end(_date, thisMin);
_engine->on_session_end();
```

#### onMinuteEnd

在一个方法中遍历执行用 "=>"

1. `WtDataReader::onMinuteEnd`: 遍历品种, 检查K线缓存, 向后执行 =>
2. `WtDtMgr::on_bar`: 检查周期, 并将所有 `on_bar` 事件插入 `_bar_notifies`, 向后执行 ->
3. `WtDtMgr::on_all_bar_updated`: 遍历品种, 执行on_bar =>
4. `WtHftEngine::on_bar`: 品种检查, 向后执行 ->
5. `HftStraContext::on_bar`: 向后执行 ->
6. `WtHftStraDemo::on_bar`: 执行策略 `on_bar` =
7. `HftStraBaseCtx::on_bar`: 保存"userdata"文件夹数据

在第1步 `WtDataReader::onMinuteEnd` 中

1. 由于会对K线缓存做检查, 若K线缓存没有更新, 则无法继续向后执行, 因此如果HFT策略中用到Bar数据, 必须开启 WtDtPorter 一直更新缓存数据

在第4步 `WtHftEngine::on_bar`中

1. 若策略文件中若未订阅K线, 则无法通过品种检查, 不会执行on_bar, 因此策略文件是否订阅K线对触发 `on_bar` 也很重要

#### on_minute_end

原本会执行 `on_schedule`, 在 HFT策略中被去掉了

#### on_session_end

收盘时间
1. `WtHftEngine::on_session_end`: 向后执行 ->
2. `WtEngine::on_session_end`: 资金结算, 保存数据 =
3. `HftStraContext::on_session_end`: 向后执行 ->
4. 执行策略 `on_session_end`
5. 保存日志

## 仿真流程总结

1. 策略编写需要注意是否订阅K线数据或Tick数据
2. 注意策略回调 `on_bar` 的条件和逻辑
3. 若HFT策略不需要K线数据(不用on_bar), 则不需要打开`WtDtPorter`
4. 若HFT策略需要K线数据, 则必须打开`WtDtPorter`
