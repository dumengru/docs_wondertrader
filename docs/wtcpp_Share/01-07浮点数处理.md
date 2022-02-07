# 01-07 浮点数处理

source: `{{ page.path }}`

## decimal.h

### decimal

在 `decimal` 命名空间中定义各种浮点数处理方法

```cpp
namespace decimal
{
	const double EPSINON = 1e-6;

    // 四舍五入
	inline double rnd(double v, int exp = 1)
	{
		return round(v*exp) / exp;
	}
    
    // 等于
	inline bool eq(double a, double b = 0.0)
	{
		return(fabs(a - b) < EPSINON);
	}

    // 大于
	inline bool gt(double a, double b = 0.0)
	{
		return a - b > EPSINON;
	}

    // 小于
	inline bool lt(double a, double b = 0.0)
	{
		return b - a > EPSINON;
	}

    // 大于等于
	inline bool ge(double a, double b = 0.0)
	{
		return gt(a, b) || eq(a, b);
	}

    // 小于等于
	inline bool le(double a, double b = 0.0)
	{
		return lt(a, b) || eq(a, b);
	}

	inline double mod(double a, double b)
	{
		return a / b - round(a / b);
	}
};
```