# 主力合约接口

source: `{{ page.path }}`

## IHotMgr.h

### IHotMgr

```cpp
// 定义主力合约切换结构体
typedef struct _HotSection
{
	std::string	_code;
	uint32_t	_s_date;
	uint32_t	_e_date;

	_HotSection(const char* code, uint32_t sdate, uint32_t edate)
		: _s_date(sdate), _e_date(edate), _code(code)
	{}
} HotSection;
typedef std::vector<HotSection>	HotSections;


#define HOTS_MARKET		"HOTS_MARKET"
#define SECONDS_MARKET	"SECONDS_MARKET"

class IHotMgr
{
public:
	// 获取分月代码, pid: 品种代码, dt: 日期(交易日)
	virtual const char* getRawCode(const char* exchg, const char* pid, uint32_t dt)	= 0;

	// 获取上一个主力合约的分月代码, pid: 品种代码, dt: 日期(交易日)
	virtual const char* getPrevRawCode(const char* exchg, const char* pid, uint32_t dt) = 0;

	// 获取主力代码, rawCode: 分月代码, dt: 日期(交易日)
	virtual const char* getHotCode(const char* exchg, const char* rawCode, uint32_t dt) = 0;

	// 是否主力合约, rawCode: 分月代码, dt: 日期(交易日)
	virtual bool		isHot(const char* exchg, const char* rawCode, uint32_t dt) = 0;

	// 将主力合约在某个时段的分月合约全部提出取来
	virtual bool		splitHotSecions(const char* exchg, const char* hotCode, uint32_t sDt, uint32_t eDt, HotSections& sections) = 0;

	// 获取次主力分月代码, pid: 品种代码, dt: 日期(交易日)
	virtual const char* getSecondRawCode(const char* exchg, const char* pid, uint32_t dt) = 0;

	// 上一个次主力合约的分月代码, pid: 品种代码, dt: 日期(交易日)
	virtual const char* getPrevSecondRawCode(const char* exchg, const char* pid, uint32_t dt) = 0;

	// 获取次主力代码, rawCode: 分月代码, dt: 日期(交易日)
	virtual const char* getSecondCode(const char* exchg, const char* rawCode, uint32_t dt) = 0;

	// 是否次主力合约, rawCode: 分月代码, dt: 日期(交易日)
	virtual bool		isSecond(const char* exchg, const char* rawCode, uint32_t dt) = 0;

	// 将次主力合约在某个时段的分月合约全部提出取来
	virtual bool		splitSecondSecions(const char* exchg, const char* hotCode, uint32_t sDt, uint32_t eDt, HotSections& sections) = 0;
};
```