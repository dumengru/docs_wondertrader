# WtCtaStraFact.h

source: `{{ page.path }}`

实现了CTA策略接口

```cpp
#pragma once
#include "../Includes/CtaStrategyDefs.h"

USING_NS_WTP;

class WtStraFact : public ICtaStrategyFact
{
public:
	WtStraFact();
	virtual ~WtStraFact();

public:
	virtual const char* getName() override;

	virtual CtaStrategy* createStrategy(const char* name, const char* id) override;

	virtual void enumStrategy(FuncEnumStrategyCallback cb) override;

	virtual bool deleteStrategy(CtaStrategy* stra) override;	
};
```

## WtCtaStraFact.cpp

```cpp
#include "WtCtaStraFact.h"
#include "WtStraDualThrust.h"

#include <string.h>
#include <boost/config.hpp>

// CTA策略工厂名称
const char* FACT_NAME = "WtCtaStraFact";

extern "C"
{
	// 导出策略工厂dll
	EXPORT_FLAG ICtaStrategyFact* createStrategyFact()
	{
		ICtaStrategyFact* fact = new WtStraFact();
		return fact;
	}
	// 删除策略
	EXPORT_FLAG void deleteStrategyFact(ICtaStrategyFact* fact)
	{
		if (fact != NULL)
			delete fact;
	}
};

WtStraFact::WtStraFact()
{}

WtStraFact::~WtStraFact()
{}

// 创建策略, 策略名称被定死了
CtaStrategy* WtStraFact::createStrategy(const char* name, const char* id)
{
	if (strcmp(name, "DualThrust") == 0)
		return new WtStraDualThrust(id);

	return NULL;
}

// 删除策略
bool WtStraFact::deleteStrategy(CtaStrategy* stra)
{
	if (stra == NULL)
		return true;

	if (strcmp(stra->getFactName(), FACT_NAME) != 0)
		return false;

	delete stra;
	return true;
}

void WtStraFact::enumStrategy(FuncEnumStrategyCallback cb)
{
	cb(FACT_NAME, "DualThrust", false);
	cb(FACT_NAME, "PairTradingFci", false);
	cb(FACT_NAME, "CtaXPA", true);
}

// 获取策略工厂名称
const char* WtStraFact::getName()
{
	return FACT_NAME;
}
```