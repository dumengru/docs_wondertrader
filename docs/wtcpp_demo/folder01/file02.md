# HFT回测详解(未完成)

source: `{{ page.path }}`

## 准备工作

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
# 回测参数
replayer:
    basefiles:
        commodity: ./common/commodities.json
        contract: ./common/contracts.json
        holiday: ./common/holidays.json
        hot: ./common/hots.json
        session: ./common/sessions.json
    mode: csv
    path: ./storage/
    fees: ./common/fees.json
    adjfactor: adjfactors.json
    stime: 202101040931
    etime: 202101121500
    tick: true
# 回测模式
env:
    mocker: hft
hft:
    error_rate: 30
    # 策略工厂dll文件名称
    module: WtHftStraFact
    # 策略配置
    strategy:
        # 策略dll文件名称
        name: HftDemoStrategy
        # 策略参数
        params:			
            code: CFFEX.IF.HOT
            count: 6
            second: 5
            freq: 20
            offset: 0
            reserve: 0
            stock: false
    use_newpx: true
```

logcfgbt.yaml

```yaml
dyn_pattern:
    strategy:
        async: false
        level: debug
        sinks:
        -   filename: BtLogs/Strategy/%s.log
            pattern: '[%Y.%m.%d %H:%M:%S - %-5l] %v'
            truncate: true
            type: basic_file_sink
root:
    async: false
    level: debug
    sinks:
    -   filename: BtLogs/BtRunner.log
        pattern: '[%Y.%m.%d %H:%M:%S,%F - %-5l] %v'
        truncate: true
        type: basic_file_sink
    -   pattern: '[%m.%d %H:%M:%S - %^%-5l%$] %v'
        type: console_sink
```

### 2. 流程详解

`WtBtRunner` 项目下的 `WtBtRunner.cpp` 文件

1. 加载配置文件, 略
2. 初始化数据回放器, 略
3. 初始化HFT策略调度器, 略
4. 将hft

### 3. 