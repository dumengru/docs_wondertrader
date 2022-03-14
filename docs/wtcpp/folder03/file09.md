# CTA仿真完整篇3: 开仓

source: `{{ page.path }}`

之前铺垫了那么多文章, 其实都没什么卵用, 只有通过计算机将订单发送到交易所的那一刻, 我们才算真正完成一笔量化交易.

本篇将要剖析 CTA 引擎如何完成一笔开仓操作.

## 准备策略

为了方便调试, 我们需要先修改一下策略源码.

**WtStraDualThrust.h**

```cpp
#pragma once
#include "../Includes/CtaStrategyDefs.h"

class WtStraDualThrust : public CtaStrategy
{
public:
	WtStraDualThrust(const char* id);
	virtual ~WtStraDualThrust();

public:
	virtual const char* getFactName() override;

	virtual const char* getName() override;

	virtual bool init(WTSVariant* cfg) override;

	virtual void on_schedule(ICtaStraCtx* ctx, uint32_t curDate, uint32_t curTime) override;

	virtual void on_init(ICtaStraCtx* ctx) override;

	virtual void on_tick(ICtaStraCtx* ctx, const char* stdCode, WTSTickData* newTick) override;


private:
	//指标参数
	double		_k1;
	double		_k2;
	uint32_t	_days;

	//数据周期
	std::string _period;
	//K线条数
	uint32_t	_count;

	//合约代码
	std::string _code;

	bool		_isstk;
};
```

**WtStraDualThrust.h**

```cpp
#include "WtStraDualThrust.h"

#include "../Includes/ICtaStraCtx.h"

#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/WTSVariant.hpp"
#include "../Includes/WTSDataDef.hpp"
#include "../Share/decimal.h"

extern const char* FACT_NAME;

//By Wesley @ 2022.01.05
#include "../Share/fmtlib.h"

WtStraDualThrust::WtStraDualThrust(const char* id)
	: CtaStrategy(id)
{
}


WtStraDualThrust::~WtStraDualThrust()
{
}

const char* WtStraDualThrust::getFactName()
{
	return FACT_NAME;
}

const char* WtStraDualThrust::getName()
{
	return "DualThrust";
}

bool WtStraDualThrust::init(WTSVariant* cfg)
{

	if (cfg == NULL)
		return false;

	_days = cfg->getUInt32("days");
	_k1 = cfg->getDouble("k1");
	_k2 = cfg->getDouble("k2");

	_period = cfg->getCString("period");
	_count = cfg->getUInt32("count");
	_code = cfg->getCString("code");

	_isstk = cfg->getBoolean("stock");

	return true;
}

void WtStraDualThrust::on_schedule(ICtaStraCtx* ctx, uint32_t curDate, uint32_t curTime)
{
	ctx->stra_log_info(fmt::format("回调 on_schedule, date: {}, time: {}", ctx->stra_get_date(), ctx->stra_get_time()).c_str());
	std::string code = _code;
	if (_isstk)
		code += "-";

	uint32_t trdUnit = 1;
	if (_isstk)
		trdUnit = 100;

	double curPos = ctx->stra_get_position(_code.c_str()) / trdUnit;
	if (decimal::eq(curPos, 0))
	{
		ctx->stra_enter_long(_code.c_str(), 1 * trdUnit, "DT_EnterLong");
		//向上突破
		ctx->stra_log_info(fmt::format("空仓, 买入 1 手多单").c_str());
	}
	else if (decimal::le(curPos, 2))
	{
		//多仓出场
		ctx->stra_enter_long(_code.c_str(), 1 * trdUnit, "DT_EnterLong");
		ctx->stra_log_info(fmt::format("有1手多单, 再买开1手多单").c_str());
	}
	else if (decimal::gt(curPos, 0))
	{
		//多仓出场
		ctx->stra_enter_short(_code.c_str(), 1 * trdUnit, "DT_EnterShort");
		ctx->stra_log_info(fmt::format("有2手多单, 开1手空单").c_str());
	}

	ctx->stra_save_user_data("test", "what");
}

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

	// 主动订阅 CFFEX.IC.2203 tick数据
	ctx->stra_sub_ticks(_code.c_str());
}

void WtStraDualThrust::on_tick(ICtaStraCtx* ctx, const char* stdCode, WTSTickData* newTick)
{
	ctx->stra_log_info(fmt::format("回调 on_tick, date: {}, time: {}", ctx->stra_get_date(), ctx->stra_get_time()).c_str());
}
```

代码逻辑很简单, 只订阅了一个品种, 并且直接在 `on_schedule` 中下单. 记得将修改的策略文件重新生成并放在对应的文件夹下.

## stra_enter_long

运行 `QuoteFactory.exe`, 并将断点打在 `WtStraDualThrust::on_schedule` 方法下的 `ctx->stra_get_position` 上.

```tip
如果你进行了多次测试, 建议删除 "WtRunner/generated" 文件夹, 因为它会记录之前测试时的持仓
```

1. 执行 `ctx->stra_enter_long`
2. 进入 "CtaStraBaseCtx.cpp", 首先会执行 `_engine->sub_tick` 订阅被下单的品种, 接着向下
3. 这里有三种下单方式: 市价单, 限价单和停止单, 这里我们会执行市价单, 执行 `append_signal`
4. 进入 `CtaStraBaseCtx::append_signal` 添加品种信号: 这里会将下单数量, 价格(最新价), 用户标签, 下单时间(精确到秒), 以及是否在调度中(on_schedule开始就会设置在调度中, 因此如果不是在 `on_schedule` 接口下的订单回是false)
5. 所有订单的信息都保存在 `_sig_map` 中, 然后将下单信号写入本地文件 "signals.csv"
6. 执行 `save_data` 保存各种数据
7. 至此, `stra_enter_long` 方法执行完毕

可是我们目前并有发送订单到交易所

## CtaStraBaseCtx

1. 继续向下执行, 回到 "WtStraDualThrust.cpp", 结束 `WtStraDualThrust::on_schedule`.
2. 进入 "CtaStraContext.cpp", 结束
3. 进入 "CtaStraBaseCtx.cpp", 向下执行至结束
4. 进入 "WtCtaEngine.cpp", 向下执行至**处理组合理论部位** `append_signal`
5. 进入 `WtEngine::append_signal`, 这里会将下单数量和时间保存到 `_sig_map`
6. 接着向下走到 `auto& m : _pos_map`, 由于这次首次下单, 持仓为空, 所以直接跳过
7. 接着向下执行 `_exec_mgr.set_positions` 
8. 进入 `WtExecMgr.cpp`, 这个是执行管理器, 同时管理多个执行器, 直接走到 `executer->set_position`
9. 进入 "WtDistExecuter.cpp", 将品种及仓位更新信息保存在 `_target_pos` 中
10. 调度器逻辑执行完毕
11. 继续向下执行回到 "WtCtaTicker.cpp"

