# TraderCTP.h

source: `{{ page.path }}`

```cpp
/*!
 * \file TraderCTP.h
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#pragma once

#include <string>
#include <queue>
#include <stdint.h>

#include "../Includes/WTSTypes.h"
#include "../Includes/ITraderApi.h"
#include "../Includes/WTSCollection.hpp"

#include "../API/CTP6.3.15/ThostFtdcTraderApi.h"

#include "../Share/IniHelper.hpp"
#include "../Share/StdUtils.hpp"
#include "../Share/DLLHelper.hpp"

USING_NS_WTP;

class TraderCTP : public ITraderApi, public CThostFtdcTraderSpi
{
public:
	TraderCTP();
	virtual ~TraderCTP();

public:
	typedef enum
	{
		WS_NOTLOGIN,		//未登录
		WS_LOGINING,		//正在登录
		WS_LOGINED,			//已登录
		WS_LOGINFAILED,		//登录失败
		WS_CONFIRM_QRYED,	//结算查询
		WS_CONFIRMED,		//结算确认
		WS_ALLREADY			//全部就绪
	} WrapperState;


private:
	// 结算确认
	int confirm();

	int queryConfirm();

	int authenticate();
	// 登录
	int doLogin();

	//////////////////////////////////////////////////////////////////////////
	//ITraderApi接口
public:
	// 初始化解析管理器
	virtual bool init(WTSVariant* params) override;
	// 释放解析管理器
	virtual void release() override;
	// 注册交易接口和数据管理器
	virtual void registerSpi(ITraderSpi *listener) override;
	// 生成委托单号
	virtual bool makeEntrustID(char* buffer, int length) override;
	// 连接服务器
	virtual void connect() override;
	// 断开连接
	virtual void disconnect() override;
	// 是否连接
	virtual bool isConnected() override;
	// 登录
	virtual int login(const char* user, const char* pass, const char* productInfo) override;
	// 登出
	virtual int logout() override;
	// 下单接口
	virtual int orderInsert(WTSEntrust* eutrust) override;
	// 订单操作接口
	virtual int orderAction(WTSEntrustAction* action) override;
	// 查询账户信息
	virtual int queryAccount() override;
	// 查询持仓信息
	virtual int queryPositions() override;
	// 查询所有订单
	virtual int queryOrders() override;
	// 查询成交明细
	virtual int queryTrades() override;
	// 查询结算单
	virtual int querySettlement(uint32_t uDate) override;


	//////////////////////////////////////////////////////////////////////////
	//CTP交易接口实现
public:
	///当客户端与交易后台建立起通信连接时（还未登录前），该方法被调用。
	virtual void OnFrontConnected() override;
	///当客户端与交易后台通信连接断开时，该方法被调用
	virtual void OnFrontDisconnected(int nReason) override;
	///心跳超时警告。当长时间未收到报文时，该方法被调用
	virtual void OnHeartBeatWarning(int nTimeLapse) override;
	///客户端认证响应
	virtual void OnRspAuthenticate(CThostFtdcRspAuthenticateField *pRspAuthenticateField, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///登录请求响应
	virtual void OnRspUserLogin(CThostFtdcRspUserLoginField *pRspUserLogin, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///登出请求响应
	virtual void OnRspUserLogout(CThostFtdcUserLogoutField *pUserLogout, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///投资者结算结果确认响应
	virtual void OnRspSettlementInfoConfirm(CThostFtdcSettlementInfoConfirmField *pSettlementInfoConfirm, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询结算信息确认响应
	virtual void OnRspQrySettlementInfoConfirm(CThostFtdcSettlementInfoConfirmField *pSettlementInfoConfirm, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询资金账户响应
	virtual void OnRspQryTradingAccount(CThostFtdcTradingAccountField *pTradingAccount, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///报单录入请求响应
	virtual void OnRspOrderInsert(CThostFtdcInputOrderField *pInputOrder, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///报单操作请求响应
	virtual void OnRspOrderAction(CThostFtdcInputOrderActionField *pInputOrderAction, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询投资者持仓响应
	virtual void OnRspQryInvestorPosition(CThostFtdcInvestorPositionField *pInvestorPosition, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询投资者结算结果响应
	virtual void OnRspQrySettlementInfo(CThostFtdcSettlementInfoField *pSettlementInfo, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询成交响应
	virtual void OnRspQryTrade(CThostFtdcTradeField *pTrade, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///请求查询报单响应
	virtual void OnRspQryOrder(CThostFtdcOrderField *pOrder, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///错误应答
	virtual void OnRspError(CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast) override;
	///报单通知
	virtual void OnRtnOrder(CThostFtdcOrderField *pOrder) override;
	///成交通知
	virtual void OnRtnTrade(CThostFtdcTradeField *pTrade) override;
	///报单录入错误回报
	virtual void OnErrRtnOrderInsert(CThostFtdcInputOrderField *pInputOrder, CThostFtdcRspInfoField *pRspInfo) override;

protected:
	/*
	*	检查错误信息
	*/
	bool IsErrorRspInfo(CThostFtdcRspInfoField *pRspInfo);

	// 将WT自定义字段类型变为CTP字段类型
	int wrapPriceType(WTSPriceType priceType, bool isCFFEX = false);
	int wrapDirectionType(WTSDirectionType dirType, WTSOffsetType offType);
	int wrapOffsetType(WTSOffsetType offType);
	int	wrapTimeCondition(WTSTimeCondition timeCond);
	int wrapActionFlag(WTSActionFlag actionFlag);

	// 将CTP字段类型变为WT自定义字段类型
	WTSPriceType		wrapPriceType(TThostFtdcOrderPriceTypeType priceType);
	WTSDirectionType	wrapDirectionType(TThostFtdcDirectionType dirType, TThostFtdcOffsetFlagType offType);
	WTSDirectionType	wrapPosDirection(TThostFtdcPosiDirectionType dirType);
	WTSOffsetType		wrapOffsetType(TThostFtdcOffsetFlagType offType);
	WTSTimeCondition	wrapTimeCondition(TThostFtdcTimeConditionType timeCond);
	WTSOrderState		wrapOrderState(TThostFtdcOrderStatusType orderState);

	// 用CTP报单填充WT自定义订单信息
	WTSOrderInfo*	makeOrderInfo(CThostFtdcOrderField* orderField);
	// 用CTP输入报单填充WT自定义委托数据结构
	WTSEntrust*		makeEntrust(CThostFtdcInputOrderField *entrustField);
	// 将CTP报错信息变为WT自定义报错信息
	WTSError*		makeError(CThostFtdcRspInfoField* rspInfo, WTSErroCode ec = WEC_NONE);
	// 将CTP成交字段变为WT自定义成交信息
	WTSTradeInfo*	makeTradeRecord(CThostFtdcTradeField *tradeField);

	// 生成录入报单ID
	std::string		generateEntrustID(uint32_t frontid, uint32_t sessionid, uint32_t orderRef);
	// 提取录入报单ID
	bool			extractEntrustID(const char* entrustid, uint32_t &frontid, uint32_t &sessionid, uint32_t &orderRef);
	// 生成请求ID
	uint32_t		genRequestID();

protected:
	// 登录字段
	std::string		m_strBroker;				// 经纪商
	std::string		m_strFront;					// 前置地址
	std::string		m_strUser;
	std::string		m_strPass;
	std::string		m_strAppID;
	std::string		m_strAuthCode;

	std::string		m_strProdInfo;				// 产品信息

	bool			m_bQuickStart;

	std::string		m_strTag;

	std::string		m_strSettleInfo;			// 结算信息

	std::string		m_strUserName;				// 用户名
	std::string		m_strFlowDir;				// 目录

	ITraderSpi*	m_sink;							// 交易接口指针
	uint64_t		m_uLastQryTime;				// 最后请求时间

	uint32_t					m_lDate;
	TThostFtdcFrontIDType		m_frontID;		//前置编号
	TThostFtdcSessionIDType		m_sessionID;	//会话编号
	std::atomic<uint32_t>		m_orderRef;		//报单引用

	WrapperState				m_wrapperState;

	CThostFtdcTraderApi*		m_pUserAPI;		// 交易API指针
	std::atomic<uint32_t>		m_iRequestID;	// 请求ID

	typedef WTSHashMap<std::string> PositionMap;
	PositionMap*				m_mapPosition;	// 持仓信息字典
	WTSArray*					m_ayTrades;		// 成交信息列表
	WTSArray*					m_ayOrders;		// 订单信息列表
	WTSArray*					m_ayPosDetail;	// 持仓详情列表

	IBaseDataMgr*				m_bdMgr;		// 数据管理器指针

	typedef std::queue<CommonExecuter>	QueryQue;
	QueryQue				m_queQuery;			// 
	bool					m_bInQuery;			// 
	StdUniqueMutex			m_mtxQuery;			// 
	uint64_t				m_lastQryTime;		// 

	bool					m_bStopped;			// 
	StdThreadPtr			m_thrdWorker;		// 

	std::string		m_strModule;				// 
	DllHandle		m_hInstCTP;					// dll句柄
	typedef CThostFtdcTraderApi* (*CTPCreator)(const char *);
	CTPCreator		m_funcCreator;				// 交易API创建

	IniHelper		m_iniHelper;				// ini文件辅助类
};
```

## TraderCTP.cpp

```cpp
/*!
 * \file TraderCTP.cpp
 * \project	WonderTrader
 *
 * \author Wesley
 * \date 2020/03/30
 * 
 * \brief 
 */
#include "TraderCTP.h"

#include "../Includes/WTSError.hpp"
#include "../Includes/WTSContractInfo.hpp"
#include "../Includes/WTSSessionInfo.hpp"
#include "../Includes/WTSTradeDef.hpp"
#include "../Includes/WTSDataDef.hpp"
#include "../Includes/WTSVariant.hpp"
#include "../Includes/IBaseDataMgr.h"

#include "../Share/decimal.h"
#include "../Share/ModuleHelper.hpp"

#include <boost/filesystem.hpp>

const char* ENTRUST_SECTION = "entrusts";
const char* ORDER_SECTION = "orders";

//By Wesley @ 2022.01.05
#include "../Share/fmtlib.h"
template<typename... Args>
inline void write_log(ITraderSpi* sink, WTSLogLevel ll, const char* format, const Args&... args)
{
	if (sink == NULL)
		return;

	static thread_local char buffer[512] = { 0 };
	fmt::format_to(buffer, format, args...);

	sink->handleTraderLog(ll, buffer);
}

uint32_t strToTime(const char* strTime)
{
	std::string str;
	const char *pos = strTime;
	while (strlen(pos) > 0)
	{
		if (pos[0] != ':')
		{
			str.append(pos, 1);
		}
		pos++;
	}

	return strtoul(str.c_str(), NULL, 10);
}

extern "C"
{
	EXPORT_FLAG ITraderApi* createTrader()
	{
		TraderCTP *instance = new TraderCTP();
		return instance;
	}

	EXPORT_FLAG void deleteTrader(ITraderApi* &trader)
	{
		if (NULL != trader)
		{
			delete trader;
			trader = NULL;
		}
	}
}

TraderCTP::TraderCTP()
	: m_pUserAPI(NULL)
	, m_mapPosition(NULL)
	, m_ayOrders(NULL)
	, m_ayTrades(NULL)
	, m_ayPosDetail(NULL)
	, m_wrapperState(WS_NOTLOGIN)
	, m_uLastQryTime(0)
	, m_iRequestID(0)
	, m_bQuickStart(false)
	, m_bInQuery(false)
	, m_bStopped(false)
	, m_lastQryTime(0)
{
}


TraderCTP::~TraderCTP()
{
}

bool TraderCTP::init(WTSVariant* params)
{
	// 登录字段
	m_strFront = params->get("front")->asCString();
	m_strBroker = params->get("broker")->asCString();
	m_strUser = params->get("user")->asCString();
	m_strPass = params->get("pass")->asCString();

	m_strAppID = params->getCString("appid");
	m_strAuthCode = params->getCString("authcode");
	m_strFlowDir = params->getCString("flowdir");

	if (m_strFlowDir.empty())
		m_strFlowDir = "CTPTDFlow";

	m_strFlowDir = StrUtil::standardisePath(m_strFlowDir);

	// 获取交易dll
	WTSVariant* param = params->get("ctpmodule");
	if (param != NULL)
		m_strModule = getBinDir() + DLLHelper::wrap_module(param->asCString());
	else
		m_strModule = getBinDir() + DLLHelper::wrap_module("thosttraderapi_se", "");

	m_hInstCTP = DLLHelper::load_library(m_strModule.c_str());
#ifdef _WIN32
#	ifdef _WIN64
	const char* creatorName = "?CreateFtdcTraderApi@CThostFtdcTraderApi@@SAPEAV1@PEBD@Z";
#	else
	const char* creatorName = "?CreateFtdcTraderApi@CThostFtdcTraderApi@@SAPAV1@PBD@Z";
#	endif
#else
	const char* creatorName = "_ZN19CThostFtdcTraderApi19CreateFtdcTraderApiEPKc";
#endif
	m_funcCreator = (CTPCreator)DLLHelper::get_symbol(m_hInstCTP, creatorName);

	m_bQuickStart = params->getBoolean("quick");

	return true;
}

void TraderCTP::release()
{
	m_bStopped = true;

	if (m_pUserAPI)
	{
		m_pUserAPI->RegisterSpi(NULL);
		m_pUserAPI->Release();
		m_pUserAPI = NULL;
	}

	if (m_ayOrders)
		m_ayOrders->clear();

	if (m_ayPosDetail)
		m_ayPosDetail->clear();

	if (m_mapPosition)
		m_mapPosition->clear();

	if (m_ayTrades)
		m_ayTrades->clear();
}

void TraderCTP::connect()
{
	std::stringstream ss;
	ss << m_strFlowDir << "flows/" << m_strBroker << "/" << m_strUser << "/";
	boost::filesystem::create_directories(ss.str().c_str());
	m_pUserAPI = m_funcCreator(ss.str().c_str());
	///注册交易API的回调接口
	m_pUserAPI->RegisterSpi(this);
	if (m_bQuickStart)
	{
		m_pUserAPI->SubscribePublicTopic(THOST_TERT_QUICK);			// 注册公有流
		m_pUserAPI->SubscribePrivateTopic(THOST_TERT_QUICK);		// 注册私有流
	}
	else
	{
		m_pUserAPI->SubscribePublicTopic(THOST_TERT_RESUME);		// 注册公有流
		m_pUserAPI->SubscribePrivateTopic(THOST_TERT_RESUME);		// 注册私有流
	}
	///注册前置机网络地址
	m_pUserAPI->RegisterFront((char*)m_strFront.c_str());

	if (m_pUserAPI)
	{
		///交易API初始化
		m_pUserAPI->Init();
	}
	// 处理流控
	if (m_thrdWorker == NULL)
	{
		m_thrdWorker.reset(new StdThread([this](){
			while (!m_bStopped)
			{
				if(m_queQuery.empty() || m_bInQuery)
				{
					std::this_thread::sleep_for(std::chrono::milliseconds(1));
					continue;
				}

				uint64_t curTime = TimeUtils::getLocalTimeNow();
				if (curTime - m_lastQryTime < 1000)
				{
					std::this_thread::sleep_for(std::chrono::milliseconds(50));
					continue;
				}


				m_bInQuery = true;
				CommonExecuter& handler = m_queQuery.front();
				handler();

				{
					StdUniqueLock lock(m_mtxQuery);
					m_queQuery.pop();
				}

				m_lastQryTime = TimeUtils::getLocalTimeNow();
			}
		}));
	}
}

void TraderCTP::disconnect()
{
	m_queQuery.push([this]() {
		release();
	});

	if (m_thrdWorker)
	{
		m_thrdWorker->join();
		m_thrdWorker = NULL;
	}
}

bool TraderCTP::makeEntrustID(char* buffer, int length)
{
	if (buffer == NULL || length == 0)
		return false;

	try
	{
		memset(buffer, 0, length);
		uint32_t orderref = m_orderRef.fetch_add(1) + 1;
		sprintf(buffer, "%06u#%010u#%06u", m_frontID, m_sessionID, orderref);
		return true;
	}
	catch (...)
	{

	}

	return false;
}

void TraderCTP::registerSpi(ITraderSpi *listener)
{
	m_sink = listener;
	if (m_sink)
	{
		m_bdMgr = listener->getBaseDataMgr();
	}
}

uint32_t TraderCTP::genRequestID()
{
	return m_iRequestID.fetch_add(1) + 1;
}

int TraderCTP::login(const char* user, const char* pass, const char* productInfo)
{
	m_strUser = user;
	m_strPass = pass;

	if (m_pUserAPI == NULL)
	{
		return -1;
	}

	m_wrapperState = WS_LOGINING;
	authenticate();

	return 0;
}

int TraderCTP::doLogin()
{
	// 创建登录字段
	CThostFtdcReqUserLoginField req;
	memset(&req, 0, sizeof(req));
	strcpy(req.BrokerID, m_strBroker.c_str());
	strcpy(req.UserID, m_strUser.c_str());
	strcpy(req.Password, m_strPass.c_str());
	strcpy(req.UserProductInfo, m_strProdInfo.c_str());
	// 调用交易API登录
	int iResult = m_pUserAPI->ReqUserLogin(&req, genRequestID());
	if (iResult != 0)
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP] Sending login request failed: {}", iResult);
	}

	return 0;
}

int TraderCTP::logout()
{
	if (m_pUserAPI == NULL)
	{
		return -1;
	}
	///用户登出请求
	CThostFtdcUserLogoutField req;
	memset(&req, 0, sizeof(req));
	strcpy(req.BrokerID, m_strBroker.c_str());
	strcpy(req.UserID, m_strUser.c_str());
	///回调交易API登出请求
	int iResult = m_pUserAPI->ReqUserLogout(&req, genRequestID());
	if (iResult != 0)
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP] Sending logout request failed: {}", iResult);
	}

	return 0;
}

int TraderCTP::orderInsert(WTSEntrust* entrust)
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}
	// 填充输入报单字段
	CThostFtdcInputOrderField req;
	memset(&req, 0, sizeof(req));
	///经纪公司代码
	strcpy(req.BrokerID, m_strBroker.c_str());
	///投资者代码
	strcpy(req.InvestorID, m_strUser.c_str());
	///合约代码
	strcpy(req.InstrumentID, entrust->getCode());

	strcpy(req.ExchangeID, entrust->getExchg());

	if (strlen(entrust->getUserTag()) == 0)
	{
		///报单引用
		sprintf(req.OrderRef, "%u", m_orderRef.fetch_add(0));
	}
	else
	{
		uint32_t fid, sid, orderref;
		extractEntrustID(entrust->getEntrustID(), fid, sid, orderref);
		///报单引用
		sprintf(req.OrderRef, "%u", orderref);
	}

	if (strlen(entrust->getUserTag()) > 0)
	{
		m_iniHelper.writeString(ENTRUST_SECTION, entrust->getEntrustID(), entrust->getUserTag());
		m_iniHelper.save();
	}
	// 获取合约信息
	WTSContractInfo* ct = m_bdMgr->getContract(entrust->getCode(), entrust->getExchg());
	if (ct == NULL)
		return -1;

	///用户代码
	//	TThostFtdcUserIDType	UserID;
	///报单价格条件: 限价
	req.OrderPriceType = wrapPriceType(entrust->getPriceType(), strcmp(entrust->getExchg(), "CFFEX") == 0);
	///买卖方向: 
	req.Direction = wrapDirectionType(entrust->getDirection(), entrust->getOffsetType());
	///组合开平标志: 开仓
	req.CombOffsetFlag[0] = wrapOffsetType(entrust->getOffsetType());
	///组合投机套保标志
	req.CombHedgeFlag[0] = THOST_FTDC_HF_Speculation;
	///价格
	req.LimitPrice = entrust->getPrice();
	///数量: 1
	req.VolumeTotalOriginal = (int)entrust->getVolume();
	///有效期类型: 当日有效
	req.TimeCondition = wrapTimeCondition(entrust->getTimeCondition());
	///GTD日期
	//	TThostFtdcDateType	GTDDate;
	///成交量类型: 任何数量
	req.VolumeCondition = THOST_FTDC_VC_AV;
	///最小成交量: 1
	req.MinVolume = 1;
	///触发条件: 立即
	req.ContingentCondition = THOST_FTDC_CC_Immediately;
	///止损价
	//	TThostFtdcPriceType	StopPrice;
	///强平原因: 非强平
	req.ForceCloseReason = THOST_FTDC_FCC_NotForceClose;
	///自动挂起标志: 否
	req.IsAutoSuspend = 0;
	///业务单元
	//	TThostFtdcBusinessUnitType	BusinessUnit;
	///请求编号
	//	TThostFtdcRequestIDType	RequestID;
	///用户强评标志: 否
	req.UserForceClose = 0;
	// 回调交易API 报单录入请求
	int iResult = m_pUserAPI->ReqOrderInsert(&req, genRequestID());
	if (iResult != 0)
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP] Order inserting failed: {}", iResult);
	}

	return 0;
}

int TraderCTP::orderAction(WTSEntrustAction* action)
{
	if (m_wrapperState != WS_ALLREADY)
		return -1;

	uint32_t frontid, sessionid, orderref;
	if (!extractEntrustID(action->getEntrustID(), frontid, sessionid, orderref))
		return -1;

	CThostFtdcInputOrderActionField req;
	memset(&req, 0, sizeof(req));
	///经纪公司代码
	strcpy(req.BrokerID, m_strBroker.c_str());
	///投资者代码
	strcpy(req.InvestorID, m_strUser.c_str());
	///报单引用
	sprintf(req.OrderRef, "%u", orderref);
	///请求编号
	///前置编号
	req.FrontID = frontid;
	///会话编号
	req.SessionID = sessionid;
	///操作标志
	req.ActionFlag = wrapActionFlag(action->getActionFlag());
	///合约代码
	strcpy(req.InstrumentID, action->getCode());

	req.LimitPrice = action->getPrice();

	req.VolumeChange = (int32_t)action->getVolume();

	strcpy(req.OrderSysID, action->getOrderID());
	strcpy(req.ExchangeID, action->getExchg());

	int iResult = m_pUserAPI->ReqOrderAction(&req, genRequestID());
	if (iResult != 0)
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP] Sending cancel request failed: {}", iResult);
	}

	return 0;
}

int TraderCTP::queryAccount()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}

	{
		StdUniqueLock lock(m_mtxQuery);
		m_queQuery.push([this]() {
			CThostFtdcQryTradingAccountField req;
			memset(&req, 0, sizeof(req));
			strcpy(req.BrokerID, m_strBroker.c_str());
			strcpy(req.InvestorID, m_strUser.c_str());
			m_pUserAPI->ReqQryTradingAccount(&req, genRequestID());
		});
	}

	//triggerQuery();

	return 0;
}

int TraderCTP::queryPositions()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}

	{
		StdUniqueLock lock(m_mtxQuery);
		m_queQuery.push([this]() {
			CThostFtdcQryInvestorPositionField req;
			memset(&req, 0, sizeof(req));
			strcpy(req.BrokerID, m_strBroker.c_str());
			strcpy(req.InvestorID, m_strUser.c_str());
			m_pUserAPI->ReqQryInvestorPosition(&req, genRequestID());
		});
	}

	//triggerQuery();

	return 0;
}

int TraderCTP::queryOrders()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}

	{
		StdUniqueLock lock(m_mtxQuery);
		m_queQuery.push([this]() {
			CThostFtdcQryOrderField req;
			memset(&req, 0, sizeof(req));
			strcpy(req.BrokerID, m_strBroker.c_str());
			strcpy(req.InvestorID, m_strUser.c_str());

			m_pUserAPI->ReqQryOrder(&req, genRequestID());
		});

		//triggerQuery();
	}

	return 0;
}

int TraderCTP::queryTrades()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}

	{
		StdUniqueLock lock(m_mtxQuery);
		m_queQuery.push([this]() {
			CThostFtdcQryTradeField req;
			memset(&req, 0, sizeof(req));
			strcpy(req.BrokerID, m_strBroker.c_str());
			strcpy(req.InvestorID, m_strUser.c_str());

			m_pUserAPI->ReqQryTrade(&req, genRequestID());
		});
	}

	//triggerQuery();

	return 0;
}

int TraderCTP::querySettlement(uint32_t uDate)
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_ALLREADY)
	{
		return -1;
	}

	m_strSettleInfo.clear();
	StdUniqueLock lock(m_mtxQuery);
	m_queQuery.push([this, uDate]() {
		CThostFtdcQrySettlementInfoField req;
		memset(&req, 0, sizeof(req));
		strcpy(req.BrokerID, m_strBroker.c_str());
		strcpy(req.InvestorID, m_strUser.c_str());
		sprintf(req.TradingDay, "%u", uDate);

		m_pUserAPI->ReqQrySettlementInfo(&req, genRequestID());
	});
	return 0;
}

void TraderCTP::OnFrontConnected()
{
	if (m_sink)
		// 回调交易接口事件处理
		m_sink->handleEvent(WTE_Connect, 0);
}

void TraderCTP::OnFrontDisconnected(int nReason)
{
	m_wrapperState = WS_NOTLOGIN;
	if (m_sink)
		// 回调交易接口事件处理
		m_sink->handleEvent(WTE_Close, nReason);
}

void TraderCTP::OnHeartBeatWarning(int nTimeLapse)
{
	write_log(m_sink, LL_DEBUG, "[TraderCTP][{}-{}] Heartbeating...", m_strBroker.c_str(), m_strUser.c_str());
}

void TraderCTP::OnRspAuthenticate(CThostFtdcRspAuthenticateField *pRspAuthenticateField, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	// 如果没有报错就登录
	if (!IsErrorRspInfo(pRspInfo))
	{
		doLogin();
	}
	else
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP][{}-{}] Authentiation failed: {}", m_strBroker.c_str(), m_strUser.c_str(), pRspInfo->ErrorMsg);
		m_wrapperState = WS_LOGINFAILED;

		if (m_sink)
			m_sink->onLoginResult(false, pRspInfo->ErrorMsg, 0);
	}

}

void TraderCTP::OnRspUserLogin(CThostFtdcRspUserLoginField *pRspUserLogin, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (!IsErrorRspInfo(pRspInfo))
	{
		m_wrapperState = WS_LOGINED;

		// 保存会话参数
		m_frontID = pRspUserLogin->FrontID;
		m_sessionID = pRspUserLogin->SessionID;
		m_orderRef = atoi(pRspUserLogin->MaxOrderRef);
		///获取当前交易日
		m_lDate = atoi(m_pUserAPI->GetTradingDay());

		write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Login succeed, AppID: {}, Sessionid: {}, login time: {}...",
			m_strBroker.c_str(), m_strUser.c_str(), m_strAppID.c_str(), m_sessionID, pRspUserLogin->LoginTime);

		std::stringstream ss;
		ss << m_strFlowDir << "local/" << m_strBroker << "/";
		std::string path = StrUtil::standardisePath(ss.str());
		if (!StdFile::exists(path.c_str()))
			boost::filesystem::create_directories(path.c_str());
		ss << m_strUser << ".dat";

		m_iniHelper.load(ss.str().c_str());
		uint32_t lastDate = m_iniHelper.readUInt("marker", "date", 0);
		if(lastDate != m_lDate)
		{
			//交易日不同,清理掉原来的数据
			m_iniHelper.removeSection(ENTRUST_SECTION);
			m_iniHelper.removeSection(ORDER_SECTION);
			m_iniHelper.writeUInt("marker", "date", m_lDate);
			m_iniHelper.save();

			write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Trading date changed[{} -> {}], local cache cleared...", m_strBroker.c_str(), m_strUser.c_str(), lastDate, m_lDate);
		}

		write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Login succeed, trading date: {}...", m_strBroker.c_str(), m_strUser.c_str(), m_lDate);

		write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Querying confirming state of settlement data...", m_strBroker.c_str(), m_strUser.c_str());
		queryConfirm();
	}
	else
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP][{}-{}] Login failed: {}", m_strBroker.c_str(), m_strUser.c_str(), pRspInfo->ErrorMsg);
		m_wrapperState = WS_LOGINFAILED;

		if (m_sink)
			m_sink->onLoginResult(false, pRspInfo->ErrorMsg, 0);
	}
}

void TraderCTP::OnRspUserLogout(CThostFtdcUserLogoutField *pUserLogout, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	m_wrapperState = WS_NOTLOGIN;
	if (m_sink)
		// 回调交易接口事件处理
		m_sink->handleEvent(WTE_Logout, 0);
}

void TraderCTP::OnRspQrySettlementInfoConfirm(CThostFtdcSettlementInfoConfirmField *pSettlementInfoConfirm, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}
	// 没报错
	if (!IsErrorRspInfo(pRspInfo))
	{
		// 结算结果不为空
		if (pSettlementInfoConfirm != NULL)
		{
			///确认日期
			uint32_t uConfirmDate = strtoul(pSettlementInfoConfirm->ConfirmDate, NULL, 10);
			if (uConfirmDate >= m_lDate)
			{
				// 结算已确认
				m_wrapperState = WS_CONFIRMED;

				write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Trading channel initialized...", m_strBroker.c_str(), m_strUser.c_str());
				m_wrapperState = WS_ALLREADY;
				if (m_sink)
					m_sink->onLoginResult(true, "", m_lDate);
			}
			else
			{
				m_wrapperState = WS_CONFIRM_QRYED;

				write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Confirming settlement data...", m_strBroker.c_str(), m_strUser.c_str());
				// 结算日期不对重新确认结算
				confirm();
			}
		}
		else
		{
			m_wrapperState = WS_CONFIRM_QRYED;
			// 结算结果为空重新确认结算
			confirm();
		}
	}

}

void TraderCTP::OnRspSettlementInfoConfirm(CThostFtdcSettlementInfoConfirmField *pSettlementInfoConfirm, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	// 如果没有报错并且 结算结果字段不为空
	if (!IsErrorRspInfo(pRspInfo) && pSettlementInfoConfirm != NULL)
	{
		if (m_wrapperState == WS_CONFIRM_QRYED)
		{
			// 结算已确认
			m_wrapperState = WS_CONFIRMED;

			write_log(m_sink, LL_INFO, "[TraderCTP][{}-{}] Trading channel initialized...", m_strBroker.c_str(), m_strUser.c_str());
			m_wrapperState = WS_ALLREADY;
			if (m_sink)
				m_sink->onLoginResult(true, "", m_lDate);
		}
	}
}

void TraderCTP::OnRspOrderInsert(CThostFtdcInputOrderField *pInputOrder, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	// 获取WT自定义委托数据结构
	WTSEntrust* entrust = makeEntrust(pInputOrder);
	if (entrust)
	{
		WTSError *err = makeError(pRspInfo, WEC_ORDERINSERT);
		// 回调交易接口指针输入报单
		if (m_sink)
			m_sink->onRspEntrust(entrust, err);
		entrust->release();
		err->release();
	}
	else if(IsErrorRspInfo(pRspInfo))
	{
		WTSError *err = makeError(pRspInfo, WEC_ORDERINSERT);
		if (m_sink)
			m_sink->onTraderError(err);
		err->release();
	}
}

void TraderCTP::OnRspOrderAction(CThostFtdcInputOrderActionField *pInputOrderAction, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (IsErrorRspInfo(pRspInfo))
	{}
	else
	{
		WTSError* error = WTSError::create(WEC_ORDERCANCEL, pRspInfo->ErrorMsg);
		if (m_sink)
			m_sink->onTraderError(error);
	}
}

void TraderCTP::OnRspQryTradingAccount(CThostFtdcTradingAccountField *pTradingAccount, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}
	// 没有报错
	if (bIsLast && !IsErrorRspInfo(pRspInfo))
	{
		// 创建账户信息类并通过查询结果进行填充
		WTSAccountInfo* accountInfo = WTSAccountInfo::create();
		accountInfo->setDescription(StrUtil::printf("{}-{}", m_strBroker.c_str(), m_strUser.c_str()).c_str());
		accountInfo->setPreBalance(pTradingAccount->PreBalance);
		accountInfo->setCloseProfit(pTradingAccount->CloseProfit);
		accountInfo->setDynProfit(pTradingAccount->PositionProfit);
		accountInfo->setMargin(pTradingAccount->CurrMargin);
		accountInfo->setAvailable(pTradingAccount->Available);
		accountInfo->setCommission(pTradingAccount->Commission);
		accountInfo->setFrozenMargin(pTradingAccount->FrozenMargin);
		accountInfo->setFrozenCommission(pTradingAccount->FrozenCommission);
		accountInfo->setDeposit(pTradingAccount->Deposit);
		accountInfo->setWithdraw(pTradingAccount->Withdraw);
		accountInfo->setBalance(accountInfo->getPreBalance() + accountInfo->getCloseProfit() - accountInfo->getCommission() + accountInfo->getDeposit() - accountInfo->getWithdraw());
		accountInfo->setCurrency("CNY");
		// 创建列表插入账户信息
		WTSArray * ay = WTSArray::create();
		ay->append(accountInfo, false);
		// 回调交易接口处理账户信息响应
		if (m_sink)
			m_sink->onRspAccount(ay);

		ay->release();
	}
}

void TraderCTP::OnRspQryInvestorPosition(CThostFtdcInvestorPositionField *pInvestorPosition, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}
	// 如果没报错并且投资者持仓信息不为空
	if (!IsErrorRspInfo(pRspInfo) && pInvestorPosition)
	{
		if (NULL == m_mapPosition)
			m_mapPosition = PositionMap::create();
		// 获取合约信息
		WTSContractInfo* contract = m_bdMgr->getContract(pInvestorPosition->InstrumentID, pInvestorPosition->ExchangeID);
		if (contract)
		{
			// 获取商品信息
			WTSCommodityInfo* commInfo = m_bdMgr->getCommodity(contract);
			std::string key = StrUtil::printf("{}-{}", pInvestorPosition->InstrumentID, pInvestorPosition->PosiDirection);
			// 创建持仓信息类并根据投资者持仓字段填充
			WTSPositionItem* pos = (WTSPositionItem*)m_mapPosition->get(key);
			if(pos == NULL)
			{
				pos = WTSPositionItem::create(pInvestorPosition->InstrumentID, commInfo->getCurrency(), commInfo->getExchg());
				m_mapPosition->add(key, pos, false);
			}
			pos->setDirection(wrapPosDirection(pInvestorPosition->PosiDirection));
			if(commInfo->getCoverMode() == CM_CoverToday)
			{
				if (pInvestorPosition->PositionDate == THOST_FTDC_PSD_Today)
					pos->setNewPosition(pInvestorPosition->Position);
				else
					pos->setPrePosition(pInvestorPosition->Position);
			}
			else
			{
				pos->setNewPosition(pInvestorPosition->TodayPosition);
				pos->setPrePosition(pInvestorPosition->Position - pInvestorPosition->TodayPosition);
			}

			pos->setMargin(pos->getMargin() + pInvestorPosition->UseMargin);
			pos->setDynProfit(pos->getDynProfit() + pInvestorPosition->PositionProfit);
			pos->setPositionCost(pos->getPositionCost() + pInvestorPosition->PositionCost);

			if (pos->getTotalPosition() != 0)
			{
				pos->setAvgPrice(pos->getPositionCost() / pos->getTotalPosition() / commInfo->getVolScale());
			}
			else
			{
				pos->setAvgPrice(0);
			}
			// 合约类别
			if (commInfo->getCategoty() != CC_Combination)
			{
				// 平仓类型
				if (commInfo->getCoverMode() == CM_CoverToday)
				{
					if (pInvestorPosition->PositionDate == THOST_FTDC_PSD_Today)
					{
						int availNew = pInvestorPosition->Position;
						if (pInvestorPosition->PosiDirection == THOST_FTDC_PD_Long)
						{
							availNew -= pInvestorPosition->ShortFrozen;
						}
						else
						{
							availNew -= pInvestorPosition->LongFrozen;
						}
						if (availNew < 0)
							availNew = 0;
						// 可平今仓
						pos->setAvailNewPos(availNew);
					}
					else
					{
						int availPre = pInvestorPosition->Position;
						if (pInvestorPosition->PosiDirection == THOST_FTDC_PD_Long)
						{
							availPre -= pInvestorPosition->ShortFrozen;
						}
						else
						{
							availPre -= pInvestorPosition->LongFrozen;
						}
						if (availPre < 0)
							availPre = 0;
						//可平昨仓
						pos->setAvailPrePos(availPre);
					}
				}
				else
				{
					int availNew = pInvestorPosition->TodayPosition;
					if (pInvestorPosition->PosiDirection == THOST_FTDC_PD_Long)
					{
						availNew -= pInvestorPosition->ShortFrozen;
					}
					else
					{
						availNew -= pInvestorPosition->LongFrozen;
					}
					if (availNew < 0)
						availNew = 0;
					pos->setAvailNewPos(availNew);

					double availPre = pos->getNewPosition() + pos->getPrePosition()
						- pInvestorPosition->LongFrozen - pInvestorPosition->ShortFrozen
						- pos->getAvailNewPos();
					pos->setAvailPrePos(availPre);
				}
			}
			else
			{

			}

			if (decimal::lt(pos->getTotalPosition(), 0.0) && decimal::eq(pos->getMargin(), 0.0))
			{
				//有仓位,但是保证金为0,则说明是套利合约,单个合约的可用持仓全部置为0
				pos->setAvailNewPos(0);
				pos->setAvailPrePos(0);
			}
		}
	}

	if (bIsLast)
	{

		WTSArray* ayPos = WTSArray::create();

		if(m_mapPosition && m_mapPosition->size() > 0)
		{
			for (auto it = m_mapPosition->begin(); it != m_mapPosition->end(); it++)
			{
				// 持仓信息列表
				ayPos->append(it->second, true);
			}
		}
		// 回调交易接口
		if (m_sink)
			m_sink->onRspPosition(ayPos);

		if (m_mapPosition)
		{
			m_mapPosition->release();
			m_mapPosition = NULL;
		}

		ayPos->release();
	}
}

void TraderCTP::OnRspQrySettlementInfo(CThostFtdcSettlementInfoField *pSettlementInfo, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}

	if (!IsErrorRspInfo(pRspInfo) && pSettlementInfo)
	{
		m_strSettleInfo += pSettlementInfo->Content;
	}

	if (bIsLast && !m_strSettleInfo.empty())
	{
		m_sink->onRspSettlementInfo(atoi(pSettlementInfo->TradingDay), m_strSettleInfo.c_str());
	}
}

void TraderCTP::OnRspQryTrade(CThostFtdcTradeField *pTrade, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}
	// 没报错且成交字段不为空
	if (!IsErrorRspInfo(pRspInfo) && pTrade)
	{
		if (NULL == m_ayTrades)
			m_ayTrades = WTSArray::create();
		// 获取成交信息
		WTSTradeInfo* trade = makeTradeRecord(pTrade);
		if (trade)
		{
			// 将成交信息添加至成交信息列表
			m_ayTrades->append(trade, false);
		}
	}

	if (bIsLast)
	{
		// 回调交易接口
		if (m_sink)
			m_sink->onRspTrades(m_ayTrades);

		if (NULL != m_ayTrades)
			m_ayTrades->clear();
	}
}

void TraderCTP::OnRspQryOrder(CThostFtdcOrderField *pOrder, CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	if (bIsLast)
	{
		m_bInQuery = false;
	}

	if (!IsErrorRspInfo(pRspInfo) && pOrder)
	{
		if (NULL == m_ayOrders)
			m_ayOrders = WTSArray::create();
		// 获取订单信息类
		WTSOrderInfo* orderInfo = makeOrderInfo(pOrder);
		if (orderInfo)
		{
			// 将订单信息类添加至订单信息列表
			m_ayOrders->append(orderInfo, false);
		}
	}
	if (bIsLast)
	{
		// 回调交易接口
		if (m_sink)
			m_sink->onRspOrders(m_ayOrders);

		if (m_ayOrders)
			m_ayOrders->clear();
	}
}

void TraderCTP::OnRspError(CThostFtdcRspInfoField *pRspInfo, int nRequestID, bool bIsLast)
{
	int x = 0;
}

void TraderCTP::OnRtnOrder(CThostFtdcOrderField *pOrder)
{
	// 获取订单信息
	WTSOrderInfo *orderInfo = makeOrderInfo(pOrder);
	if (orderInfo)
	{
		// 回调交易接口
		if (m_sink)
			m_sink->onPushOrder(orderInfo);

		orderInfo->release();
	}
}

void TraderCTP::OnRtnTrade(CThostFtdcTradeField *pTrade)
{
	// 获取成交信息类
	WTSTradeInfo *tRecord = makeTradeRecord(pTrade);
	if (tRecord)
	{
		// 回调交易接口
		if (m_sink)
			m_sink->onPushTrade(tRecord);

		tRecord->release();
	}
}

int TraderCTP::wrapDirectionType(WTSDirectionType dirType, WTSOffsetType offsetType)
{
	if (WDT_LONG == dirType)
		if (offsetType == WOT_OPEN)
			return THOST_FTDC_D_Buy;
		else
			return THOST_FTDC_D_Sell;
	else
		if (offsetType == WOT_OPEN)
			return THOST_FTDC_D_Sell;
		else
			return THOST_FTDC_D_Buy;
}

WTSDirectionType TraderCTP::wrapDirectionType(TThostFtdcDirectionType dirType, TThostFtdcOffsetFlagType offsetType)
{
	if (THOST_FTDC_D_Buy == dirType)
		if (offsetType == THOST_FTDC_OF_Open)
			return WDT_LONG;
		else
			return WDT_SHORT;
	else
		if (offsetType == THOST_FTDC_OF_Open)
			return WDT_SHORT;
		else
			return WDT_LONG;
}

WTSDirectionType TraderCTP::wrapPosDirection(TThostFtdcPosiDirectionType dirType)
{
	if (THOST_FTDC_PD_Long == dirType)
		return WDT_LONG;
	else
		return WDT_SHORT;
}

int TraderCTP::wrapOffsetType(WTSOffsetType offType)
{
	if (WOT_OPEN == offType)
		return THOST_FTDC_OF_Open;
	else if (WOT_CLOSE == offType)
		return THOST_FTDC_OF_Close;
	else if (WOT_CLOSETODAY == offType)
		return THOST_FTDC_OF_CloseToday;
	else if (WOT_CLOSEYESTERDAY == offType)
		return THOST_FTDC_OF_Close;
	else
		return THOST_FTDC_OF_ForceClose;
}

WTSOffsetType TraderCTP::wrapOffsetType(TThostFtdcOffsetFlagType offType)
{
	if (THOST_FTDC_OF_Open == offType)
		return WOT_OPEN;
	else if (THOST_FTDC_OF_Close == offType)
		return WOT_CLOSE;
	else if (THOST_FTDC_OF_CloseToday == offType)
		return WOT_CLOSETODAY;
	else
		return WOT_FORCECLOSE;
}

int TraderCTP::wrapPriceType(WTSPriceType priceType, bool isCFFEX /* = false */)
{
	if (WPT_ANYPRICE == priceType)
		return isCFFEX ? THOST_FTDC_OPT_FiveLevelPrice : THOST_FTDC_OPT_AnyPrice;
	else if (WPT_LIMITPRICE == priceType)
		return THOST_FTDC_OPT_LimitPrice;
	else if (WPT_BESTPRICE == priceType)
		return THOST_FTDC_OPT_BestPrice;
	else
		return THOST_FTDC_OPT_LastPrice;
}

WTSPriceType TraderCTP::wrapPriceType(TThostFtdcOrderPriceTypeType priceType)
{
	if (THOST_FTDC_OPT_AnyPrice == priceType || THOST_FTDC_OPT_FiveLevelPrice == priceType)
		return WPT_ANYPRICE;
	else if (THOST_FTDC_OPT_LimitPrice == priceType)
		return WPT_LIMITPRICE;
	else if (THOST_FTDC_OPT_BestPrice == priceType)
		return WPT_BESTPRICE;
	else
		return WPT_LASTPRICE;
}

int TraderCTP::wrapTimeCondition(WTSTimeCondition timeCond)
{
	if (WTC_IOC == timeCond)
		return THOST_FTDC_TC_IOC;
	else if (WTC_GFD == timeCond)
		return THOST_FTDC_TC_GFD;
	else
		return THOST_FTDC_TC_GFS;
}

WTSTimeCondition TraderCTP::wrapTimeCondition(TThostFtdcTimeConditionType timeCond)
{
	if (THOST_FTDC_TC_IOC == timeCond)
		return WTC_IOC;
	else if (THOST_FTDC_TC_GFD == timeCond)
		return WTC_GFD;
	else
		return WTC_GFS;
}

WTSOrderState TraderCTP::wrapOrderState(TThostFtdcOrderStatusType orderState)
{
	if (orderState != THOST_FTDC_OST_Unknown)
		return (WTSOrderState)orderState;
	else
		return WOS_Submitting;
}

int TraderCTP::wrapActionFlag(WTSActionFlag actionFlag)
{
	if (WAF_CANCEL == actionFlag)
		return THOST_FTDC_AF_Delete;
	else
		return THOST_FTDC_AF_Modify;
}


WTSOrderInfo* TraderCTP::makeOrderInfo(CThostFtdcOrderField* orderField)
{
	// 获取合约信息
	WTSContractInfo* contract = m_bdMgr->getContract(orderField->InstrumentID, orderField->ExchangeID);
	if (contract == NULL)
		return NULL;
	// 创建订单信息类并填充
	WTSOrderInfo* pRet = WTSOrderInfo::create();
	pRet->setPrice(orderField->LimitPrice);
	pRet->setVolume(orderField->VolumeTotalOriginal);
	pRet->setDirection(wrapDirectionType(orderField->Direction, orderField->CombOffsetFlag[0]));
	pRet->setPriceType(wrapPriceType(orderField->OrderPriceType));
	pRet->setTimeCondition(wrapTimeCondition(orderField->TimeCondition));
	pRet->setOffsetType(wrapOffsetType(orderField->CombOffsetFlag[0]));

	pRet->setVolTraded(orderField->VolumeTraded);
	pRet->setVolLeft(orderField->VolumeTotal);

	pRet->setCode(orderField->InstrumentID);
	pRet->setExchange(contract->getExchg());

	uint32_t uDate = strtoul(orderField->InsertDate, NULL, 10);
	std::string strTime = orderField->InsertTime;
	StrUtil::replace(strTime, ":", "");
	uint32_t uTime = strtoul(strTime.c_str(), NULL, 10);
	if (uTime >= 210000 && uDate == m_lDate)
	{
		uDate = TimeUtils::getNextDate(uDate, -1);
	}

	pRet->setOrderDate(uDate);
	pRet->setOrderTime(TimeUtils::makeTime(uDate, uTime * 1000));

	pRet->setOrderState(wrapOrderState(orderField->OrderStatus));
	if (orderField->OrderSubmitStatus >= THOST_FTDC_OSS_InsertRejected)
		pRet->setError(true);		

	pRet->setEntrustID(generateEntrustID(orderField->FrontID, orderField->SessionID, atoi(orderField->OrderRef)).c_str());
	pRet->setOrderID(orderField->OrderSysID);

	pRet->setStateMsg(orderField->StatusMsg);


	std::string usertag = m_iniHelper.readString(ENTRUST_SECTION, pRet->getEntrustID(), "");
	if(usertag.empty())
	{
		pRet->setUserTag(pRet->getEntrustID());
	}
	else
	{
		pRet->setUserTag(usertag.c_str());

		if (strlen(pRet->getOrderID()) > 0)
		{
			m_iniHelper.writeString(ORDER_SECTION, StrUtil::trim(pRet->getOrderID()).c_str(), usertag.c_str());
			m_iniHelper.save();
		}
	}

	return pRet;
}

WTSEntrust* TraderCTP::makeEntrust(CThostFtdcInputOrderField *entrustField)
{
	// 获取合约信息
	WTSContractInfo* ct = m_bdMgr->getContract(entrustField->InstrumentID, entrustField->ExchangeID);
	if (ct == NULL)
		return NULL;
	// 创建委托数据结构类并填充
	WTSEntrust* pRet = WTSEntrust::create(
		entrustField->InstrumentID,
		entrustField->VolumeTotalOriginal,
		entrustField->LimitPrice,
		ct->getExchg());

	pRet->setDirection(wrapDirectionType(entrustField->Direction, entrustField->CombOffsetFlag[0]));
	pRet->setPriceType(wrapPriceType(entrustField->OrderPriceType));
	pRet->setOffsetType(wrapOffsetType(entrustField->CombOffsetFlag[0]));
	pRet->setTimeCondition(wrapTimeCondition(entrustField->TimeCondition));
	pRet->setEntrustID(generateEntrustID(m_frontID, m_sessionID, atoi(entrustField->OrderRef)).c_str());

	std::string usertag = m_iniHelper.readString(ENTRUST_SECTION, pRet->getEntrustID());
	if (!usertag.empty())
		pRet->setUserTag(usertag.c_str());

	return pRet;
}

WTSError* TraderCTP::makeError(CThostFtdcRspInfoField* rspInfo, WTSErroCode ec /* = WEC_NONE */)
{
	WTSError* pRet = WTSError::create(ec, rspInfo->ErrorMsg);
	return pRet;
}

WTSTradeInfo* TraderCTP::makeTradeRecord(CThostFtdcTradeField *tradeField)
{
	// 获取合约信息
	WTSContractInfo* contract = m_bdMgr->getContract(tradeField->InstrumentID, tradeField->ExchangeID);
	if (contract == NULL)
		return NULL;
	// 获取品种信息
	WTSCommodityInfo* commInfo = m_bdMgr->getCommodity(contract);
	// 获取交易时间段
	WTSSessionInfo* sInfo = m_bdMgr->getSession(commInfo->getSession());
	// 创建成交信息类并用响应的交易字段填充
	WTSTradeInfo *pRet = WTSTradeInfo::create(tradeField->InstrumentID, commInfo->getExchg());
	pRet->setVolume(tradeField->Volume);
	pRet->setPrice(tradeField->Price);
	pRet->setTradeID(tradeField->TradeID);

	// 成交时间和日期
	std::string strTime = tradeField->TradeTime;
	StrUtil::replace(strTime, ":", "");
	uint32_t uTime = strtoul(strTime.c_str(), NULL, 10);
	uint32_t uDate = strtoul(tradeField->TradeDate, NULL, 10);
	
	//如果是夜盘时间，并且成交日期等于交易日，说明成交日期是需要修正
	//因为夜盘是提前的，成交日期必然小于交易日
	//但是这里只能做一个简单修正了
	if(uTime >= 210000 && uDate == m_lDate)
	{
		uDate = TimeUtils::getNextDate(uDate, -1);
	}

	pRet->setTradeDate(uDate);
	pRet->setTradeTime(TimeUtils::makeTime(uDate, uTime * 1000));

	WTSDirectionType dType = wrapDirectionType(tradeField->Direction, tradeField->OffsetFlag);

	pRet->setDirection(dType);
	pRet->setOffsetType(wrapOffsetType(tradeField->OffsetFlag));
	pRet->setRefOrder(tradeField->OrderSysID);
	pRet->setTradeType((WTSTradeType)tradeField->TradeType);

	double amount = commInfo->getVolScale()*tradeField->Volume*pRet->getPrice();
	pRet->setAmount(amount);
	std::string usertag = m_iniHelper.readString(ORDER_SECTION, StrUtil::trim(pRet->getRefOrder()).c_str());
	if (!usertag.empty())
		pRet->setUserTag(usertag.c_str());

	return pRet;
}

std::string TraderCTP::generateEntrustID(uint32_t frontid, uint32_t sessionid, uint32_t orderRef)
{
	return StrUtil::printf("%06u#%010u#%06u", frontid, sessionid, orderRef);
}

bool TraderCTP::extractEntrustID(const char* entrustid, uint32_t &frontid, uint32_t &sessionid, uint32_t &orderRef)
{
	//Market.FrontID.SessionID.OrderRef
	const StringVector &vecString = StrUtil::split(entrustid, "#");
	if (vecString.size() != 3)
		return false;

	frontid = strtoul(vecString[0].c_str(), NULL, 10);
	sessionid = strtoul(vecString[1].c_str(), NULL, 10);
	orderRef = strtoul(vecString[2].c_str(), NULL, 10);

	return true;
}

bool TraderCTP::IsErrorRspInfo(CThostFtdcRspInfoField *pRspInfo)
{
	if (pRspInfo && pRspInfo->ErrorID != 0)
		return true;

	return false;
}

void TraderCTP::OnErrRtnOrderInsert(CThostFtdcInputOrderField *pInputOrder, CThostFtdcRspInfoField *pRspInfo)
{
	WTSEntrust* entrust = makeEntrust(pInputOrder);
	if (entrust)
	{
		WTSError *err = makeError(pRspInfo, WEC_ORDERINSERT);
		if (m_sink)
			m_sink->onRspEntrust(entrust, err);
		entrust->release();
		err->release();
	}
}

bool TraderCTP::isConnected()
{
	return (m_wrapperState == WS_ALLREADY);
}

int TraderCTP::queryConfirm()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_LOGINED)
	{
		return -1;
	}

	{
		StdUniqueLock lock(m_mtxQuery);
		m_queQuery.push([this]() {
			CThostFtdcQrySettlementInfoConfirmField req;
			memset(&req, 0, sizeof(req));
			strcpy(req.BrokerID, m_strBroker.c_str());
			strcpy(req.InvestorID, m_strUser.c_str());

			int iResult = m_pUserAPI->ReqQrySettlementInfoConfirm(&req, genRequestID());
			if (iResult != 0)
			{
				write_log(m_sink, LL_ERROR, "[TraderCTP][{}-{}] Sending query of settlement data confirming state failed: {}", m_strBroker.c_str(), m_strUser.c_str(), iResult);
			}
		});
	}

	//triggerQuery();

	return 0;
}

int TraderCTP::confirm()
{
	if (m_pUserAPI == NULL || m_wrapperState != WS_CONFIRM_QRYED)
	{
		return -1;
	}
	///投资者结算结果确认信息
	CThostFtdcSettlementInfoConfirmField req;
	memset(&req, 0, sizeof(req));
	strcpy(req.BrokerID, m_strBroker.c_str());
	strcpy(req.InvestorID, m_strUser.c_str());

	sprintf(req.ConfirmDate, "%u", TimeUtils::getCurDate());
	strncpy(req.ConfirmTime, TimeUtils::getLocalTime().c_str(), 8);
	// 回调交易API确认结算单
	int iResult = m_pUserAPI->ReqSettlementInfoConfirm(&req, genRequestID());
	if (iResult != 0)
	{
		write_log(m_sink, LL_ERROR, "[TraderCTP][{}-{}] Sending confirming of settlement data failed: {}", m_strBroker.c_str(), m_strUser.c_str(), iResult);
		return -1;
	}

	return 0;
}

int TraderCTP::authenticate()
{
	///客户端认证请求字段
	CThostFtdcReqAuthenticateField req;
	memset(&req, 0, sizeof(req));
	strcpy(req.BrokerID, m_strBroker.c_str());
	strcpy(req.UserID, m_strUser.c_str());
	strcpy(req.AuthCode, m_strAuthCode.c_str());
	strcpy(req.AppID, m_strAppID.c_str());
	///回调交易API 客户端认证请求
	m_pUserAPI->ReqAuthenticate(&req, genRequestID());

	return 0;
}

```