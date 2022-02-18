# 编译器版本问题

source: `{{ page.path }}`

WonderTrader使用VS2017开发，但也可以在其他更新版本的VS下运行。只需要通过vs工具栏中的获取工具和功能获取对应的VS2017的平台工具集即可。

![获取.png](../../../assets/images/wt/hej_getToolsAndFunction1.png)

![获取.png](../../../assets/images/wt/hej_getToolsAndFunction2.png)

更新后，确认每个项目属性中的平台工具集项为`Visual Studio 2017 (v141)`即可在其他版本下编译。

![获取.png](../../../assets/images/wt/hej_getToolsAndFunction3.png)

```tip
使用非2017版本的vs打开项目文件时，会提示是否重定向项目，点击否
```