# 01-03 消息通知

source: `{{ page.path }}`

## EventNotifier.h

### EventNotifier

```cpp
// 为输出 WtMsgQue.dll 中的函数准备
typedef unsigned long(*FuncCreateMQServer)(const char*, bool);
typedef void(*FuncDestroyMQServer)(unsigned long);
typedef void(*FundPublishMessage)(unsigned long, const char*, const char*, unsigned long);

typedef void(*FuncLogCallback)(unsigned long, const char*, bool);
typedef void(*FuncRegCallbacks)(FuncLogCallback);

class EventNotifier
{
public:
	EventNotifier();
	~EventNotifier();

public:
	bool	init(WTSVariant* cfg);
	void	notifyEvent(const char* evtType);
	void	notifyData(const char* topic, void* data , uint32_t dataLen);
	void	notifyFund(const char* topic, uint32_t uDate, double total_profit, double dynprofit, double dynbalance, double total_fee);
	

private:
	std::string		m_strURL;
	uint32_t		_mq_sid;
	FuncCreateMQServer	_creator;       // MQ函数: 注册
	FuncDestroyMQServer	_remover;       // MQ函数: 删除
	FundPublishMessage	_publisher;     // MQ函数: 通知
	FuncRegCallbacks	_register;      // MQ函数: 注册
};
```

## EventNotifier.cpp

```cpp
// 回调函数
void on_mq_log(unsigned long id, const char* message, bool bServer)
{}

EventNotifier::EventNotifier()
	: _mq_sid(0)
{}

EventNotifier::~EventNotifier()
{
	if (_remover && _mq_sid != 0)
		_remover(_mq_sid);
}

// 从加载的配置文件中初始化消息通知对象
bool EventNotifier::init(WTSVariant* cfg)
{
	if (!cfg->getBoolean("active"))
		return false;

	m_strURL = cfg->getCString("url");
	std::string module = DLLHelper::wrap_module("WtMsgQue", "lib");
	//先看工作目录下是否有对应模块
	std::string dllpath = WtHelper::getCWD() + module;
	//如果没有,则再看模块目录,即dll同目录下
	if (!StdFile::exists(dllpath.c_str()))
		dllpath = WtHelper::getInstDir() + module;

    // 加载 WtMsgQue.dll 文件
	DllHandle dllInst = DLLHelper::load_library(dllpath.c_str());
	if (dllInst == NULL)
	{
		WTSLogger::error("MQ module %s loading failed", dllpath.c_str());
		return false;
	}

    // 获取 dll 中的函数对象
	_creator = (FuncCreateMQServer)DLLHelper::get_symbol(dllInst, "create_server");
	if (_creator == NULL)
	{
		DLLHelper::free_library(dllInst);
		WTSLogger::error("MQ module %s is not compatible", dllpath.c_str());
		return false;
	}

	_remover = (FuncDestroyMQServer)DLLHelper::get_symbol(dllInst, "destroy_server");
	_publisher = (FundPublishMessage)DLLHelper::get_symbol(dllInst, "publish_message");
	_register = (FuncRegCallbacks)DLLHelper::get_symbol(dllInst, "regiter_callbacks");

	//注册回调函数
	_register(on_mq_log);
	
	//创建一个MQServer
	_mq_sid = _creator(m_strURL.c_str(), true);

	WTSLogger::info("EventNotifier initialized with channel %s", m_strURL.c_str());

	return true;
}

// 通知事件
void EventNotifier::notifyEvent(const char* evtType)
{
	if (_publisher)
		_publisher(_mq_sid, "BT_EVENT", evtType, strlen(evtType));
}

// 通知数据
void EventNotifier::notifyData(const char* topic, void* data , uint32_t dataLen)
{
	if (_publisher)
		_publisher(_mq_sid, topic, (const char*)data, dataLen);
}

void EventNotifier::notifyFund(const char* topic, uint32_t uDate, double total_profit, double dynprofit, double dynbalance, double total_fee)
{
	std::string output;
	{
		rj::Document root(rj::kObjectType);
		rj::Document::AllocatorType &allocator = root.GetAllocator();

		root.AddMember("date", uDate, allocator);
		root.AddMember("total_profit", total_profit, allocator);
		root.AddMember("dynprofit", dynprofit, allocator);
		root.AddMember("dynbalance", dynbalance, allocator);
		root.AddMember("total_fee", total_fee, allocator);

		rj::StringBuffer sb;
		rj::PrettyWriter<rj::StringBuffer> writer(sb);
		root.Accept(writer);

		output = sb.GetString();
	}

	if (_publisher)
		_publisher(_mq_sid, topic, (const char*)output.c_str(), output.size());
}
```