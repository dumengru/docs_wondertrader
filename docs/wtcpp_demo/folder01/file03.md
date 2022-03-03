# HFT回测进阶2: 源码解析

source: `{{ page.path }}`

```tip
建议先阅读 "HFT回测进阶1: 源码解析"

请使用最新版0.9dev源码(20220303)
```

## 本文主要解决问题

1. 回测如何加载数据
2. 回测如何加载策略

## replayer.prepare()

`WtBtRunner.cpp` 文件中两句代码最重要

```cpp
replayer.prepare();
replayer.run();
```

### 关键代码 handle_init()

进入 `bool HisDataReplayer::prepare()`有一句代码最重要

```cpp
_listener->handle_init();
```

#### 数据加载路径

我们通过打断点逐步查看他的执行路径

1.进入策略文件
`HftMocker::handle_init()` -> `HftMocker::on_init()` -> `WtHftStraDemo::on_init`

2.加载tick数据
 `WtHftStraDemo::on_init` -> `HftMocker::stra_get_ticks` -> `HisDataReplayer::get_tick_slice` -> `HisDataReplayer::checkTicks` -> `HisDataReplayer::cacheRawTicksFromCSV`(这里已经开始从csv文件加载tick数据了)
- 此处如果存在文件 "./storage/bin/ticks/CUSTOM.FX.EURUSD_tick_20220221.dsb" 则加载dsb文件, 否则加载 "./storage/csv/ticks/CUSTOM.FX.EURUSD_tick_20220221.csv" 并自动转存储为 "./storage/bin/ticks/CUSTOM.FX.EURUSD_tick_20220221.dsb"

```tip
这段csv转dsb逻辑似乎有点问题, csv数据列和tick结构体不好对齐, 最好还是按照文章 "数据压缩/解压" 自己将csv转dsb并放在对应位置
```

继续往下执行直到再次进入 `WtHftStraDemo.cpp`

3.加载bar数据
`WtHftStraDemo::on_init` -> ` HftMocker::stra_get_bars` -> `HisDataReplayer::get_kline_slice` -> `HisDataReplayer::cacheRawBarsFromCSV`
- 此处如果存在文件 "./storage/his/min1/CUSTOM/EURUSD.dsb" 则直接加载dsb文件, 否则加载 "./storage/csv/CUSTOM.FX.EURUSD_m1.csv" 并自动转存储为 "./storage/his/min1/CUSTOM/EURUSD.dsb"

继续往下执行直到再次进入 `WtHftStraDemo.cpp`

4.主动订阅tick数据(也比较关键, 策略中记得订阅)
`WtHftStraDemo::on_init` -> `HftMocker::stra_sub_ticks` -> `HisDataReplayer::sub_tick`

## replayer.run()

只有一句代码最重要
```cpp
if(!_main_key.empty())
{
    //如果订阅了K线，则按照主K线进行回放
    run_by_bars(bNeedDump);
}
```

### 主K回放

我们通过打断点逐步查看他的执行路径

1.进入策略文件`on_session_begin`
`HisDataReplayer::run_by_bars` -> `HftMocker::handle_session_begin` -> `HftMocker::on_session_begin` -> `_strategy->on_session_begin` (策略文件没有实现该方法, 到头了)

继续往下执行直到再次进入`HisDataReplayer.cpp`

2.进入策略文件`on_tick`
`HisDataReplayer::run_by_bars` -> `HisDataReplayer::replayHftDatas` -> `HftMocker::handle_tick` -> `HftMocker::on_tick` -> `HftMocker::on_tick_updated` -> `WtHftStraDemo::on_tick`

继续往下执行直到第一阶段tick数据回放完毕并再次进入`HisDataReplayer::run_by_bars`
3.进入策略文件`on_bar`
`HisDataReplayer::run_by_bars` -> `HisDataReplayer::onMinuteEnd` -> `HftMocker::handle_bar_close` -> `HftMocker::on_bar` -> `WtHftStraDemo::on_bar`

4.逻辑梳理完毕. 等待数据回放完毕即可