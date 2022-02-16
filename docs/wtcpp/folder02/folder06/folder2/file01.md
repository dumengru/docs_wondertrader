# HFT策略工厂

source: `{{ page.path }}`

实现了HFT策略接口

```cpp
#pragma once
#include "../Includes/HftStrategyDefs.h"

USING_NS_WTP;

class WtHftStraFact : public IHftStrategyFact
{
public:
	WtHftStraFact();
	virtual ~WtHftStraFact();

public:
	virtual const char* getName() override;

	virtual void enumStrategy(FuncEnumHftStrategyCallback cb) override;

	virtual HftStrategy* createStrategy(const char* name, const char* id) override;

	virtual bool deleteStrategy(HftStrategy* stra) override;
};
```

## WtHftStraFact.cpp

```cpp
#include "WtHftStraFact.h"
#include "WtHftStraDemo.h"

#include <string.h>

// HFT策略工厂名称
const char* FACT_NAME = "WtHftStraFact";

extern "C"
{
	// 导出策略工厂dll
	EXPORT_FLAG IHftStrategyFact* createStrategyFact()
	{
		IHftStrategyFact* fact = new WtHftStraFact();
		return fact;
	}

	EXPORT_FLAG void deleteStrategyFact(IHftStrategyFact* fact)
	{
		if (fact != NULL)
			delete fact;
	}
}

WtHftStraFact::WtHftStraFact()
{}

WtHftStraFact::~WtHftStraFact()
{}

const char* WtHftStraFact::getName()
{
	return FACT_NAME;
}

void WtHftStraFact::enumStrategy(FuncEnumHftStrategyCallback cb)
{
	cb(FACT_NAME, "HftDemoStrategy", true);
}

// 创建策略对象
HftStrategy* WtHftStraFact::createStrategy(const char* name, const char* id)
{
	if(strcmp(name, "HftDemoStrategy") == 0)
	{
		return new WtHftStraDemo(id);
	}

	return NULL;
}

// 删除策略
bool WtHftStraFact::deleteStrategy(HftStrategy* stra)
{
	if (stra == NULL)
		return true;

	if (strcmp(stra->getFactName(), FACT_NAME) != 0)
		return false;

	delete stra;
	return true;
}
```