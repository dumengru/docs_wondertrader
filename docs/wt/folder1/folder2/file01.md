# 日志报错

source: `{{ page.path }}`

## 1. 日志输出报错

1. 报错内容
```danger
0x00007FF65363356C 处有未经处理的异常(在 WtBtRunner.exe 中): 堆栈 Cookie 检测代码检测到基于堆栈的缓冲区溢出。
```

2. 报错描述
```cpp
WTSLogger::info_f("Reading data from {}...", filename);
```
使用 `WTSLogger::info_f()` 输出日志时可能会报这个错误

3. 解决方案
```cpp
1. 错误原因暂时未知
2. debug版本可能出错, release版本正常
3. 临时性解决方法: `filename.c_str()`
```
