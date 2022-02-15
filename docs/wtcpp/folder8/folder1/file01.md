# ParserCTP.h

source: `{{ page.path }}`

```cpp
/*!
 * \file ParserCTP.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#pragma once
#include "../Includes/IParserApi.h"
#include "../Share/DLLHelper.hpp"
#include "../API/CTP6.3.15/ThostFtdcMdApi.h"
#include <map>

NS_WTP_BEGIN
class WTSTickData;
NS_WTP_END

USING_NS_WTP;

class ParserCTP :	public IParserApi, public CThostFtdcMdSpi
{
public:
	ParserCTP();
	virtual ~ParserCTP();

public:
	// 登陆状态
	enum LoginStatus
	{
		LS_NOTLOGIN,	// 未登录
		LS_LOGINING,	// 正登录
		LS_LOGINED		// 已登录
	};

// IParserApi 接口
public:
	// 初始化
	virtual bool init(WTSVariant* config) override;
	// 释放
	virtual void release() override;
	// 连接
	virtual bool connect() override;
	// 断开连接
	virtual bool disconnect() override;
	// 判断是否连接
	virtual bool isConnected() override;
	// 订阅行情
	virtual void subscribe(const CodeSet &vecSymbols) override;
	// 取消订阅
	virtual void unsubscribe(const CodeSet &vecSymbols) override;
	// 注册行情解析接口和数据管理接口
	virtual void registerSpi(IParserSpi* listener) override;

// CThostFtdcMdSpi 接口(行情响应函数, 需要自己实现)
public:
	///错误应答
	virtual void OnRspError( CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast );
	///当客户端与交易后台建立起通信连接时（还未登录前），该方法被调用。
	virtual void OnFrontConnected();
	///登录请求响应
	virtual void OnRspUserLogin( CThostFtdcRspUserLoginField *pRspUserLogin, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast );
	// 登出请求响应
	virtual void OnRspUserLogout(CThostFtdcUserLogoutField *pUserLogout, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast);
	///当客户端与交易后台通信连接断开时，该方法被调用
	virtual void OnFrontDisconnected( int nReason );
	///取消订阅行情应答
	virtual void OnRspUnSubMarketData( CThostFtdcSpecificInstrumentField *pSpecificInstrument, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast );
	///深度行情通知
	virtual void OnRtnDepthMarketData( CThostFtdcDepthMarketDataField *pDepthMarketData );
	///订阅行情应答
	virtual void OnRspSubMarketData( CThostFtdcSpecificInstrumentField *pSpecificInstrument, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast );
	///当客户端与交易后台通信连接断开时，该方法被调用
	virtual void OnHeartBeatWarning( int nTimeLapse );

// MdSpi 接口的回调函数
private:
	/*
	 *	发送登录请求
	 */
	void ReqUserLogin();
	/*
	 *	订阅品种行情
	 */
	void SubscribeMarketData();
	/*
	 *	检查错误信息
	 */
	bool IsErrorRspInfo(CThostFtdcRspInfoField *pRspInfo);

private:
	uint32_t			m_uTradingDate;		// 当前交易日
	LoginStatus			m_loginState;		// 登录状态
	CThostFtdcMdApi*	m_pUserAPI;			// 行情API指针

	// 行情登录相关字段
	std::string			m_strFrontAddr;		// 地址
	std::string			m_strBroker;		// 经纪商
	std::string			m_strUserID;		// 用户ID
	std::string			m_strPassword;		// 密码
	std::string			m_strFlowDir;		// 目录

	CodeSet				m_filterSubs;		// 行情订阅列表

	int					m_iRequestID;		// 请求ID

	IParserSpi*			m_sink;				// 行情解析接口, 该对象中的函数将被回调
	IBaseDataMgr*		m_pBaseDataMgr;		// 基础数据管理接口

	DllHandle		m_hInstCTP;				// dll文件句柄
	typedef CThostFtdcMdApi* (*CTPCreator)(const char *, const bool, const bool);
	CTPCreator		m_funcCreator;			// 创建行情API指针
};
```

## ParserCTP.cpp

```cpp
/*!
 * \file ParserCTP.cpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#include "ParserCTP.h"

#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/WTSVariant.hpp"
#include "../Includes/IBaseDataMgr.h"

#include "../Share/ModuleHelper.hpp"
#include "../Share/TimeUtils.hpp"
#include "../Share/StdUtils.hpp"

#include <boost/filesystem.hpp>

 //By Wesley @ 2022.01.05
#include "../Share/fmtlib.h"
template<typename... Args>
inline void write_log(IParserSpi* sink, WTSLogLevel ll, const char* format, const Args&... args)
{
	if (sink == NULL)
		return;

	static thread_local char buffer[512] = { 0 };
	memset(buffer, 0, 512);
	fmt::format_to(buffer, format, args...);

	sink->handleParserLog(ll, buffer);
}

extern "C"
{
	EXPORT_FLAG IParserApi* createParser()
	{
		ParserCTP* parser = new ParserCTP();
		return parser;
	}

	EXPORT_FLAG void deleteParser(IParserApi* &parser)
	{
		if (NULL != parser)
		{
			delete parser;
			parser = NULL;
		}
	}
};


inline uint32_t strToTime(const char* strTime)
{
	static char str[10] = { 0 };
	const char *pos = strTime;
	int idx = 0;
	auto len = strlen(strTime);
	for(auto i = 0; i < len; i++)
	{
		if(strTime[i] != ':')
		{
			str[idx] = strTime[i];
			idx++;
		}
	}
	str[idx] = '\0';

	return strtoul(str, NULL, 10);
}

// 检查数据是否有效
inline double checkValid(double val)
{
	if (val == DBL_MAX || val == FLT_MAX)
		return 0;

	return val;
}

ParserCTP::ParserCTP()
	:m_pUserAPI(NULL)
	,m_iRequestID(0)
	,m_uTradingDate(0)
{
}


ParserCTP::~ParserCTP()
{
	m_pUserAPI = NULL;
}

// 通过配置文件初始化CTP接口
bool ParserCTP::init(WTSVariant* config)
{
	m_strFrontAddr = config->getCString("front");
	m_strBroker = config->getCString("broker");
	m_strUserID = config->getCString("user");
	m_strPassword = config->getCString("pass");
	m_strFlowDir = config->getCString("flowdir");

	if (m_strFlowDir.empty())
		m_strFlowDir = "CTPMDFlow";

	m_strFlowDir = StrUtil::standardisePath(m_strFlowDir);

	std::string module = config->getCString("ctpmodule");
	if (module.empty())
		module = "thostmduserapi_se";

	// 加载CTP行情dll文件 thostmduserapi_se.dll
	std::string dllpath = getBinDir() + DLLHelper::wrap_module(module.c_str(), "");
	m_hInstCTP = DLLHelper::load_library(dllpath.c_str());
	std::string path = StrUtil::printf("%s%s/%s/", m_strFlowDir.c_str(), m_strBroker.c_str(), m_strUserID.c_str());
	if (!StdFile::exists(path.c_str()))
	{
		boost::filesystem::create_directories(boost::filesystem::path(path));
	}	
#ifdef _WIN32
#	ifdef _WIN64
	const char* creatorName = "?CreateFtdcMdApi@CThostFtdcMdApi@@SAPEAV1@PEBD_N1@Z";
#	else
	const char* creatorName = "?CreateFtdcMdApi@CThostFtdcMdApi@@SAPAV1@PBD_N1@Z";
#	endif
#else
	const char* creatorName = "_ZN15CThostFtdcMdApi15CreateFtdcMdApiEPKcbb";
#endif
	// 创建行情API指针
	m_funcCreator = (CTPCreator)DLLHelper::get_symbol(m_hInstCTP, creatorName);
	m_pUserAPI = m_funcCreator(path.c_str(), false, false);
	m_pUserAPI->RegisterSpi(this);								// 利用行情API指针注册回调接口
	m_pUserAPI->RegisterFront((char*)m_strFrontAddr.c_str());	// 利用行情API指针注册前置机网络地址

	return true;
}

void ParserCTP::release()
{
	disconnect();
}

bool ParserCTP::connect()
{
	if(m_pUserAPI)
	{
		///回调行情API指针初始化
		m_pUserAPI->Init();
	}

	return true;
}

bool ParserCTP::disconnect()
{
	if(m_pUserAPI)
	{
		///注册回调接口
		m_pUserAPI->RegisterSpi(NULL);
		///删除接口对象本身
		m_pUserAPI->Release();
		m_pUserAPI = NULL;
	}

	return true;
}

void ParserCTP::OnRspError( CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast )
{
	// 回调检查错误信息函数
	IsErrorRspInfo(pRspInfo);
}

void ParserCTP::OnFrontConnected()
{
	if(m_sink)
	{
		write_log(m_sink, LL_INFO, "[ParserCTP] Market data server connected");
		// 回调行情解析接口的事件处理函数(ParserCTP未实现)
		m_sink->handleEvent(WPE_Connect, 0);
	}
	// 回调发送登录请求函数
	ReqUserLogin();
}

void ParserCTP::OnRspUserLogin( CThostFtdcRspUserLoginField *pRspUserLogin, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast )
{
	if(bIsLast && !IsErrorRspInfo(pRspInfo))
	{
		// 获取当前交易日
		m_uTradingDate = strtoul(m_pUserAPI->GetTradingDay(), NULL, 10);
		
		if(m_sink)
		{
			m_sink->handleEvent(WPE_Login, 0);
		}

		//回调订阅行情数据函数
		SubscribeMarketData();
	}
}

void ParserCTP::OnRspUserLogout(CThostFtdcUserLogoutField *pUserLogout, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if(m_sink)
	{
		m_sink->handleEvent(WPE_Logout, 0);
	}
}

void ParserCTP::OnFrontDisconnected( int nReason )
{
	if(m_sink)
	{
		write_log(m_sink, LL_ERROR, "[ParserCTP] Market data server disconnected: {}", nReason);
		m_sink->handleEvent(WPE_Close, 0);
	}
}

void ParserCTP::OnRspUnSubMarketData( CThostFtdcSpecificInstrumentField *pSpecificInstrument, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast )
{

}

void ParserCTP::OnRtnDepthMarketData( CThostFtdcDepthMarketDataField *pDepthMarketData )
{	
	if(m_pBaseDataMgr == NULL)
	{
		return;
	}

	///业务日期
	uint32_t actDate = strtoul(pDepthMarketData->ActionDay, NULL, 10);
	///最后修改时间 + 最后修改毫秒
	uint32_t actTime = strToTime(pDepthMarketData->UpdateTime) * 1000 + pDepthMarketData->UpdateMillisec;
	uint32_t actHour = actTime / 10000000;

	if (actDate == m_uTradingDate && actHour >= 20)
	{
		//这样的时间是有问题,因为夜盘时发生日期不可能等于交易日
		//这就需要手动设置一下
		uint32_t curDate, curTime;
		TimeUtils::getDateTime(curDate, curTime);
		uint32_t curHour = curTime / 10000000;

		//早上启动以后,会收到昨晚12点以前收盘的行情,这个时候可能会有发生日期=交易日的情况出现
		//这笔数据直接丢掉
		if (curHour >= 3 && curHour < 9)
			return;

		actDate = curDate;

		if (actHour == 23 && curHour == 0)
		{
			//行情时间慢于系统时间
			actDate = TimeUtils::getNextDate(curDate, -1);
		}
		else if (actHour == 0 && curHour == 23)
		{
			//系统时间慢于行情时间
			actDate = TimeUtils::getNextDate(curDate, 1);
		}
	}
	// 获取合约信息
	WTSContractInfo* contract = m_pBaseDataMgr->getContract(pDepthMarketData->InstrumentID, pDepthMarketData->ExchangeID);
	if (contract == NULL)
		return;
	// 获取商品信息
	WTSCommodityInfo* pCommInfo = m_pBaseDataMgr->getCommodity(contract);
	// 创建一个tick数据对象
	WTSTickData* tick = WTSTickData::create(pDepthMarketData->InstrumentID);
	// 获取tick对象中的tick结构体引用, 并填充它
	WTSTickStruct& quote = tick->getTickStruct();
	strcpy(quote.exchg, pCommInfo->getExchg());
	
	quote.action_date = actDate;
	quote.action_time = actTime;
	
	quote.price = checkValid(pDepthMarketData->LastPrice);
	quote.open = checkValid(pDepthMarketData->OpenPrice);
	quote.high = checkValid(pDepthMarketData->HighestPrice);
	quote.low = checkValid(pDepthMarketData->LowestPrice);
	quote.total_volume = pDepthMarketData->Volume;
	quote.trading_date = m_uTradingDate;
	if(pDepthMarketData->SettlementPrice != DBL_MAX)
		quote.settle_price = checkValid(pDepthMarketData->SettlementPrice);
	if(strcmp(quote.exchg, "CZCE") == 0)
	{
		quote.total_turnover = pDepthMarketData->Turnover*pCommInfo->getVolScale();
	}
	else
	{
		if(pDepthMarketData->Turnover != DBL_MAX)
			quote.total_turnover = pDepthMarketData->Turnover;
	}

	quote.open_interest = (uint32_t)pDepthMarketData->OpenInterest;

	quote.upper_limit = checkValid(pDepthMarketData->UpperLimitPrice);
	quote.lower_limit = checkValid(pDepthMarketData->LowerLimitPrice);

	quote.pre_close = checkValid(pDepthMarketData->PreClosePrice);
	quote.pre_settle = checkValid(pDepthMarketData->PreSettlementPrice);
	quote.pre_interest = (uint32_t)pDepthMarketData->PreOpenInterest;

	//委卖价格
	quote.ask_prices[0] = checkValid(pDepthMarketData->AskPrice1);
	quote.ask_prices[1] = checkValid(pDepthMarketData->AskPrice2);
	quote.ask_prices[2] = checkValid(pDepthMarketData->AskPrice3);
	quote.ask_prices[3] = checkValid(pDepthMarketData->AskPrice4);
	quote.ask_prices[4] = checkValid(pDepthMarketData->AskPrice5);

	//委买价格
	quote.bid_prices[0] = checkValid(pDepthMarketData->BidPrice1);
	quote.bid_prices[1] = checkValid(pDepthMarketData->BidPrice2);
	quote.bid_prices[2] = checkValid(pDepthMarketData->BidPrice3);
	quote.bid_prices[3] = checkValid(pDepthMarketData->BidPrice4);
	quote.bid_prices[4] = checkValid(pDepthMarketData->BidPrice5);

	//委卖量
	quote.ask_qty[0] = pDepthMarketData->AskVolume1;
	quote.ask_qty[1] = pDepthMarketData->AskVolume2;
	quote.ask_qty[2] = pDepthMarketData->AskVolume3;
	quote.ask_qty[3] = pDepthMarketData->AskVolume4;
	quote.ask_qty[4] = pDepthMarketData->AskVolume5;

	//委买量
	quote.bid_qty[0] = pDepthMarketData->BidVolume1;
	quote.bid_qty[1] = pDepthMarketData->BidVolume2;
	quote.bid_qty[2] = pDepthMarketData->BidVolume3;
	quote.bid_qty[3] = pDepthMarketData->BidVolume4;
	quote.bid_qty[4] = pDepthMarketData->BidVolume5;

	// 通过数据管理器接口转发数据
	if(m_sink)
		m_sink->handleQuote(tick, 1);

	tick->release();
}

void ParserCTP::OnRspSubMarketData( CThostFtdcSpecificInstrumentField *pSpecificInstrument, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast )
{
	if(!IsErrorRspInfo(pRspInfo))
	{

	}
	else
	{

	}
}

void ParserCTP::OnHeartBeatWarning( int nTimeLapse )
{
	if(m_sink)
		write_log(m_sink, LL_INFO, "[ParserCTP] Heartbeating, elapse: {}", nTimeLapse);
}

void ParserCTP::ReqUserLogin()
{
	if(m_pUserAPI == NULL)
	{
		return;
	}
	// 创建登录字段
	CThostFtdcReqUserLoginField req;
	memset(&req, 0, sizeof(req));
	strcpy(req.BrokerID, m_strBroker.c_str());
	strcpy(req.UserID, m_strUserID.c_str());
	strcpy(req.Password, m_strPassword.c_str());
	// 通过行情API指针回调 用户登录请求 函数
	int iResult = m_pUserAPI->ReqUserLogin(&req, ++m_iRequestID);
	if(iResult != 0)
	{
		if(m_sink)
			write_log(m_sink, LL_ERROR, "[ParserCTP] Sending login request failed: {}", iResult);
	}
}

void ParserCTP::SubscribeMarketData()
{
	CodeSet codeFilter = m_filterSubs;	// 获取行情订阅列表
	if(codeFilter.empty())
	{//如果订阅礼包只空的,则取出全部合约列表
		return;
	}

	char ** subscribe = new char*[codeFilter.size()];
	int nCount = 0;
	for(auto& code : codeFilter)
	{
		std::size_t pos = code.find(".");
		if (pos != std::string::npos)
			subscribe[nCount++] = (char*)code.c_str() + pos + 1;
		else
			subscribe[nCount++] = (char*)code.c_str();
	}

	if(m_pUserAPI && nCount > 0)
	{
		// 回调行情API的 订阅行情 函数
		int iResult = m_pUserAPI->SubscribeMarketData(subscribe, nCount);
		if(iResult != 0)
		{
			if(m_sink)
				write_log(m_sink, LL_ERROR, "[ParserCTP] Sending md subscribe request failed: {}", iResult);
		}
		else
		{
			if(m_sink)
				write_log(m_sink, LL_INFO, "[ParserCTP] Market data of {} contracts subscribed in total", nCount);
		}
	}
	codeFilter.clear();
	delete[] subscribe;
}

bool ParserCTP::IsErrorRspInfo(CThostFtdcRspInfoField *pRspInfo)
{
	return false;
}

void ParserCTP::subscribe(const CodeSet &vecSymbols)
{
	if(m_uTradingDate == 0)
	{
		m_filterSubs = vecSymbols;
	}
	else
	{
		m_filterSubs = vecSymbols;
		// 回调
		SubscribeMarketData();
	}
}

void ParserCTP::unsubscribe(const CodeSet &vecSymbols)
{
}

bool ParserCTP::isConnected()
{
	return m_pUserAPI!=NULL;
}

void ParserCTP::registerSpi(IParserSpi* listener)
{
	m_sink = listener;

	if(m_sink)
		m_pBaseDataMgr = m_sink->getBaseDataMgr();
}
```