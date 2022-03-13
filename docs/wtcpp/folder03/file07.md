# CTA仿真完整篇2: 策略初始化

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

## stra_get_ticks

## stra_get_bars

## stra_sub_ticks