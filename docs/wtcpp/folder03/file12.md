# CTA仿真完整篇4: 配置文件

source: `{{ page.path }}`

本章内容将讲述5个配置文件相关内容, 主要包括
1. actpolicy.yaml
2. filters.yaml
3. config.yaml内的riskmon
4. common目录下的fees.json
5. tdtraders.yaml内的riskmon

## 修改配置文件

1.修改config.yaml配置文件

```yaml
#基础配置文件
basefiles:
    commodity: ../common/commodities.json   #品种列表
    contract: ../common/contracts.json      #合约列表
    holiday: ../common/holidays.json        #节假日列表
    hot: ../common/hots.json                #主力合约映射表
    session: ../common/sessions.json        #交易时间模板
#数据存储
data:
    store:
        path: ../FUT_Data/      #数据存储根目录

#环境配置
env:
    name: cta               #引擎名称：cta/hft/sel
    product:
        session: ALLDAY    #驱动交易时间模板，TRADING是一个覆盖国内全部交易品种的最大的交易时间模板，从夜盘21点到凌晨1点，再到第二天15:15，详见sessions.json
    filters: filters.yaml       #过滤器配置文件，这个主要是用于盘中不停机干预的
    fees: ../common/fees.json   #佣金配置文件
    riskmon:                #组合风控设置
        active: true            #是否开启
        module: WtRiskMonFact   #风控模块名，会根据平台自动补齐模块前缀和后缀
        name: SimpleRiskMon     #风控策略名，会自动创建对应的风控策略
        #以下为风控指标参数，该风控策略的主要逻辑就是日内和多日的跟踪止损风控，如果回撤超过阈值，则降低仓位
        base_amount: 5000000    #组合基础资金，WonderTrader只记录资金的增量，基础资金是用来模拟组合的基本资金用的，和增量相加得到动态权益
        basic_ratio: 101        #日内高点百分比，即当日最高动态权益是上一次的101%才会触发跟踪侄止损
        calc_span: 5            #计算时间间隔，单位s
        inner_day_active: true  #日内跟踪止损是否启用
        inner_day_fd: 20.0      #日内跟踪止损阈值，即如果收益率从高点回撤20%，则触发风控
        multi_day_active: false #多日跟踪止损是否启用
        multi_day_fd: 60.0      #多日跟踪止损阈值
        risk_scale: 0.3         #风控系数，即组合给执行器的目标仓位，是组合理论仓位的0.3倍，即真实仓位是三成仓
        risk_span: 30           #风控触发时间间隔，单位s。因为风控计算很频繁，如果已经触发风控，不需要每次重算都输出风控日志，加一个时间间隔，友好一些

strategies:
    # CTA策略配置，当mocker为cta时会读取该配置项
    cta:
    -   active: true       # 模块名，linux下为xxxx.so
        id: dt_if          # 策略ID，自定义的
        name: WtCtaStraFact.DualThrust   # 策略名，要和factory中的匹配
        params:                         # 策略初始化参数，这个根据策略的需要提供
            code: SHFE.au.2206
            count: 1
            days: 30
            k1: 0.6
            k2: 0.6
            period: m1
            stock: false

executers: executers.yaml   #执行器配置文件
parsers: tdparsers.yaml     #行情通达配置文件
traders: tdtraders.yaml     #交易通道配置文件
bspolicy: actpolicy.yaml    #开平策略配置文件
```

1. 修改 filters.yaml

```yaml
code_filters:
    SHFE.au:
        action: ignore
        target: 0
strategy_filters:
    dt_if:
        action: redirect
        target: 0
executer_filters:
    exec: true
```

## 初始化

书接上文, 首先打开"QuoteFactory.exe", 然后将断点打在"WtRunner.cpp"下的 `initEngine()` 函数内的 `_cta_engine.init`, 启动程序

进入 `WtCtaEngine::init`, 执行 `WtEngine::init`, 进入 `WtEngine::init`

### 信号过滤器 filters.yaml

1. 执行 `_filter_mgr.load_filters` 初始化信号过滤器, 此时进入 `WtFilterMgr::load_filters`
2. 首先加载策略过滤器配置, 内容保存至 `_stra_filters`
3. 接着加载代码过滤器配置, 内容保存至 `_code_filters`
4. 最后加载通道过滤器配置, 内容保存至 ``

### 手续费 common->fees.json

1. 执行 `load_fees(cfg->getCString("fees"))`, 进入 `WtEngine::load_fees` 加载手续费, 所有手续费相关配置都保存在 `_fee_map` 中.

### 风控管理 config.ymal->riskmon

1. 执行 `init_riskmon(cfgRisk)`, 进入 `WtEngine::init_riskmon`初始化风控工厂, 这里会从程序文件夹下加载 `WtRiskMonFact.dll`, 然后创建并初始化相应的风险管理器.
2. 执行 `_risk_mon->self()->init`, 进入 `WtSimpleRiskMon::init` 初始化风险管理器, 这里的参数和"config.yaml"内的`riskmon`参数一一对应
3. 风控和策略创建模式是一模一样的

### 开平仓管理 actpolicy.yaml

开平仓管理和上述三个配置文件初始化地方不大一致. 将断点打在 `WtRunner::initActionPolicy` 下.

1.进入 `ActionPolicyMgr::init`, 所有开平规则和ID映射存放在 `_rules` 中, 品种名称和ID映射存放在 `_comm_rule_map` 中.

2.开平规则有三种(limit 字段)

```cpp
uint32_t	_limit;		//手数限制
uint32_t	_limit_l;	//多头手数限制
uint32_t	_limit_s;	//空头手数限制
```
同时还包括四种动作类型(action 字段)

```cpp
	AT_Open = 9999,		//开仓
	AT_Close,			//平仓
	AT_CloseToday,		//平今
	AT_CloseYestoday	//平昨
```

### 流量风控 tdtraders.yaml->riskmo

流量风控的配置隐藏在交易端口内. 将断点打在 `WtRunner::initTraders` 下的 `adapter->init`

进入 `TraderAdapter::init`, 可以看到"解析流量风控参数"部分, 对应着 "tdtraders.yaml" 配置文件下的 `riskmon` 部分, 所有配置内容保存在 `_risk_params_map` 中.

## 运行

### 流量风控

流量风控主要检查下单和取消订单的限制. 它的调用包含在 `TraderAdapter::checkOrderLimits` 和 `TraderAdapter::checkCancelLimits` 中, 因此每次下单和撤单前, 只要调用这两个方法, 都会去检查流量控制. 

下单流程很长, 但真正的下单动作只在最后几步, 回看前一篇文章, 下单动作进入到 `WtLocalExecuter::set_position` 这一步时, 就会调用 `checkOrderLimits`.

### 风控管理

风控管理会启动单独的线程. 将断点打在 "WtCtaEngine.cpp" 下的 `_risk_mon->self()->run();`, 进入 `WtSimpleRiskMon::run()`

这里包含了风控管理的所有逻辑, 当风控触发时, 会修改引擎中的 `_risk_volscale` 参数, 每次下单前都会对 `_risk_volscale` 做检查, 以此达到风控目的.

### 信号过滤器

打开 "WtCtaEngine.cpp" 直接 Ctrl+f 搜索 "_filter" 就可以看到那些地方调用了信号过滤器, 除了初始化部分, 在 `WtCtaEngine::on_schedule` 和 `WtCtaEngine::handle_pos_change` 执行时会调用.

然后打开 "CtaStraBaseCtx", 搜索 `handle_pos_change` 即可知道信号过滤器的调用过程.

### 手续费

打开 "WtEngine.cpp" 直接 Ctrl+f 搜索 "_fee_map" 就可以看到那些地方计算手续费, 除了初始化部分, 在 `WtEngine::calc_fee` 执行时会计算.

然后打开 "CtaStraBaseCtx", 搜索 `calc_fee`, 就知道下单过程中那些步骤会计算手续费了.

### 开平仓管理

打开 "TraderAdapter.cpp" 直接 Ctrl+f 搜索 "_policy" 就可以看到那些地方调用该方法, 除了初始化部分, 在 `TraderAdapter::sell` 和 `TraderAdapter::buy` 执行时会调用.

## 总结

流量风控和开平仓管理都在 "TraderAdapter.cpp" 中被调用, 风控管理和信号过滤器主要在 "WtCtaEngine.cpp" 中被调用, 手续费主要在 "WtEngine.cpp" 中计算.

```tip
因为上一章内容讲述了下单流程, 所以这一些配置的调用过程没有按照流程表述, 而是看起来很零碎. 建议两章内容互相查看, 更容易理解.
```