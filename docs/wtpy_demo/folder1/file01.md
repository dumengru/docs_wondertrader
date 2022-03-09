# 目录结构

source: `{{ page.path }}`

写来写去发现还是群主写的简单明了~

## wtpy子框架简介
+ apps子模块
    > - WtBtAnalyst.py	回测分析模块，主要是利用回测生成的数据，计算各项回测指标，并输出到`excel`文件
    > - WtCtaOptimizer  `CTA`优化器，主要是利用`multiprocessing`并行回测，并统计各项交易指标，最后将统计结果汇总输出到`csv`文件
	> - WtHotPicker		国内期货换月规则辅助模块，支持从交易所网站页面爬取数据确定换月规则，也支持解析`datakit`每日收盘生成的snapshot.csv来确定换月规则
+ wrapper子模块
	> 该模块主要包含了所有和`C++`底层对接的接口模块
	> - ContractLoader.py	主要用于通过`CTP`等接口加载基础的`commodities.json`和`contracts.json`文件
	> - WtBtWrapper.py	主要用于和回测引擎`C++`核心模块对接
	> - WtDtWrapper.py	主要用于和数据组件`C++`核心模块对接
	> - WtDtHelper.py	主要提供将用户自己的数据和`WonderTrader`内部数据格式进行转换的功能
	> - WtDtServoApi.py	主要向用户提供直接通过`python`访问`datakit`落地的数据的接口	
	> - WtExecApi.py	主要用于和`C++`独立执行模块`WtExecMon`对接
	> - WtWrapper.py	主要用于和实盘交易引擎`C++`核心模块对接
	> - WtMQWrapper.py	主要提供直接使用底层WtMsgQue模块的对接
    > - WtDtHelper.py   主要用于和底层的`WtDtHelper`数据辅助模块对接
+ monitor子模块
	> 该模块主要包含了内置的监控服务，提供了`Http`和`websocket`两种连接方式
	> - DataMgr.py	主要用于读取并缓存组合数据
	> - EventReceiver.py	主要用于在指定的`udp`端口接收组合转发的各种事件
	> - PushSvr.py	主要用于向`web`提供`websocket`服务
	> - WatchDog.py	主要用于自动调度服务端的进程
	> - WtBtMon.py	主要进行回测的管理
	> - WtMonSvr.py	监控服务核心服务模块 ，利用`flask`实现了一个`http`服务接口
	> - static		`webui`静态文件
+ 其他模块
	> 主要位于根节点下，包含了各个子模块的入口组件
	> - WtCoreDefs.py	主要定义的`Python`版本的策略基类，方便用户重写
	> - CodeHelper.py   品种代码辅助模块，内置了一些方法，方便使用
	> - ContractMgr.py  合约管理器模块，用于加载`contracts.json`或`stocks.json`文件，并提供查询方法
	> - CtaContext.py	主要定义了`CTA`策略的上下文，可以理解为单策略的运行环境
	> - HftContext.py	主要定义了`HFT`策略的上下文，可以理解为单策略的运行环境
	> - SelContext.py	主要定义了`SEL`策略的上下文，可以理解为单策略的运行环境
	> - ExtToolDefs.py	扩展模块定义文件，主要定义了一些扩展模块的基础接口
	> - ProductMgr.py	品种管理器，主要用于`Python`环境中的合约属性、品种属性查询
	> - SelContext.py	选股策略上下文，即选股策略直接交互的`API`
	> - SessionMgr.py	交易时间模板管理器，主要用于`Python`环境中的交易时段模板管理
	> - StrategyDefs.py	各引擎策略基础定义模块，定义了`CTA`、`HFT`、`SEL`三种策略基类
	> - WtBtEngine.py	回测引擎转换模块，主要封装底层接口调用
	> - WtDtEngine.py	数据引擎转换模块，主要封装底层接口调用
	> -	WtEngine.py		交易引擎转换模块，主要封装底层接口调用
