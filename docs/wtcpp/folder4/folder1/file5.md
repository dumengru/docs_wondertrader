# CTA策略调度器

source: `{{ page.path }}`

```tip
**策略接口调用逻辑**

1. 策略调度器都会继承 `ICtaStraCtx`(上下文管理器接口) 和 `IDataSink`(历史数据回放器回调函数接口), 并实现相应的方法

2. 其中 `IDataSink` 中主要是回调函数接口, 在调度器中实现接口的过程中, 实现 `IDataSink` 的回调函数都会调用 `ICtaStraCtx` 接口对应的函数(如 `handle_init()` 中调用 `this->on_init()`)

3. 实现 `ICtaStraCtx` 接口的函数在添加部分逻辑之后又会调用对应策略中的函数(如 `on_init()` 中调用 `_strategy->on_init(this);`)

4. 因此策略中的 `on_init()` 等函数实际是整个回测框架的最后一步
```

## CtaMocker.h

```cpp
const char COND_ACTION_OL = 0;	    //开多
const char COND_ACTION_CL = 1;	    //平多
const char COND_ACTION_OS = 2;	    //开空
const char COND_ACTION_CS = 3;	    //平空
const char COND_ACTION_SP = 4;	    //直接设置仓位

typedef struct _CondEntrust
{
	WTSCompareField _field;
	WTSCompareType	_alg;
	double			_target;
	double			_qty;
	char			_action;	//0-开多,1-平多,2-开空,3-平空
	char			_code[MAX_INSTRUMENT_LENGTH];
	char			_usertag[32];
	_CondEntrust()
	{
		memset(this, 0, sizeof(_CondEntrust));
	}

} CondEntrust;

typedef std::vector<CondEntrust>	CondList;
typedef faster_hashmap<std::string, CondList>	CondEntrustMap;

class CtaMocker : public ICtaStraCtx, public IDataSink
{
public:
	CtaMocker(HisDataReplayer* replayer, const char* name, int32_t slippage = 0, bool persistData = true, EventNotifier* notifier = NULL);
	virtual ~CtaMocker();

private:
	void	dump_outputs();
	inline void log_signal(const char* stdCode, double target, double price, uint64_t gentime, const char* usertag = "");
	inline void	log_trade(const char* stdCode, bool isLong, bool isOpen, uint64_t curTime, double price, double qty, const char* userTag = "", double fee = 0.0, uint32_t barNo = 0);
	inline void	log_close(const char* stdCode, bool isLong, uint64_t openTime, double openpx, uint64_t closeTime, double closepx, double qty,
		double profit, double maxprofit, double maxloss, double totalprofit = 0, const char* enterTag = "", const char* exitTag = "", uint32_t openBarNo = 0, uint32_t closeBarNo = 0);
	void	update_dyn_profit(const char* stdCode, double price);
	void	do_set_position(const char* stdCode, double qty, double price = 0.0, const char* userTag = "", bool bTriggered = false);
	void	append_signal(const char* stdCode, double qty, const char* userTag = "", double price = 0.0);
	inline CondList& get_cond_entrusts(const char* stdCode);

public:
	bool	init_cta_factory(WTSVariant* cfg);
	void	install_hook();
	void	enable_hook(bool bEnabled = true);
	bool	step_calc();

public:
	// IDataSink 接口实现
	virtual void	handle_tick(const char* stdCode, WTSTickData* curTick) override;
	virtual void	handle_bar_close(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar) override;
	virtual void	handle_schedule(uint32_t uDate, uint32_t uTime) override;
	virtual void	handle_init() override;
	virtual void	handle_session_begin(uint32_t curTDate) override;
	virtual void	handle_session_end(uint32_t curTDate) override;
	virtual void	handle_replay_done() override;

	// ICtaStraCtx 接口实现
	virtual uint32_t id() { return _context_id; }
	    //回调函数
	virtual void on_init() override;
	virtual void on_session_begin(uint32_t curTDate) override;
	virtual void on_session_end(uint32_t curTDate) override;
	virtual void on_tick(const char* stdCode, WTSTickData* newTick, bool bEmitStrategy = true) override;
	virtual void on_bar(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar) override;
	virtual bool on_schedule(uint32_t curDate, uint32_t curTime) override;
	virtual void enum_position(FuncEnumCtaPosCallBack cb) override;
	virtual void on_tick_updated(const char* stdCode, WTSTickData* newTick) override;
	virtual void on_bar_close(const char* stdCode, const char* period, WTSBarStruct* newBar) override;
	virtual void on_calculate(uint32_t curDate, uint32_t curTime) override;
        // 策略接口
	virtual void stra_enter_long(const char* stdCode, double qty, const char* userTag = "", double limitprice = 0.0, double stopprice = 0.0) override;
	virtual void stra_enter_short(const char* stdCode, double qty, const char* userTag = "", double limitprice = 0.0, double stopprice = 0.0) override;
	virtual void stra_exit_long(const char* stdCode, double qty, const char* userTag = "", double limitprice = 0.0, double stopprice = 0.0) override;
	virtual void stra_exit_short(const char* stdCode, double qty, const char* userTag = "", double limitprice = 0.0, double stopprice = 0.0) override;
        // 获取策略相关信息
	virtual double stra_get_position(const char* stdCode, bool bOnlyValid = false, const char* userTag = "") override;
	virtual void stra_set_position(const char* stdCode, double qty, const char* userTag = "", double limitprice = 0.0, double stopprice = 0.0) override;
	virtual double stra_get_price(const char* stdCode) override;
	virtual uint32_t stra_get_tdate() override;
	virtual uint32_t stra_get_date() override;
	virtual uint32_t stra_get_time() override;
	virtual double stra_get_fund_data(int flag = 0) override;
	virtual uint64_t stra_get_first_entertime(const char* stdCode) override;
	virtual uint64_t stra_get_last_entertime(const char* stdCode) override;
	virtual uint64_t stra_get_last_exittime(const char* stdCode) override;
	virtual double stra_get_last_enterprice(const char* stdCode) override;
	virtual double stra_get_position_avgpx(const char* stdCode) override;
	virtual double stra_get_position_profit(const char* stdCode) override;
	virtual uint64_t stra_get_detail_entertime(const char* stdCode, const char* userTag) override;
	virtual double stra_get_detail_cost(const char* stdCode, const char* userTag) override;
	virtual double stra_get_detail_profit(const char* stdCode, const char* userTag, int flag = 0) override;
	virtual WTSCommodityInfo* stra_get_comminfo(const char* stdCode) override;
	virtual WTSKlineSlice*	stra_get_bars(const char* stdCode, const char* period, uint32_t count, bool isMain = false) override;
	virtual WTSTickSlice*	stra_get_ticks(const char* stdCode, uint32_t count) override;
	virtual WTSTickData*	stra_get_last_tick(const char* stdCode) override;
	virtual void stra_sub_ticks(const char* stdCode) override;
	virtual void stra_log_info(const char* fmt, ...) override;
	virtual void stra_log_debug(const char* fmt, ...) override;
	virtual void stra_log_error(const char* fmt, ...) override;
	virtual void stra_save_user_data(const char* key, const char* val) override;
	virtual const char* stra_load_user_data(const char* key, const char* defVal = "") override;

protected:
	uint32_t			_context_id;    // 策略上下文ID
	HisDataReplayer*	_replayer;      // 回测引擎
	uint64_t		_total_calc_time;	// 总计算时间
	uint32_t		_emit_times;		// 总计算次数
	int32_t			_slippage;			// 成交滑点
	uint32_t		_schedule_times;	// 调度次数
	std::string		_main_key;          // 

	typedef struct _KlineTag
	{
		bool			_closed;
		_KlineTag() :_closed(false){}
	} KlineTag;
	typedef faster_hashmap<std::string, KlineTag> KlineTags;
	KlineTags	_kline_tags;            // 

	typedef faster_hashmap<std::string, double> PriceMap;
	PriceMap		_price_map;         // 

	typedef struct _DetailInfo
	{
		bool		_long;
		double		_prlume;
		uint64_t	_opice;
		double		_voentime;
		uint32_t	_opentdate;
		double		_max_profit;
		double		_max_loss;
		double		_profit;
		char		_opentag[32];
		uint32_t	_open_barno;

		_DetailInfo()
		{
			memset(this, 0, sizeof(_DetailInfo));
		}
	} DetailInfo;                       // 

	typedef struct _PosInfo
	{
		double		_volume;
		double		_closeprofit;
		double		_dynprofit;
		uint64_t	_last_entertime;
		uint64_t	_last_exittime;
		double		_frozen;
		std::vector<DetailInfo> _details;
		_PosInfo()
		{
			_volume = 0;
			_closeprofit = 0;
			_dynprofit = 0;
			_frozen = 0;
		}

		inline double valid() const { return _volume - _frozen; }
	} PosInfo;                      // 持仓信息
	typedef faster_hashmap<std::string, PosInfo> PositionMap;
	PositionMap		_pos_map;
	double	_total_closeprofit;

	typedef struct _SigInfo
	{
		double		_volume;
		std::string	_usertag;
		double		_sigprice;
		double		_desprice;
		bool		_triggered;
		uint64_t	_gentime;
		_SigInfo()
		{
			_volume = 0;
			_sigprice = 0;
			_desprice = 0;
			_triggered = false;
			_gentime = 0;
		}
	} SigInfo;
	typedef faster_hashmap<std::string, SigInfo>	SignalMap;
	SignalMap		_sig_map;			// 信号信息

	std::stringstream	_trade_logs;    // 成交记录: trades.csv
	std::stringstream	_close_logs;    // 平仓记录: closes.csv
	std::stringstream	_fund_logs;     // 资金记录: funds.csv
	std::stringstream	_sig_logs;      // 开仓记录: signals.csv
	CondEntrustMap	_condtions;         // 条件列表

	bool			_is_in_schedule;	//是否在自动调度中

	// 用户数据
	typedef faster_hashmap<std::string, std::string> StringHashMap;
	StringHashMap	_user_datas;
	bool			_ud_modified;

	typedef struct _StraFundInfo
	{
		double	_total_profit;
		double	_total_dynprofit;
		double	_total_fees;
		_StraFundInfo()
		{
			memset(this, 0, sizeof(_StraFundInfo));
		}
	} StraFundInfo;                     
	StraFundInfo		_fund_info;     // 策略资金详情

	typedef struct _StraFactInfo
	{
		std::string		_module_path;
		DllHandle		_module_inst;
		ICtaStrategyFact*	_fact;
		FuncCreateStraFact	_creator;
		FuncDeleteStraFact	_remover;
		_StraFactInfo()
		{
			_module_inst = NULL;
			_fact = NULL;
		}
		~_StraFactInfo()
		{
			if (_fact)
				_remover(_fact);
		}
	} StraFactInfo;
	StraFactInfo	_factory;           // 策略工厂详情
	CtaStrategy*	_strategy;          // cta策略指针
	EventNotifier*	_notifier;          // 消息指针
	StdUniqueMutex	_mtx_calc;
	StdCondVariable	_cond_calc;
	bool			_has_hook;		    //这是人为控制是否启用钩子
	bool			_hook_valid;	    //这是根据是否是异步回测模式而确定钩子是否可用
	std::atomic<uint32_t>		_cur_step;	//临时变量，用于控制状态
	bool			_in_backtest;			// 回测是否结束
	bool			_wait_calc;
	bool			_persist_data;			// 是否保存数据
};
```

## CtaMocker.cpp

```cpp
const char* CMP_ALG_NAMES[] =
{
	"＝",
	"＞",
	"＜",
	"≥",
	"≤"
};

const char* ACTION_NAMES[] =
{
	"OL",
	"CL",
	"OS",
	"CS",
	"SYN"
};


inline uint32_t makeCtxId()
{
	static std::atomic<uint32_t> _auto_context_id{ 1 };
	return _auto_context_id.fetch_add(1);
}

CtaMocker::CtaMocker(HisDataReplayer* replayer, const char* name, int32_t slippage /* = 0 */, bool persistData /* = true */, EventNotifier* notifier /* = NULL */)
	: ICtaStraCtx(name)
	, _replayer(replayer)
	, _total_calc_time(0)
	, _emit_times(0)
	, _is_in_schedule(false)
	, _ud_modified(false)
	, _strategy(NULL)
	, _slippage(slippage)
	, _schedule_times(0)
	, _total_closeprofit(0)
	, _notifier(notifier)
	, _has_hook(false)
	, _hook_valid(true)
	, _cur_step(0)
	, _wait_calc(false)
	, _in_backtest(false)
	, _persist_data(persistData)
{
	_context_id = makeCtxId();
}

CtaMocker::~CtaMocker()
{}

// 保存信息到本地
void CtaMocker::dump_outputs()
{
	if (!_persist_data)
		return;

	std::string folder = WtHelper::getOutputDir();
	folder += _name;		// _name是策略ID, 在配置文件中
	folder += "/";
	boost::filesystem::create_directories(folder.c_str());

	// 创建各类csv文件
	std::string filename = folder + "trades.csv";
	std::string content = "code,time,direct,action,price,qty,tag,fee,barno\n";
	content += _trade_logs.str();
	StdFile::write_file_content(filename.c_str(), (void*)content.c_str(), content.size());

	filename = folder + "closes.csv";
	content = "code,direct,opentime,openprice,closetime,closeprice,qty,profit,maxprofit,maxloss,totalprofit,entertag,exittag,openbarno,closebarno\n";
	content += _close_logs.str();
	StdFile::write_file_content(filename.c_str(), (void*)content.c_str(), content.size());


	filename = folder + "funds.csv";
	content = "date,closeprofit,positionprofit,dynbalance,fee\n";
	content += _fund_logs.str();
	StdFile::write_file_content(filename.c_str(), (void*)content.c_str(), content.size());


	filename = folder + "signals.csv";
	content = "code,target,sigprice,gentime,usertag\n";
	content += _sig_logs.str();
	StdFile::write_file_content(filename.c_str(), (void*)content.c_str(), content.size());
}

// 输出对应日志信息
void CtaMocker::log_signal(const char* stdCode, double target, double price, uint64_t gentime, const char* usertag /* = "" */)
{
	_sig_logs << stdCode << "," << target << "," << price << "," << gentime << "," << usertag << "\n";
}
void CtaMocker::log_trade(const char* stdCode, bool isLong, bool isOpen, uint64_t curTime, double price, double qty, const char* userTag, double fee, uint32_t barNo)
{
	_trade_logs << stdCode << "," << curTime << "," << (isLong ? "LONG" : "SHORT") << "," << (isOpen ? "OPEN" : "CLOSE") 
		<< "," << price << "," << qty << "," << userTag << "," << fee << "," << barNo << "\n";
}
void CtaMocker::log_close(const char* stdCode, bool isLong, uint64_t openTime, double openpx, uint64_t closeTime, double closepx, double qty, double profit, double maxprofit, double maxloss, 
	double totalprofit /* = 0 */, const char* enterTag /* = "" */, const char* exitTag /* = "" */, uint32_t openBarNo /* = 0 */, uint32_t closeBarNo /* = 0 */)
{
	_close_logs << stdCode << "," << (isLong ? "LONG" : "SHORT") << "," << openTime << "," << openpx
		<< "," << closeTime << "," << closepx << "," << qty << "," << profit << "," << maxprofit << "," << maxloss << ","
		<< totalprofit << "," << enterTag << "," << exitTag << "," << openBarNo << "," << closeBarNo << "\n";
}

// 根据配置文件初始化cta策略工厂
bool CtaMocker::init_cta_factory(WTSVariant* cfg)
{
	if (cfg == NULL)
		return false;
	const char* module = cfg->getCString("module");		// 文件名
	DllHandle hInst = DLLHelper::load_library(module);	// 加载dll文件
	if (hInst == NULL)
		return false;
	// 策略工厂指针
	FuncCreateStraFact creator = (FuncCreateStraFact)DLLHelper::get_symbol(hInst, "createStrategyFact");
	if (creator == NULL)
	{
		DLLHelper::free_library(hInst);
		return false;
	}

	// 填充策略工厂结构体
	_factory._module_inst = hInst;
	_factory._module_path = module;
	_factory._creator = creator;
	_factory._remover = (FuncDeleteStraFact)DLLHelper::get_symbol(hInst, "deleteStrategyFact");
	_factory._fact = _factory._creator();

	WTSVariant* cfgStra = cfg->get("strategy");
	if (cfgStra)
	{
		// 创建策略
		_strategy = _factory._fact->createStrategy(cfgStra->getCString("name"), cfgStra->getCString("id"));
		if(_strategy)
		{
			WTSLogger::info("Strategy %s.%s is created,strategy ID: %s", _factory._fact->getName(), _strategy->getName(), _strategy->id());
		}
		// 初始化策略参数
		_strategy->init(cfgStra->get("params"));
		_name = _strategy->id();
	}
	return true;
}

// 回调调度器的 on_init
void CtaMocker::handle_init()
{
	this->on_init();
}

// 回调调度器的on_bar
void CtaMocker::handle_bar_close(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar)
{
	this->on_bar(stdCode, period, times, newBar);
}

// 回调调度器的 on_schedule
void CtaMocker::handle_schedule(uint32_t uDate, uint32_t uTime)
{
	this->on_schedule(uDate, uTime);
}

// 回调调度器的 on_session_begin
void CtaMocker::handle_session_begin(uint32_t curTDate)
{
	this->on_session_begin(curTDate);
}

// 回调调度器的 on_session_end
void CtaMocker::handle_session_end(uint32_t curTDate)
{
	this->on_session_end(curTDate);
}

// 回测结束回调
void CtaMocker::handle_replay_done()
{
	_in_backtest = false;

	if(_emit_times > 0)
	{
		WTSLogger::log_dyn_raw("strategy", _name.c_str(), LL_INFO, fmt::format("Strategy has been scheduled for {} times,totally taking {} microsecs,average of {} microsecs",
			_emit_times, _total_calc_time, _total_calc_time / _emit_times).c_str());
	}
	else
	{
		WTSLogger::log_dyn_raw("strategy", _name.c_str(), LL_INFO, fmt::format("Strategy has been scheduled for {} times", _emit_times).c_str());
	}

	dump_outputs();

	if (_has_hook && _hook_valid)
	{
		WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Replay done, notify control thread");
		while(_wait_calc)
			_cond_calc.notify_all();
		WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Notify control thread the end done");
	}

	WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Notify strategy the end of backtest");
	
	// 逻辑未实现
	this->on_bactest_end();
}

// 回调调度器的 on_tick
void CtaMocker::handle_tick(const char* stdCode, WTSTickData* curTick)
{
	this->on_tick(stdCode, curTick, true);
}

// 回调策略的 on_bar
void CtaMocker::on_bar(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar)
{
	if (newBar == NULL)
		return;

	std::string realPeriod;
	if (period[0] == 'd')
		realPeriod = StrUtil::printf("%s%u", period, times);
	else
		realPeriod = StrUtil::printf("m%u", times);

	std::string key = StrUtil::printf("%s#%s", stdCode, realPeriod.c_str());
	KlineTag& tag = _kline_tags[key];
	tag._closed = true;

	on_bar_close(stdCode, realPeriod.c_str(), newBar);
}

// 回调策略的 on_init
void CtaMocker::on_init()
{
	_in_backtest = true;
	if (_strategy)
		_strategy->on_init(this);

	WTSLogger::info("CTA Strategy initialized, with slippage: %d", _slippage);
}

// 计算资金信息
void CtaMocker::update_dyn_profit(const char* stdCode, double price)
{
	auto it = _pos_map.find(stdCode);
	if (it != _pos_map.end())
	{
		PosInfo& pInfo = (PosInfo&)it->second;
		if (pInfo._volume == 0)
		{
			pInfo._dynprofit = 0;
		}
		else
		{
			WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
			double dynprofit = 0;
			for (auto pit = pInfo._details.begin(); pit != pInfo._details.end(); pit++)
			{
				DetailInfo& dInfo = *pit;
				dInfo._profit = dInfo._volume*(price - dInfo._price)*commInfo->getVolScale()*(dInfo._long ? 1 : -1);
				if (dInfo._profit > 0)
					dInfo._max_profit = max(dInfo._profit, dInfo._max_profit);
				else if (dInfo._profit < 0)
					dInfo._max_loss = min(dInfo._profit, dInfo._max_loss);

				dynprofit += dInfo._profit;
			}
			pInfo._dynprofit = dynprofit;
		}
	}
	double total_dynprofit = 0;
	for (auto& v : _pos_map)
	{
		const PosInfo& pInfo = v.second;
		total_dynprofit += pInfo._dynprofit;
	}
	_fund_info._total_dynprofit = total_dynprofit;
}

// 调度器回调的 on_tick
void CtaMocker::on_tick(const char* stdCode, WTSTickData* newTick, bool bEmitStrategy /* = true */)
{
	double last_px = _price_map[stdCode];
	double cur_px = newTick->price();
	_price_map[stdCode] = cur_px;

	// 先检查是否要信号要触发
	{
		auto it = _sig_map.find(stdCode);
		if (it != _sig_map.end())
		{
			{
				const SigInfo& sInfo = it->second;
				double price;
				if (decimal::eq(sInfo._desprice, 0.0))
					price = newTick->price();
				else
					price = sInfo._desprice;
				do_set_position(stdCode, sInfo._volume, price, sInfo._usertag.c_str(), sInfo._triggered);
				_sig_map.erase(it);
			}

		}
	}

	update_dyn_profit(stdCode, newTick->price());

	// 检查条件单
	if (!_condtions.empty())
	{
		auto it = _condtions.find(stdCode);
		if (it == _condtions.end())
			return;

		const CondList& condList = it->second;
		double curPrice = cur_px;
		for (const CondEntrust& entrust : condList)
		{
			/*
			 * 如果开启了tick模式，就正常比较
			 * 但是如果没有开启tick模式，逻辑就非常复杂
			 * 因为不开回测的时候tick是用开高低收模拟出来的，如果直接按照目标价格触发，可能是有问题的
			 * 首先要拿到上一笔价格，和当前最新价格做一个比价，得到左边界和右边界
			 * 这里只能假设前后两笔价格之间是连续的，这样需要将两笔价格都加入判断
			 * 当条件是等于时，如果目标价格在左右边界之间，说明目标价格在这期间是出现过的，则认为价格匹配
			 * 当条件是大于的时候，我们需要判断右边界，即稍大的值是否满足条件，并取左边界与目标价中稍大的作为当前价
			 * 当条件是小于的时候，我们需要判断左边界，即稍小的值是否满足条件，并取右边界与目标价中稍小的作为当前价
			 */

			double left_px = min(last_px, cur_px);
			double right_px = max(last_px, cur_px);

			bool isMatched = false;	
			if(!_replayer->is_tick_simulated())
			{
				//如果tick数据不是模拟的，则使用最新价格
				switch (entrust._alg)
				{
				case WCT_Equal:
					isMatched = decimal::eq(curPrice, entrust._target);
					break;
				case WCT_Larger:
					isMatched = decimal::gt(curPrice, entrust._target);
					break;
				case WCT_LargerOrEqual:
					isMatched = decimal::ge(curPrice, entrust._target);
					break;
				case WCT_Smaller:
					isMatched = decimal::lt(curPrice, entrust._target);
					break;
				case WCT_SmallerOrEqual:
					isMatched = decimal::le(curPrice, entrust._target);
					break;
				default:
					break;
				}
			}
			else
			{
				// 如果tick数据是模拟的，则要处理一下
				switch (entrust._alg)
				{
				case WCT_Equal:
					isMatched = decimal::le(left_px, entrust._target) && decimal::ge(right_px, entrust._target);
					curPrice = entrust._target;
					break;
				case WCT_Larger:
					isMatched = decimal::gt(right_px, entrust._target);
					curPrice = max(left_px, entrust._target);
					break;
				case WCT_LargerOrEqual:
					isMatched = decimal::ge(right_px, entrust._target);
					curPrice = max(left_px, entrust._target);
					break;
				case WCT_Smaller:
					isMatched = decimal::lt(left_px, entrust._target);
					curPrice = min(right_px, entrust._target);
					break;
				case WCT_SmallerOrEqual:
					isMatched = decimal::le(left_px, entrust._target);
					curPrice = min(right_px, entrust._target);
					break;
				default:
					break;
				}
			}
			

			if (isMatched)
			{
				double price = curPrice;
				double curQty = stra_get_position(stdCode);
				//_replayer->is_tick_enabled() ? newTick->price() : entrust._target;	//如果开启了tick回测,则用tick数据的价格,如果没有开启,则只能用条件单价格
				WTSLogger::log_dyn_raw("strategy", _name.c_str(), LL_INFO, fmt::format("Condition order triggered[newprice: {}{}{}], instrument: {}, {} {}", cur_px, CMP_ALG_NAMES[entrust._alg], entrust._target, stdCode, ACTION_NAMES[entrust._action], entrust._qty).c_str());
				switch (entrust._action)
				{
				case COND_ACTION_OL:
				{
					if(decimal::lt(curQty, 0))
						append_signal(stdCode, entrust._qty, entrust._usertag, price);
					else
						append_signal(stdCode, curQty + entrust._qty, entrust._usertag, price);
				}
				break;
				case COND_ACTION_CL:
				{
					double maxQty = min(curQty, entrust._qty);
					append_signal(stdCode, curQty - maxQty, entrust._usertag, price);
				}
				break;
				case COND_ACTION_OS:
				{
					if(decimal::gt(curQty, 0))
						append_signal(stdCode, -entrust._qty, entrust._usertag, price);
					else
						append_signal(stdCode, curQty - entrust._qty, entrust._usertag, price);
				}
				break;
				case COND_ACTION_CS:
				{
					double maxQty = min(abs(curQty), entrust._qty);
					append_signal(stdCode, curQty + maxQty, entrust._usertag, price);
				}
				break;
				case COND_ACTION_SP:
				{
					append_signal(stdCode, entrust._qty, entrust._usertag, price);
				}
				default: break;
				}

				//同一个bar设置针对同一个合约的条件单,只可能触发一条
				//所以这里直接清理掉即可
				_condtions.erase(it);
				break;
			}
		}
	}

	if (bEmitStrategy)
		on_tick_updated(stdCode, newTick);
}

// 回调策略的 on_bar
void CtaMocker::on_bar_close(const char* code, const char* period, WTSBarStruct* newBar)
{
	if (_strategy)
		_strategy->on_bar(this, code, period, newBar);
}
// 回调策略的 on_tick
void CtaMocker::on_tick_updated(const char* code, WTSTickData* newTick)
{
	if (_strategy)
		_strategy->on_tick(this, code, newTick);
}
// 回调策略的 on_schedule
void CtaMocker::on_calculate(uint32_t curDate, uint32_t curTime)
{
	if (_strategy)
		_strategy->on_schedule(this, curDate, curTime);
}

// 外挂逻辑?
void CtaMocker::enable_hook(bool bEnabled /* = true */)
{
	_hook_valid = bEnabled;

	WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calculating hook %s", bEnabled?"enabled":"disabled");
}
void CtaMocker::install_hook()
{
	_has_hook = true;

	WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "CTA hook installed");
}
bool CtaMocker::step_calc()
{
	if (!_has_hook)
	{
		return false;
	}

	//总共分为4个状态
	//0-初始状态，1-oncalc，2-oncalc结束，3-oncalcdone
	//所以，如果出于0/2，则说明没有在执行中，需要notify
	bool bNotify = false;
	while (_in_backtest && (_cur_step == 0 || _cur_step == 2))
	{
		_cond_calc.notify_all();
		bNotify = true;
	}

	if(bNotify)
		WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Notify calc thread, wait for calc done");

	if(_in_backtest)
	{
		_wait_calc = true;
		StdUniqueLock lock(_mtx_calc);
		_cond_calc.wait(_mtx_calc);
		_wait_calc = false;
		WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calc done notified");
		_cur_step = (_cur_step + 1) % 4;

		return true;
	}
	else
	{
		_hook_valid = false;
		WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Backtest exit automatically");
		return false;
	}
}

// 回调策略的 on_schedule
bool CtaMocker::on_schedule(uint32_t curDate, uint32_t curTime)
{
	_is_in_schedule = true;			// 开始调度,修改标记
	_schedule_times++;
	bool isMainUdt = false;
	bool emmited = false;

	for (auto it = _kline_tags.begin(); it != _kline_tags.end(); it++)
	{
		const std::string& key = it->first;
		KlineTag& marker = (KlineTag&)it->second;

		StringVector ay = StrUtil::split(key, "#");
		const char* stdCode = ay[0].c_str();

		if (key == _main_key)
		{
			if (marker._closed)
			{
				isMainUdt = true;
				marker._closed = false;
			}
			else
			{
				isMainUdt = false;
				break;
			}
		}
		WTSSessionInfo* sInfo = _replayer->get_session_info(stdCode, true);
		if (isMainUdt || _kline_tags.empty())
		{
			TimeUtils::Ticker ticker;
			uint32_t offTime = sInfo->offsetTime(curTime);
			if (offTime <= sInfo->getCloseTime(true))
			{
				_condtions.clear();
				if(_has_hook && _hook_valid)
				{
					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Waiting for resume notify");
					StdUniqueLock lock(_mtx_calc);
					_cond_calc.wait(_mtx_calc);
					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calc resumed");
					_cur_step = 1;
				}

				on_calculate(curDate, curTime);

				if (_has_hook && _hook_valid)
				{
					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calc done, notify control thread");
					while (_cur_step==1)
						_cond_calc.notify_all();

					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Waiting for resume notify");
					StdUniqueLock lock(_mtx_calc);
					_cond_calc.wait(_mtx_calc);
					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calc resumed");
					_cur_step = 3;
				}
				if(_has_hook)
					on_calculate_done(curDate, curTime);
				emmited = true;

				_emit_times++;
				_total_calc_time += ticker.micro_seconds();

				if (_has_hook && _hook_valid)
				{
					WTSLogger::log_dyn("strategy", _name.c_str(), LL_DEBUG, "Calc done, notify control thread");
					while(_cur_step == 3)
						_cond_calc.notify_all();
				}
			}
			else
			{
				WTSLogger::log_dyn("strategy", _name.c_str(), LL_INFO, "%u is not trading time,strategy will not be scheduled", curTime);
			}
			break;
		}
	}
	_is_in_schedule = false;	// 调度结束,修改标记
	return emmited;
}

// 每个交易日开始，要把冻结持仓置零
void CtaMocker::on_session_begin(uint32_t curTDate)
{
	for (auto& it : _pos_map)
	{
		const char* stdCode = it.first.c_str();
		PosInfo& pInfo = (PosInfo&)it.second;
		if (!decimal::eq(pInfo._frozen, 0))
		{
			stra_log_debug("%.0f of %s frozen released on %u", pInfo._frozen, stdCode, curTDate);
			pInfo._frozen = 0;
		}
	}
}

// 获取持仓并调用 cb 处理
void CtaMocker::enum_position(FuncEnumCtaPosCallBack cb)
{
	faster_hashmap<std::string, double> desPos;
	for (auto& it : _pos_map)
	{
		const char* stdCode = it.first.c_str();
		const PosInfo& pInfo = it.second;
		desPos[stdCode] = pInfo._volume;
	}
	for (auto sit : _sig_map)
	{
		const char* stdCode = sit.first.c_str();
		const SigInfo& sInfo = sit.second;
		desPos[stdCode] = sInfo._volume;
	}
	for (auto v : desPos)
	{
		cb(v.first.c_str(), v.second);
	}
}

// 计算资金信息并输出
void CtaMocker::on_session_end(uint32_t curTDate)
{
	uint32_t curDate = curTDate;

	double total_profit = 0;
	double total_dynprofit = 0;

	for (auto it = _pos_map.begin(); it != _pos_map.end(); it++)
	{
		const char* stdCode = it->first.c_str();
		const PosInfo& pInfo = it->second;
		total_profit += pInfo._closeprofit;
		total_dynprofit += pInfo._dynprofit;
	}
	_fund_logs << StrUtil::printf("%d,%.2f,%.2f,%.2f,%.2f\n", curDate,
		_fund_info._total_profit, _fund_info._total_dynprofit,
		_fund_info._total_profit + _fund_info._total_dynprofit - _fund_info._total_fees, _fund_info._total_fees);

	if (_notifier)
		_notifier->notifyFund("BT_FUND", curDate, _fund_info._total_profit, _fund_info._total_dynprofit,
			_fund_info._total_profit + _fund_info._total_dynprofit - _fund_info._total_fees, _fund_info._total_fees);
}

// 
CondList& CtaMocker::get_cond_entrusts(const char* stdCode)
{
	CondList& ce = _condtions[stdCode];
	return ce;
}

// 开多
void CtaMocker::stra_enter_long(const char* stdCode, double qty, const char* userTag /* = "" */, double limitprice, double stopprice)
{
	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if(commInfo == NULL)
	{
		stra_log_error("Cannot find corresponding commodity info of %s", stdCode);
		return;
	}

	_replayer->sub_tick(_context_id, stdCode);
	if (decimal::eq(limitprice, 0.0) && decimal::eq(stopprice, 0.0))	//如果不是动态下单模式,则直接触发
	{
		double curQty = stra_get_position(stdCode);
		if(decimal::lt(curQty, 0))
			append_signal(stdCode, qty, userTag);
		else
			append_signal(stdCode, curQty + qty, userTag);
	}
	else
	{
		CondList& condList = get_cond_entrusts(stdCode);

		CondEntrust entrust;
		strcpy(entrust._code, stdCode);
		strcpy(entrust._usertag, userTag);

		entrust._qty = qty;
		entrust._field = WCF_NEWPRICE;
		if (!decimal::eq(limitprice))
		{
			entrust._target = limitprice;
			entrust._alg = WCT_SmallerOrEqual;
		}
		else if (!decimal::eq(stopprice))
		{
			entrust._target = stopprice;
			entrust._alg = WCT_LargerOrEqual;
		}

		entrust._action = COND_ACTION_OL;

		condList.emplace_back(entrust);
	}
}

// 开空
void CtaMocker::stra_enter_short(const char* stdCode, double qty, const char* userTag /* = "" */, double limitprice, double stopprice)
{
	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if (commInfo == NULL)
	{
		stra_log_error("Cannot find corresponding commodity info of %s", stdCode);
		return;
	}

	if(!commInfo->canShort())
	{
		stra_log_error("Cannot short on %s", stdCode);
		return;
	}

	_replayer->sub_tick(_context_id, stdCode);
	if (decimal::eq(limitprice, 0.0) && decimal::eq(stopprice, 0.0))	//如果不是动态下单模式,则直接触发
	{
		double curQty = stra_get_position(stdCode);
		if(decimal::gt(curQty, 0))
			append_signal(stdCode, -qty, userTag);
		else
			append_signal(stdCode, curQty - qty, userTag);

	}
	else
	{
		CondList& condList = get_cond_entrusts(stdCode);
		CondEntrust entrust;
		strcpy(entrust._code, stdCode);
		strcpy(entrust._usertag, userTag);
		entrust._qty = qty;
		entrust._field = WCF_NEWPRICE;
		if (!decimal::eq(limitprice))
		{
			entrust._target = limitprice;
			entrust._alg = WCT_LargerOrEqual;
		}
		else if (!decimal::eq(stopprice))
		{
			entrust._target = stopprice;
			entrust._alg = WCT_SmallerOrEqual;
		}
		entrust._action = COND_ACTION_OS;
		condList.emplace_back(entrust);
	}
}

// 平多
void CtaMocker::stra_exit_long(const char* stdCode, double qty, const char* userTag /* = "" */, double limitprice, double stopprice)
{
	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if (commInfo == NULL)
	{
		stra_log_error("Cannot find corresponding commodity info of %s", stdCode);
		return;
	}

	//读取可平持仓
	double curQty = stra_get_position(stdCode, true);
	if (decimal::le(curQty, 0))
		return;

	if (decimal::eq(limitprice, 0.0) && decimal::eq(stopprice, 0.0))	//如果不是动态下单模式,则直接触发
	{
		double maxQty = min(curQty, qty);
		append_signal(stdCode, curQty - qty, userTag);
	}
	else
	{
		CondList& condList = get_cond_entrusts(stdCode);
		CondEntrust entrust;
		strcpy(entrust._code, stdCode);
		strcpy(entrust._usertag, userTag);
		entrust._qty = qty;
		entrust._field = WCF_NEWPRICE;
		if (!decimal::eq(limitprice))
		{
			entrust._target = limitprice;
			entrust._alg = WCT_LargerOrEqual;
		}
		else if (!decimal::eq(stopprice))
		{
			entrust._target = stopprice;
			entrust._alg = WCT_SmallerOrEqual;
		}
		entrust._action = COND_ACTION_CL;
		condList.emplace_back(entrust);
	}
}

// 平空
void CtaMocker::stra_exit_short(const char* stdCode, double qty, const char* userTag /* = "" */, double limitprice, double stopprice)
{
	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if (commInfo == NULL)
	{
		stra_log_error("Cannot find corresponding commodity info of %s", stdCode);
		return;
	}

	if (!commInfo->canShort())
	{
		stra_log_error("Cannot short on %s", stdCode);
		return;
	}

	double curQty = stra_get_position(stdCode);
	if (decimal::ge(curQty, 0))
		return;

	if (decimal::eq(limitprice, 0.0) && decimal::eq(stopprice, 0.0))	//如果不是动态下单模式,则直接触发
	{
		double maxQty = min(abs(curQty), qty);
		append_signal(stdCode, curQty + maxQty, userTag);
	}
	else
	{
		CondList& condList = get_cond_entrusts(stdCode);

		CondEntrust entrust;
		strcpy(entrust._code, stdCode);
		strcpy(entrust._usertag, userTag);

		entrust._qty = qty;
		entrust._field = WCF_NEWPRICE;
		if (!decimal::eq(limitprice))
		{
			entrust._target = limitprice;
			entrust._alg = WCT_SmallerOrEqual;
		}
		else if (!decimal::eq(stopprice))
		{
			entrust._target = stopprice;
			entrust._alg = WCT_LargerOrEqual;
		}

		entrust._action = COND_ACTION_CS;

		condList.emplace_back(entrust);
	}
}

// 调用回测引擎获取当前价格
double CtaMocker::stra_get_price(const char* stdCode)
{
	if (_replayer)
		return _replayer->get_cur_price(stdCode);
	return 0.0;
}

// 设置目标仓位
void CtaMocker::stra_set_position(const char* stdCode, double qty, const char* userTag /* = "" */, double limitprice /* = 0.0 */, double stopprice /* = 0.0 */)
{
	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if (commInfo == NULL)
	{
		stra_log_error("Cannot find corresponding commodity info of %s", stdCode);
		return;
	}

	// 如果不能做空，则目标仓位不能设置负数
	if (!commInfo->canShort() && decimal::lt(qty, 0))
	{
		stra_log_error("Cannot short on %s", stdCode);
		return;
	}
	double total = stra_get_position(stdCode, false);
	// 如果目标仓位和当前仓位是一致的，直接退出
	if (decimal::eq(total, qty))
		return;
	if(commInfo->isT1())
	{
		double valid = stra_get_position(stdCode, true);
		double frozen = total - valid;
		// 如果是T+1规则，则目标仓位不能小于冻结仓位
		if(decimal::lt(qty, frozen))
		{
			stra_log_error(fmt::format("New position of {} cannot be set to {} due to {} being frozen", stdCode, qty, frozen).c_str());
			return;
		}
	}
	_replayer->sub_tick(_context_id, stdCode);
	if (decimal::eq(limitprice, 0.0) && decimal::eq(stopprice, 0.0))	// 没有设置触发条件，则直接添加信号
	{
		append_signal(stdCode, qty, userTag);
	}
	else
	{
		CondList& condList = get_cond_entrusts(stdCode);
		bool isBuy = decimal::gt(qty, total);
		CondEntrust entrust;
		strcpy(entrust._code, stdCode);
		strcpy(entrust._usertag, userTag);
		entrust._qty = qty;
		entrust._field = WCF_NEWPRICE;
		if (!decimal::eq(limitprice))
		{
			entrust._target = limitprice;
			entrust._alg = isBuy ? WCT_SmallerOrEqual : WCT_LargerOrEqual;
		}
		else if (!decimal::eq(stopprice))
		{
			entrust._target = stopprice;
			entrust._alg = isBuy ? WCT_LargerOrEqual : WCT_SmallerOrEqual;
		}
		entrust._action = COND_ACTION_SP;
		condList.emplace_back(entrust);
	}
}

// 记录交易信号
void CtaMocker::append_signal(const char* stdCode, double qty, const char* userTag /* = "" */, double price/* = 0.0*/)
{
	double curPx = _price_map[stdCode];

	SigInfo& sInfo = _sig_map[stdCode];
	sInfo._volume = qty;
	sInfo._sigprice = curPx;
	sInfo._desprice = price;
	sInfo._usertag = userTag;
	sInfo._gentime = (uint64_t)_replayer->get_date() * 1000000000 + (uint64_t)_replayer->get_raw_time() * 100000 + _replayer->get_secs();
	sInfo._triggered = !_is_in_schedule;

	log_signal(stdCode, qty, curPx, sInfo._gentime, userTag);
}

// 调整持仓到目标持仓
void CtaMocker::do_set_position(const char* stdCode, double qty, double price /* = 0.0 */, const char* userTag /* = "" */, bool bTriggered /* = false */)
{
	PosInfo& pInfo = _pos_map[stdCode];
	double curPx = price;
	if (decimal::eq(price, 0.0))
		curPx = _price_map[stdCode];
	uint64_t curTm = (uint64_t)_replayer->get_date() * 10000 + _replayer->get_min_time();
	uint32_t curTDate = _replayer->get_trading_date();

	// 手数相等则不用操作了
	if (decimal::eq(pInfo._volume, qty))
		return;

	WTSCommodityInfo* commInfo = _replayer->get_commodity_info(stdCode);
	if (commInfo == NULL)
		return;

	// 成交价
	double trdPx = curPx;

	double diff = qty - pInfo._volume;
	bool isBuy = decimal::gt(diff, 0.0);
	if (decimal::gt(pInfo._volume*diff, 0))			// 当前持仓和仓位变化方向一致, 增加一条明细, 增加数量即可
	{
		pInfo._volume = qty;

		// 如果T+1，则冻结仓位要增加
		if (commInfo->isT1())
		{
			pInfo._frozen += diff;
			stra_log_debug("%s frozen position up to %.0f", stdCode, pInfo._frozen);
		}
		
		if (_slippage != 0)
		{
			trdPx += _slippage * commInfo->getPriceTick()*(isBuy ? 1 : -1);
		}
		DetailInfo dInfo;
		dInfo._long = decimal::gt(qty, 0);
		dInfo._price = trdPx;
		dInfo._volume = abs(diff);
		dInfo._opentime = curTm;
		dInfo._opentdate = curTDate;
		strcpy(dInfo._opentag, userTag);
		dInfo._open_barno = _schedule_times;
		pInfo._details.emplace_back(dInfo);
		pInfo._last_entertime = curTm;
		double fee = _replayer->calc_fee(stdCode, trdPx, abs(diff), 0);
		_fund_info._total_fees += fee;

		log_trade(stdCode, dInfo._long, true, curTm, trdPx, abs(diff), userTag, fee, _schedule_times);
	}
	else
	{
		//持仓方向和仓位变化方向不一致,需要平仓
		double left = abs(diff);
		if (_slippage != 0)
			trdPx += _slippage * commInfo->getPriceTick()*(isBuy ? 1 : -1);

		pInfo._volume = qty;
		if (decimal::eq(pInfo._volume, 0))
			pInfo._dynprofit = 0;
		uint32_t count = 0;
		for (auto it = pInfo._details.begin(); it != pInfo._details.end(); it++)
		{
			DetailInfo& dInfo = *it;
			double maxQty = min(dInfo._volume, left);
			if (decimal::eq(maxQty, 0))
				continue;
			double maxProf = dInfo._max_profit * maxQty / dInfo._volume;
			double maxLoss = dInfo._max_loss * maxQty / dInfo._volume;
			dInfo._volume -= maxQty;
			left -= maxQty;
			if (decimal::eq(dInfo._volume, 0))
				count++;

			double profit = (trdPx - dInfo._price) * maxQty * commInfo->getVolScale();
			if (!dInfo._long)
				profit *= -1;
			pInfo._closeprofit += profit;
			_total_closeprofit += profit;
			pInfo._dynprofit = pInfo._dynprofit*dInfo._volume / (dInfo._volume + maxQty);//浮盈也要做等比缩放
			pInfo._last_exittime = curTm;
			_fund_info._total_profit += profit;

			double fee = _replayer->calc_fee(stdCode, trdPx, maxQty, dInfo._opentdate == curTDate ? 2 : 1);
			_fund_info._total_fees += fee;
			//这里写成交记录
			log_trade(stdCode, dInfo._long, false, curTm, trdPx, maxQty, userTag, fee, _schedule_times);
			//这里写平仓记录
			log_close(stdCode, dInfo._long, dInfo._opentime, dInfo._price, curTm, trdPx, maxQty, profit, maxProf, maxLoss, 
				_total_closeprofit - _fund_info._total_fees, dInfo._opentag, userTag, dInfo._open_barno, _schedule_times);
			if (left == 0)
				break;
		}

		// 需要清理掉已经平仓完的明细
		while (count > 0)
		{
			auto it = pInfo._details.begin();
			pInfo._details.erase(it);
			count--;
		}
		// 最后,如果还有剩余的,则需要反手了
		if (left > 0)
		{
			left = left * qty / abs(qty);

			// 如果T+1，则冻结仓位要增加
			if (commInfo->isT1())
			{
				pInfo._frozen += left;
				stra_log_debug("%s frozen position up to %.0f", stdCode, pInfo._frozen);
			}
			DetailInfo dInfo;
			dInfo._long = decimal::gt(qty, 0);
			dInfo._price = trdPx;
			dInfo._volume = abs(left);
			dInfo._opentime = curTm;
			dInfo._opentdate = curTDate;
			dInfo._open_barno = _schedule_times;
			strcpy(dInfo._opentag, userTag);
			pInfo._details.emplace_back(dInfo);
			// 这里还需要写一笔成交记录
			double fee = _replayer->calc_fee(stdCode, trdPx, abs(left), 0);
			_fund_info._total_fees += fee;
			log_trade(stdCode, dInfo._long, true, curTm, trdPx, abs(left), userTag, fee, _schedule_times);

			pInfo._last_entertime = curTm;
		}
	}
}

WTSKlineSlice* CtaMocker::stra_get_bars(const char* stdCode, const char* period, uint32_t count, bool isMain /* = false */)
{
	std::string key = StrUtil::printf("%s#%s", stdCode, period);
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
		key.append("1");
	}
	if (isMain)
	{
		if (_main_key.empty())
			_main_key = key;
		else if (_main_key != key)
			throw std::runtime_error("Main k bars can only be setup once");
	}

	WTSKlineSlice* kline = _replayer->get_kline_slice(stdCode, basePeriod.c_str(), count, times, isMain);
	bool bFirst = (_kline_tags.find(key) == _kline_tags.end());
	KlineTag& tag = _kline_tags[key];
	tag._closed = false;

	if (kline)
	{
		CodeHelper::CodeInfo cInfo = CodeHelper::extractStdCode(stdCode);
		std::string realCode = stdCode;
		if(cInfo._category == CC_Stock && cInfo.isExright())
			realCode = StrUtil::printf("%s.%s.%s", cInfo._exchg, cInfo._product, cInfo._code);
		_replayer->sub_tick(id(), realCode.c_str());
	}
	return kline;
}

// 回调回测引擎对应的函数
WTSTickSlice* CtaMocker::stra_get_ticks(const char* stdCode, uint32_t count)
{
	return _replayer->get_tick_slice(stdCode, count);
}
WTSTickData* CtaMocker::stra_get_last_tick(const char* stdCode)
{
	return _replayer->get_last_tick(stdCode);
}
void CtaMocker::stra_sub_ticks(const char* code)
{
	_replayer->sub_tick(_context_id, code);
}
WTSCommodityInfo* CtaMocker::stra_get_comminfo(const char* stdCode)
{
	return _replayer->get_commodity_info(stdCode);
}
uint32_t CtaMocker::stra_get_tdate()
{
	return _replayer->get_trading_date();
}
uint32_t CtaMocker::stra_get_date()
{
	return _replayer->get_date();
}
uint32_t CtaMocker::stra_get_time()
{
	return _replayer->get_min_time();
}

// 获取对应资金信息
double CtaMocker::stra_get_fund_data(int flag)
{
	switch (flag)
	{
	case 0:
		return _fund_info._total_profit - _fund_info._total_fees + _fund_info._total_dynprofit;
	case 1:
		return _fund_info._total_profit;
	case 2:
		return _fund_info._total_dynprofit;
	case 3:
		return _fund_info._total_fees;
	default:
		return 0.0;
	}
}
	
// 输出不同级别日志
void CtaMocker::stra_log_info(const char* fmt, ...)
{
	va_list args;
	va_start(args, fmt);
	WTSLogger::vlog_dyn("strategy", _name.c_str(), LL_INFO, fmt, args);
	va_end(args);
}
void CtaMocker::stra_log_debug(const char* fmt, ...)
{
	va_list args;
	va_start(args, fmt);
	WTSLogger::vlog_dyn("strategy", _name.c_str(), LL_DEBUG, fmt, args);
	va_end(args);
}
void CtaMocker::stra_log_error(const char* fmt, ...)
{
	va_list args;
	va_start(args, fmt);
	WTSLogger::vlog_dyn("strategy", _name.c_str(), LL_ERROR, fmt, args);
	va_end(args);
}

// 加载用户数据
const char* CtaMocker::stra_load_user_data(const char* key, const char* defVal /*= ""*/)
{
	auto it = _user_datas.find(key);
	if (it != _user_datas.end())
		return it->second.c_str();

	return defVal;
}
// 更新用户数据
void CtaMocker::stra_save_user_data(const char* key, const char* val)
{
	_user_datas[key] = val;
	_ud_modified = true;
}

// 获取首次开仓时间
uint64_t CtaMocker::stra_get_first_entertime(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return INVALID_UINT64;

	const PosInfo& pInfo = it->second;
	if (pInfo._details.empty())
		return INVALID_UINT64;

	return pInfo._details[0]._opentime;
}

// 获取最后一次开仓时间
uint64_t CtaMocker::stra_get_last_entertime(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return INVALID_UINT64;

	const PosInfo& pInfo = it->second;
	if (pInfo._details.empty())
		return INVALID_UINT64;

	return pInfo._details[pInfo._details.size() - 1]._opentime;
}

// 获取最后一次退出平仓时间
uint64_t CtaMocker::stra_get_last_exittime(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return INVALID_UINT64;

	const PosInfo& pInfo = it->second;
	return pInfo._last_exittime;
}

// 获取最后一次开仓价格
double CtaMocker::stra_get_last_enterprice(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return INVALID_DOUBLE;

	const PosInfo& pInfo = it->second;
	if (pInfo._details.empty())
		return INVALID_DOUBLE;

	return pInfo._details[pInfo._details.size() - 1]._price;
}

// 获取持仓信息
double CtaMocker::stra_get_position(const char* stdCode, bool bOnlyValid /* = false */, const char* userTag /* = "" */)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;

	const PosInfo& pInfo = it->second;
	if (strlen(userTag) == 0)
	{
		//只有userTag为空的时候时候，才会用bOnlyValid
		if(bOnlyValid)
		{
			//这里理论上，只有多头才会进到这里
			//其他地方要保证，空头持仓的话，_frozen要为0
			return pInfo._volume - pInfo._frozen;
		}
		else
			return pInfo._volume;
	}

	for (auto it = pInfo._details.begin(); it != pInfo._details.end(); it++)
	{
		const DetailInfo& dInfo = (*it);
		if (strcmp(dInfo._opentag, userTag) != 0)
			continue;
		return dInfo._volume;
	}
	return 0;
}

// 获取持仓均价
double CtaMocker::stra_get_position_avgpx(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;

	const PosInfo& pInfo = it->second;
	if (pInfo._volume == 0)
		return 0.0;

	double amount = 0.0;
	for (auto dit = pInfo._details.begin(); dit != pInfo._details.end(); dit++)
	{
		const DetailInfo& dInfo = *dit;
		amount += dInfo._price*dInfo._volume;
	}
	return amount / pInfo._volume;
}

// 获取持仓利润
double CtaMocker::stra_get_position_profit(const char* stdCode)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;
	const PosInfo& pInfo = it->second;
	return pInfo._dynprofit;
}

// 查询开仓时间
uint64_t CtaMocker::stra_get_detail_entertime(const char* stdCode, const char* userTag)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;

	const PosInfo& pInfo = it->second;
	for (auto it = pInfo._details.begin(); it != pInfo._details.end(); it++)
	{
		const DetailInfo& dInfo = (*it);
		if (strcmp(dInfo._opentag, userTag) != 0)
			continue;

		return dInfo._opentime;
	}

	return 0;
}

// 查询开仓价格
double CtaMocker::stra_get_detail_cost(const char* stdCode, const char* userTag)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;

	const PosInfo& pInfo = it->second;
	for (auto it = pInfo._details.begin(); it != pInfo._details.end(); it++)
	{
		const DetailInfo& dInfo = (*it);
		if (strcmp(dInfo._opentag, userTag) != 0)
			continue;

		return dInfo._price;
	}

	return 0.0;
}

// 查询利润
double CtaMocker::stra_get_detail_profit(const char* stdCode, const char* userTag, int flag /* = 0 */)
{
	auto it = _pos_map.find(stdCode);
	if (it == _pos_map.end())
		return 0;

	const PosInfo& pInfo = it->second;
	for (auto it = pInfo._details.begin(); it != pInfo._details.end(); it++)
	{
		const DetailInfo& dInfo = (*it);
		if (strcmp(dInfo._opentag, userTag) != 0)
			continue;
		if (flag == 0)
			return dInfo._profit;
		else if (flag > 0)
			return dInfo._max_profit;
		else
			return dInfo._max_loss;
	}
	return 0.0;
}
```