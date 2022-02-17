# WtStraDualThrust.h

source: `{{ page.path }}`

CTA策略demo

```cpp
#pragma once
#include "../Includes/CtaStrategyDefs.h"

class WtStraDualThrust : public CtaStrategy
{
public:
	WtStraDualThrust(const char* id);
	virtual ~WtStraDualThrust();

public:
	// 获取策略工厂名称
	virtual const char* getFactName() override;
	// 获取策略名称
	virtual const char* getName() override;
	// 通过配置文件初始化策略
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
	uint32_t	_count
	//合约代码
	std::string _code;
	bool		_isstk;
};
```

## WtStraDualThrust.cpp

```cpp

extern const char* FACT_NAME;

WtStraDualThrust::WtStraDualThrust(const char* id)
	: CtaStrategy(id)
{}


WtStraDualThrust::~WtStraDualThrust()
{}

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
	// 如果是股票, 添加后缀
	std::string code = _code;
	if (_isstk)
		code += "Q";
	
	// 获取K线信息
	WTSKlineSlice *kline = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
	if(kline == NULL)
	{
		// 这里可以输出一些日志
		return;
	}

	if (kline->size() == 0)
	{
		kline->release();
		return;
	}

	// 设置交易单位
	uint32_t trdUnit = 1;
	if (_isstk)
		trdUnit = 100;

	int32_t days = (int32_t)_days;

	// 获取价格相关信息
	double hh = kline->maxprice(-days, -2);
	double ll = kline->minprice(-days, -2);

	// 提取某个价格
	WTSValueArray* closes = kline->extractData(KFT_CLOSE);
	double hc = closes->maxvalue(-days, -2);
	double lc = closes->minvalue(-days, -2);
	double curPx = closes->at(-1);
	closes->release();		///!!!这个释放一定要做

	double openPx = kline->open(-1);
	double highPx = kline->high(-1);
	double lowPx = kline->low(-1);

	// 计算上下轨
	double upper_bound = openPx + _k1 * (max(hh - lc, hc - ll));
	double lower_bound = openPx - _k2 * max(hh - lc, hc - ll);

	// 获取合约信息
	WTSCommodityInfo* commInfo = ctx->stra_get_comminfo(_code.c_str());
	// 获取当前持仓
	double curPos = ctx->stra_get_position(_code.c_str()) / trdUnit;
	if(decimal::eq(curPos,0))
	{
		if(highPx >= upper_bound)
		{
			ctx->stra_enter_long(_code.c_str(), 2 * trdUnit, "DT_EnterLong");
			//向上突破
			ctx->stra_log_info("向上突破%.2f>=%.2f,多仓进场", highPx, upper_bound);
		}
		else if (lowPx <= lower_bound && !_isstk)
		{
			ctx->stra_enter_short(_code.c_str(), 2 * trdUnit, "DT_EnterShort");
			//向下突破
			ctx->stra_log_info("向下突破%.2f<=%.2f,空仓进场", lowPx, lower_bound);
		}
	}
	else if (decimal::gt(curPos, 0))
	{
		if(lowPx <= lower_bound)
		{
			//多仓出场
			ctx->stra_exit_long(_code.c_str(), 2 * trdUnit, "DT_ExitLong");
			ctx->stra_log_info("向下突破%.2f<=%.2f,多仓出场", lowPx, lower_bound);
		}
	}
	else if (decimal::lt(curPos, 0))
	{
		if (highPx >= upper_bound && !_isstk)
		{
			//空仓出场
			ctx->stra_exit_short(_code.c_str(), 2 * trdUnit, "DT_ExitShort");
			ctx->stra_log_info("向上突破%.2f>=%.2f,空仓出场", highPx, upper_bound);
		}
	}
	// 保存用户数据
	ctx->stra_save_user_data("test", "waht");

	// 这个释放一定要做
	kline->release();
}

// 策略初始化
void WtStraDualThrust::on_init(ICtaStraCtx* ctx)
{
	std::string code = _code;
	if (_isstk)
		code += "Q";
	WTSKlineSlice *kline = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
	if (kline == NULL)
	{
		//这里可以输出一些日志
		return;
	}

	kline->release();
}

void WtStraDualThrust::on_tick(ICtaStraCtx* ctx, const char* stdCode, WTSTickData* newTick)
{
	//没有什么要处理
}
```