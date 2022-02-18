# 常见报错

source: `{{ page.path }}`

## 1. C2589 "(":"::"右边的非法标记

1. 报错内容
![img](../assets/images/../../../../assets/images/wt/nullptr_001.png)

2. 解决方案
![img](../assets/images/../../../../assets/images/wt/nullptr_002.png)


## 2. wtuftcore这个项目报未定义标识符错误

1. 报错内容
![img](../assets/images/../../../../assets/images/wt/nullptr_003.jpg)


2. 解决方案

在TraderAdapter.cpp添加using namespace std;即可

![img](../assets/images/../../../../assets/images/wt/nullptr_004.png)