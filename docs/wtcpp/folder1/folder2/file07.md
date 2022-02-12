# WTSHotltem.hpp

source: `{{ page.path }}`

```cpp
/*!
 * \file WTSHotItem.hpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief Wt主力切换规则对象定义文件
 */
#pragma once
#include "WTSObject.hpp"

NS_WTP_BEGIN
class WTSHotItem : public WTSObject
{
protected:
	WTSHotItem(){}
	virtual ~WTSHotItem(){}

public:
	static WTSHotItem* create(const char* exchg, const char* product, const char* from, const char* to, uint32_t dt, double oldclose = 0, double newclose = 0)
	{
		WTSHotItem* pRet = new WTSHotItem();
		pRet->_exchg = exchg;
		pRet->_product = product;
		pRet->_hot = pRet->_product + "0001";
		pRet->_from = from;
		pRet->_to = to;
		pRet->_dt = dt;
		pRet->_oldclose = oldclose;
		pRet->_newclose = newclose;
		return pRet;
	}

	const char*		exchg() const{return _exchg.c_str();}
	const char*		product() const{return _product.c_str();}
	const char*		hot() const{return _hot.c_str();}
	const char*		from() const{return _from.c_str();}
	const char*		to() const{return _to.c_str();}
	uint32_t		switchdate() const{return _dt;}

private:
	std::string		_exchg;		// 交易所
	std::string		_product;	// 品种
	std::string		_hot;		// 品种 + "0001"
	std::string		_from;		// 旧合约
	std::string		_to;		// 新合约
	uint32_t		_dt;		// 切换日期
	double		_oldclose;		// 旧收盘价
	double		_newclose;		// 新收盘价
};
NS_WTP_END
```