# CTA仿真进阶篇2: 策略初始化

source: `{{ page.path }}`

为甚么策略初始化总是要单独讲解呢, 因为策略初始化对后续触发 `on_tick` 和 `on_shcedule` 和 `on_bar` 很重要

## on_init

在策略 `on_init` 中主要做了3件事

1. 获取 CFFEX.IC.2203 的tick数据(`code` 是通过配置文件中的参数传入的品种代码, 这里是指 `CFFEX.IC.2203`)
2. 获取 CFFEX.IC.2203 的bar数据
3. 主动订阅 CFFEX.IH.2203 tick数据

```cpp
void WtStraDualThrust::on_init(ICtaStraCtx* ctx)
{
	ctx->stra_log_info(fmt::format("回调 on_init, date: {}, time: {}", ctx->stra_get_date(), ctx->stra_get_time()).c_str());

	std::string code = _code;
	if (_isstk)
		code += "-";

	// 获取 CFFEX.IC.2203 的tick数据
	WTSTickSlice* ticks = ctx->stra_get_ticks(_code.c_str(), 30);
	if (ticks)
		ticks->release();

	// 获取 CFFEX.IC.2203 的bar数据
	WTSKlineSlice *kline1 = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
	if (kline1 == NULL)
	{
		//这里可以输出一些日志
		return;
	}
	kline1->release();

	// 主动订阅 CFFEX.IH.2203 tick数据
	ctx->stra_sub_ticks("CFFEX.IH.2203");
}
```

为什么控制台显示的 `on_tick` 回调只有 `CFFEX.IH.2203`, `on_bar` 回调只有 `CFFEX.IC.2203` 呢?(这里和 HFT 仿真区别很大)
接下来详细解析这三步都做了哪些事情

```tip
切记程序调试时打开 `QuoteFactory`
```

## stra_get_ticks

### 运行流程

1. 断点在策略文件 `on_init` 函数下 `ctx->stra_get_ticks`
2. 进入 "CtaStraBaseCtx.cpp", 调用 `_engine->get_tick_slice`
3. 进入 "WtEngine.cpp", 调用 `_data_mgr->get_tick_slice`
4. 进入 "WtDtMgr.cpp", 调用 `_reader->readTickSlice`
5. 进入 "WtDataReader.cpp", 调用  `WtDataReader::readTickSlice` 正式读取数据(该函数若是调用历史数据, 则从dsb加载历史Tick数据, 若是调用当天数据则从 dmb 加载缓存Tick数据)
6. 数据加载完毕回到 "CtaStraBaseCtx.cpp", 若数据加载成功则调用 `_engine->sub_tick`
7. 进入 "WtEngine::sub_tick", 将成功订阅的品种存放在 `_tick_sub_map` 中
8. 如果没有成功则不会通过引擎订阅tick数据
9. 函数结束

上述关键 6, 7点很重要

## stra_get_bars

1. 断点在 `ctx->stra_get_bars` 
2. 进入 `CtaStraBaseCtx.cpp`, 
3. 此处如果还没有主K, 则将当前品种设置为主K, 如果已有主K, 则会抛出异常, 因此 **CTA策略中只能有一个品种作为主K, 如果想订阅多个品种K线, 注意`isMain`, 这个参数, 订阅其他K线需要设置为false, (默认也是false)**
4. 继续向下执行, 调用 `_engine->get_kline_slice`
5. 进入 "WtEngine.cpp", 首先对周期进行了判断, **1min, 5min和1Day是WT内置的基础周期**, 最后调用 `_data_mgr->get_kline_slice`
6. 进入 "WtDtMgr.cpp", 首先会将品种存放到 `_subed_basic_bars` 中, 然后调用 `_reader->readKlineSlice`
7. 进入 "WtDataReader.cpp" 中正式读取数据, 首先会从 `extloader` 尝试加载数据, 然后会从文件中加载数据(具体细节参考文章"HFT仿真")
8. 数据加载完毕回到 "CtaStraBaseCtx.cpp", 若未加载成功, 则会跳过后续逻辑, 因此凡是需要订阅K线的策略(也包括HFT), 都必须打开 `QuoteFactory`
9. 如果数据加载成功, 继续向下调用 `_engine->sub_tick`
10. 进入 "WtEngine::sub_tick", 将成功订阅的品种存放在 `_tick_sub_map` 中, 因此**只要成功调用 `stra_get_bars`, 会默认订阅 tick 数据**
11. 函数结束

## stra_sub_ticks

1. 断点在 `ctx->stra_sub_ticks`
2. 进入 "CtaStraBaseCtx.cpp", 首先将品种名称存放在 `_tick_subs` 中, 然后调用 `_engine->sub_tick`, 后续逻辑和之前一样
3. 函数结束

这里和之前对比, 主要多出一步, 将品种名称放在 `CtaStraBaseCtx` 下的 `_tick_subs` 中, 这点很重要.

## 总结 CTA 和 HFT 策略初始化区别

订阅数据路径都已一模一样的, 唯一的区别是 "CtaStraBaseCtx.cpp" 和 "HftStraBaseCtx.cpp" 下的 `stra_get_bars` 方法中, 很明显 CTA 的更复杂, 首先会对主 K 做判断, 这点限制了 CTA 只能订阅一个品种的 K 线, 其次对数据是否获取成功做判断, 这点限制了 CTA 必须要有数据. HFT 对此都无要求.

cta

```cpp

WTSKlineSlice* CtaStraBaseCtx::stra_get_bars(const char* stdCode, const char* period, uint32_t count, bool isMain /* = false */)
{
	std::string key = StrUtil::printf("%s#%s", stdCode, period);
	if (isMain)
	{
		if (_main_key.empty())
		{
			_main_key = key;
			log_debug("Main KBars confirmed：%s", key.c_str());
		}
		else if (_main_key != key)
			throw std::runtime_error("Main KBars already confirmed");
	}

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

	WTSKlineSlice* kline = _engine->get_kline_slice(_context_id, stdCode, basePeriod.c_str(), count, times);
	if(kline)
	{
	// 代码略
	}
```

hft

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

	WTSKlineSlice* ret = _engine->get_kline_slice(_context_id, stdCode, basePeriod.c_str(), count, times);

	if (ret)
		_engine->sub_tick(id(), stdCode);

	return ret;
}
```