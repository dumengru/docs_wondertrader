# IDataManager.h

source: `{{ page.path }}`

```cpp
#pragma once
#include "../Includes/WTSTypes.h"

NS_WTP_BEGIN
class WTSTickSlice;
class WTSKlineSlice;
class WTSTickData;
class WTSOrdQueSlice;
class WTSOrdDtlSlice;
class WTSTransSlice;

/*
交易数据管理器接口
主要处理K线, 订单等交易数据切片
*/
class IDataManager
{
public:
	// 获取各类交易数据切片
	virtual WTSTickSlice* get_tick_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSOrdQueSlice* get_order_queue_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSOrdDtlSlice* get_order_detail_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSTransSlice* get_transaction_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSKlineSlice* get_kline_slice(const char* stdCode, WTSKlinePeriod period, uint32_t times, uint32_t count, uint64_t etime = 0) { return NULL; }
	// 获取最新tick数据
	virtual WTSTickData* grab_last_tick(const char* stdCode) { return NULL; }

	virtual double get_adjusting_factor(const char* stdCode, uint32_t uDate) { return 1.0; }
};

NS_WTP_END

```