# 数据进阶: 全天候环境准备

source: `{{ page.path }}`

```tip
如果你只是在盘中正常交易时间段做测试, 本章内容可忽略

本章内容多处牵涉到**非正常**修改源码部分, 小白慎入!
```

WT对数据时间的检查极为严苛, 这对实盘自然是非常好的, 但是却给非交易时段做测试带来诸多不便, 本章内容是本人在非交易时段做测试时的踩坑记录.

执行 `TestDtPorter` 下的 main.cpp

## 1. 规避状态机检测

在之前的文章 "UDP数据转发" 中提到了如何随时获取数据, 之后在 "状态机详解" 中又提到了状态机工作机制. 随着测试的深入, 我发现自己可能随时需要某个品种的 dmb和dsb数据文件(仿真测试需要), 这就不得不使用状态机的收盘作业机制帮我准备数据(即我需要随时模拟收盘作业)

### 如何规避状态机检测呢?
我们仍然使用openctp的7*24小时环境, 同时在 "状态机详解"的基础上修改三个文件

**sessions.json**

```json
{
    "TTS24":{
        "name":"TTS24测试",
        "offset": -480,
        "auction":{
            "from": 802,
            "to": 803
        },
        "sections":[
            {
                "from": 803,
                "to": 758
            }
        ]
    }
}
```

`offset` 即将所有时间向前偏移480分钟, 因为wt处理夜盘机制是通过时间偏移实现的.

`auction` 集合竞价偏移后变为 1-2(0点01分用数字表示)

`sections` 交易时段偏移后变为 2-2359(0点02分到23点59分), 这样几乎覆盖全天交易时段

**commodities.json示例**

```json
{
    "CFFEX": {
        "IC": {
            "covermode": 0,
            "pricemode": 0,
            "category": 1,
            "precision": 1,
            "pricetick": 0.2,
            "volscale": 200,
            "name": "中证֤",
            "exchg": "CFFEX",
            "session": "TTS24",
            "holiday": "CHINA"
        }
    }
}
```
将你想使用的品种 `session` 字段修改和 `session.json` 一一对应

**statemonitor.yaml**

```yaml
TTS24:
    closetime: 2356     # 停止获取行情时间
    inittime: 0001      # 开盘前准备时间
    name: TTS24测试
    proctime: 2357      # 处理收盘作业时间
```

最后记得注意 `dtcfg.yaml` 文件需要将 `allday:` 配置字段删掉或设为 `false`

然后运行程序即可, 当电脑时间大于 `closetime` 时便停止获取数据, 当时间大于`proctime` 时便开始收盘作业, 成功运行截图参考文章"状态机详解"

### 节假日规避

上述配置还是只能做到5*24小时, 因为状态机还会自动过滤周末和节日

如果是在周末测试, 通过简单的文件配置是行不通的, 还必须修改源码.

在 "WtDtCore/StateMonitor.cpp" 中修改 `StateMonitor::run()` 方法, 将 `isAllHoliday` 初始状态由 `true` 改为 `false`.

```cpp
bool isAllHoliday = false;  // 默认是true
```

```danger
!!!非必要别改源码, 测试完记得改回来

!!!非必要别改源码, 测试完记得改回来

!!!非必要别改源码, 测试完记得改回来
```

## 2. 规避仿真时间校验

在文章 "对接openctp" 中我提到如何在wt中使用openctp做仿真交易, 当时只是填充了 `trading_date` 字段, 然而随着源码阅读的深入, 简单的填充已经无法满足 7*24 小时环境测试了, 因为在仿真交易中会对行情的 `action_time` 字段和电脑当前时间字段做校验, 试想在非交易时段, 服务器回放的都是历史数据, `action_time` 自然是小于当前时间的, 于是程序便会一直卡住, 不再继续往下了.(**虽然给测试造成了很大阻碍, 但是在实盘中还是非常必要的**)

### 如何规避仿真的时间校验呢?

在 `ParserCTP::OnRtnDepthMarketData` 方法末尾添加如下代码

```cpp
/* ----- 添加代码0 ------*/
if (m_sink && m_uTradingDate == 0)
{
    /* 对接openctp, 添加 trading_date 字段
    const char* session = pCommInfo->getSession();
    uint32_t trading_date = m_sink->getBaseDataMgr()->calcTradingDate(session, actDate, actTime);
    quote.trading_date = trading_date;
    */

    // 在7*24测试环境下跳过框架中的时间校验
    uint32_t curDate, curTime;
    TimeUtils::getDateTime(curDate, curTime);
    quote.action_date = curDate;
    quote.action_time = curTime;
    quote.trading_date = curDate;

}
/* ----- 添加代码1 ------*/
if(m_sink)
    m_sink->handleQuote(tick, 1);
```

1. 在文章 "对接openctp" 中也是修改的这里, 因为openctp不提供trading_date字段, 我便自己给填充了
2. 这里直接用当前时间和日期填充 `action_date` 和 `action_time`(顺便直接填充了 `trading_date`) 便可以规避掉时间校验

```danger
!!!非必要别改源码, 仅供测试使用

!!!非必要别改源码, 仅供测试使用

!!!非必要别改源码, 仅供测试使用
```