# QuoteFactory模块(中泰XTP)

source: `{{ page.path }}`

**说明：行情接口为中泰证券的XTP，该接口为市场常用，WT已经实现了它的parser，开发环境为WIndows。**

## 执行程序配置

按如下的配置说明进行配置即可以使用QuoteFactory.exe来接收行情，作者建议早上9点15分启动，下午4点半关闭。
首先是配置文件，以下的文件要放在QuoteFactory可执行文件的同目录下，具体包括dtcfg.yaml、mdparsers.yaml、logcfgdt.yaml和statemonitor.yaml，这些配置文件都可以从WT主目录下的dist/QuoteFactory文件夹下复制到，其中要修改的是dtcfg.yaml和mdparsers.yaml两个文件，具体如下，配置项如果没有特别说明的则可以直接复制粘贴。

### **dtcfg.yaml**

```yaml
basefiles:
    # 这下面对应的四个文件要在../common/下面放好，哪个不需要放目前还没确定过，但holidays.json和stk_sessions.json是确定需要的，stk_sessions.json文件缺失的话会导致服务没法保存行情数据，这些文件可以从wtpy的dev分支下的demos/commons找到。
    commodity: ../common/stk_comms.json
    contract: ../common/stocks.json
    holiday: ../common/holidays.json
    session: ../common/stk_sessions.json
# 顶点广播配置，所以本机WTRunner接收数据的时候可以采用ParserUDP通过UDP获取数据，但win11调试的时候会提示权限不足（调试的时候可以换成其他Parser），但exe运行的时候可以用管理员运行解决该问题
broadcaster:
    active: true
    bport: 3997
    broadcast:
    -   host: 255.255.255.255
        port: 9001
        type: 2
    multicast_:
    -   host: 224.169.169.169
        port: 9002
        sendport: 8997
        type: 0
    -   host: 224.169.169.169
        port: 9003
        sendport: 8998
        type: 1
    -   host: 224.169.169.169
        port: 9004
        sendport: 8999
        type: 2
parsers: mdparsers.yaml
# statemonitor.yaml定义的历史数据的处理时间
statemonitor: statemonitor.yaml
writer:
    async: true
    groupsize: 20
    # 行情数据的保存目录
    path: ../STK_Data
    savelog: false
```

### ** mdparsers.yaml**

```yaml
parsers:
-   active: true
    buffsize: 128
    clientid: 1
    #你想要订阅的股票代码，不做code的配置则认为是订阅全市场代码    code: SSE.000001,SSE.600009,SSE.600036,SSE.600276,SZSE.000001
    hbinterval: 15
    host: xtp行情服务地址
    id: parser
    module: ParserXTP
    pass: 你的xtp密码
    port: xtp行情服务端口
    protocol: 1
    user: 你的xtp用户名
```

## 调试配置

1. 调试时候相应的配置文件要放在src/QuoteFactory的相应路径下面，src/QuoteFactory对应于QuoteFactory.exe的当前路径。
2. 把WtDataStorage.dll复制到src/QuoteFactory下面，还要把xtpquoteapi.dll和生成好的parserXTP.dll复制到src/QuoteFactory/parsers下面，如果使用最新生成的dll文件可以直接进入dll对应的源代码调试。
3. 然后你就可以在**Visual Studio**下对程序断点、单步...愉快地调试代码了。

## QuoteFactory main.cpp主要代码解析

```cpp
void initialize()
{
	WtHelper::set_module_dir(getBinDir());

    # 最新的配置文件只有dtcfg.yaml
	std::string filename("QFConfig.json");
	if (!StdFile::exists(filename.c_str()))
		filename = "QFConfig.yaml";
	if (!StdFile::exists(filename.c_str()))
		filename = "dtcfg.json";
	if (!StdFile::exists(filename.c_str()))
		filename = "dtcfg.yaml";

	WTSVariant* config = WTSCfgLoader::load_from_file(filename.c_str(), true);
	if(config == NULL)
	{
		WTSLogger::error_f("Loading config file {} failed", filename);
		return;
	}

	// 加载市场信息
	WTSVariant* cfgBF = config->get("basefiles");
	bool isUTF8 = cfgBF->getBoolean("utf-8");
	if (cfgBF->get("session"))
	{
		g_baseDataMgr.loadSessions(cfgBF->getCString("session"), isUTF8);
		WTSLogger::info("Trading sessions loaded");
	}

    // 如果是股票的话是加载市场和品种信息
	WTSVariant* cfgItem = cfgBF->get("commodity");
	if (cfgItem)
	{
		if (cfgItem->type() == WTSVariant::VT_String)
		{
			g_baseDataMgr.loadCommodities(cfgItem->asCString(), isUTF8);
		}
		else if (cfgItem->type() == WTSVariant::VT_Array)
		{
			for (uint32_t i = 0; i < cfgItem->size(); i++)
			{
				g_baseDataMgr.loadCommodities(cfgItem->get(i)->asCString(), isUTF8);
			}
		}
	}
    // 如果是股票的话是加载指数信息
	cfgItem = cfgBF->get("contract");
	if (cfgItem)
	{
		if (cfgItem->type() == WTSVariant::VT_String)
		{
			g_baseDataMgr.loadContracts(cfgItem->asCString(), isUTF8);
		}
		else if (cfgItem->type() == WTSVariant::VT_Array)
		{
			for (uint32_t i = 0; i < cfgItem->size(); i++)
			{
				g_baseDataMgr.loadContracts(cfgItem->get(i)->asCString(), isUTF8);
			}
		}
	}
    // 加载节假日信息
	if (cfgBF->get("holiday"))
	{
		g_baseDataMgr.loadHolidays(cfgBF->getCString("holiday"));
		WTSLogger::info("Holidays loaded");
	}

	//By Wesley @ 2021.12.27
	//datakit不需要主力映射规则
	//if (cfgBF->get("hot"))
	//{
	//	g_hotMgr.loadHots(cfgBF->getCString("hot"));
	//	WTSLogger::info("Hot rules loaded");
	//}

	//if (cfgBF->get("second"))
	//{
	//	g_hotMgr.loadSeconds(cfgBF->getCString("second"));
	//	WTSLogger::info("Second rules loaded");
	//}
    #初始化需要UDP广播的信息
	g_udpCaster.init(config->get("broadcaster"), &g_baseDataMgr, &g_dataMgr);

	//By Wesley @ 2021.12.27
	// 全天候模式，不需要再使用状态机
	bool bAlldayMode = config->getBoolean("allday");
	if (!bAlldayMode)
	{
		g_stateMon.initialize(config->getCString("statemonitor"), &g_baseDataMgr, &g_dataMgr);
	}
	else
	{
		WTSLogger::info("QuoteFactory will run in allday mode");
	}
	initDataMgr(config->get("writer"), bAlldayMode);

    // 初始化Parser
	WTSVariant* cfgParser = config->get("parsers");
	if (cfgParser)
	{
		if (cfgParser->type() == WTSVariant::VT_String)
		{
			const char* filename = cfgParser->asCString();
			if (StdFile::exists(filename))
			{
				WTSLogger::info_f("Reading parser config from {}...", filename);
				WTSVariant* var = WTSCfgLoader::load_from_file(filename, isUTF8);
				if (var)
				{
                    // 实际上是通过一个叫ParserAdapter的适配器去管理Parser
					initParsers(var->get("parsers"));
					var->release();
				}
				else
				{
					WTSLogger::error_f("Loading parser config {} failed", filename);
				}
			}
			else
			{
				WTSLogger::error_f("Parser configuration {} not exists", filename);
			}
		}
		else if (cfgParser->type() == WTSVariant::VT_Array)
		{
			initParsers(cfgParser);
		}
	}

	config->release();

    // 这里就是下文main提到的需要异步执行的任务
	g_asyncIO.post([bAlldayMode](){
        // ParserAdaper运行会让它适配的Parser去登录数据源和接收数据
		g_parsers.run();

		//全天候模式，不启动状态机
		if(!bAlldayMode)
		{
			std::this_thread::sleep_for(std::chrono::milliseconds(5));
            // 状态机起来可以监控交易时间和历史数据处理时间等
			g_stateMon.run();
		}
	});
}

int main()
{ 
    // 获取日志配置文件名称
	std::string filename = "logcfgdt.json";
	if (!StdFile::exists(filename.c_str()))
		filename = "logcfgdt.yaml";
    // 根据配置文件名称来初始化日志对象
	WTSLogger::init(filename.c_str());

#ifdef _MSC_VER
	_CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_DEBUG);
	_CrtSetReportMode(_CRT_ERROR, _CRTDBG_MODE_DEBUG);
	_CrtSetReportMode(_CRT_ASSERT, _CRTDBG_MODE_DEBUG);

	_set_error_mode(_OUT_TO_STDERR);
	_set_abort_behavior(0, _WRITE_ABORT_MSG);

	g_dwMainThreadId = GetCurrentThreadId();
	SetConsoleCtrlHandler(ConsoleCtrlhandler, TRUE);

	CMiniDumper::Enable("QuoteFactory.exe", true);
#endif

#if _WIN32
#pragma message("Signal hooks disabled in WIN32")
#else
#pragma message("Signal hooks enabled in UNIX")
	install_signal_hooks([](const char* message) {
		WTSLogger::error(message);
	});
#endif
    // 初始化QF
	initialize();

    // 通过一个asio::io_service::work对象来守护io_service。这样，即使所有io任务都执行完成，也不会退出，继续等待新的io任务。
	boost::asio::io_service::work work(g_asyncIO);
    // 异步执行g_asyncIO，initialize()函数里面的g_asyncIO.post的lambda函数就是要异步执行的任务
	g_asyncIO.run();
}
```