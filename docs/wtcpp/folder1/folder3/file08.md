# 日志接口

source: `{{ page.path }}`

## ILogHandler.h

### ILogHandler

```cpp
class ILogHandler
{
public:
	virtual void handleLogAppend(WTSLogLevel ll, const char* msg)	= 0;
};
```