# 编译器问题

WonderTrader使用VS2017开发，但也可以在其他更新版本的VS下运行。只需要通过vs工具栏中的获取工具和功能获取对应的VS2017的平台工具集即可。

![获取.png](/docs/wt/folder1/folder1/pic/getToolsAndFunction1.png)

![获取.png](/docs/wt/folder1/folder1/pic/getToolsAndFunction2.png)

更新后，确认每个项目属性中的平台工具集项为`Visual Studio 2017 (v141)`即可在其他版本下编译。

![获取.png](/docs/wt/folder1/folder1/pic/getToolsAndFunction3.png)

```tip
使用非2017版本的vs打开项目文件时，会提示是否重定向项目，点击否
```

# 定义问题

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