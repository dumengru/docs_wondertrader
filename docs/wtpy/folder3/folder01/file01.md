# test_datafact

source: `{{ page.path }}`

## 简介

该示例主要是从外部API接口获取相关数据并保存到本地, 

目前有三个API接口
1. "baostock": 可以有股票免费数据
2. "tushare": 要积分
3. "rqdata": 收费

这三个接口在使用前都需要安装对应的python包(pip install ...)等, 实际上在不改源码的情况下你需要一次性把三个包都安装, 否则运行示例会提示缺少对应包.

API详情建议自己对应官网查看, 包括如何申请账号, 哪些数据直接免费, 哪些数据需要花钱等.

[tushare网址](https://tushare.pro/)

[baostock网址](http://baostock.com/baostock/index.php/%E9%A6%96%E9%A1%B5)

[rqdata网址](https://www.ricequant.com/doc/rqdata/python/)

## 运行流程

下载数据到文件

1. "test_datafact" 首先调用 `DHFactory` 创建对应的接口对象
2. 接口对象认证/不认证(与对应的接口有关)
3. 调用 `dmpBarsToFile` 方法下载数据到本地文件

下载数据到数据库

1. "test_datafact" 首先调用 `DHFactory` 创建对应的接口对象
2. 接口对象认证/不认证(与对应的接口有关)
3. 通过 `MysqlHelper` 创建数据库对象, 然后初始化数据库
4. 最后通过 `dmpBarsToDB` 方法下载数据到本地数据库

### 代码解析

典型的设计模式

1.`DHFactory` 负责创建数据接口对象
2.创建完后真正执行程序的是 "wtpy/apps/datahelper/"下对应的py文件
- "DHBaostock.py"
- "DHFactory.py"
- "DHRqData.py"
3.这三个文件内容基本一致, 封装的也非常简单明了, (感兴趣的小伙伴可以仿照编写其他的接口, 比如聚宽)
4."DHDefs.py" 定义了数据下载接口 `BaseDataHelper` 和数据库保存接口 `DBHelper`
5."wtpy/apps/datahelper/db"里 "MysqlHelper.py" 实现了Mysql 数据库接口(感兴趣的小伙伴可以仿照编写其他的接口, 比如sqlite)
6."initdb_mysql.sql", 即sql语句, 主要是新建一些数据表

## 成功示例

1. 如果下载数据到文件, 当前文件夹下应该有对应的csv文件
2. 如果下载数据到数据库, 数据文件应该会保存到数据库里