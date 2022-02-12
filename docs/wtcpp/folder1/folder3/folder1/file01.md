# IBaseDataMgr.h

source: `{{ page.path }}`

```cpp
/*!
 * \file IBaseDataMgr.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 基础数据管理器接口定义
 */
#pragma once
#include <string>
#include <stdint.h>

#include "WTSMarcos.h"
#include "FasterDefs.h"

typedef faster_hashset<std::string> ContractSet;

NS_WTP_BEGIN
class WTSContractInfo;
class WTSArray;
class WTSSessionInfo;
class WTSCommodityInfo;

// 节日元组
typedef faster_hashset<uint32_t> HolidaySet;
// 节日模板结构体
typedef struct _TradingDayTpl
{
	uint32_t	_cur_tdate;
	HolidaySet	_holidays;

	_TradingDayTpl() :_cur_tdate(0){}
} TradingDayTpl;

// 数据管理器接口类
class IBaseDataMgr
{
public:
	// 获取品种信息
	virtual WTSCommodityInfo*	getCommodity(const char* exchgpid)						= 0;
	virtual WTSCommodityInfo*	getCommodity(const char* exchg, const char* pid)		= 0;
	virtual WTSCommodityInfo*	getCommodity(WTSContractInfo* ct)						= 0;
	// 获取合约信息
	virtual WTSContractInfo*	getContract(const char* code, const char* exchg = "")	= 0;
	virtual WTSArray*			getContracts(const char* exchg = "")					= 0; 
	// 获取交易时间段信息
	virtual WTSSessionInfo*		getSession(const char* sid)						= 0;
	virtual WTSSessionInfo*		getSessionByCode(const char* code, const char* exchg = "") = 0;
	virtual WTSArray*			getAllSessions() = 0;
	// 判断是否是节日
	virtual bool				isHoliday(const char* pid, uint32_t uDate, bool isTpl = false) = 0;
	// 计算交易日
	virtual uint32_t			calcTradingDate(const char* stdPID, uint32_t uDate, uint32_t uTime, bool isSession = false) = 0;
	// 获取分割时间
	virtual uint64_t			getBoundaryTime(const char* stdPID, uint32_t tDate, bool isSession = false, bool isStart = true) = 0;
};
NS_WTP_END
```