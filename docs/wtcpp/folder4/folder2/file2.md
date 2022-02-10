# 执行引擎

source: `{{ page.path }}`

## WtBtRunner.h

### WtBtRunner

继承了历史数据加载器接口

```cpp
typedef enum tagEngineType
{
	ET_CTA = 999,	// CTA引擎	
	ET_HFT,			// 高频引擎
	ET_SEL			// 选股引擎
} EngineType;

class WtBtRunner : public IBtDataLoader
{
public:
	WtBtRunner();
	~WtBtRunner();

	// IBtDataLoader
	virtual bool loadFinalHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb) override;

	virtual bool loadRawHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb) override;

	virtual bool loadAllAdjFactors(void* obj, FuncReadFactors cb) override;

	virtual bool loadAdjFactors(void* obj, const char* stdCode, FuncReadFactors cb) override;

	virtual bool loadRawHisTicks(void* obj, const char* stdCode, uint32_t uDate, FuncReadTicks cb) override;

	virtual bool isAutoTrans() override
	{
		return _loader_auto_trans;
	}

	void feedRawBars(WTSBarStruct* bars, uint32_t count);
	void feedRawTicks(WTSTickStruct* ticks, uint32_t count);
	void feedAdjFactors(const char* stdCode, uint32_t* dates, double* factors, uint32_t count);

public:
	// 注册CTA策略回调函数
	void	registerCtaCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraCalcCallback cbCalc, 
		FuncStraBarCallback cbBar, FuncSessionEvtCallback cbSessEvt, FuncStraCalcCallback cbCalcDone = NULL);
	void	registerSelCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraCalcCallback cbCalc, 
		FuncStraBarCallback cbBar, FuncSessionEvtCallback cbSessEvt, FuncStraCalcCallback cbCalcDone = NULL);
	void registerHftCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraBarCallback cbBar,
		FuncHftChannelCallback cbChnl, FuncHftOrdCallback cbOrd, FuncHftTrdCallback cbTrd, FuncHftEntrustCallback cbEntrust,
		FuncStraOrdDtlCallback cbOrdDtl, FuncStraOrdQueCallback cbOrdQue, FuncStraTransCallback cbTrans, FuncSessionEvtCallback cbSessEvt);

	// 注册事件回调
	void registerEvtCallback(FuncEventCallback cbEvt)
	{
		_cb_evt = cbEvt;
	}

	void		registerExtDataLoader(FuncLoadFnlBars fnlBarLoader, FuncLoadRawBars rawBarLoader, FuncLoadAdjFactors fctLoader, FuncLoadRawTicks tickLoader, bool bAutoTrans = true)
	{
		_ext_fnl_bar_loader = fnlBarLoader;
		_ext_raw_bar_loader = rawBarLoader;
		_ext_adj_fct_loader = fctLoader;
		_ext_tick_loader = tickLoader;
		_loader_auto_trans = bAutoTrans;
	}

	// 初始化CTA策略调度器
	uint32_t	initCtaMocker(const char* name, int32_t slippage = 0, bool hook = false, bool persistData = true);
	uint32_t	initHftMocker(const char* name, bool hook = false);
	uint32_t	initSelMocker(const char* name, uint32_t date, uint32_t time, const char* period, const char* trdtpl = "CHINA", const char* session = "TRADING", int32_t slippage = 0);

	// 通过配置初始化事件通知
	bool	initEvtNotifier(WTSVariant* cfg);

	// 
	void	ctx_on_init(uint32_t id, EngineType eType);
	void	ctx_on_session_event(uint32_t id, uint32_t curTDate, bool isBegin = true, EngineType eType = ET_CTA);
	void	ctx_on_tick(uint32_t id, const char* stdCode, WTSTickData* newTick, EngineType eType);
	void	ctx_on_calc(uint32_t id, uint32_t uDate, uint32_t uTime, EngineType eType);
	void	ctx_on_calc_done(uint32_t id, uint32_t uDate, uint32_t uTime, EngineType eType);
	void	ctx_on_bar(uint32_t id, const char* stdCode, const char* period, WTSBarStruct* newBar, EngineType eType);

	void	hft_on_order_queue(uint32_t id, const char* stdCode, WTSOrdQueData* newOrdQue);
	void	hft_on_order_detail(uint32_t id, const char* stdCode, WTSOrdDtlData* newOrdDtl);
	void	hft_on_transaction(uint32_t id, const char* stdCode, WTSTransData* newTranns);

	void	hft_on_channel_ready(uint32_t cHandle, const char* trader);
	void	hft_on_order(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool isBuy, double totalQty, double leftQty, double price, bool isCanceled, const char* userTag);
	void	hft_on_trade(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool isBuy, double vol, double price, const char* userTag);
	void	hft_on_entrust(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool bSuccess, const char* message, const char* userTag);

	void	init(const char* logProfile = "", bool isFile = true, const char* outDir = "./outputs_bt");
	void	config(const char* cfgFile, bool isFile = true);
	void	run(bool bNeedDump = false, bool bAsync = false);
	void	release();
	void	stop();

	void	set_time_range(WtUInt64 stime, WtUInt64 etime);

	void	enable_tick(bool bEnabled = true);

	void	clear_cache();

	inline CtaMocker*		cta_mocker() { return _cta_mocker; }
	inline SelMocker*		sel_mocker() { return _sel_mocker; }
	inline HftMocker*		hft_mocker() { return _hft_mocker; }
	inline HisDataReplayer&	replayer() { return _replayer; }

	inline bool	isAsync() const { return _async; }

public:
	// 框架初始化事件
	inline void on_initialize_event()
	{
		if (_cb_evt)
			_cb_evt(EVENT_ENGINE_INIT, 0, 0);
	}

	// 框架调度事件
	inline void on_schedule_event(uint32_t uDate, uint32_t uTime)
	{
		if (_cb_evt)
			_cb_evt(EVENT_ENGINE_SCHDL, uDate, uTime);
	}

	// 交易日开始/结束事件
	inline void on_session_event(uint32_t uDate, bool isBegin = true)
	{
		if (_cb_evt)
		{
			_cb_evt(isBegin ? EVENT_SESSION_BEGIN : EVENT_SESSION_END, uDate, 0);
		}
	}

	// 回测结束事件
	inline void on_backtest_end()
	{
		if (_cb_evt)
			_cb_evt(EVENT_BACKTEST_END, 0, 0);
	}

private:
	// CTA策略相关回调函数, 通过 registerCtaCallbacks 初始化
	FuncStraInitCallback	_cb_cta_init;
	FuncSessionEvtCallback	_cb_cta_sessevt;
	FuncStraTickCallback	_cb_cta_tick;
	FuncStraCalcCallback	_cb_cta_calc;
	FuncStraCalcCallback	_cb_cta_calc_done;
	FuncStraBarCallback		_cb_cta_bar;

	FuncStraInitCallback	_cb_sel_init;
	FuncSessionEvtCallback	_cb_sel_sessevt;
	FuncStraTickCallback	_cb_sel_tick;
	FuncStraCalcCallback	_cb_sel_calc;
	FuncStraCalcCallback	_cb_sel_calc_done;
	FuncStraBarCallback		_cb_sel_bar;

	FuncStraInitCallback	_cb_hft_init;
	FuncSessionEvtCallback	_cb_hft_sessevt;
	FuncStraTickCallback	_cb_hft_tick;
	FuncStraBarCallback		_cb_hft_bar;
	FuncHftChannelCallback	_cb_hft_chnl;
	FuncHftOrdCallback		_cb_hft_ord;
	FuncHftTrdCallback		_cb_hft_trd;
	FuncHftEntrustCallback	_cb_hft_entrust;

	FuncStraOrdQueCallback	_cb_hft_ordque;
	FuncStraOrdDtlCallback	_cb_hft_orddtl;
	FuncStraTransCallback	_cb_hft_trans;

	FuncEventCallback		_cb_evt;				// 事件回调

	FuncLoadFnlBars			_ext_fnl_bar_loader;	// 最终K线加载器
	FuncLoadRawBars			_ext_raw_bar_loader;	// 原始K线加载器
	FuncLoadAdjFactors		_ext_adj_fct_loader;	// 复权因子加载器
	FuncLoadRawTicks		_ext_tick_loader;		// tick加载器
	bool					_loader_auto_trans;		// 是否自动转储

	CtaMocker*		_cta_mocker;					// CTA策略调度器
	SelMocker*		_sel_mocker;
	ExecMocker*		_exec_mocker;
	HftMocker*		_hft_mocker;
	HisDataReplayer	_replayer;						// 回测引擎
	EventNotifier	_notifier;						// 消息通知

	bool			_inited;
	bool			_running;

	StdThreadPtr	_worker;
	bool			_async;

	void*			_feed_obj;
	FuncReadBars	_feeder_bars;					// 历史数据加载器的回调函数 bars
	FuncReadTicks	_feeder_ticks;					// 历史数据加载器的回调函数 ticks
	FuncReadFactors	_feeder_fcts;					// 历史数据加载器的回调函数 factors
	StdUniqueMutex	_feed_mtx;
};
```

## WtBtRunner.cpp

```cpp
#ifdef _MSC_VER
#include "../Common/mdump.h"
#include <boost/filesystem.hpp>
// 这个主要是给MiniDumper用的
const char* getModuleName()
{
	static char MODULE_NAME[250] = { 0 };
	if (strlen(MODULE_NAME) == 0)
	{
		GetModuleFileName(g_dllModule, MODULE_NAME, 250);
		boost::filesystem::path p(MODULE_NAME);
		strcpy(MODULE_NAME, p.filename().string().c_str());
	}
	return MODULE_NAME;
}
#endif

WtBtRunner::WtBtRunner()
	: _cta_mocker(NULL)
	, _sel_mocker(NULL)

	, _cb_cta_init(NULL)
	, _cb_cta_tick(NULL)
	, _cb_cta_calc(NULL)
	, _cb_cta_calc_done(NULL)
	, _cb_cta_bar(NULL)
	, _cb_cta_sessevt(NULL)

	, _cb_sel_init(NULL)
	, _cb_sel_tick(NULL)
	, _cb_sel_calc(NULL)
	, _cb_sel_calc_done(NULL)
	, _cb_sel_bar(NULL)
	, _cb_sel_sessevt(NULL)

	, _cb_hft_init(NULL)
	, _cb_hft_tick(NULL)
	, _cb_hft_bar(NULL)
	, _cb_hft_ord(NULL)
	, _cb_hft_trd(NULL)
	, _cb_hft_entrust(NULL)
	, _cb_hft_chnl(NULL)

	, _cb_hft_orddtl(NULL)
	, _cb_hft_ordque(NULL)
	, _cb_hft_trans(NULL)

	, _cb_hft_sessevt(NULL)

	, _ext_fnl_bar_loader(NULL)
	, _ext_raw_bar_loader(NULL)
	, _ext_adj_fct_loader(NULL)
	, _ext_tick_loader(NULL)

	, _inited(false)
	, _running(false)
	, _async(false)
{
	install_signal_hooks([](const char* message) {
		WTSLogger::error(message);
	});
}

WtBtRunner::~WtBtRunner()
{}

// 调用原始K线加载器加载对应K线
bool WtBtRunner::loadRawHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb)
{
	StdUniqueLock lock(_feed_mtx);
	if (_ext_raw_bar_loader == NULL)
		return false;

	_feed_obj = obj;
	_feeder_bars = cb;

	switch (period)
	{
	case KP_DAY:
        return _ext_raw_bar_loader(stdCode, "d1");
	case KP_Minute1:
        return _ext_raw_bar_loader(stdCode, "m1");
	case KP_Minute5:
        return _ext_raw_bar_loader(stdCode, "m5");
	default:
		{
			WTSLogger::error("Unsupported period of extended data loader");
			return false;
		}
	}
}

// 调用最终K线加载器加载K线
bool WtBtRunner::loadFinalHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb)
{
	StdUniqueLock lock(_feed_mtx);
	if (_ext_fnl_bar_loader == NULL)
		return false;

	_feed_obj = obj;
	_feeder_bars = cb;

	switch (period)
	{
	case KP_DAY:
		return _ext_fnl_bar_loader(stdCode, "d1");
	case KP_Minute1:
		return _ext_fnl_bar_loader(stdCode, "m1");
	case KP_Minute5:
		return _ext_fnl_bar_loader(stdCode, "m5");
	default:
		{
			WTSLogger::error("Unsupported period of extended data loader");
			return false;
		}
	}
}

// 调用复权因子加载器加载全品种数据
bool WtBtRunner::loadAllAdjFactors(void* obj, FuncReadFactors cb)
{
	StdUniqueLock lock(_feed_mtx);
	if (_ext_adj_fct_loader == NULL)
		return false;

	_feed_obj = obj;
	_feeder_fcts = cb;

	return _ext_adj_fct_loader("");
}

// 调用复权因子加载器加载单品种数据
bool WtBtRunner::loadAdjFactors(void* obj, const char* stdCode, FuncReadFactors cb)
{
	StdUniqueLock lock(_feed_mtx);
	if (_ext_adj_fct_loader == NULL)
		return false;

	_feed_obj = obj;
	_feeder_fcts = cb;

	return _ext_adj_fct_loader(stdCode);
}

// 调用tick加载器加载数据
bool WtBtRunner::loadRawHisTicks(void* obj, const char* stdCode, uint32_t uDate, FuncReadTicks cb)
{
	StdUniqueLock lock(_feed_mtx);
	if (_ext_tick_loader == NULL)
		return false;

	_feed_obj = obj;
	_feeder_ticks = cb;

	return _ext_tick_loader(stdCode, uDate);
}

// 历史数据加载器的回调函数: bars
void WtBtRunner::feedRawBars(WTSBarStruct* bars, uint32_t count)
{
	if(_ext_fnl_bar_loader == NULL && _ext_raw_bar_loader == NULL)
	{
		WTSLogger::error("Cannot feed bars because of no extented bar loader registered.");
		return;
	}

	_feeder_bars(_feed_obj, bars, count);
}

// 历史数据加载器的回调函数: factors
void WtBtRunner::feedAdjFactors(const char* stdCode, uint32_t* dates, double* factors, uint32_t count)
{
	if(_ext_adj_fct_loader == NULL)
	{
		WTSLogger::error("Cannot feed adjusting factors because of no extented adjusting factor loader registered.");
		return;
	}

	_feeder_fcts(_feed_obj, stdCode, dates, factors, count);
}

// 历史数据加载器的回调函数: ticks
void WtBtRunner::feedRawTicks(WTSTickStruct* ticks, uint32_t count)
{
	if (_ext_tick_loader == NULL)
	{
		WTSLogger::error("Cannot feed ticks because of no extented tick loader registered.");
		return;
	}

	_feeder_ticks(_feed_obj, ticks, count);
}

// 注册CTA策略的回调函数
void WtBtRunner::registerCtaCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraCalcCallback cbCalc, 
	FuncStraBarCallback cbBar, FuncSessionEvtCallback cbSessEvt, FuncStraCalcCallback cbCalcDone/* = NULL*/)
{
	_cb_cta_init = cbInit;
	_cb_cta_tick = cbTick;
	_cb_cta_calc = cbCalc;
	_cb_cta_bar = cbBar;
	_cb_cta_sessevt = cbSessEvt;
	_cb_cta_calc_done = cbCalcDone;

	WTSLogger::info("Callbacks of CTA engine registration done");
}

void WtBtRunner::registerSelCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraCalcCallback cbCalc, 
	FuncStraBarCallback cbBar, FuncSessionEvtCallback cbSessEvt, FuncStraCalcCallback cbCalcDone/* = NULL*/)
{
	_cb_sel_init = cbInit;
	_cb_sel_tick = cbTick;
	_cb_sel_calc = cbCalc;
	_cb_sel_bar = cbBar;
	_cb_sel_sessevt = cbSessEvt;

	_cb_sel_calc_done = cbCalcDone;

	WTSLogger::info("Callbacks of SEL engine registration done");
}

void WtBtRunner::registerHftCallbacks(FuncStraInitCallback cbInit, FuncStraTickCallback cbTick, FuncStraBarCallback cbBar,
	FuncHftChannelCallback cbChnl, FuncHftOrdCallback cbOrd, FuncHftTrdCallback cbTrd, FuncHftEntrustCallback cbEntrust, 
	FuncStraOrdDtlCallback cbOrdDtl, FuncStraOrdQueCallback cbOrdQue, FuncStraTransCallback cbTrans, FuncSessionEvtCallback cbSessEvt)
{
	_cb_hft_init = cbInit;
	_cb_hft_tick = cbTick;
	_cb_hft_bar = cbBar;

	_cb_hft_chnl = cbChnl;
	_cb_hft_ord = cbOrd;
	_cb_hft_trd = cbTrd;
	_cb_hft_entrust = cbEntrust;

	_cb_hft_orddtl = cbOrdDtl;
	_cb_hft_ordque = cbOrdQue;
	_cb_hft_trans = cbTrans;

	_cb_hft_sessevt = cbSessEvt;

	WTSLogger::info("Callbacks of HFT engine registration done");
}

// 初始化CTA调度器模板
uint32_t WtBtRunner::initCtaMocker(const char* name, int32_t slippage /* = 0 */, bool hook /* = false */, bool persistData /* = true */)
{
	if(_cta_mocker)
	{
		delete _cta_mocker;
		_cta_mocker = NULL;
	}

	_cta_mocker = new ExpCtaMocker(&_replayer, name, slippage, persistData, &_notifier);
	if(hook) _cta_mocker->install_hook();
	_replayer.register_sink(_cta_mocker, name);
	return _cta_mocker->id();
}

uint32_t WtBtRunner::initHftMocker(const char* name, bool hook/* = false*/)
{
	if (_hft_mocker)
	{
		delete _hft_mocker;
		_hft_mocker = NULL;
	}

	_hft_mocker = new ExpHftMocker(&_replayer, name);
	if (hook) _hft_mocker->install_hook();
	_replayer.register_sink(_hft_mocker, name);
	return _hft_mocker->id();
}

uint32_t WtBtRunner::initSelMocker(const char* name, uint32_t date, uint32_t time, const char* period, const char* trdtpl /* = "CHINA" */, const char* session /* = "TRADING" */, int32_t slippage/* = 0*/)
{
	if (_sel_mocker)
	{
		delete _sel_mocker;
		_sel_mocker = NULL;
	}

	_sel_mocker = new ExpSelMocker(&_replayer, name, slippage);
	_replayer.register_sink(_sel_mocker, name);

	_replayer.register_task(_sel_mocker->id(), date, time, period, trdtpl, session);
	return _sel_mocker->id();
}

// 执行回调函数
void WtBtRunner::ctx_on_bar(uint32_t id, const char* stdCode, const char* period, WTSBarStruct* newBar, EngineType eType/*= ET_CTA*/)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_bar) _cb_cta_bar(id, stdCode, period, newBar); break;
	case ET_HFT: if (_cb_hft_bar) _cb_hft_bar(id, stdCode, period, newBar); break;
	case ET_SEL: if (_cb_sel_bar) _cb_sel_bar(id, stdCode, period, newBar); break;
	default:
		break;
	}
}

void WtBtRunner::ctx_on_calc(uint32_t id, uint32_t curDate, uint32_t curTime, EngineType eType /* = ET_CTA */)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_calc) _cb_cta_calc(id, curDate, curTime); break;
	case ET_SEL: if (_cb_sel_calc) _cb_sel_calc(id, curDate, curTime); break;
	default:
		break;
	}
}

void WtBtRunner::ctx_on_calc_done(uint32_t id, uint32_t curDate, uint32_t curTime, EngineType eType /* = ET_CTA */)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_calc_done) _cb_cta_calc_done(id, curDate, curTime); break;
	case ET_SEL: if (_cb_sel_calc_done) _cb_sel_calc_done(id, curDate, curTime); break;
	default:
		break;
	}
}

void WtBtRunner::ctx_on_init(uint32_t id, EngineType eType/*= ET_CTA*/)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_init) _cb_cta_init(id); break;
	case ET_HFT: if (_cb_hft_init) _cb_hft_init(id); break;
	case ET_SEL: if (_cb_sel_init) _cb_sel_init(id); break;
	default:
		break;
	}
}

void WtBtRunner::ctx_on_session_event(uint32_t id, uint32_t curTDate, bool isBegin /* = true */, EngineType eType /* = ET_CTA */)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_sessevt) _cb_cta_sessevt(id, curTDate, isBegin); break;
	case ET_HFT: if (_cb_hft_sessevt) _cb_hft_sessevt(id, curTDate, isBegin); break;
	case ET_SEL: if (_cb_sel_sessevt) _cb_sel_sessevt(id, curTDate, isBegin); break;
	default:
		break;
	}
}

void WtBtRunner::ctx_on_tick(uint32_t id, const char* stdCode, WTSTickData* newTick, EngineType eType/*= ET_CTA*/)
{
	switch (eType)
	{
	case ET_CTA: if (_cb_cta_tick) _cb_cta_tick(id, stdCode, &newTick->getTickStruct()); break;
	case ET_HFT: if (_cb_hft_tick) _cb_hft_tick(id, stdCode, &newTick->getTickStruct()); break;
	case ET_SEL: if (_cb_sel_tick) _cb_sel_tick(id, stdCode, &newTick->getTickStruct()); break;
	default:
		break;
	}
}

void WtBtRunner::hft_on_order_queue(uint32_t id, const char* stdCode, WTSOrdQueData* newOrdQue)
{
	if (_cb_hft_ordque)
		_cb_hft_ordque(id, stdCode, &newOrdQue->getOrdQueStruct());
}

void WtBtRunner::hft_on_order_detail(uint32_t id, const char* stdCode, WTSOrdDtlData* newOrdDtl)
{
	if (_cb_hft_orddtl)
		_cb_hft_orddtl(id, stdCode, &newOrdDtl->getOrdDtlStruct());
}

void WtBtRunner::hft_on_transaction(uint32_t id, const char* stdCode, WTSTransData* newTrans)
{
	if (_cb_hft_trans)
		_cb_hft_trans(id, stdCode, &newTrans->getTransStruct());
}

void WtBtRunner::hft_on_channel_ready(uint32_t cHandle, const char* trader)
{
	if (_cb_hft_chnl)
		_cb_hft_chnl(cHandle, trader, 1000/*CHNL_EVENT_READY*/);
}

void WtBtRunner::hft_on_entrust(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool bSuccess, const char* message, const char* userTag)
{
	if (_cb_hft_entrust)
		_cb_hft_entrust(cHandle, localid, stdCode, bSuccess, message, userTag);
}

void WtBtRunner::hft_on_order(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool isBuy, double totalQty, double leftQty, double price, bool isCanceled, const char* userTag)
{
	if (_cb_hft_ord)
		_cb_hft_ord(cHandle, localid, stdCode, isBuy, totalQty, leftQty, price, isCanceled, userTag);
}

void WtBtRunner::hft_on_trade(uint32_t cHandle, WtUInt32 localid, const char* stdCode, bool isBuy, double vol, double price, const char* userTag)
{
	if (_cb_hft_trd)
		_cb_hft_trd(cHandle, localid, stdCode, isBuy, vol, price, userTag);
}

// 初始化: 设置路径
void WtBtRunner::init(const char* logProfile /* = "" */, bool isFile /* = true */, const char* outDir/* = "./outputs_bt"*/)
{
#ifdef _MSC_VER
	CMiniDumper::Enable(getModuleName(), true, WtHelper::getCWD().c_str());
#endif

	WTSLogger::init(logProfile, isFile);
	WtHelper::setInstDir(getBinDir());
	WtHelper::setOutputDir(outDir);
}

// 通过配置文件初始化
void WtBtRunner::config(const char* cfgFile, bool isFile /* = true */)
{
	if(_inited)
	{
		WTSLogger::error("WtBtEngine has already been inited");
		return;
	}
	// 读取 config.json 文件
	std::string content;
	if (isFile)
		StdFile::read_file_content(cfgFile, content);
	else
		content = cfgFile;

	// 将 json 数据转为 WTSVariant 类型
	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::info("Parsing configuration file failed");
		return;
	}
	WTSVariant* cfg = WTSVariant::createObject();
	jsonToVariant(root, cfg);

	// 初始化事件推送器
	initEvtNotifier(cfg->get("notifier"));
	// 初始化回测引擎
	_replayer.init(cfg->get("replayer"), &_notifier, this);

	// 初始化 CTA 调度器模板
	WTSVariant* cfgEnv = cfg->get("env");
	const char* mode = cfgEnv->getCString("mocker");
	WTSVariant* cfgMode = cfg->get(mode);
	if (strcmp(mode, "cta") == 0 && cfgMode)
	{
		const char* name = cfgMode->getCString("name");
		int32_t slippage = cfgMode->getInt32("slippage");
		_cta_mocker = new ExpCtaMocker(&_replayer, name, slippage, &_notifier);
		_cta_mocker->init_cta_factory(cfgMode);
		_replayer.register_sink(_cta_mocker, name);
	}
	else if (strcmp(mode, "hft") == 0 && cfgMode)
	{
		const char* name = cfgMode->getCString("name");
		_hft_mocker = new ExpHftMocker(&_replayer, name);
		_hft_mocker->init_hft_factory(cfgMode);
		_replayer.register_sink(_hft_mocker, name);
	}
	else if (strcmp(mode, "sel") == 0 && cfgMode)
	{
		const char* name = cfgMode->getCString("name");
		int32_t slippage = cfgMode->getInt32("slippage");
		_sel_mocker = new ExpSelMocker(&_replayer, name, slippage);
		_sel_mocker->init_sel_factory(cfgMode);
		_replayer.register_sink(_sel_mocker, name);

		WTSVariant* cfgTask = cfgMode->get("task");
		if(cfgTask)
			_replayer.register_task(_sel_mocker->id(), cfgTask->getUInt32("date"), cfgTask->getUInt32("time"), 
				cfgTask->getCString("period"), cfgTask->getCString("trdtpl"), cfgTask->getCString("session"));
	}
	else if (strcmp(mode, "exec") == 0 && cfgMode)
	{
		const char* name = cfgMode->getCString("name");
		_exec_mocker = new ExecMocker(&_replayer);
		_exec_mocker->init(cfgMode);
		_replayer.register_sink(_exec_mocker, name);
	}
}

// 执行
void WtBtRunner::run(bool bNeedDump /* = false */, bool bAsync /* = false */)
{
	if (_running)
		return;

	_async = bAsync;

	WTSLogger::info("Backtesting will run in %s mode", _async ? "async" : "sync");

	if (_cta_mocker)
		_cta_mocker->enable_hook(_async);
	else if (_hft_mocker)
		_hft_mocker->enable_hook(_async);

	// 回测引擎准备
	_replayer.prepare();

	_worker.reset(new StdThread([this, bNeedDump]() {
		_running = true;
		try
		{
			_replayer.run(bNeedDump);
		}
		catch (...)
		{
			WTSLogger::error("Exception raised while worker running");
		}
		WTSLogger::debug("Worker thread of backtest finished");
		_running = false;
	}));

	if (!bAsync)
		_worker->join();
}

// 停止
void WtBtRunner::stop()
{
	if (!_running)
	{
		if (_worker)
		{
			_worker->join();
			_worker.reset();
		}
		return;
	}

	_replayer.stop();

	WTSLogger::debug("Notify to finish last round");

	if (_cta_mocker)
		_cta_mocker->step_calc();
	if (_hft_mocker)
		_hft_mocker->step_tick();
	WTSLogger::debug("Last round ended");
	if (_worker)
	{
		_worker->join();
		_worker.reset();
	}
	WTSLogger::freeAllDynLoggers();
	WTSLogger::debug("Backtest stopped");
}

// 释放
void WtBtRunner::release()
{
	WTSLogger::stop();
}

// 通过调用回测引擎设置回测时间区间
void WtBtRunner::set_time_range(WtUInt64 stime, WtUInt64 etime)
{
	_replayer.set_time_range(stime, etime);

	WTSLogger::info(fmt::format("Backtest time range is set to be [{},{}] mannually", stime, etime).c_str());
}

void WtBtRunner::enable_tick(bool bEnabled /* = true */)
{
	_replayer.enable_tick(bEnabled);
	WTSLogger::info("Tick data replaying is %s", bEnabled ? "enabled" : "disabled");
}

void WtBtRunner::clear_cache()
{
	_replayer.clear_cache();
}

// 日志级别
const char* LOG_TAGS[] = {
	"all",
	"debug",
	"info",
	"warn",
	"error",
	"fatal",
	"none",
};

// 通过配置初始化消息通知对象
bool WtBtRunner::initEvtNotifier(WTSVariant* cfg)
{
	if (cfg == NULL || cfg->type() != WTSVariant::VT_Object)
		return false;

	_notifier.init(cfg);

	return true;
}
```