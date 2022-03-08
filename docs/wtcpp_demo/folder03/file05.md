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
WtEngine | 主引擎 | 承上启下: 继承 `WtPortContext` 为策略组合提供环境支持,继承 `IParserStub` 触发Tick事件
WtDtMgr | 数据管理器 | 为策略提供数据支持

### Tick数据传递流程

市场推送一笔Tick数据 -> `ParserCTP::OnRtnDepthMarketData` -> `ParserAdapter::handleQuote` -> `WtHftEngine::handle_push_quote` -> `WtHftRtTicker::on_tick` = `WtHftRtTicker::trigger_price` -> `WtHftEngine::on_tick` `WtEngine::on_tick` = `WtDtMgr::handle_push_quote` = `HftStraContext::on_tick` -> `WtHftStraDemo::on_tick`


