# CTA仿真完整篇2: 初始化

source: `{{ page.path }}`

```tip
在测试过程中发现: 
1. 给的demo配置文件有错误, 因此上一篇文章中出现的配置文件内容尽量复制粘贴

2. 源码也有错误, 具体在 "WtRunner.cpp" 文件222行, 应该是 `initExecuters`(时间20220320, 该问题已提交issue, 注意检查自己下载的源码是否已修改)
```

## 修改策略文件

为了方便测试(这句话已经重复N遍), 首先需要修改策略文件.

修改项目"Plugins/WtCtaStraFact/DualThrust/WtStraDualThrust.cpp", 文件内容如下(在上一篇文章的基础上, 简化`on_schedule`内容, 直接下单.)

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
	ctx->stra_log_info(fmt::format("策略回调on_schedule, {}", ctx->stra_get_time()).c_str());

	std::string code = _code;
	if (_isstk)
		code += "-";
	WTSKlineSlice *kline = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
	if(kline == NULL)
	{
		//这里可以输出一些日志
		return;
	}

	if (kline->size() == 0)
	{
		kline->release();
		return;
	}

	//这个释放一定要做
	kline->release();

	// 每笔交易手数
	uint32_t trdUnit = 1;
	if (_isstk)
		trdUnit = 100;

	double curPos = ctx->stra_get_position(_code.c_str()) / trdUnit;
	if(decimal::eq(curPos,0))
	{
		ctx->stra_enter_long(_code.c_str(), 2 * trdUnit, "DT_EnterLong");
		//向上突破
		ctx->stra_log_info(fmt::format("多仓进场").c_str());

	}
	else if (decimal::gt(curPos, 1))
	{

		//多仓出场
		ctx->stra_exit_long(_code.c_str(), 2 * trdUnit, "DT_ExitLong");
		ctx->stra_log_info(fmt::format("多单离场").c_str());

		ctx->stra_save_user_data("test", "waht");
	}

}

void WtStraDualThrust::on_init(ICtaStraCtx* ctx)
{
	ctx->stra_log_info(fmt::format("策略回调on_init, {}", ctx->stra_get_time()).c_str());

	std::string code = _code;
	if (_isstk)
		code += "-";
	WTSKlineSlice *kline = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
	if (kline == NULL)
	{
		//这里可以输出一些日志
		return;
	}

	ctx->stra_sub_ticks(code.c_str());

	kline->release();
}

void WtStraDualThrust::on_tick(ICtaStraCtx* ctx, const char* stdCode, WTSTickData* newTick)
{
	//没有什么要处理
	ctx->stra_log_info(fmt::format("策略回调on_tick, {}", ctx->stra_get_time()).c_str());
}
```

修改完策略源码后, 右击项目"WtCtaStraFact", 重新生成, 然后将"Debug"目录下的"WtCtaStraFact.dll"复制到"Debug/WtRunner/cta"目录下.

启动"QuoteFactory.exe", 准备测试"WtRunner"

## 文件初始化

其他文件初始化在之前许多文章中都讲过, 因此不再赘述. 将断点打在"WtRunner.cpp"222行(对, 就是前面提示源码有错误的那一行), 然后进入`WtRunner::initExecuters` 

"Executers"是CTA策略的执行器, 主要是为了实现多个CTA策略组合(这是CTA策略组合管理的核心, 也是比HFT复杂的原因)

关键代码如下

```cpp
// 1. executers.yaml 配置文件内容
WTSVariant* cfgItem = cfgExecuter->get(idx);
if (!cfgItem->getBoolean("active"))
    continue;

const char* id = cfgItem->getCString("id");
bool bLocal = cfgItem->getBoolean("local");

if (bLocal)
{
    // 2. 创建本地执行器对象
    WtLocalExecuter* executer = new WtLocalExecuter(&_exe_factory, id, &_data_mgr);
    if (!executer->init(cfgItem))
        return false;

    // 3. 将执行器和交易接口相互绑定
    TraderAdapterPtr trader = _traders.getAdapter(cfgItem->getCString("trader"));
    executer->setTrader(trader.get());
    trader->addSink(executer);
    // 4. 将执行器添加到cta引擎中
    _cta_engine.addExecuter(ExecCmdPtr(executer));
}
```

1. 这里初始化内容对应了"executers.yaml"配置文件内容, 注意配置文件里需要添加`local: true` 后续内容才能顺利进行
2. 这里的执行器对应的就是"WtLocalExecuter.h"下的`WtLocalExecuter`, CTA策略中的所有下单操作都需要通过执行器才能执行.
3. 这里的相互绑定在WT中很常见, 因为将来需要在各自类中互相调用.
4. 策略中的各种动作是通过引擎调度的, 下单有需要执行器完成, 所以需要将执行器添加到CTA引擎中(进一步跟踪会发现实际添加到了引擎的执行器管理器中)

## 策略初始化

这部分内容在之前的文章也重复过多次, 这里再简要啰嗦一下.

### 确定主K

将断点打在`WtStraDualThrust::on_init`下的`WTSKlineSlice *kline = ctx->stra_get_bars`这句代码上

进入 `CtaStraBaseCtx::stra_get_bars` , 首先会检查是否有主K. 如果主K不存在(首次进入), 默认当前品种是主K; 如果主K存在, 会抛出异常. 

因此CTA策略中只能有一个品种作为主K, 如果想订阅多个品种K线, 注意`isMain`, 这个参数, 订阅其他K线需要设置为false, (默认也是false)

### 获取K线数据

当执行到 `_engine->get_kline_slice` 这句代码时, 会从本地读取历史K线数据, 如果读取不到, 后边逻辑也就不会实现, 因此在需要K线的策略中(不仅CTA, 还包括HFT), 必须打开 "QuoteFactory"

```tip
注意揣摩这句话, 在我看来CTA和HFT只是策略形式的符号, 并不是某种特定策略类型的称谓.
```

### 订阅tick

结束 `CtaStraBaseCtx::stra_get_bars` 函数, 回到 `WtStraDualThrust::on_init`, 走到 `ctx->stra_sub_ticks`, 这里会将品种代码添加到`_tick_subs`中.

如果不主动订阅tick, 该品种后续就不会执行策略中的 `on_tick` 回调.(逻辑在之前的文章中有详细描述)
