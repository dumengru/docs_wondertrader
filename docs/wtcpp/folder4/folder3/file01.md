# 配置文件预览

source: `{{ page.path }}`

```tip
config.json
```

```json
{
    "replayer":{
        "mode":"csv",
        "path":"./storage/", 
        "stime":201905010900,
        "etime":201910011500,
        "basefiles":{
            "session":"./common/sessions.json",
           	"commodity":"./common/commodities.json",
            "contract":"./common/contracts.json",
            "holiday":"./common/holidays.json",
            "hot":"./common/hots.json"
        },
        "fees":"./common/fees.json"
    },
    "env":{
        "mocker":"cta",
        "slippage": 1
    },
    "cta":{
        "module":"WtCtaStraFact.dll",
        "strategy":{
            "id": "dt_if",
            "name": "DualThrust",
            "params": {
                "code": "CFFEX.IF.HOT",
                "count": 50,
                "period": "m5",
                "days": 30,
                "k1": 0.6,
                "k2": 0.6,
                "stock":false
            }
        }
    }
}
```

```tip
session.json
```

```json
{
    "root":{
        "level":"debug",
        "async":false,
        "sinks":[
            {
                "type":"basic_file_sink",
                "filename":"BtLogs/Runner.log",
                "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                "truncate":true
            },
            {
                "type":"console_sink",
                "pattern":"[%m.%d %H:%M:%S - %^%-5l%$] %v"
            }
        ]
    },
    "dyn_pattern":{
        "strategy":{
            "level":"debug",
            "async":false,
            "sinks":[
                {
                    "type":"basic_file_sink",
                    "filename":"BtLogs/Strategy_%s.log",
                    "pattern":"[%Y.%m.%d %H:%M:%S - %-5l] %v",
                    "truncate":false
                }
            ]
        }
    }
}
```

```tip

```

```json
{
    "FD0915":{
        "name":"期货白盘0915",
        "offset": 0,
        "auction":{
            "from": 929,
            "to": 930
        },
        "sections":[
            {
                "from": 930,
                "to": 1130
            },
            {
                "from": 1300,
                "to": 1515
            }
        ]
    },
    "TRADING":{
        "name":"全市场回测",
        "offset": 300,
        "auction":{
            "from": 2059,
            "to": 2100
        },
        "sections":[
            {
                "from": 2100,
                "to": 230
            },
            {
                "from": 900,
                "to": 1130
            },
            {
                "from": 1300,
                "to": 1515
            }
        ]
    }
}
```

```tip
commodities.json
```

```json
{
    "CFFEX": {
        "IC": {
            "covermode": 0,
            "pricemode": 0,
            "category": 1,
            "precision": 1,
            "pricetick": 0.2,
            "volscale": 200,
            "name": "中证",
            "exchg": "CFFEX",
            "session": "SD0930",
            "holiday": "CHINA"
        },
        "IF": {
            "covermode": 0,
            "pricemode": 0,
            "category": 1,
            "precision": 1,
            "pricetick": 0.2,
            "volscale": 300,
            "name": "沪深",
            "exchg": "CFFEX",
            "session": "SD0930",
            "holiday": "CHINA"
        }
    }
}
```

```tip
contracts.json
```

```json
{
    "CFFEX": {
        "IC2108": {
            "name": "中证2108",
            "code": "IC2108",
            "exchg": "CFFEX",
            "product": "IC",
            "maxlimitqty": 20,
            "maxmarketqty": 10
        },
        "IC2109": {
            "name": "中证2109",
            "code": "IC2109",
            "exchg": "CFFEX",
            "product": "IC",
            "maxlimitqty": 20,
            "maxmarketqty": 10
        }
    }
}
```

```tip
holidays.json
```

```json
{
    "CHINA" : [
        20180101,20180215,20181005,20181231,
        20190101,20190913,20191001,20191002,20191003,20191004,20191007,
        20200101,20200124,20200127,20200128,20200129,20200130,20200131,20200406,20200501,20200504,20200505,20200625,20200626,20201001,20201002,20201005,20201006,20201007,20201008,
        20210101,20210211,20210212,20210215,20210216,20210217,20210405,20210503,20210504,20210505,20210614,20210920,20210921,20211001,20211004,20211005,20211006,20211007]
}
```

```tip
hots.json
```

```json
{
    "CFFEX": {
        "IC": [
            {
                "date": 20190102,
                "from": "",
                "newclose": 4096.8,
                "oldclose": 0.0,
                "to": "IC1901"
            },
            {
                "date": 20190118,
                "from": "IC1901",
                "newclose": 4274.4,
                "oldclose": 4307.4,
                "to": "IC1903"
            }
        ]
    }
}
```

```tip
fees.json
```

```json
{
    "CFFEX.IF":
    {
        "open":0.000023,
        "close":0.000023,
        "closetoday":0.000023,
        "byvolume":false
    },
    "CFFEX.IC":
    {
        "open":0.000023,
        "close":0.000023,
        "closetoday":0.000023,
        "byvolume":false
    }
}
```