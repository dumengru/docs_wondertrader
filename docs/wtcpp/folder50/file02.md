# 关于WT使用一个礼拜后的一些积累

source: `{{ page.path }}`

## 直接使用执行器做算法交易
开始使用WT到现在有一个礼拜了，目前已经跑通了QuoteFactory，然后根据当前不通过策略直接做算法交易的需求，开始使用WTExeMon做算法交易，目前有两种方法，一个是在WTExecMon上自己建main.cpp，代码如下：
```cpp
#include <boost/asio.hpp>
#include "WtExecRunner.h"


 //#include <vld.h>

#ifdef _MSC_VER
#include "../Common/mdump.h"
#endif

//#include <vld.h>
boost::asio::io_service g_asyncIO;
int main()
{
#ifdef _MSC_VER
 CMiniDumper::Enable("WtExecMon.exe", true);
#endif

 WtExecRunner runner;
 std::string cfgFile = "config.json";
 if (!StdFile::exists(cfgFile.c_str()))
  cfgFile = "config.yaml";

 runner.init();
 runner.config(cfgFile.c_str());


 runner.run();
 //开仓1万股
 runner.setPosition("SSE.600276", 10000);
 //等待IO，如果下面两句话不写main在执行完之后就会直接退出控制台
 boost::asio::io_service::work work(g_asyncIO);
 g_asyncIO.run();

 return 0;
}
```
代码里面使用的cofig.yaml文件，Excecutors目前只能直接使用Excecutors配置，而不能指定Excecutors.ymal，配置参考如下：
```yaml
basefiles:
    commodity: ../common/stk_comms.json
    contract: ../common/stocks.json
    holiday: ../common/holidays.json
    session: ../common/stk_sessions.json
data:
    store:
        #这个session不一定要指定，否则会报莫名其妙的错误然后粗
        session: SD0930
        path: ../STK_Data/
executers:
-   active: true    #是否启用
    id: exec        #执行器id，不可重复
    trader: simnow  #执行器绑定的交易通道id，如果不存在，无法执行
    scale: 1        #数量放大倍数，即该执行器的目标仓位，是组合理论目标仓位的多少倍，可以为小数 
    policy:         #执行单元分配策略，系统根据该策略创建对一个的执行单元
        default:    #默认策略，根据品种ID设置，如SHFE.rb，如果没有针对品种设置，则使用默认策略
            name: WtExeFact.WtMinImpactExeUnit #执行单元名称
            offset: 0      #委托价偏移跳数
            expire: 5      #订单超时没秒数
            pricemode: 1   #基础价格模式，-1-己方最优，0-最新价，1-对手价
            span: 500      #下单时间间隔（tick驱动的）
            byrate: false  #是否按对手盘挂单量的比例挂单，配合rate使用
            lots: 100       #固定数量
            rate: 0         #挂单比例，配合byrate使用

    clear:                  #过期主力自动清理配置
        active: false       #是否启用
        excludes:           #排除列表
        - CFFEX.IF
        - CFFEX.IC
        includes:           #包含列表
        - SHFE.rb

fees: ../common/fees_stk.json
filters: filters.yaml       #过滤器配置文件，这个主要是用于盘中不停机干预的
parsers: tdparsers.yaml     #行情通达配置文件
traders: tdtraders.yaml     #交易通道配置文件
bspolicy: actpolicy.yaml
```
另外一种方法就是修改WTExecPorter，所谓的porter就是dll提供的外部接口。这里面的代码其实本质上和第一种方法一致，但WTExecPorter的实现是通过调用WTExecMon的dll文件对外提供的接口，使用WTExecPorter的时候也要注意加上：
```cpp
 boost::asio::io_service::work work(g_asyncIO);
 g_asyncIO.run();
```
否则同样在程序执行完main函数后会直接推出，而不是等待行情的传入、交易的执行。
## WT的架构理解
以前没做过这种量化交易系统，但通过这几天的使用和代码调试，终于理解了这类系统的根本业务逻辑还是使用行情和交易回调函数来驱动策略（或者直接交易），理解了这一点，策略或者执行器的入口都可以通过在ParserAdapter和TraderAdapter适配的具体的Parser和Trader中找到，举个例子，执行器中的on_tick触发，可以从Parser的接受行情回调函数中去找，像ParserXTP中的onDepthMarketData函数中就有handlequote的调用，跟下代码你就会发现有handletick的逻辑，handletick就会去调用策略或者执行器中的on_tick。