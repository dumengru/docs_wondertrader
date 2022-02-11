# 基础数据管理

source: `{{ page.path }}`

## WTSBaseDataMgr.h

加载配置文件数据到内存, 并获取相关数据

```cpp
// 自定义类型

typedef faster_hashmap<std::string, TradingDayTpl>	TradingDayTplMap;
typedef WTSHashMap<std::string>		WTSContractList;
typedef WTSHashMap<std::string>		WTSExchgContract;
typedef WTSHashMap<std::string>		WTSContractMap;
typedef WTSHashMap<std::string>		WTSSessionMap;
typedef WTSHashMap<std::string>		WTSCommodityMap;
typedef faster_hashset<std::string> CodeSet;
typedef faster_hashmap<std::string, CodeSet> SessionCodeMap;
```

### WTSBaseDataMgr

基础数据管理器实现(继承了基础数据管理接口)

```cpp
class WTSBaseDataMgr : public IBaseDataMgr
{
public:
	WTSBaseDataMgr();
	~WTSBaseDataMgr();

public:
	virtual WTSCommodityInfo*	getCommodity(const char* stdPID) override;
	virtual WTSCommodityInfo*	getCommodity(const char* exchg, const char* pid) override;
	virtual WTSCommodityInfo*	getCommodity(WTSContractInfo* ct) override;
	virtual WTSContractInfo*	getContract(const char* code, const char* exchg = "") override;
	virtual WTSArray*			getContracts(const char* exchg = "") override;
	virtual WTSSessionInfo*		getSession(const char* sid) override;
	virtual WTSSessionInfo*		getSessionByCode(const char* code, const char* exchg = "") override;
	virtual WTSArray*			getAllSessions() override;
	virtual bool				isHoliday(const char* stdPID, uint32_t uDate, bool isTpl = false) override;

	virtual uint32_t			calcTradingDate(const char* stdPID, uint32_t uDate, uint32_t uTime, bool isSession = false) override;
	virtual uint64_t			getBoundaryTime(const char* stdPID, uint32_t tDate, bool isSession = false, bool isStart = true) override;
	void		release();

	// 从对应配置文件加载数据到内存中
	bool		loadSessions(const char* filename);
	bool		loadCommodities(const char* filename);
	bool		loadContracts(const char* filename);
	bool		loadHolidays(const char* filename);

public:
	uint32_t	getTradingDate(const char* stdPID, uint32_t uOffDate = 0, uint32_t uOffMinute = 0, bool isTpl = false);
	uint32_t	getNextTDate(const char* stdPID, uint32_t uDate, int days = 1, bool isTpl = false);
	uint32_t	getPrevTDate(const char* stdPID, uint32_t uDate, int days = 1, bool isTpl = false);
	bool		isTradingDate(const char* stdPID, uint32_t uDate, bool isTpl = false);
	void		setTradingDate(const char* stdPID, uint32_t uDate, bool isTpl = false);
    CodeSet*	getSessionComms(const char* sid);
private:
	const char* getTplIDByPID(const char* stdPID);

private:
	TradingDayTplMap	m_mapTradingDay;        // 交易日模板
	SessionCodeMap		m_mapSessionCode;       // 
	WTSExchgContract*	m_mapExchgContract;     // 交易所合约
	WTSSessionMap*		m_mapSessions;          // 交易时间信息字典
	WTSCommodityMap*	m_mapCommodities;       // 品种信息字典
	WTSContractMap*		m_mapContracts;         // 合约信息字典
};
```

## WTSBaseDataMgr.cpp

实现逻辑

```cpp
// 节日模板
const char* DEFAULT_HOLIDAY_TPL = "CHINA";
namespace rj = rapidjson;

// 类初始化时创建 字典对象
WTSBaseDataMgr::WTSBaseDataMgr()
	: m_mapExchgContract(NULL)
	, m_mapSessions(NULL)
	, m_mapCommodities(NULL)
	, m_mapContracts(NULL)
{
	m_mapExchgContract = WTSExchgContract::create();
	m_mapSessions = WTSSessionMap::create();
	m_mapCommodities = WTSCommodityMap::create();
	m_mapContracts = WTSContractMap::create();
}

// 类析构时释放对象指针
WTSBaseDataMgr::~WTSBaseDataMgr()
{
	if (m_mapExchgContract)
	{
		m_mapExchgContract->release();
		m_mapExchgContract = NULL;
	}
	if (m_mapSessions)
	{
		m_mapSessions->release();
		m_mapSessions = NULL;
	}
	if (m_mapCommodities)
	{
		m_mapCommodities->release();
		m_mapCommodities = NULL;
	}
	if(m_mapContracts)
	{
		m_mapContracts->release();
		m_mapContracts = NULL;
	}
}

// 获取品种信息
WTSCommodityInfo* WTSBaseDataMgr::getCommodity(const char* exchgpid)
{
	return (WTSCommodityInfo*)m_mapCommodities->get(exchgpid);
}

WTSCommodityInfo* WTSBaseDataMgr::getCommodity(const char* exchg, const char* pid)
{
	if (m_mapCommodities == NULL)
		return NULL;

	char key[64] = { 0 };
	fmt::format_to(key, "{}.{}", exchg, pid);

	return (WTSCommodityInfo*)m_mapCommodities->get(key);
}

WTSCommodityInfo* WTSBaseDataMgr::getCommodity(WTSContractInfo* ct)
{
	if (ct == NULL)
		return NULL;

	return getCommodity(ct->getExchg(), ct->getProduct());
}

// 获取合约信息
WTSContractInfo* WTSBaseDataMgr::getContract(const char* code, const char* exchg)
{
	//如果直接找到对应的市场代码,则直接
	auto it = m_mapExchgContract->find(exchg);
	if(it != m_mapExchgContract->end())
	{
		WTSContractList* contractList = (WTSContractList*)it->second;
		auto it = contractList->find(code);
		if(it != contractList->end())
		{
			return (WTSContractInfo*)it->second;
		}
	}
	else if (strlen(exchg) == 0)
	{
		auto it = m_mapContracts->find(code);
		if (it == m_mapContracts->end())
			return NULL;
		WTSArray* ayInst = (WTSArray*)it->second;
		if (ayInst == NULL || ayInst->size() == 0)
			return NULL;
		return (WTSContractInfo*)ayInst->at(0);
	}
	return NULL;
}

// 根据交易所名称获取全部合约
WTSArray* WTSBaseDataMgr::getContracts(const char* exchg /* = "" */)
{
	WTSArray* ay = WTSArray::create();
	if(strlen(exchg) > 0)
	{
		auto it = m_mapExchgContract->find(exchg);
		if (it != m_mapExchgContract->end())
		{
			WTSContractList* contractList = (WTSContractList*)it->second;
			auto it2 = contractList->begin();
			for (; it2 != contractList->end(); it2++)
			{
				ay->append(it2->second, true);
			}
		}
	}
	else
	{
		auto it = m_mapExchgContract->begin();
		for (; it != m_mapExchgContract->end(); it++)
		{
			WTSContractList* contractList = (WTSContractList*)it->second;
			auto it2 = contractList->begin();
			for (; it2 != contractList->end(); it2++)
			{
				ay->append(it2->second, true);
			}
		}
	}

	return ay;
}

// 获取全部交易时段
WTSArray* WTSBaseDataMgr::getAllSessions()
{
	WTSArray* ay = WTSArray::create();
	for (auto it = m_mapSessions->begin(); it != m_mapSessions->end(); it++)
	{
		ay->append(it->second, true);
	}
	return ay;
}

// 获取交易时段信息
WTSSessionInfo* WTSBaseDataMgr::getSession(const char* sid)
{
	return (WTSSessionInfo*)m_mapSessions->get(sid);
}

WTSSessionInfo* WTSBaseDataMgr::getSessionByCode(const char* code, const char* exchg /* = "" */)
{
	WTSContractInfo* ct = getContract(code, exchg);
	if (ct == NULL)
		return NULL;

	WTSCommodityInfo* commInfo = getCommodity(ct);
	if (commInfo == NULL)
		return NULL;

	return getSession(commInfo->getSession());
}

// 判断是否是节日
bool WTSBaseDataMgr::isHoliday(const char* pid, uint32_t uDate, bool isTpl)
{
	// 判断是否是周末, 这部分代码可以用 TimeUtils::isWeekends 替换
	uint32_t wd = TimeUtils::getWeekDay(uDate);  
	if (wd == 0 || wd == 6)
		return true;

	// 对比节日模板
	std::string tplid = pid;
	if (!isTpl)
		tplid = getTplIDByPID(pid);

	auto it = m_mapTradingDay.find(tplid);
	if(it != m_mapTradingDay.end())
	{
		const TradingDayTpl& tpl = it->second;
		return (tpl._holidays.find(uDate) != tpl._holidays.end());
	}

	return false;
}

// 释放对象指针
void WTSBaseDataMgr::release()
{
	if (m_mapExchgContract)
	{
		m_mapExchgContract->release();
		m_mapExchgContract = NULL;
	}

	if (m_mapSessions)
	{
		m_mapSessions->release();
		m_mapSessions = NULL;
	}

	if (m_mapCommodities)
	{
		m_mapCommodities->release();
		m_mapCommodities = NULL;
	}
}

// 读取 sessions.json 文件, 加载到 m_mapSessions 中
bool WTSBaseDataMgr::loadSessions(const char* filename)
{
	if (!StdFile::exists(filename))
	{
		WTSLogger::error("Trading sessions configuration file %s not exists", filename);
		return false;
	}

	std::string content;
	StdFile::read_file_content(filename, content);
	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::error("Parsing trading session configuration file failed");
		return false;
	}

	for(auto& m : root.GetObject())
	{
		const std::string& id = m.name.GetString();
		const rj::Value& jVal = m.value;

		const char* name = jVal["name"].GetString();
		int32_t offset = jVal["offset"].GetInt();

		WTSSessionInfo* sInfo = WTSSessionInfo::create(id.c_str(), name, offset);

		if (jVal.HasMember("auction"))
		{
			const rj::Value& jAuc = jVal["auction"];
			sInfo->setAuctionTime(jAuc["from"].GetUint(), jAuc["to"].GetUint());
		}

		const rj::Value& jSecs = jVal["sections"];
		if (jSecs.IsNull() || !jSecs.IsArray())
			continue;

		for (const rj::Value& jSec : jSecs.GetArray())
		{
			sInfo->addTradingSection(jSec["from"].GetUint(), jSec["to"].GetUint());
		}

		m_mapSessions->add(id, sInfo);
	}

	return true;
}

// 加载 commodities.json 文件内容到 m_mapSessionCode
bool WTSBaseDataMgr::loadCommodities(const char* filename)
{
	if (!StdFile::exists(filename))
	{
		WTSLogger::error("Commodities configuration file %s not exists", filename);
		return false;
	}

	std::string content;
	StdFile::read_file_content(filename, content);
	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::error("Loading commodities configuration file failed");
		return false;
	}

	for(auto& m : root.GetObject())
	{
		const std::string& exchg = m.name.GetString();
		const rj::Value& jExchg = m.value;

		for (auto& mExchg : jExchg.GetObject())
		{
			const std::string& pid = mExchg.name.GetString();
			const rj::Value& jPInfo = mExchg.value;

			const char* name = jPInfo["name"].GetString();
			const char* sid = jPInfo["session"].GetString();
			const char* hid = jPInfo["holiday"].GetString();

			if (strlen(sid) == 0)
			{
				WTSLogger::error("Not corresponding session ID of instrument %s", pid.c_str());
				continue;
			}

			WTSCommodityInfo* pCommInfo = WTSCommodityInfo::create(pid.c_str(), name, exchg.c_str(), sid, hid);
			pCommInfo->setPrecision(jPInfo["precision"].GetUint());
			pCommInfo->setPriceTick(jPInfo["pricetick"].GetDouble());
			pCommInfo->setVolScale(jPInfo["volscale"].GetUint());

			if (jPInfo.HasMember("category"))
				pCommInfo->setCategory((ContractCategory)jPInfo["category"].GetUint());
			else
				pCommInfo->setCategory(CC_Future);

			pCommInfo->setCoverMode((CoverMode)jPInfo["covermode"].GetUint());
			pCommInfo->setPriceMode((PriceMode)jPInfo["pricemode"].GetUint());

			if (jPInfo.HasMember("trademode"))
				pCommInfo->setTradingMode((TradingMode)jPInfo["trademode"].GetUint());
			else
				pCommInfo->setTradingMode(TM_Both);

			uint32_t buyQtyUnit = 1;
			uint32_t sellQtyUnit = 1;
			if (jPInfo.HasMember("buyqtyunit"))
				buyQtyUnit = jPInfo["buyqtyunit"].GetUint();
			if (jPInfo.HasMember("sellqtyunit"))
				sellQtyUnit = jPInfo["sellqtyunit"].GetUint();
			pCommInfo->setEntrustQtyUnit(buyQtyUnit, sellQtyUnit);

			std::string key = StrUtil::printf("%s.%s", exchg.c_str(), pid.c_str());
			if (m_mapCommodities == NULL)
				m_mapCommodities = WTSCommodityMap::create();

			m_mapCommodities->add(key, pCommInfo, false);

			m_mapSessionCode[sid].insert(key);
		}
	}
	WTSLogger::info("Commodities configuration file %s loaded", filename);
	return true;
}

// 加载 contracts.json 内容到 m_mapContracts
bool WTSBaseDataMgr::loadContracts(const char* filename)
{
	if (!StdFile::exists(filename))
	{
		WTSLogger::error("Contracts configuration file %s not exists", filename);
		return false;
	}

	std::string content;
	StdFile::read_file_content(filename, content);
	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::error("Parsing contracts configuration file failed");
		return false;
	}

	for(auto& m : root.GetObject())
	{
		const std::string& exchg = m.name.GetString();
		const rj::Value& jExchg = m.value;

		for(auto& cMember : jExchg.GetObject())
		{
			const std::string& code = cMember.name.GetString();
			const rj::Value& jcInfo = cMember.value;

			WTSCommodityInfo* commInfo = getCommodity(jcInfo["exchg"].GetString(), jcInfo["product"].GetString());
			if (commInfo == NULL)
				continue;

			WTSContractInfo* cInfo = WTSContractInfo::create(code.c_str(),
				jcInfo["name"].GetString(),
				jcInfo["exchg"].GetString(),
				jcInfo["product"].GetString());

			uint32_t maxMktQty = 0;
			uint32_t maxLmtQty = 0;
			if (jcInfo.HasMember("maxmarketqty"))
				maxMktQty = jcInfo["maxmarketqty"].GetUint();
			if (jcInfo.HasMember("maxlimitqty"))
				maxLmtQty = jcInfo["maxlimitqty"].GetUint();
			cInfo->setVolumeLimits(maxMktQty, maxLmtQty);

			WTSContractList* contractList = (WTSContractList*)m_mapExchgContract->get(cInfo->getExchg());
			if (contractList == NULL)
			{
				contractList = WTSContractList::create();
				m_mapExchgContract->add(cInfo->getExchg(), contractList, false);
			}
			contractList->add(cInfo->getCode(), cInfo, false);

			commInfo->addCode(code.c_str());

			WTSArray* ayInst = (WTSArray*)m_mapContracts->get(cInfo->getCode());
			if(ayInst == NULL)
			{
				ayInst = WTSArray::create();
				m_mapContracts->add(cInfo->getCode(), ayInst, false);
			}
			ayInst->append(cInfo, true);
		}
	}
	WTSLogger::info("Contracts configuration file %s loaded", filename);
	return true;
}

// 加载 holidays.json 文件内容到 m_mapTradingDay
bool WTSBaseDataMgr::loadHolidays(const char* filename)
{
	if (!StdFile::exists(filename))
	{
		WTSLogger::error("Holidays configuration file %s not exists", filename);
		return false;
	}

	std::string content;
	StdFile::read_file_content(filename, content);

	rj::Document root;
	if (root.Parse(content.c_str()).HasParseError())
	{
		WTSLogger::error("Parsing holidays configuration file failed");
		return false;
	}

	for (auto& m : root.GetObject())
	{
		const std::string& hid = m.name.GetString();
		const rj::Value& jHolidays = m.value;
		if(!jHolidays.IsArray())
			continue;

		TradingDayTpl& trdDayTpl = m_mapTradingDay[hid];
		for(const rj::Value& hItem : jHolidays.GetArray())
		{
			trdDayTpl._holidays.insert(hItem.GetUint());
		}
	}
	return true;
}

// 获取下一个开盘/收盘时间
uint64_t WTSBaseDataMgr::getBoundaryTime(const char* stdPID, uint32_t tDate, bool isSession, bool isStart)
{
	if(tDate == 0)
		tDate = TimeUtils::getCurDate();
	
	std::string tplid = stdPID;
	bool isTpl = false;
	WTSSessionInfo* sInfo = NULL;
	if (isSession)
	{
		sInfo = getSession(stdPID);
		tplid = DEFAULT_HOLIDAY_TPL;
		isTpl = true;
	}
	else
	{
		WTSCommodityInfo* cInfo = getCommodity(stdPID);
		if (cInfo == NULL)
			return 0;

		sInfo = getSession(cInfo->getSession());
	}

	if (sInfo == NULL)
		return 0;

	uint32_t weekday = TimeUtils::getWeekDay(tDate);
	if (weekday == 6 || weekday == 0)
	{
		if (isStart)
			tDate = getNextTDate(tplid.c_str(), tDate, 1, isTpl);
		else
			tDate = getPrevTDate(tplid.c_str(), tDate, 1, isTpl);
	}

	//不偏移的最简单,只需要直接返回开盘和收盘时间即可
	if (sInfo->getOffsetMins() == 0)
	{
		if (isStart)
			return (uint64_t)tDate * 10000 + sInfo->getOpenTime();
		else
			return (uint64_t)tDate * 10000 + sInfo->getCloseTime();
	}

	if(sInfo->getOffsetMins() < 0)
	{
		//往前偏移,就是交易日推后,一般用于外盘
		//这个比较简单,只需要按自然日取即可
		if (isStart)
			return (uint64_t)tDate * 10000 + sInfo->getOpenTime();
		else
			return (uint64_t)TimeUtils::getNextDate(tDate) * 10000 + sInfo->getCloseTime();
	}
	else
	{
		//往后偏移,一般国内期货夜盘都是这个,即夜盘算是第二个交易日
		//这个比较复杂,主要节假日后第一天的边界很麻烦（如一般情况的周一）
		//这种情况唯一方便的就是,收盘时间不需要处理
		if(!isStart)
			return (uint64_t)tDate * 10000 + sInfo->getCloseTime();

		//想到一个简单的办法,就是不管怎么样,开始时间一定是上一个交易日的晚上
		//所以我只需要拿到上一个交易日即可
		tDate = getPrevTDate(tplid.c_str(), tDate, 1, isTpl);
		return (uint64_t)tDate * 10000 + sInfo->getOpenTime();
	}
}

// 计算时间偏移后的交易日(过滤节假日)
uint32_t WTSBaseDataMgr::calcTradingDate(const char* stdPID, uint32_t uDate, uint32_t uTime, bool isSession /* = false */)
{
	if (uDate == 0)
	{
		TimeUtils::getDateTime(uDate, uTime);
		uTime /= 100000;
	}

	std::string tplid = stdPID;
	bool isTpl = false;
	WTSSessionInfo* sInfo = NULL;
	if(isSession)
	{
		sInfo = getSession(stdPID);		// 获取交易时间段
		tplid = DEFAULT_HOLIDAY_TPL;	// "CHINA"
		isTpl = true;
	}
	else
	{
		WTSCommodityInfo* cInfo = getCommodity(stdPID);		// 获取品种信息
		if (cInfo == NULL)
			return uDate;
		
		sInfo = getSession(cInfo->getSession());			// 通过品种信息获取交易时间段信息
	}

	if (sInfo == NULL)
		return uDate;
	

	uint32_t weekday = TimeUtils::getWeekDay(uDate);	// 是否是周末

	uint32_t offMin = sInfo->offsetTime(uTime);			// 偏移后的时间
	if (sInfo->getOffsetMins() > 0)
	{
		// 如果向后偏移,且当前时间大于偏移时间,说明向后跨日了
		// 这时交易日=下一个交易日
		if (uTime > offMin)
		{
			// 如,20151016 23:00,偏移300分钟,为5:00
			return getNextTDate(tplid.c_str(), uDate, 1, isTpl);
		}
		else if (weekday == 6 || weekday == 0)
		{
			// 如,20151017 1:00,周六,交易日为20151019
			return getNextTDate(tplid.c_str(), uDate, 1, isTpl);
		}
	}
	else if (sInfo->getOffsetMins() < 0)
	{
		// 如果向前偏移,且当前时间小于偏移时间,说明还是前一个交易日
		// 这时交易日=前一个交易日
		if (uTime < offMin)
		{
			//如20151017 1:00,偏移-300分钟,为20:00
			return getPrevTDate(tplid.c_str(), uDate, 1, isTpl);
		}
		else if (weekday == 6 || weekday == 0)
		{
			//因为向前偏移,如果在周末,则直接到下一个交易日
			return getNextTDate(tplid.c_str(), uDate, 1, isTpl);
		}
	}
	else if (weekday == 6 || weekday == 0)
	{
		// 如果没有偏移,且在周末,则直接读取下一个交易日
		return getNextTDate(tplid.c_str(), uDate, 1, isTpl);;
	}

	// 其他情况,交易日=自然日
	return uDate;
}

// 获取交易日
uint32_t WTSBaseDataMgr::getTradingDate(const char* pid, uint32_t uOffDate /* = 0 */, uint32_t uOffMinute /* = 0 */, bool isTpl /* = false */)
{
	const char* tplID = isTpl ? pid : getTplIDByPID(pid);

	uint32_t curDate = TimeUtils::getCurDate();
	auto it = m_mapTradingDay.find(tplID);
	if (it == m_mapTradingDay.end())
	{
		return curDate;
	}

	TradingDayTpl* tpl = (TradingDayTpl*)&it->second;
	if (tpl->_cur_tdate != 0 && uOffDate == 0)
		return tpl->_cur_tdate;

	if (uOffDate == 0)
		uOffDate = curDate;

	uint32_t weekday = TimeUtils::getWeekDay(uOffDate);

	if (weekday == 6 || weekday == 0)
	{
		//如果没有偏移,且在周末,则直接读取下一个交易日
		tpl->_cur_tdate = getNextTDate(tplID, uOffDate, 1, true);
		uOffDate = tpl->_cur_tdate;
	}

	//其他情况,交易日=自然日
	return uOffDate;
}

// 获取下一个交易日(调过节假日)
uint32_t WTSBaseDataMgr::getNextTDate(const char* pid, uint32_t uDate, int days /* = 1 */, bool isTpl /* = false */)
{
	uint32_t curDate = uDate;
	int left = days;
	while (true)
	{
		tm t;
		memset(&t, 0, sizeof(tm));
		t.tm_year = curDate / 10000 - 1900;
		t.tm_mon = (curDate % 10000) / 100 - 1;
		t.tm_mday = curDate % 100;
		//t.tm_isdst 	
		time_t ts = mktime(&t);
		ts += 86400;

		tm* newT = localtime(&ts);
		curDate = (newT->tm_year + 1900) * 10000 + (newT->tm_mon + 1) * 100 + newT->tm_mday;
		if (newT->tm_wday != 0 && newT->tm_wday != 6 && !isHoliday(pid, curDate, isTpl))
		{
			//如果不是周末,也不是节假日,则剩余的天数-1
			left--;
			if (left == 0)
				return curDate;
		}
	}
}

uint32_t WTSBaseDataMgr::getPrevTDate(const char* pid, uint32_t uDate, int days /* = 1 */, bool isTpl /* = false */)
{
	uint32_t curDate = uDate;
	int left = days;
	while (true)
	{
		tm t;
		memset(&t, 0, sizeof(tm));
		t.tm_year = curDate / 10000 - 1900;
		t.tm_mon = (curDate % 10000) / 100 - 1;
		t.tm_mday = curDate % 100;
		//t.tm_isdst 	
		time_t ts = mktime(&t);
		ts -= 86400;

		tm* newT = localtime(&ts);
		curDate = (newT->tm_year + 1900) * 10000 + (newT->tm_mon + 1) * 100 + newT->tm_mday;
		if (newT->tm_wday != 0 && newT->tm_wday != 6 && !isHoliday(pid, curDate, isTpl))
		{
			//如果不是周末,也不是节假日,则剩余的天数-1
			left--;
			if (left == 0)
				return curDate;
		}
	}
}

// 判断是否是交易日
bool WTSBaseDataMgr::isTradingDate(const char* pid, uint32_t uDate, bool isTpl /* = false */)
{
	uint32_t wd = TimeUtils::getWeekDay(uDate);
	if (wd != 0 && wd != 6 && !isHoliday(pid, uDate, isTpl))
	{
		return true;
	}

	return false;
}

// 设置交易日
void WTSBaseDataMgr::setTradingDate(const char* pid, uint32_t uDate, bool isTpl /* = false */)
{
	std::string tplID = pid;
	if (!isTpl)
		tplID = getTplIDByPID(pid);

	auto it = m_mapTradingDay.find(tplID);
	if (it == m_mapTradingDay.end())
		return;

	TradingDayTpl* tpl = (TradingDayTpl*)&it->second;
	tpl->_cur_tdate = uDate;
}

// 根据 pid 获取交易
CodeSet* WTSBaseDataMgr::getSessionComms(const char* sid)
{
	auto it = m_mapSessionCode.find(sid);
	if (it == m_mapSessionCode.end())
		return NULL;

	return (CodeSet*)&it->second;
}

// 根据 pid 获取品种信息中的节日模板
const char* WTSBaseDataMgr::getTplIDByPID(const char* pid)
{
	const StringVector& ay = StrUtil::split(pid, ".");
	WTSCommodityInfo* commInfo = getCommodity(ay[0].c_str(), ay[1].c_str());
	if (commInfo == NULL)
		return "";

	return commInfo->getTradingTpl();
}
```