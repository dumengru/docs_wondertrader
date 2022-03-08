# HFT仿真进阶2: 策略初始化

source: `{{ page.path }}`

既然是进阶篇, 基础的内容就不赘述了. 本文的环境配置承接上篇环境准备篇

## 解决目标

本文主要解决的问题是策略初始化时 `on_init` 中调用的三个方法及相关的判断逻辑

1. `ctx->stra_get_ticks`
2. `ctx->stra_sub_ticks`
3. `ctx->stra_get_bars`


### stra_get_ticks

1.加载Tick切片数据路径

`HftStraBaseCtx::stra_get_ticks` -> `WtEngine::get_tick_slice` -> `WtDtMgr::get_tick_slice` -> `WtDataReader::readTickSlice`

2.`WtDataReader` 加载Tick数据逻辑

```cpp
// 加载Tick切片数据
WTSTickSlice* WtDataReader::readTickSlice(const char* stdCode, uint32_t count, uint64_t etime /* = 0 */)
{
	// 1. 获取合约标准码(合约类)
	CodeHelper::CodeInfo cInfo = CodeHelper::extractStdCode(stdCode);
	// 2. 获取合约对应的商品信息
	WTSCommodityInfo* commInfo = _base_data_mgr->getCommodity(cInfo._exchg, cInfo._product);
	// 3. 获取合约名称
	std::string stdPID = StrUtil::printf("%s.%s", cInfo._exchg, cInfo._product);
	// 4. 获取当前时间, 精确到毫秒
	uint32_t curDate, curTime, curSecs;
	if (etime == 0)
	{
		curDate = _sink->get_date();
		curTime = _sink->get_min_time();
		curSecs = _sink->get_secs();

		etime = (uint64_t)curDate * 1000000000 + curTime * 100000 + curSecs;
	}
	else
	{
		//20190807124533900
		curDate = (uint32_t)(etime / 1000000000);
		curTime = (uint32_t)(etime % 1000000000) / 100000;
		curSecs = (uint32_t)(etime % 100000);
	}
	/*
	如果不指定etime, 默认为 0, curDate和curTime是最新时间和日期,
	endTDate 会获取最新时间的交易日期
	curTDate也会获取最新时间的交易日期

	如果指定e_time, curDate和curTime是历史时间和日期
	endTDate 是历史时间的交易日期
	curTDate 是最新时间的交易日期

	之后如果两者不在同一天则从dsb加载历史Tick数据, 若在同一天则从 dmb 加载缓存Tick数据
	*/
	uint32_t endTDate = _base_data_mgr->calcTradingDate(stdPID.c_str(), curDate, curTime, false);
	uint32_t curTDate = _base_data_mgr->calcTradingDate(stdPID.c_str(), 0, 0, false);

	bool isToday = (endTDate == curTDate);
	// 6. 主力合约单独处理
	std::string curCode = cInfo._code;
	if (cInfo.isHot() && commInfo->isFuture())
		curCode = _hot_mgr->getRawCode(cInfo._exchg, cInfo._product, endTDate);
	else if (cInfo.isSecond() && commInfo->isFuture())
		curCode = _hot_mgr->getSecondRawCode(cInfo._exchg, cInfo._product, endTDate);

	//比较时间的对象
	WTSTickStruct eTick;
	eTick.action_date = curDate;
	eTick.action_time = curTime * 100000 + curSecs;
	// 如果在同一天(盘中只有dmb缓存文件)
	if (isToday)
	{
		// 7. 获取dmb文件的Tick数据映射, 如果没有就返回NULL, 如果有就返回切片数据
		TickBlockPair* tPair = getRTTickBlock(cInfo._exchg, curCode.c_str());
		if (tPair == NULL)
			return NULL;

		// ... 代码略
	}
	// 若不在同一天(收盘后会有dsb历史文件)
	else
	{
		std::string key = StrUtil::printf("%s-%d", stdCode, endTDate);
		// 9. 从dsb中加载数据切片
		auto it = _his_tick_map.find(key);
		// 代码略...
```

3.加载完Tick数据之后

如果加载成功, 就通过引擎订阅tick行情, 如果没有tick数据则tick行情不会被订阅

```cpp
WTSTickSlice* HftStraBaseCtx::stra_get_ticks(const char* stdCode, uint32_t count)
{
	WTSTickSlice* ticks = _engine->get_tick_slice(_context_id, stdCode, count);

	if (ticks)
		_engine->sub_tick(id(), stdCode);
	return ticks;
}
```

### stra_sub_ticks

`stra_sub_ticks` 是主动订阅Tick行情, 但是和通过 `stra_get_ticks` 被动订阅行情略有不同, `stra_sub_ticks` 会首先将品种名称添加到 `_tick_subs` 中, 如此在策略其他地方都可以通过 `ctx->` 接口调用

```cpp
void HftStraBaseCtx::stra_sub_ticks(const char* stdCode)
{
	/*
	 *	By Wesley @ 2022.03.01
	 *	主动订阅tick会在本地记一下
	 *	tick数据回调的时候先检查一下
	 */
	_tick_subs.insert(stdCode);

	_engine->sub_tick(id(), stdCode);
	log_info("Market Data subscribed: %s", stdCode);
}
```

### stra_get_bars

1.加载Bar切片数据路径

`HftStraBaseCtx::stra_get_bars` -> `WtEngine::get_kline_slice` -> `WtDtMgr::get_kline_slice` -> `WtDataReader::readKlineSlice`

和Tick数据加载路径一致

2.`WtDataReader` 加载Bar数据逻辑

```cpp

// 获取K线数据切片
WTSKlineSlice* WtDataReader::readKlineSlice(const char* stdCode, WTSKlinePeriod period, uint32_t count, uint64_t etime /* = 0 */)
{
	// 获取合约类, 商品信息, 标准码
	CodeHelper::CodeInfo cInfo = CodeHelper::extractStdCode(stdCode);
	WTSCommodityInfo* commInfo = _base_data_mgr->getCommodity(cInfo._exchg, cInfo._product);
	std::string stdPID = StrUtil::printf("%s.%s", cInfo._exchg, cInfo._product);

	// 判断是否可以加载历史Bar数据
	std::string key = StrUtil::printf("%s#%u", stdCode, period);
	auto it = _bars_cache.find(key);
	bool bHasHisData = false;
	if (it == _bars_cache.end())
	{
		/*
		 *	By Wesley @ 2021.12.20
		 *	先从extloader加载最终的K线数据（如果是复权）
		 *	如果加载失败，则再从文件加载K线数据
		 */
		bHasHisData = cacheFinalBarsFromLoader(key, stdCode, period);

		// 1. 判断是否可以从文件正常加载 dsb 数据(这一步会填充_bars_cache)
		if(!bHasHisData)
			bHasHisData = cacheHisBarsFromFile(key, stdCode, period);
	}
	else
	{
		bHasHisData = true;
	}

	// endTDate 和 Tick数据同样的的判断逻辑
	uint32_t curDate, curTime;
	// ...代码略

	//是否获取最新数据
	bool bHasToday = (endTDate == curTDate);
	// 主力合约判断逻辑
	if (cInfo.isHot() && commInfo->isFuture())
	// ...代码略

	// 若在同一天, 则读取 dmb 数据
	if (bHasToday)
	{
		WTSBarStruct bar;
		bar.date = curDate;
		bar.time = (curDate - 19900000) * 10000 + curTime;

		const char* curCode = _bars_cache[key]._raw_code.c_str();

		// 2. 读取实时的 dmb 数据
		RTKlineBlockPair* kPair = getRTKilneBlock(cInfo._exchg, curCode, period);
		if (kPair != NULL)
		{
			//读取当日的数据
			WTSBarStruct* pBar = NULL;
			// ...代码略
		}
	}

	// 3. 缓存数据不够, 判断需要从历史数据加载多少
	if (left > 0 && bHasHisData)
	{
		hisCnt = left;
		//历史数据，直接从缓存的历史数据尾部截取
		BarsList& barList = _bars_cache[key];
		hisCnt = min(hisCnt, (uint32_t)barList._bars.size());
		hisHead = &barList._bars[barList._bars.size() - hisCnt];//indexBarFromCache(key, etime, hisCnt, period == KP_DAY);
	}

	pipe_reader_log(_sink, LL_DEBUG, "His {} bars of {} loaded, {} from history, {} from realtime", PERIOD_NAME[period], stdCode, hisCnt, rtCnt);
	// 3. 返回数据切片
	if (hisCnt + rtCnt > 0)
	{
		// 3.1 创建历史数据切片
		WTSKlineSlice* slice = WTSKlineSlice::create(stdCode, period, 1, hisHead, hisCnt);
		// 3.2 添加缓存数据切片
		if (rtCnt > 0)
			slice->appendBlock(rtHead, rtCnt);
		return slice;
	}
	return NULL;
}
```

Bar数据加载逻辑和tick数据加载逻辑非常相似, 应该很容易理解.

3. 加载完Bar数据之后, 如果成功, 也会通过引擎订阅tick行情

```cpp
WTSKlineSlice* HftStraBaseCtx::stra_get_bars(const char* stdCode, const char* period, uint32_t count)
{
	std::string basePeriod = "";
	uint32_t times = 1;
	if (strlen(period) > 1)
	{
		basePeriod.append(period, 1);
		times = strtoul(period + 1, NULL, 10);
	}
	else
	{
		basePeriod = period;
	}
    // 加载Bar数据
	WTSKlineSlice* ret = _engine->get_kline_slice(_context_id, stdCode, basePeriod.c_str(), count, times);

    // 订阅tick
	if (ret)
		_engine->sub_tick(id(), stdCode);

	return ret;
}
```

## 总结

1. `stra_get_ticks` 和 `stra_get_bars` 会分别从历史dsb和当天dmb文件加载数据, 没有数据则会加载失败
2. `stra_get_ticks` 和 `stra_get_bars` **如果加载数据成功**则会被动订阅Tick行情
3. `stra_sub_ticks` 主动订阅某品种Tick行情, 且会保留在订阅列表中