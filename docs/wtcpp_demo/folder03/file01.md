# HFT仿真demo

source: `{{ page.path }}`

## 准备工作

### 需要先完成`生成解决方案`

### 1. 配置文件

- logcfg.json 
- config.json
- hft
- parsers
- traders
- common目录
  - sessions.json",
  - commodities.json",
  - contracts.json",
  - holidays.json",
  - hots.json"

config.json 内容
```yaml
{
    "basefiles":{
        "session":"../common/sessions.json",
        "commodity":"../common/commodities.json",
        "contract":"../common/contracts.json",
        "holiday":"../common/holidays.json",
        "hot":"../common/hots.json"
    },
    "env":{
        "name":"hft",
        "mode": "product",
        "product":{
            "session":"TRADING"
        },
        "filters":"filters.json",
        "fees":"../common/fees.json"
    },
    "bspolicy":"actpolicy.json",
    "data":{
        "store0":{
            "path":"../FUT_Data/"
        }
    },
    "strategies":{
		"hft":[
            {
                "active":false,
                "id":"hft_demo",
                "name":"WtHftStraFact.SimpleHft",
                "params":{
                    "code":"CFFEX.IF.HOT",
                    "count":10,
                    "second":10,
                    "offset":1,
                    "count": 50,
                    "stock":false
                },
				"trader":"simnow"
            }
        ]
    },
    "traders":[
        {
            "active":true,
            "id":"simnow",
            "module":"TraderCTP",
            "front":"tcp://180.168.146.187:10101",
            "broker":"9999",
            "user":"your account",
            "pass":"your passwd",
            "appid":"simnow_client_test",
            "authcode":"0000000000000000",
            "quick":true,
            "riskmon":
            {
                "active":false,
                "policy":
                {
                    "default":
                    {
                        "order_times_boundary": 20,
                        "order_stat_timespan": 10,
                        
                        "cancel_times_boundary": 20,
                        "cancel_stat_timespan": 10,
                        "cancel_total_limits": 470
                    }
                }
            }
        }
    ],
    "parsers":[
        {
			"id":"parser0",
            "active":true,
            "module":"ParserCTP",
            "front":"tcp://180.168.146.187:10211",
            "broker":"9999",
            "user":"your account",
            "pass":"your passwd"
        }
    ]
}
```

```note
仿真采用simnow实现，账户需要自行前往[simnow官网](https://www.simnow.com.cn/)注册，注册后需要重置一次密码才能够使用。config.json中的user对应simnow账户中的investorID,pass则为账户密码。
```
![project_setting.png](../../assets/images/wt/hej_02_simnow.png)

logcfg.json 内容

```yaml
{
    "root":{
        "level":"debug",
        "async":false,
        "sinks":[
            {
                "type":"daily_file_sink",
                "filename":"Logs/Runner.log",
                "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                "truncate":true
            },
            {
                "type":"ostream_sink",
                "pattern":"[%m.%d %H:%M:%S - %-5l] %v"
            }
        ]
    },
    "dyn_pattern":{
        "strategy":{
            "level":"debug",
            "async":false,
            "sinks":[
                {
                    "type":"daily_file_sink",
                    "filename":"Logs/Strategy/%s.log",
                    "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                    "truncate":true
                }
            ]
        },
        "parser":{
            "level":"debug",
            "async":false,
            "sinks":[
                {
                    "type":"daily_file_sink",
                    "filename":"Logs/Parser/%s.log",
                    "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                    "truncate":true
                }
            ]
        },
        "trader":{
            "level":"debug",
            "async":false,
            "sinks":[
                {
                    "type":"daily_file_sink",
                    "filename":"Logs/Trader/%s.log",
                    "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                    "truncate":true
                }
            ]
        },
        "executer":{
            "level":"debug",
            "async":false,
            "sinks":[
                {
                    "type":"daily_file_sink",
                    "filename":"Logs/Executer/%s.log",
                    "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                    "truncate":true
                }
            ]
        }
    },
    "risk":{
        "level":"debug",
        "async":false,
        "sinks":[
            {
                "type":"daily_file_sink",
                "filename":"Logs/Riskmon/Riskmon.log",
                "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                "truncate":true
            }
        ]
    }
}
```

上述两个配置文件含义与回测类似，具体参考`CTA回测流程逐步解析`

### 3. 策略工程(dll文件，需要完成WtCtaStraFact工程的编译)

WonderTrader给出了一个HFT引擎的demo，位于WtHftStraFact工程中，将编译后得到的`WtHftStraFact.dll`文件移入生成目录下的`hft`目录（如果没有则自己创建，下同）

### 4. CTP配置
CTP分为行情与交易两块，文件位于src/API/CTP6.3.15目录下，根据编译环境选择对应操作系统的文件，包含两个

- thostmduserapi_se.dll
- thosttraderapi_se.dll

thostmduserapi_se.dll为行情模块，放在生成目录的`parsers`文件夹下，thosttraderapi_se.dll为交易库，放在`traders`下。

### 文件结构

以X64 Debug编译为例，需要添加的配置文件的文件结构如下

- src
  - x64/x86
    - Debug/Release
      - common
      - WtRunner
        - executer
        - hft
          - WtHftStraFact.dll
        - parsers
          - thostmduserapi_se.dll
        - traders
          - thosttraderapi_se.dll
        - logcfg.json
        - config.json
        - actpolicy.json

### 编译运行程序（WtRunner工程）

将WtBtRunner工程设置为启动项目，并将项目属性-配置属性-调试-工作目录设置为$(OutDir)

```tip
编译过程中如果出现问题，可以考虑将使用管理员权限启动Visual Studio，并将编译模式由Debug修改为Release。
```
