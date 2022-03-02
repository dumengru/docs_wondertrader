# CTA回测详解

source: `{{ page.path }}`

## 准备工作

### 需要先完成`生成解决方案`

### 1. 配置文件

- logcfgbt.yaml 
- configbt.yaml
- storage数据目录
- common配置目录
  - sessions.json"
  - commodities.json"
  - contracts.json"
  - holidays.json"
  - hots.json"

configbt.yaml 样式

```yaml
replayer:                   # 回放器配置，包块数据格式、存放路径、交易费用等
    mode: csv               # 获取数据的模式，如csv，bin等
    path: ./storage/        # 数据存放的路径
    stime: 201905010900     # 回测开始时间
    etime: 201910011500     # 回测结束时间
    basefiles:              # common下的配置文件路径，(ToDo:每个文件的意义单独展开介绍)   
        session: ./common/sessions.json
        commodity: ./common/commodities.json
        contract: ./common/contracts.json
        holiday: ./common/holidays.json
        hot: ./common/hots.json
    fees: ./common/fees.json
env:                # 环境设置，与交易品种和撮合有关
    mocker: cta     # 模式
    slippage: 1     # 滑点
cta:                # 策略工厂配置
    module: WtCtaStraFact.dll       # 策略生成器模块名
    strategy:                       # 策略配置
        id: dt_if                   # 策略id
        name: DualThrust            # 策略名
        params:                     # 策略参数
            code: CFFEX.IF.HOT
            count: 50
            period: m5
            days: 30
            k1: 0.6
            k2: 0.6
            stock: false
```

logcfg.yaml 样式

```yaml
root:                   # 根日志对象的配置，通常是框架信息
    level: debug        # 日志等级
    async: false        #是否异步
    sinks:              # 输出流的配置，即日志信息会被输出到哪些位置
    -   type: basic_file_sink                           # 类型，基本的文件输出
        filename: BtLogs/Runner.log                     # 日志文件保存位置
        pattern: '[%Y.%m.%d %H: %M: %S - %-5l] %v'      # 日志输出模板
        truncate: true                                  # 日志是否自动截断，截断可以避免日志文件过大
        type: console_sink                              # 输出到控制台
        pattern: '[%m.%d %H: %M: %S - %^%-5l%$] %v'
dyn_pattern: 
    strategy: 
        level: debug
        async: false
        sinks: 
        -   type: basic_file_sink
            filename: BtLogs/Strategy_%s.log
            pattern: '[%Y.%m.%d %H: %M: %S - %-5l] %v'
            truncate: false
```

### 2. 数据文件

- storage目录
  - csv/CFFEX.IF.HOT_m5.csv

### 3. 策略工程(dll文件，需要完成WtCtaStraFact工程的编译)

- WtCtaStraFact.dll

### 文件结构

以X64 Debug编译为例，需要添加的配置文件的文件结构如下

- src
  - x64/x86
    - Debug/Release
      - WtBtRunner
        - common
        - storage
        - logcfg.json
        - config.json
        - WtCtaStraFact.dll

### 编译运行程序（WtBtRunner工程）

将WtBtRunner工程设置为启动项目，并将项目属性-配置属性-调试-工作目录设置为$(OutDir)

![project_setting.png](../../assets/images/wt/hej_01_project_setting.png)

WtBtRunner.cpp

```cpp
int main()
{
    // 1. 加载 logcfgbt.yaml
    std::string filename = "logcfgbt.json";
	if (!StdFile::exists(filename.c_str()))
		filename = "logcfgbt.yaml";
	WTSLogger::init(filename.c_str());

    // 2. 加载 configbt.yaml 转为 WTSVariant 类型
	filename = "configbt.json";
	if(!StdFile::exists(filename.c_str()))
		filename = "configbt.yaml";

	WTSVariant* cfg = WTSCfgLoader::load_from_file(filename.c_str(), true);
	if (cfg == NULL)
	{
		WTSLogger::info_f("Loading configuration file {} failed", filename);
		return -1;
	}

    // 3. 初始化历史数据回放器
    HisDataReplayer replayer;
    replayer.init(cfg->get("replayer"));

    // 4. 创建cta调度器对象
    WTSVariant* cfgEnv = cfg->get("env");
    const char* mode = cfgEnv->getCString("mocker");
    int32_t slippage = cfgEnv->getInt32("slippage");
    if (strcmp(mode, "cta") == 0)
    {
        CtaMocker* mocker = new CtaMocker(&replayer, "cta", slippage);

        // 5. 利用调度器初始化cta策略工厂
        mocker->init_cta_factory(cfg->get("cta"));
        // 6. 再回放器中注册cta调度器接口
        replayer.register_sink(mocker, "cta");
    }

    // 7. 回放器准备回放历史数据
    replayer.prepare();

    // 8. 回放器回放历史数据
    replayer.run();

    printf("press enter key to exit\r\n");
    getchar();

    // 9. 停止输出日志
    WTSLogger::stop();
}
```

## 逐步解析

### 1. 加载 logcfgbt.yaml

```cpp
// 加载 logcfgbt.yaml 文件初始化日志对象
void WTSLogger::init(const char* propFile /* = "logcfg.json" */, bool isFile /* = true */, ILogHandler* handler /* = NULL */, WTSLogLevel logLevel /* = LL_INFO */)
{
	if (m_bInited)
		return;

	// 判断文件是否存在
	if (isFile && !StdFile::exists(propFile))
		return;
	// 从文件/缓存中加载数据
	WTSVariant* cfg = isFile ? WTSCfgLoader::load_from_file(propFile, true) : WTSCfgLoader::load_from_content(propFile, false, true);
	if (cfg == NULL)
		return;

	// 获取一级节点键名并遍历
	auto keys = cfg->memberNames();
	for (std::string& key : keys)
	{
		WTSVariant* cfgItem = cfg->get(key.c_str());
		if (key == DYN_PATTERN)		// "dyn_pattern"
		{
			auto pkeys = cfgItem->memberNames();
			for(std::string& pkey : pkeys)
			{
				WTSVariant* cfgPattern = cfgItem->get(pkey.c_str());
				if (m_mapPatterns == NULL)
					m_mapPatterns = LogPatterns::create();

				m_mapPatterns->add(pkey.c_str(), cfgPattern, true);
			}
			continue;
		}
		// 根据配置文件初始化对应的日志对象
		initLogger(key.c_str(), cfgItem);
	}
	// 日志设置
	m_rootLogger = getLogger("root");
	spdlog::set_default_logger(m_rootLogger);		// 设置默认logger
	spdlog::flush_every(std::chrono::seconds(2));	// 日志刷新频率: 2s

	m_logHandler = handler;		// 日志句柄
	m_logLevel = logLevel;		// 日志级别

	m_bInited = true;			// 日志对象初始化完成标志
}
```

```note
5. m_mapPatterns 是一个map容器指针, 可以添加多个日志对象, 用来记录策略日志.(背后逻辑暂时不懂, 不重要)
```

### 2. 加载 configbt.yaml 转为 WTSVariant 类型

```cpp
WTSVariant* WTSCfgLoader::load_from_file(const char* filename, bool isUTF8 /* = true */)
{
	if (!StdFile::exists(filename))
		return NULL;

	std::string content;
	StdFile::read_file_content(filename, content);
	if (content.empty())
		return NULL;

	//By Wesley @ 2022.01.07
	//Linux下得是UTF8
	//Win下得是GBK
#ifdef _WIN32
	if(isUTF8)
		content = UTF8toChar(content);
#endif
	// 通过文件名判断读取 json 或 ymal 格式文件
	if (StrUtil::endsWith(filename, ".json"))
		return load_from_json(content.c_str());
	else if (StrUtil::endsWith(filename, ".yaml") || StrUtil::endsWith(filename, ".yml"))
		return load_from_yaml(content.c_str());

	return NULL;
}
```

### 3. 初始化历史数据回放器

```cpp

bool HisDataReplayer::init(WTSVariant* cfg, EventNotifier* notifier /* = NULL */, IBtDataLoader* dataLoader /* = NULL */)
{
	_notifier = notifier;				// 消息通知对象
	_bt_loader = dataLoader;			// 数据加载器对象

	_mode = cfg->getCString("mode");	// 数据模式
	/*
	 *	By Wesley @ 2022.01.11
	 *	因为store可能会变复杂，所以这里做一个兼容处理
	 *	如果有store就读取store的path，如果没有store，就还读取root的path
	 */
	if (cfg->has("store"))
	{
		_base_dir = StrUtil::standardisePath(cfg->get("store")->getCString("path"));	
	}
	else
	{
		_base_dir = StrUtil::standardisePath(cfg->getCString("path"));
	}

	if(_mode == "storage" || _mode == "bin")
	{
		if (cfg->has("store"))
		{
			_his_dt_mgr.init(cfg->get("store"));
		}
		else
		{
			WTSVariant* item = WTSVariant::createObject();
			item->append("path", _base_dir.c_str());
			_his_dt_mgr.init(item);
			item->release();
		}
	}
	
	bool isRangeCfg = (_begin_time == 0 || _end_time == 0);	//是否从配置文件读取回测区间
	if(_begin_time == 0)
		_begin_time = cfg->getUInt64("stime");

	if(_end_time == 0)
		_end_time = cfg->getUInt64("etime");

	WTSLogger::info_f("Backtest time range is set to be [{},{}] via config", _begin_time, _end_time);

	_tick_enabled = cfg->getBoolean("tick");	// 是否回放tick数据
	WTSLogger::info_f("Tick data replaying is {}", _tick_enabled ? "enabled" : "disabled");

	// 基础数据文件
	WTSVariant* cfgBF = cfg->get("basefiles");
	// 基础配置文件的编码，这样可以兼容原来的配置
	bool isUTF8 = cfgBF->getBoolean("utf-8");
	// 加载交易时段配置文件
	if (cfgBF->get("session"))
		_bd_mgr.loadSessions(cfgBF->getCString("session"), isUTF8);
	// 加载品种信息配置文件
	WTSVariant* cfgItem = cfgBF->get("commodity");
	if (cfgItem)
	{
		if (cfgItem->type() == WTSVariant::VT_String)
		{
			_bd_mgr.loadCommodities(cfgItem->asCString(), isUTF8);
		}
		else if (cfgItem->type() == WTSVariant::VT_Array)
		{
			for(uint32_t i = 0; i < cfgItem->size(); i ++)
			{
				_bd_mgr.loadCommodities(cfgItem->get(i)->asCString(), isUTF8);
			}
		}
	}
	// 加载合约信息配置文件
	cfgItem = cfgBF->get("contract");
	if (cfgItem)
	{
		if (cfgItem->type() == WTSVariant::VT_String)
		{
			_bd_mgr.loadContracts(cfgItem->asCString(), isUTF8);
		}
		else if (cfgItem->type() == WTSVariant::VT_Array)
		{
			for (uint32_t i = 0; i < cfgItem->size(); i++)
			{
				_bd_mgr.loadContracts(cfgItem->get(i)->asCString(), isUTF8);
			}
		}
	}
	// 加载节日信息配置文件
	if (cfgBF->get("holiday"))
		_bd_mgr.loadHolidays(cfgBF->getCString("holiday"));
	// 加载主力合约切换配置文件
	if (cfgBF->get("hot"))
		_hot_mgr.loadHots(cfgBF->getCString("hot"));
	// 加载次主力合约切换配置文件(无)
	if (cfgBF->get("second"))
		_hot_mgr.loadSeconds(cfgBF->getCString("second"));
	// 加载手续费配置文件
	loadFees(cfg->getCString("fees"));

	/*
	 *	By Wesley @ 2021.12.20
	 *	先从extloader加载除权因子
	 *	如果加载失败，并且配置了除权因子文件，再加载除权因子文件
	 */
	bool bLoaded = loadStkAdjFactorsFromLoader();

	if (!bLoaded && cfg->has("adjfactor"))
		loadStkAdjFactorsFromFile(cfg->getCString("adjfactor"));

	return true;
}
```

### 4. 创建cta调度器对象

1. 通过配置文件 "env", 确定加载 cta 调度器
2. cta 调度器初始化时传入了调度器中的回放器对象 _replayer

### 5. 利用调度器初始化cta策略工厂

```cpp
// 根据配置文件初始化cta策略工厂
bool CtaMocker::init_cta_factory(WTSVariant* cfg)
{
	if (cfg == NULL)
		return false;

	const char* module = cfg->getCString("module");		// 文件名
	// 1. 加载策略工厂dll文件: WtCtaStraFact.dll
	DllHandle hInst = DLLHelper::load_library(module);	// 加载dll文件
	if (hInst == NULL)
		return false;
	// 2. 获取dll文件中创建策略的方法: createStrategyFact
	FuncCreateStraFact creator = (FuncCreateStraFact)DLLHelper::get_symbol(hInst, "createStrategyFact");
	if (creator == NULL)
	{
		DLLHelper::free_library(hInst);
		return false;
	}
	// 3. 填充自定义的工厂结构体对象 _factory
	_factory._module_inst = hInst;
	_factory._module_path = module;
	_factory._creator = creator;
	_factory._remover = (FuncDeleteStraFact)DLLHelper::get_symbol(hInst, "deleteStrategyFact");
	_factory._fact = _factory._creator();

	WTSVariant* cfgStra = cfg->get("strategy");
	if (cfgStra)
	{
		// 4. 利用策略工创建策略对象, 并传递给调度器中的策略对象: _strategy
		_strategy = _factory._fact->createStrategy(cfgStra->getCString("name"), cfgStra->getCString("id"));
		if(_strategy)
		{
			WTSLogger::info("Strategy %s.%s is created,strategy ID: %s", _factory._fact->getName(), _strategy->getName(), _strategy->id());
		}
		// 5. 利用策略文件对策略进行初始化
		_strategy->init(cfgStra->get("params"));
		_name = _strategy->id();	// 调度器中的策略ID
	}
	return true;
}
```

### 6. 在回放器中注册cta调度器接口

```cpp
inline void register_sink(IDataSink* listener, const char* sinkName) 
{
    _listener = listener;         // 初始化完毕的调度器
    _stra_name = sinkName;        // 回放器中的策略工厂名称"cta"
}
```

```note
再来梳理一下逻辑: 
1. 初始化回放器: `HisDataReplayer replayer`;
2. 将`replayer`传入`mocker`初始化调度器: `CtaMocker* mocker = new CtaMocker(&replayer, "cta", slippage);`
3. 将初始化完毕的调度器回传给回放器: `replayer.register_sink(mocker, "cta");`

总结: 调度器中的 `_replayer` 是回放器, 回放器中 `_listener` 是调度器
```

### 7. 回放器准备回放历史数据

```cpp
bool HisDataReplayer::prepare()
{
	if (_running)
	{
		WTSLogger::log_raw(LL_ERROR, "Cannot run more than one backtesting task at the same time");
		return false;
	}

	_running = true;
	_terminated = false;
	reset();	// 数据全部初始化
	// 1. begin_time是整数型日期(精确到分, 在配置文件中)如: 201905010900
	_cur_date = (uint32_t)(_begin_time / 10000);
	_cur_time = (uint32_t)(_begin_time % 10000);
	_cur_secs = 0;
	// 2. 计算时间偏移后的交易日(过滤节假日)
	_cur_tdate = _bd_mgr.calcTradingDate(DEFAULT_SESSIONID, _cur_date, _cur_time, true);
	// 3. 通知事件: 回测开始
	if (_notifier)
		_notifier->notifyEvent("BT_START");
	// 4. 调度器中的回调函数
	_listener->handle_init();
	// 5. 如果没有tick数据就加载Bar数据
	if (!_tick_enabled)
		checkUnbars();

	return true;
}
```

```note
1. WTCPP中的日期和时间常用整数型, 如整数 20220211 表示 2022年2月11日, 整数 0910 表示 9:10, 在阅读源码过程中需要注意这种日期时间格式的**精度**

2. 逻辑比较复杂, 可查看源码注释(搜索框输入: WTSBaseDataMgr::calcTradingDate, 如果注释有误, 欢迎QQ群提出)

4. 调度器中的 handle_init 会调用 `this->on_init();`, 而 on_init() 又调用 `_strategy->on_init(this);`(注意: 这里又把调度器对象传递到策略中去了, 所以记得策略文件中的参数 CtaMocker 是怎么来的了吧。。。山路十八弯~)

5. 数据加载逻辑重要但是复杂(见下一部分内容)
```

### 8. 回放器回放历史数据

```cpp
void HisDataReplayer::run(bool bNeedDump/* = false*/)
{
    // 1. 如果没有时间调度任务, 则采用主K线回放的模式
    if(_task == NULL)
    {
        // 2. 如果没有确定主K线(多周期), 则确定一个周期最短的主K线
        if (_main_key.empty() && !_bars_cache.empty())
        {
            // 2.1 假设回放的K线的周期是D1(日线数据)
            WTSKlinePeriod minPeriod = KP_DAY;
            uint32_t minTimes = 1;
            // 2.2 遍历缓存中的K线数据
            for(auto& m : _bars_cache)
            {
                // 2.3 获取自定义的K线列表结构体
                const BarsList& barList = m.second;
                // 2.4 当前周期小于D1则更新minPeriod和minTimes
                if (barList._period < minPeriod)
                {
                    minPeriod = barList._period;
                    minTimes = barList._times;
                    _main_key = m.first;
                }
                // 2.5 否则主K周期保持不变
                else if(barList._period == minPeriod)
                {
                    if(barList._times < minTimes)
                    {
                        _main_key = m.first;
                        minTimes = barList._times;
                    }
                }
            }

            WTSLogger::info("Main K bars automatic determined: %s", _main_key.c_str());
        }

        // 3. 确定主K后按照主K回放
        if(!_main_key.empty())
        {
            // 如果订阅了K线，则按照主K线进行回放
            run_by_bars(bNeedDump);
        }
        // 4. 没有主K按照tick回放
        else if(_tick_enabled)
        {
            run_by_ticks(bNeedDump);
        }
        else
        {
            // WTSLogger::info("没有订阅主力K线且未开放tick回测, 回放直接退出");
            WTSLogger::info("Main K bars not subscribed and backtesting of tick data not available , replaying done");
            _listener->handle_replay_done();
            if (_notifier)
                _notifier->notifyEvent("BT_END");
        }
    }
    else //if(_task != NULL)
    {
        run_by_tasks(bNeedDump);
    }

    _running = false;
}
```

```note
1. 数据回放有两种模式: 时间调度和主K回放

2. 实际就是K线周期大小的比较过程, 最小的K线周期作为主K.(这里的逻辑可能有点小问题)

3. 数据回放逻辑重要且复杂(见下一部分内容)
```

### 9. 停止输出日志

```cpp
void WTSLogger::stop()
{
    m_bStopped = true;
    // 1. 释放 m_mmapPatterns 对象
    if (m_mapPatterns)
        m_mapPatterns->release();
    spdlog::shutdown();
}
```

## 历史数据回放器中重要的代码逻辑

### 加载Bars数据(checkUnbars)

**_tick_sub_map订阅路径**

checkUnbars函数第一句代码就是遍历 `_tick_sub_map`, 但是 `_tick_sub_map`从哪来?

数据回放器准备阶段 `bool HisDataReplayer::prepare()` 中有两句代

```cpp
reset()        // 初始化变量
...
// 4. 调度器中的回调函数
_listener->handle_init();
```

1. `reset()`之后 `_tick_sub_map` 首先被初始化
2. `_listener->handle_init();`函数调用路径如下:
    1. `_listener->handle_init();`
    2. `this->on_init();`
    3. `_strategy->on_init(this);`
3. 到这里已经进入策略文件中了, 具体代码如下

```cpp
void WtStraDualThrust::on_init(ICtaStraCtx* ctx)
{
    std::string code = _code;
    if (_isstk)
        code += "Q";
    WTSKlineSlice *kline = ctx->stra_get_bars(code.c_str(), _period.c_str(), _count, true);
    if (kline == NULL)
    {
        //这里可以输出一些日志
        return;
    }
    kline->release();
}
```

1. 这里又回调了调度器函数 `ctx->stra_get_bars;`, 具体内容如下

```cpp
WTSKlineSlice* CtaMocker::stra_get_bars(const char* stdCode, const char* period, uint32_t count, bool isMain /* = false */)
{
    std::string key = StrUtil::printf("%s#%s", stdCode, period);    // config.json文件中的"code":"CFFEX.IF.HOT", "period": "m5"
    std::string basePeriod = "";                    // 记录 m, 即分钟级别回测
    uint32_t times = 1;
    if (strlen(period) > 1)
    {
        basePeriod.append(period, 1);
        times = strtoul(period + 1, NULL, 10);        // 提取 5, 即5分钟
    }
    else
    {
        basePeriod = period;
        key.append("1");
    }
    // 确定主K
    if (isMain)
    {
        if (_main_key.empty())
            _main_key = key;
        else if (_main_key != key)
            throw std::runtime_error("Main k bars can only be setup once");
    }
    // 调用回放器获取K线切片
    WTSKlineSlice* kline = _replayer->get_kline_slice(stdCode, basePeriod.c_str(), count, times, isMain);

    bool bFirst = (_kline_tags.find(key) == _kline_tags.end());
    KlineTag& tag = _kline_tags[key];
    tag._closed = false;

    if (kline)
    {
        // 提取标准合约代码
        CodeHelper::CodeInfo cInfo = CodeHelper::extractStdCode(stdCode);

        // 如果是股票 ...
        std::string realCode = stdCode;
        if(cInfo._category == CC_Stock && cInfo.isExright())
            realCode = StrUtil::printf("%s.%s.%s", cInfo._exchg, cInfo._product, cInfo._code);
        
        // 订阅tick数据
        _replayer->sub_tick(id(), realCode.c_str());
    }
    return kline;
}
```

1. 最后一句`_replayer->sub_tick`又回调了回放器函数, 具体如下

```cpp
void HisDataReplayer::sub_tick(uint32_t sid, const char* stdCode)
{
    if (strlen(stdCode) == 0)
        return;
    // 订阅tick行情
    SIDSet& sids = _tick_sub_map[stdCode];
    sids.insert(sid);
}
```

```tip
整个逻辑在回放器 HisDataReplayer, 调度器 CtaMocker 和策略对象 WtStraDualThrust 中来回切换, 感兴趣建议自己打断点细细品味.
```

**_bars_cache加载路径**

还是 `_tick_sub_map` 订阅的那条路径, 不过从第4步开始分叉, 在 `WTSKlineSlice* kline = _replayer->get_kline_slice`中进入到了回放器获取K线切片. 这里在加载数据的过程中填充了 `bars_cache`.

**数据加载逻辑**

```cpp
void HisDataReplayer::checkUnbars()
{
    // 1. 遍历Tick数据订阅表
    for(auto& m : _tick_sub_map)
    {
        const char* stdCode = m.first.c_str();
        bool bHasBars = false;
        // 2. 遍历未订阅的K线缓存(空)
        for(auto& m : _unbars_cache)
        {
            const std::string& key = m.first;
            auto ay = StrUtil::split(key, "#");
            if (ay[0].compare(stdCode) == 0)
            {
                bHasBars = true;
                break;
            }
        }
        if(bHasBars)
            continue;

        // 3. 遍历K线缓存
        for (auto& m : _bars_cache)
        {
            const std::string& key = m.first;
            auto ay = StrUtil::split(key, "#");
            if (ay[0].compare(stdCode) == 0)
            {
                bHasBars = true;
                break;
            }
        }
        
        if (bHasBars)
            continue;

        //如果订阅了tick,但是没有对应的K线数据,则自动加载1分钟线到内存中
        bool bHasHisData = false;
        std::string key = StrUtil::printf("%s#m#1", stdCode);

        /*
         *    By Wesley @ 2021.12.20
         *    先从extloader加载最终的K线数据
         *    如果加载不到，再从配置的历史数据存储引擎加载数据
         */
        if (NULL != _bt_loader)
        {
            bHasHisData = cacheFinalBarsFromLoader(key, stdCode, KP_Minute1, false);
        }

        if(!bHasHisData)
        {
            if (_mode == "csv")
            {
                bHasHisData = cacheRawBarsFromCSV(key, stdCode, KP_Minute1, false);
            }
            else
            {
                bHasHisData = cacheRawBarsFromBin(key, stdCode, KP_Minute1, false);
            }
        }
        
        if (!bHasHisData)
            continue;

        WTSSessionInfo* sInfo = get_session_info(stdCode, true);

        BarsList& kBlkPair = _unbars_cache[key];
        
        //还没有经过初始定位
        WTSBarStruct bar;
        bar.date = _cur_tdate;
        bar.time = (_cur_date - 19900000) * 10000 + _cur_time;

        auto it = std::lower_bound(kBlkPair._bars.begin(), kBlkPair._bars.end(), bar, [](const WTSBarStruct& a, const WTSBarStruct& b) {
            return a.time < b.time;
        });

        uint32_t eIdx = it - kBlkPair._bars.begin();

        if (it != kBlkPair._bars.end())
        {
            WTSBarStruct& curBar = *it;
            if (curBar.time > bar.time)
            {
                if (eIdx > 0)
                {
                    it--;
                    eIdx--;
                }
            }
            kBlkPair._cursor = eIdx + 1;
        }
    }
}
```

### 主K回放(run_by_bars)

在replayer.run()中，如果订阅了主K线，则按照主K线进行回放，执行run_by_bars。

run_by_bars位于HisDataReplayer.cpp中，位于WtBtCore工程下，下面将逐行解析run_by_bars的回测逻辑。

run_by_bars

```cpp
void HisDataReplayer::run_by_bars(bool bNeedDump /* = false */)
{
    // 获取当前纳秒级别的时间戳
    int64_t now = TimeUtils::getLocalTimeNano();
    
    // 从缓存中加载主K的数据
    BarsList& barList = _bars_cache[_main_key];
    // 获取交易时间信息
    WTSSessionInfo* sInfo = get_session_info(barList._code.c_str(), true);
    // 解析品种的代码，如CFFEX.IC.HOT -> CFFEX.IC
    std::string commId = CodeHelper::stdCodeToStdCommID(barList._code.c_str());

    // 计算开始和结束bar的索引号
    uint32_t sIdx = locate_barindex(_main_key, _begin_time, false);
    uint32_t eIdx = locate_barindex(_main_key, _end_time, true);

    // 获取全部bar的数目
    uint32_t total_barcnt = eIdx - sIdx + 1;
    // 初始化已回测过的bar数
    uint32_t replayed_barcnt = 0;
    // 输出回测概况
    notify_state(barList._code.c_str(), barList._period, barList._times, _begin_time, _end_time, 0);

    // 本地记录回测概况
    if (bNeedDump)
        dump_btstate(barList._code.c_str(), barList._period, barList._times, _begin_time, _end_time, 100.0, TimeUtils::getLocalTimeNano() - now);

    WTSLogger::info_f("Start to replay back data from {}...", _begin_time);

    // 开始回测循环
    for (; !_terminated;)
    {
        // 判断bar的频率是否是日线，从而采取不同的处理方法
        bool isDay = barList._period == KP_DAY;
        if (barList._cursor != UINT_MAX)
        {
            // 计算下一个bar的时间
            uint64_t nextBarTime = 0;
            if (isDay)
                // 如果是日线数据，使用当前日期的收盘时间作为下一个bar的时间
                nextBarTime = (uint64_t)barList._bars[barList._cursor].date * 10000 + sInfo->getCloseTime();
            else
            {
                // 非日线时间，使用当前时间
                nextBarTime = (uint64_t)barList._bars[barList._cursor].time;
                // 由于bar的时间是自1990年为起点，所以要加上起点时间转换为当前时间
                nextBarTime += 199000000000;
            }

            // 检查下一个bar的时间是否超过回测的结束时间
            if (nextBarTime > _end_time)
            {
                WTSLogger::info_f("{} is beyond ending time {},replaying done", nextBarTime, _end_time);
                break;
            }

            // 计算日期和时间
            uint32_t nextDate = (uint32_t)(nextBarTime / 10000);
            uint32_t nextTime = (uint32_t)(nextBarTime % 10000);

            //By Wesley @ 2022.01.10
            //如果和收盘时间一样，进行这个判断
            //主要针对7*24小时的品种，其他的品种不需要
            uint32_t nextTDate = _opened_tdate;
            if(isDay || (!isDay && sInfo->offsetTime(nextTime, false) != sInfo->getCloseTime(true)))
            {
                nextTDate = _bd_mgr.calcTradingDate(commId.c_str(), nextDate, nextTime, false);
                // 判断是否到达了新的交易日
                if (_opened_tdate != nextTDate)
                {
                    WTSLogger::debug_f("Tradingday {} begins", nextTDate);
                    _listener->handle_session_begin(nextTDate);
                    _opened_tdate = nextTDate;
                    _cur_tdate = nextTDate;
                }
            }

            uint64_t curBarTime = (uint64_t)_cur_date * 10000 + _cur_time;
            if (_tick_enabled)
            {
                //如果开启了tick回放,则直接回放tick数据
                //如果tick回放失败，说明tick数据不存在，则需要模拟tick
                _tick_simulated = !replayHftDatas(curBarTime, nextBarTime);
            }

            _cur_date = nextDate;
            _cur_time = nextTime;
            _cur_secs = 0;
            
            // 判断当前交易日是否结束
            bool isEndTDate = (sInfo->offsetTime(_cur_time, false) >= sInfo->getCloseTime(true));

            if (!_tick_enabled)
            {
                // 检查是否未订阅数据，如果未订阅，则会自动订阅一个1分钟bar作为unbar来进行撮合
                checkUnbars();
                // 使用unbar数据进行回放撮合
                replayUnbars(curBarTime, nextBarTime, (isDay || isEndTDate) ? nextTDate : 0);
            }

            // 检查K线闭合
            onMinuteEnd(nextDate, nextTime, (isDay || isEndTDate) ? nextTDate : 0, _tick_simulated);

            replayed_barcnt += 1;

            // 若交易日结束，则触发交易日结束事件
            if (isEndTDate && _closed_tdate != _cur_tdate)
            {
                WTSLogger::debug_f("Tradingday {} ends", _cur_tdate);
                _listener->handle_session_end(_cur_tdate);
                _closed_tdate = _cur_tdate;
                _day_cache.clear();
            }

            // 记录回测状态
            notify_state(barList._code.c_str(), barList._period, barList._times, _begin_time, _end_time, replayed_barcnt*100.0 / total_barcnt);
            
            // 所有数据都回测完成，结束回测
            if (barList._cursor >= barList._bars.size())
            {
                WTSLogger::log_raw(LL_INFO, "All back data replayed, replaying done");
                break;
            }
        }
        else
        {
            WTSLogger::log_raw(LL_ERROR, "No back data initialized, replaying canceled");
            break;
        }
    }

    if (_terminated)
        WTSLogger::debug("Replaying by bars terminated forcely");

    // 回测结束
    notify_state(barList._code.c_str(), barList._period, barList._times, _begin_time, _end_time, 100);
    if (_notifier)
        _notifier->notifyEvent("BT_END");

    // 最后检查是否触发交易日结束事件
    if (_closed_tdate != _cur_tdate)
    {
        WTSLogger::debug_f("Tradingday {} ends", _cur_tdate);
        _listener->handle_session_end(_cur_tdate);
    }

     // 检查是否需要本地化
    if (bNeedDump)
    {
        dump_btstate(barList._code.c_str(), barList._period, barList._times, _begin_time, _end_time, 100.0, TimeUtils::getLocalTimeNano() - now);
    }

    // 触发回测结束事件
    _listener->handle_replay_done();
}
```

### Tick回放(run_by_ticks)

如果没有订阅K线，且tick回测是打开的，则按照每日的tick进行回放

run_by_ticks位于HisDataReplayer.cpp中，位于WtBtCore工程下，run_by_ticks较run_by_bars简单，下面将逐行解析run_by_ticks的回测逻辑。

run_by_ticks

```cpp

void HisDataReplayer::run_by_ticks(bool bNeedDump /* = false */)
{
    //如果没有订阅K线，且tick回测是打开的，则按照每日的tick进行回放
    // 计算结束的日期和时间
    uint32_t edt = (uint32_t)(_end_time / 10000);
    uint32_t etime = (uint32_t)(_end_time % 10000);
    // 计算当前交易日
    uint64_t end_tdate = _bd_mgr.calcTradingDate(DEFAULT_SESSIONID, edt, etime, true);

    // 开始回测循环
    while (_cur_tdate <= end_tdate && !_terminated)
    {
        // 加载当前交易日的所有tick数据
        if (checkAllTicks(_cur_tdate))
        {
            WTSLogger::info_f("Start to replay tick data of {}...", _cur_tdate);
            // 触发交易日开始事件
            _listener->handle_session_begin(_cur_tdate);
            // 回放该交易日的tick
            replayHftDatasByDay(_cur_tdate);
            // 触发交易日结束事件
            _listener->handle_session_end(_cur_tdate);
        }
        
        // 获取下一个交易日
        _cur_tdate = TimeUtils::getNextDate(_cur_tdate);
    }

    // 回测结束
    if (_terminated)
        WTSLogger::debug("Replaying by ticks terminated forcely");

    WTSLogger::log_raw(LL_INFO, "All back data replayed, replaying done");
    _listener->handle_replay_done();
    if (_notifier)
        _notifier->notifyEvent("BT_END");
}
```

### 时间调度(run_by_tasks)

时间调度任务不为空,则按照时间调度任务回放

run_by_tasks位于HisDataReplayer.cpp中，位于WtBtCore工程下，下面将逐行解析run_by_tasks的回测逻辑。

run_by_tasks

```cpp
void HisDataReplayer::run_by_tasks(bool bNeedDump /* = false */)
{
    //时间调度任务不为空,则按照时间调度任务回放
    WTSSessionInfo* sInfo = NULL;
    const char* DEF_SESS = (strlen(_task->_session) == 0) ? DEFAULT_SESSIONID : _task->_session;
    sInfo = _bd_mgr.getSession(DEF_SESS);
    WTSLogger::info_f("Start to backtest with task frequency from {}...", _begin_time);

    //分钟即任务和日级别任务分开写
    if (_task->_period != TPT_Minute)
    {
        uint32_t endtime = TimeUtils::getNextMinute(_task->_time, -1);
        bool bIsPreDay = endtime > _task->_time;
        if (bIsPreDay)
            _cur_date = TimeUtils::getNextDate(_cur_date, -1);

        for (; !_terminated;)
        {
            bool fired = false;
            //获取上一个交易日的日期
            uint32_t preTDate = TimeUtils::getNextDate(_cur_tdate, -1);
            if (_cur_time == endtime)
            {
                if (!_bd_mgr.isHoliday(_task->_trdtpl, _cur_date, true))
                {
                    uint32_t weekDay = TimeUtils::getWeekDay(_cur_date);


                    bool bHasHoliday = false;
                    uint32_t days = 1;
                    while (_bd_mgr.isHoliday(_task->_trdtpl, preTDate, true))
                    {
                        bHasHoliday = true;
                        preTDate = TimeUtils::getNextDate(preTDate, -1);
                        days++;
                    }
                    uint32_t preWD = TimeUtils::getWeekDay(preTDate);

                    switch (_task->_period)
                    {
                    case TPT_Daily:
                        fired = true;
                        break;
                    case TPT_Minute:
                        break;
                    case TPT_Monthly:
                        //if (preTDate % 1000000 < _task->_day && _cur_date % 1000000 >= _task->_day)
                        //	fired = true;
                        if (_cur_date % 1000000 == _task->_day)
                            fired = true;
                        else if (bHasHoliday)
                        {
                            //上一个交易日在上个月,且当前日期大于触发日期
                            //说明这个月的开始日期在节假日内,顺延到今天
                            if ((preTDate % 10000 / 100 < _cur_date % 10000 / 100) && _cur_date % 1000000 > _task->_day)
                            {
                                fired = true;
                            }
                            else if (preTDate % 1000000 < _task->_day && _cur_date % 1000000 > _task->_day)
                            {
                                //上一个交易日在同一个月,且小于触发日期,但是今天大于触发日期,说明正确触发日期到节假日内,顺延到今天
                                fired = true;
                            }
                        }
                        break;
                    case TPT_Weekly:
                        //if (preWD < _task->_day && weekDay >= _task->_day)
                        //	fired = true;
                        if (weekDay == _task->_day)
                            fired = true;
                        else if (bHasHoliday)
                        {
                            if (days >= 7 && weekDay > _task->_day)
                            {
                                fired = true;
                            }
                            else if (preWD > weekDay && weekDay > _task->_day)
                            {
                                //上一个交易日的星期大于今天的星期,说明换了一周了
                                fired = true;
                            }
                            else if (preWD < _task->_day && weekDay > _task->_day)
                            {
                                fired = true;
                            }
                        }
                        break;
                    case TPT_Yearly:
                        if (preTDate % 10000 < _task->_day && _cur_date % 10000 >= _task->_day)
                            fired = true;
                        break;
                    }
                }
            }

            if (!fired)
            {
                //调整时间
                //如果当前时间小于任务时间,则直接赋值即可
                //如果当前时间大于任务时间,则至少要等下一天
                if (_cur_time < endtime)
                {
                    _cur_time = endtime;
                    continue;
                }

                uint32_t newTDate = _bd_mgr.calcTradingDate(DEF_SESS, _cur_date, _cur_time, true);

                if (newTDate != _cur_tdate)
                {
                    _cur_tdate = newTDate;
                    if (_listener)
                        _listener->handle_session_begin(newTDate);
                    if (_listener)
                        _listener->handle_session_end(newTDate);
                }
            }
            else
            {
                //用前一分钟作为结束时间
                uint32_t curDate = _cur_date;
                uint32_t curTime = endtime;
                bool bEndSession = sInfo->offsetTime(curTime, true) >= sInfo->getCloseTime(true);
                if (_listener)
                    _listener->handle_session_begin(_cur_tdate);
                onMinuteEnd(curDate, curTime, bEndSession ? _cur_tdate : preTDate);
                if (_listener)
                    _listener->handle_session_end(_cur_tdate);
            }

            _cur_date = TimeUtils::getNextDate(_cur_date);
            _cur_time = endtime;
            _cur_tdate = _bd_mgr.calcTradingDate(DEF_SESS, _cur_date, _cur_time, true);

            uint64_t nextTime = (uint64_t)_cur_date * 10000 + _cur_time;
            if (nextTime > _end_time)
            {
                WTSLogger::log_raw(LL_INFO, "Backtesting with task frequency is done");
                if (_listener)
                {
                    _listener->handle_session_end(_cur_tdate);
                    _listener->handle_replay_done();
                    if (_notifier)
                        _notifier->notifyEvent("BT_END");
                }

                break;
            }
        }
    }
    else
    {
        if (_listener)
            _listener->handle_session_begin(_cur_tdate);

        for (; !_terminated;)
        {
            //要考虑到跨日的情况
            uint32_t mins = sInfo->timeToMinutes(_cur_time);
            //如果一开始不能整除,则直接修正一下
            if (mins % _task->_time != 0)
            {
                mins = mins / _task->_time + _task->_time;
                _cur_time = sInfo->minuteToTime(mins);
            }

            bool bNewTDate = false;
            if (mins < sInfo->getTradingMins())
            {
                onMinuteEnd(_cur_date, _cur_time, 0);
            }
            else
            {
                bNewTDate = true;
                mins = sInfo->getTradingMins();
                _cur_time = sInfo->getCloseTime();

                onMinuteEnd(_cur_date, _cur_time, _cur_tdate);

                if (_listener)
                    _listener->handle_session_end(_cur_tdate);
            }


            if (bNewTDate)
            {
                //换日了
                mins = _task->_time;
                uint32_t nextTDate = _bd_mgr.getNextTDate(_task->_trdtpl, _cur_tdate, 1, true);

                if (sInfo->getOffsetMins() != 0)
                {
                    if (sInfo->getOffsetMins() > 0)
                    {
                        //真实时间后移,说明夜盘算作下一天的
                        _cur_date = _cur_tdate;
                        _cur_tdate = nextTDate;
                    }
                    else
                    {
                        //真实时间前移,说明夜盘是上一天的,这种情况就不需要动了
                        _cur_tdate = nextTDate;
                        _cur_date = _cur_tdate;
                    }
                }

                _cur_time = sInfo->minuteToTime(mins);

                if (_listener)
                    _listener->handle_session_begin(nextTDate);
            }
            else
            {
                mins += _task->_time;
                uint32_t newTime = sInfo->minuteToTime(mins);
                bool bNewDay = newTime < _cur_time;
                if (bNewDay)
                    _cur_date = TimeUtils::getNextDate(_cur_date);

                uint32_t dayMins = _cur_time / 100 * 60 + _cur_time % 100;
                uint32_t nextDMins = newTime / 100 * 60 + newTime % 100;

                //是否到了一个新的小节
                bool bNewSec = (nextDMins - dayMins > _task->_time) && !bNewDay;

                while (bNewSec && _bd_mgr.isHoliday(_task->_trdtpl, _cur_date, true))
                    _cur_date = TimeUtils::getNextDate(_cur_date);

                _cur_time = newTime;
            }

            uint64_t nextTime = (uint64_t)_cur_date * 10000 + _cur_time;
            if (nextTime > _end_time)
            {
                WTSLogger::log_raw(LL_INFO, "Backtesting with task frequency is done");
                if (_listener)
                {
                    _listener->handle_session_end(_cur_tdate);
                    _listener->handle_replay_done();
                    if (_notifier)
                        _notifier->notifyEvent("BT_END");
                }
                break;
            }
        }
    }
}
```

### 回测撮合逻辑

#### on_tick

回测中的订单撮合主要在on_tick事件中执行，这里的on_tick事件不是指策略中的on_tick，而是指调度器CtaMocker中的on_tick，由handle_tick触发。主要完成以下工作

1. 检查是否要信号要触发

2. 检查条件单

3. 调用策略的on_tick

#### 信号函数

当在策略中调用stra_enter_long/stra_enter_short/stra_exit_long/stra_exit_short 等函数时，会进行以下流程：

1. 如果没有指定条件单价格，则视为动态下单，直接添加到信号列表中。

2. 如果指定了条件单价格，则视为条件下单，根据信号方向生成条件单对象，记录目标价格以及触发规则

    WCT_Equal,          //等于
    WCT_Larger,         //大于
    WCT_Smaller,        //小于
    WCT_LargerOrEqual,  //大于等于
    WCT_SmallerOrEqual  //小于等于

3. 信号列表以及停止单列表，将会依次在on_tick事件中处理。

#### 信号处理

在on_tick中，首先处理信号列表，通过执行do_set_position函数，将当前持仓调整到信号期望的目标仓位。

#### 信号生成

除了在策略代码中，直接调用stra_enter_long/stra_enter_short/stra_exit_long/stra_exit_short进行动态下单直接添加到信号列表中外，在on_tick中对停止单进行检查，如果判断触发，生成信号并添加到信号列表中。这个信号会在下一轮on_tick中被执行。