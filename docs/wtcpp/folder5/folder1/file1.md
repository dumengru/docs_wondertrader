# 01-01 CTA策略工厂

source: `{{ page.path }}`

## WtCtaStraFact.h

### WtStraFact

实现了CTA策略接口

```cpp
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

### WtCtaStraFact.cpp

```cpp
const char* FACT_NAME = "WtCtaStraFact";

extern "C"
{
	EXPORT_FLAG ICtaStrategyFact* createStrategyFact()
	{
		ICtaStrategyFact* fact = new WtStraFact();
		return fact;
	}
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

// 创建策略对象(传入策略名称, 和配置文件中的"name"一致)
CtaStrategy* WtStraFact::createStrategy(const char* name, const char* id)
{
	if (strcmp(name, "DualThrust") == 0)
		return new WtStraDualThrust(id);

	return NULL;
}

// 删除策略对象
bool WtStraFact::deleteStrategy(CtaStrategy* stra)
{
	if (stra == NULL)
		return true;

	if (strcmp(stra->getFactName(), FACT_NAME) != 0)
		return false;

	delete stra;
	return true;
}

// 枚举策略(有回调函数)
void WtStraFact::enumStrategy(FuncEnumStrategyCallback cb)
{
	cb(FACT_NAME, "DualThrust", false);
	cb(FACT_NAME, "PairTradingFci", false);
	cb(FACT_NAME, "CtaXPA", true);
}

// 获取策略名称
const char* WtStraFact::getName()
{
	return FACT_NAME;
}
```