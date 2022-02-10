# 04-10 HFT接口

source: `{{ page.path }}`

## HftStrategyDefs.h

### HftStrategy

HFT策略接口, 将在编写HFT策略时被继承

```cpp
class HftStrategy
{
public:
	HftStrategy(const char* id) :_id(id){}
	virtual ~HftStrategy(){}

public:
	// 执行单元名称
	virtual const char* getName() = 0;

	// 所属执行器工厂名称
	virtual const char* getFactName() = 0;

	// 初始化
	virtual bool init(WTSVariant* cfg){ return true; }

	virtual const char* id() const { return _id.c_str(); }

	// 回调函数
	virtual void on_init(IHftStraCtx* ctx) = 0;
	virtual void on_session_begin(IHftStraCtx* ctx, uint32_t uTDate) {}
	virtual void on_session_end(IHftStraCtx* ctx, uint32_t uTDate) {}
	virtual void on_tick(IHftStraCtx* ctx, const char* code, WTSTickData* newTick) = 0;
	virtual void on_order_queue(IHftStraCtx* ctx, const char* code, WTSOrdQueData* newOrdQue) {}
	virtual void on_order_detail (IHftStraCtx* ctx, const char* code, WTSOrdDtlData* newOrdDtl) {}
	virtual void on_transaction(IHftStraCtx* ctx, const char* code, WTSTransData* newTrans) {}
	virtual void on_bar(IHftStraCtx* ctx, const char* code, const char* period, uint32_t times, WTSBarStruct* newBar) = 0;
	virtual void on_trade(IHftStraCtx* ctx, uint32_t localid, const char* stdCode, bool isBuy, double vol, double price, const char* userTag) = 0;
	virtual void on_position(IHftStraCtx* ctx, const char* stdCode, bool isLong, double prevol, double preavail, double newvol, double newavail) = 0;
	virtual void on_order(IHftStraCtx* ctx, uint32_t localid, const char* stdCode, bool isBuy, double totalQty, double leftQty, double price, bool isCanceled, const char* userTag) = 0;
	virtual void on_channel_ready(IHftStraCtx* ctx) = 0;
	virtual void on_channel_lost(IHftStraCtx* ctx) = 0;
	virtual void on_entrust(uint32_t localid, bool bSuccess, const char* message, const char* userTag) = 0;

protected:
	std::string _id;
};
```

### IHftStrategyFact

HFT策略工厂接口

```cpp
typedef void(*FuncEnumHftStrategyCallback)(const char* factName, const char* straName, bool isLast);

class IHftStrategyFact
{
public:
	IHftStrategyFact(){}
	virtual ~IHftStrategyFact(){}

public:
	// 获取工厂名
	virtual const char* getName() = 0;

	// 枚举策略
	virtual void enumStrategy(FuncEnumHftStrategyCallback cb) = 0;

	// 根据名称创建执行单元
	virtual HftStrategy* createStrategy(const char* name, const char* id) = 0;

	// 删除执行单元
	virtual bool deleteStrategy(HftStrategy* stra) = 0;
};

// 创建执行工厂
typedef IHftStrategyFact* (*FuncCreateHftStraFact)();
// 删除执行工厂
typedef void(*FuncDeleteHftStraFact)(IHftStrategyFact* &fact);
```

## IHftStraCtx.h

### IHftStraCtx

HFT策略上下文接口

```cpp
class IHftStraCtx
{
public:
	IHftStraCtx(const char* name) :_name(name) {}
	virtual ~IHftStraCtx() {}

	const char* name() const { return _name.c_str(); }

public:
	virtual uint32_t id() = 0;

	// 回调函数
	virtual void on_init() = 0;
	virtual void on_tick(const char* stdCode, WTSTickData* newTick) = 0;
	virtual void on_order_queue(const char* stdCode, WTSOrdQueData* newOrdQue) = 0;
	virtual void on_order_detail(const char* stdCode, WTSOrdDtlData* newOrdDtl) = 0;
	virtual void on_transaction(const char* stdCode, WTSTransData* newTrans) = 0;
	virtual void on_bar(const char* stdCode, const char* period, uint32_t times, WTSBarStruct* newBar) {}
	virtual void on_session_begin(uint32_t uTDate) {}
	virtual void on_session_end(uint32_t uTDate) {}
	
	// 回测结束事件, 只在回测下才会触发
	virtual void on_bactest_end() {};

	virtual void on_tick_updated(const char* stdCode, WTSTickData* newTick) {}
	virtual void on_ordque_updated(const char* stdCode, WTSOrdQueData* newOrdQue) {}
	virtual void on_orddtl_updated(const char* stdCode, WTSOrdDtlData* newOrdDtl) {}
	virtual void on_trans_updated(const char* stdCode, WTSTransData* newTrans) {}

	// 策略接口
	virtual bool		stra_cancel(uint32_t localid) = 0;
	virtual OrderIDs	stra_cancel(const char* stdCode, bool isBuy, double qty) = 0;
	virtual OrderIDs	stra_buy(const char* stdCode, double price, double qty, const char* userTag) = 0;
	virtual OrderIDs	stra_sell(const char* stdCode, double price, double qty, const char* userTag) = 0;

	virtual uint32_t	stra_enter_long(const char* stdCode, double price, double qty, const char* userTag) { return 0; }
	virtual uint32_t	stra_enter_short(const char* stdCode, double price, double qty, const char* userTag) { return 0; }
	virtual uint32_t	stra_exit_long(const char* stdCode, double price, double qty, const char* userTag, bool isToday = false) { return 0; }
	virtual uint32_t	stra_exit_short(const char* stdCode, double price, double qty, const char* userTag, bool isToday = false) { return 0; }

	virtual WTSCommodityInfo* stra_get_comminfo(const char* stdCode) = 0;
	virtual WTSKlineSlice*	stra_get_bars(const char* stdCode, const char* period, uint32_t count) = 0;
	virtual WTSTickSlice*	stra_get_ticks(const char* stdCode, uint32_t count) = 0;
	virtual WTSOrdDtlSlice*	stra_get_order_detail(const char* stdCode, uint32_t count) = 0;
	virtual WTSOrdQueSlice*	stra_get_order_queue(const char* stdCode, uint32_t count) = 0;
	virtual WTSTransSlice*	stra_get_transaction(const char* stdCode, uint32_t count) = 0;
	virtual WTSTickData*	stra_get_last_tick(const char* stdCode) = 0;

	virtual double stra_get_position(const char* stdCode, bool bOnlyValid = false) = 0;
	virtual double stra_get_position_profit(const char* stdCode) = 0;
	virtual double stra_get_price(const char* stdCode) = 0;
	virtual double stra_get_undone(const char* stdCode) = 0;

	virtual uint32_t stra_get_date() = 0;
	virtual uint32_t stra_get_time() = 0;
	virtual uint32_t stra_get_secs() = 0;

	virtual void stra_sub_ticks(const char* stdCode) = 0;
	virtual void stra_sub_order_queues(const char* stdCode) = 0;
	virtual void stra_sub_order_details(const char* stdCode) = 0;
	virtual void stra_sub_transactions(const char* stdCode) = 0;

	virtual void stra_log_info(const char* fmt, ...) = 0;
	virtual void stra_log_debug(const char* fmt, ...) = 0;
	virtual void stra_log_error(const char* fmt, ...) = 0;

	virtual void stra_save_user_data(const char* key, const char* val) {}

	virtual const char* stra_load_user_data(const char* key, const char* defVal = "") { return defVal; }

protected:
	std::string _name;
};
```