# 概念解释

source: `{{ page.path }}`

# unbar

```note
在策略运行中，如果交易未订阅的标的，则由于没有数据传入而导致无法进行撮合，所以需要订阅一个bar来进行撮合，这个bar便称为unbar。在wonderTrader中，unbar默认为1分钟频率，且不触发on_bar事件，仅用于撮合。
```

# filters文件

```note
用于盘中干预用的文件。
```
