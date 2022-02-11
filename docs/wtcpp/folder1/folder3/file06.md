# 指标接口

source: `{{ page.path }}`

## IExpFactory.h

### IExpFactory

指标工厂接口定义

```cpp
class IExpFactory
{
public:
	virtual WTSExpressData *calcKlineExpress(const char* expName, WTSKlineData* klineData, WTSExpressParams* params) = 0;
	virtual WTSExpressData *calcTrendExpress(const char* expName, WTSHisTrendData* trendData, WTSExpressParams* params) = 0;
};
```