# file1 01-01 基础宏

source: `{{ page.path }}`

## WTSMarcos.h

```cpp
// 合约/交易所代码最大长度
#define MAX_INSTRUMENT_LENGTH	32
#define MAX_EXCHANGE_LENGTH		10

// 数据类型转换
#define STATIC_CONVERT(x,T)		static_cast<T>(x)

// 最大浮点数
#ifndef DBL_MAX
#define DBL_MAX 1.7976931348623158e+308
#endif

// 最大浮点数
#ifndef FLT_MAX
#define FLT_MAX 3.402823466e+38F        /* max value */
#endif

// 无效数值类型数据
#ifdef _MSC_VER
#define INVALID_DOUBLE		DBL_MAX
#define INVALID_INT32		INT_MAX
#define INVALID_UINT32		UINT_MAX
#define INVALID_INT64		_I64_MAX
#define INVALID_UINT64		_UI64_MAX
#else
#define INVALID_DOUBLE		1.7976931348623158e+308 /* max value */
#define INVALID_INT32		2147483647
#define INVALID_UINT32		0xffffffffUL
#define INVALID_INT64		9223372036854775807LL
#define INVALID_UINT64		0xffffffffffffffffULL
#endif

// 空值类型
#ifndef NULL
#ifdef __cplusplus
#define NULL 0
#else
#define NULL ((void *)0)
#endif
#endif

// 全局命名空间
#define NS_OTP_BEGIN	namespace otp{
#define NS_OTP_END	}//namespace opt
#define	USING_NS_OTP	using namespace otp

// 函数导出
#ifndef EXPORT_FLAG
#ifdef _MSC_VER
#	define EXPORT_FLAG __declspec(dllexport)
#else
#	define EXPORT_FLAG __attribute__((__visibility__("default")))
#endif
#endif

// 函数调用方法
#ifndef PORTER_FLAG
#ifdef _MSC_VER
#	define PORTER_FLAG _cdecl
#else
#	define PORTER_FLAG 
#endif
#endif

// 定义类型
typedef unsigned long		WtUInt32;
typedef unsigned long long	WtUInt64;
typedef const char*			WtString;

// 忽略大小写比较字符串
#ifdef _MSC_VER
#define wt_stricmp _stricmp
#else
#define wt_stricmp strcasecmp
#endif
```