# MatchEngine.h

source: `{{ page.path }}`

```cpp
#pragma once
#include <stdint.h>
#include <map>
#include <vector>
#include <functional>
#include <string.h>

#include "../Includes/WTSMarcos.h"
#include "../Includes/WTSCollection.hpp"
#include "../Includes/FasterDefs.h"

NS_WTP_BEGIN
class WTSTickData;
class WTSVariant;
NS_WTP_END

USING_NS_WTP;

typedef std::vector<uint32_t> OrderIDs;			// 订单列表
typedef WTSHashMap<std::string>	WTSTickCache;

// "MatchEngine": 尚不清楚具体细节, 暂命名为 匹配引擎吧

// 匹配引擎回调接口
class IMatchSink
{
public:
	/*
	 *	成交回报
	 *	code	合约代码
	 *	isBuy	买or卖
	 *	vol		成交数量, 这里没有正负, 通过isBuy确定买入还是卖出
	 *	price	成交价格
	 */
	virtual void handle_trade(uint32_t localid, const char* stdCode, bool isBuy, double vol, double fireprice, double price, uint64_t ordTime) = 0;

	/*
	 *	订单回报
	 *	localid	本地单号
	 *	code	合约代码
	 *	isBuy	买or卖
	 *	leftover	剩余数量
	 *	price	委托价格
	 *	isCanceled	是否已撤销
	 */
	virtual void handle_order(uint32_t localid, const char* stdCode, bool isBuy, double leftover, double price, bool isCanceled, uint64_t ordTime) = 0;

	/*
	 * 输入订单
	 */
	virtual void handle_entrust(uint32_t localid, const char* stdCode, bool bSuccess, const char* message, uint64_t ordTime) = 0;
};

typedef std::function<void(double)> FuncCancelCallback;

// 匹配引擎
class MatchEngine
{
public:
	MatchEngine() : _tick_cache(NULL),_cancelrate(0), _sink(NULL)
	{

	}
private:
	void	fire_orders(const char* stdCode, OrderIDs& to_erase);
	void	match_orders(WTSTickData* curTick, OrderIDs& to_erase);
	void	update_lob(WTSTickData* curTick);
	// 获取品种对应最新价
	inline WTSTickData*	grab_last_tick(const char* stdCode);

public:
	// 初始化
	void	init(WTSVariant* cfg);
	// 注册回调接口
	void	regisSink(IMatchSink* sink) { _sink = sink; }

	void	clear();

	void	handle_tick(const char* stdCode, WTSTickData* curTick);

	// 买入, 卖出撤单
	OrderIDs	buy(const char* stdCode, double price, double qty, uint64_t curTime);
	OrderIDs	sell(const char* stdCode, double price, double qty, uint64_t curTime);
	virtual OrderIDs cancel(const char* stdCode, bool isBuy, double qty, FuncCancelCallback cb);
	// 修改订单状态: 撤单
	double		cancel(uint32_t localid);

private:
	typedef struct _OrderInfo
	{
		char		_code[32];
		bool		_buy;			// 买入/卖出
		double		_qty;			// 可成交量
		double		_left;			// 市场成交量
		double		_traded;
		double		_limit;			// 限价
		double		_price;			// 最新成交价
		uint32_t	_state;			// 订单状态
		uint64_t	_time;
		double		_queue;			// 排队
		bool		_positive;		// 是否主动成交

		_OrderInfo()
		{
			memset(this, 0, sizeof(_OrderInfo));
		}
	} OrderInfo;

	typedef faster_hashmap<uint32_t, OrderInfo> Orders;
	Orders	_orders;		// 订单字典

	typedef std::map<uint32_t, double>	LOBItems;

	typedef struct _LmtOrdBook
	{
		LOBItems	_items;
		uint32_t	_cur_px;	// 最新价
		uint32_t	_ask_px;	// ask
		uint32_t	_bid_px;	// bid

		void clear()
		{
			_items.clear();
			_cur_px = 0;
			_ask_px = 0;
			_bid_px = 0;
		}

		_LmtOrdBook()
		{
			_cur_px = 0;
			_ask_px = 0;
			_bid_px = 0;
		}
	} LmtOrdBook;
	typedef faster_hashmap<std::string, LmtOrdBook> LmtOrdBooks;
	LmtOrdBooks	_lmt_ord_books;		// 订单簿字典

	IMatchSink*	_sink;				// 匹配引擎回调接口

	double			_cancelrate;
	WTSTickCache*	_tick_cache;	// Tick数据缓存
};
```

## MatchEngine.cpp

```cpp
#include "MatchEngine.h"
#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSVariant.hpp"

#include "../Share/TimeUtils.hpp"
#include "../Share/decimal.h"
#include "../WTSTools/WTSLogger.h"

// 处理数据精度
#define PRICE_DOUBLE_TO_INT_P(x) ((int32_t)((x)*10000.0 + 0.5))
#define PRICE_DOUBLE_TO_INT_N(x) ((int32_t)((x)*10000.0 - 0.5))
#define PRICE_DOUBLE_TO_INT(x) (((x)==DBL_MAX)?0:((x)>0?PRICE_DOUBLE_TO_INT_P(x):PRICE_DOUBLE_TO_INT_N(x)))

extern uint32_t makeLocalOrderID();

// 通过配置初始化匹配引擎
void MatchEngine::init(WTSVariant* cfg)
{
	if (cfg == NULL)
		return;
	// cancelrate?
	_cancelrate = cfg->getDouble("cancelrate");
}

void MatchEngine::clear()
{
	_orders.clear();
}

// 
void MatchEngine::fire_orders(const char* stdCode, OrderIDs& to_erase)
{
	// 遍历订单簿字典
	for (auto& v : _orders)
	{
		uint32_t localid = v.first;						// 编号
		OrderInfo& ordInfo = (OrderInfo&)v.second;		// 订单簿

		if (ordInfo._state == 0)	//需要激活
		{
			// 输入订单, 订单回报
			_sink->handle_entrust(localid, stdCode, true, "", ordInfo._time);
			_sink->handle_order(localid, stdCode, ordInfo._buy, ordInfo._left, ordInfo._limit, false, ordInfo._time);
			ordInfo._state = 1;
		}
	}
}

void MatchEngine::match_orders(WTSTickData* curTick, OrderIDs& to_erase)
{
	// 当前时间
	uint64_t curTime = (uint64_t)curTick->actiondate() * 1000000000 + curTick->actiontime();
	uint64_t curUnixTime = TimeUtils::makeTime(curTick->actiondate(), curTick->actiontime());

	// 遍历订单字典
	for (auto& v : _orders)
	{
		uint32_t localid = v.first;
		OrderInfo& ordInfo = (OrderInfo&)v.second;

		if (ordInfo._state == 9)//要撤单
		{
			_sink->handle_order(localid, ordInfo._code, ordInfo._buy, 0, ordInfo._limit, true, ordInfo._time);
			ordInfo._state = 99;

			to_erase.emplace_back(localid);

			WTSLogger::info("订单%u已撤销, 剩余数量: %d", localid, ordInfo._left*(ordInfo._buy ? 1 : -1));
			ordInfo._left = 0;
			continue;
		}

		if (ordInfo._state != 1 || curTick->volume() == 0)
			continue;

		if (ordInfo._buy)
		{
			double price;
			double volume;

			//主动订单就按照对手价
			if (ordInfo._positive)
			{
				price = curTick->askprice(0);
				volume = curTick->askqty(0);
			}
			else
			{
				price = curTick->price();
				volume = curTick->volume();
			}

			if (decimal::le(price, ordInfo._limit))
			{
				//如果价格相等,需要先看排队位置,如果价格不等说明已经全部被大单吃掉了
				if (!ordInfo._positive && decimal::eq(price, ordInfo._limit))
				{
					double& quepos = ordInfo._queue;

					//如果成交量小于排队位置,则不能成交
					if (volume <= quepos)
					{
						quepos -= volume;
						continue;
					}
					else if (quepos != 0)
					{
						//如果成交量大于排队位置,则可以成交
						volume -= quepos;
						quepos = 0;
					}
				}
				else if (!ordInfo._positive)
				{
					volume = ordInfo._left;
				}

				double qty = min(volume, ordInfo._left);
				// 回调成交回报
				_sink->handle_trade(localid, ordInfo._code, ordInfo._buy, qty, ordInfo._price, price, ordInfo._time);

				ordInfo._traded += qty;
				ordInfo._left -= qty;

				// 回调订单回报
				_sink->handle_order(localid, ordInfo._code, ordInfo._buy, ordInfo._left, price, false, ordInfo._time);

				if (ordInfo._left == 0)
					to_erase.emplace_back(localid);
			}
		}

		if (!ordInfo._buy)
		{
			double price;
			double volume;

			//主动订单就按照对手价
			if (ordInfo._positive)
			{
				price = curTick->bidprice(0);
				volume = curTick->bidqty(0);
			}
			else
			{
				price = curTick->price();
				volume = curTick->volume();
			}

			if (decimal::ge(price, ordInfo._limit))
			{
				//如果价格相等,需要先看排队位置,如果价格不等说明已经全部被大单吃掉了
				if (!ordInfo._positive && decimal::eq(price, ordInfo._limit))
				{
					double& quepos = ordInfo._queue;

					//如果成交量小于排队位置,则不能成交
					if (volume <= quepos)
					{
						quepos -= volume;
						continue;
					}
					else if (quepos != 0)
					{
						//如果成交量大于排队位置,则可以成交
						volume -= quepos;
						quepos = 0;
					}
				}
				else if (!ordInfo._positive)
				{
					volume = ordInfo._left;
				}

				double qty = min(volume, ordInfo._left);
				_sink->handle_trade(localid, ordInfo._code, ordInfo._buy, qty, ordInfo._price, price, ordInfo._time);
				ordInfo._traded += qty;
				ordInfo._left -= qty;

				_sink->handle_order(localid, ordInfo._code, ordInfo._buy, ordInfo._left, price, false, ordInfo._time);

				if (ordInfo._left == 0)
					to_erase.emplace_back(localid);
			}
		}
	}
}

// 更新啥？
void MatchEngine::update_lob(WTSTickData* curTick)
{
	LmtOrdBook& curBook = _lmt_ord_books[curTick->code()];
	curBook._cur_px = PRICE_DOUBLE_TO_INT(curTick->price());
	curBook._ask_px = PRICE_DOUBLE_TO_INT(curTick->askprice(0));
	curBook._bid_px = PRICE_DOUBLE_TO_INT(curTick->bidprice(0));

	for (uint32_t i = 0; i < 10; i++)
	{
		if (PRICE_DOUBLE_TO_INT(curTick->askprice(i)) == 0 && PRICE_DOUBLE_TO_INT(curTick->bidprice(i)) == 0)
			break;

		uint32_t px = PRICE_DOUBLE_TO_INT(curTick->askprice(i));
		if (px != 0)
		{
			double& volume = curBook._items[px];
			volume = curTick->askqty(i);
		}

		px = PRICE_DOUBLE_TO_INT(curTick->bidprice(i));
		if (px != 0)
		{
			double& volume = curBook._items[px];
			volume = curTick->askqty(i);
		}
	}

	//卖一和买一之间的报价必须全部清除掉
	if (!curBook._items.empty())
	{
		auto sit = curBook._items.lower_bound(curBook._bid_px);
		if (sit->first == curBook._bid_px)
			sit++;

		auto eit = curBook._items.lower_bound(curBook._ask_px);

		if (sit->first <= eit->first)
			curBook._items.erase(sit, eit);
	}
}

// 
OrderIDs MatchEngine::buy(const char* stdCode, double price, double qty, uint64_t curTime)
{
	WTSTickData* lastTick = grab_last_tick(stdCode);
	if (lastTick == NULL)
		return OrderIDs();

	// 获取本地订单ID和并填充订单信息
	uint32_t localid = makeLocalOrderID();
	OrderInfo& ordInfo = _orders[localid];
	strcpy(ordInfo._code, stdCode);
	ordInfo._buy = true;
	ordInfo._limit = price;
	ordInfo._qty = qty;
	ordInfo._left = qty;
	ordInfo._price = lastTick->price();

	//订单排队,如果是对手价,则按照对手价的挂单量来排队
	//如果是最新价,则按照买一卖一的加权平均
	if (decimal::ge(price, lastTick->askprice(0)))
		ordInfo._positive = true;
	else if (decimal::eq(price, lastTick->bidprice(0)))
		ordInfo._queue = lastTick->bidqty(0);
	if (decimal::eq(price, lastTick->price()))
		ordInfo._queue = (uint32_t)round((lastTick->askqty(0)*lastTick->askprice(0) + lastTick->bidqty(0)*lastTick->bidprice(0)) / (lastTick->askprice(0) + lastTick->bidprice(0)));

	//排队位置按照平均撤单率,撤销掉部分
	ordInfo._queue -= (uint32_t)round(ordInfo._queue*_cancelrate);
	ordInfo._time = curTime;

	lastTick->release();

	OrderIDs ret;
	ret.emplace_back(localid);
	return ret;
}

OrderIDs MatchEngine::sell(const char* stdCode, double price, double qty, uint64_t curTime)
{
	WTSTickData* lastTick = grab_last_tick(stdCode);
	if (lastTick == NULL)
		return OrderIDs();

	uint32_t localid = makeLocalOrderID();
	OrderInfo& ordInfo = _orders[localid];
	strcpy(ordInfo._code, stdCode);
	ordInfo._buy = false;
	ordInfo._limit = price;
	ordInfo._qty = qty;
	ordInfo._left = qty;
	ordInfo._price = lastTick->price();

	//订单排队,如果是对手价,则按照对手价的挂单量来排队
	//如果是最新价,则按照买一卖一的加权平均
	if (decimal::eq(price, lastTick->askprice(0)))
		ordInfo._queue = lastTick->askqty(0);
	else if (decimal::le(price, lastTick->bidprice(0)))
		ordInfo._positive = true;
	if (decimal::eq(price, lastTick->price()))
		ordInfo._queue = (uint32_t)round((lastTick->askqty(0)*lastTick->askprice(0) + lastTick->bidqty(0)*lastTick->bidprice(0)) / (lastTick->askprice(0) + lastTick->bidprice(0)));

	ordInfo._queue -= (uint32_t)round(ordInfo._queue*_cancelrate);
	ordInfo._time = curTime;

	lastTick->release();

	OrderIDs ret;
	ret.emplace_back(localid);
	return ret;
}

OrderIDs MatchEngine::cancel(const char* stdCode, bool isBuy, double qty, FuncCancelCallback cb)
{
	OrderIDs ret;
	for (auto& v : _orders)
	{
		OrderInfo& ordInfo = (OrderInfo&)v.second;
		if (ordInfo._state != 1)
			continue;

		double left = qty;
		if (ordInfo._buy == isBuy)
		{
			uint32_t localid = v.first;
			ret.emplace_back(localid);
			ordInfo._state = 9;
			cb(ordInfo._left*(ordInfo._buy ? 1 : -1));

			if (qty != 0)
			{
				if ((int32_t)left <= ordInfo._left)
					break;

				left -= ordInfo._left;
			}
		}
	}

	return ret;
}

double MatchEngine::cancel(uint32_t localid)
{
	auto it = _orders.find(localid);
	if (it == _orders.end())
		return 0.0;

	OrderInfo& ordInfo = (OrderInfo&)it->second;
	ordInfo._state = 9;

	return ordInfo._left*(ordInfo._buy ? 1 : -1);
}

void MatchEngine::handle_tick(const char* stdCode, WTSTickData* curTick)
{
	if (NULL == curTick)
		return;

	if (NULL == _tick_cache)
		_tick_cache = WTSTickCache::create();

	_tick_cache->add(stdCode, curTick, true);

	update_lob(curTick);

	OrderIDs to_erase;
	//检查订单状态
	fire_orders(stdCode, to_erase);

	//撮合
	match_orders(curTick, to_erase);

	for (uint32_t localid : to_erase)
	{
		auto it = _orders.find(localid);
		if (it != _orders.end())
			_orders.erase(it);
	}
}

// 获取品种最新tick数据
WTSTickData* MatchEngine::grab_last_tick(const char* stdCode)
{
	if (NULL == _tick_cache)
		return NULL;

	return (WTSTickData*)_tick_cache->grab(stdCode);
}
```