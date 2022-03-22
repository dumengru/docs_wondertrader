# CTA仿真完整篇3: 下单流程

source: `{{ page.path }}`

本节内容开始阐述CTA策略下单流程, 其实"CTA完整篇"系列内容本应该在"CTA进阶篇"之后, 无奈工作繁忙, 正准备接着"进阶篇3"之后继续写下单流程时被搁置了, 那几天恰好又赶上WT新版本发布, 于是乎就重新整理了"CTA完整篇".

可能有人会问为什么不选择"HFT下单流程", 因为"HFT"整个逻辑比较简单, "CTA"逻辑复杂, 要想写一个完整流程, 干脆直接挑一个"刺儿头"(经常看群里人抱怨没文档, 说实话, 我看WT源码也很苦逼啊...就是一点点磨), 如果你把CTA下单流程看明白了, "HFT下单流程"自然不在话下.

书接上文, 初始化完毕...

## stra_get_position

1.将断点打在 `WtStraDualThrust::on_schedule` 下的 `if(decimal::eq(curPos,0))`. (注意, 保证"QuoteFactory"在开启状态), 当**分钟结束**时, 程序回调 `on_schedule` 会在断点处自动停止

2.执行 `ctx->stra_enter_long`, 进入 `CtaStraBaseCtx::stra_enter_long`, 这里通过 `limitprice` 和 `stopprice` 设置了三种下单方式: 市价单(最新价), 限价单和停止单

3.我们默认是市价单, 执行 `append_signal`, 进入 `CtaStraBaseCtx::append_signal`, 填充 `sInfo` 字段(信号信息), 然后存放在 `_sig_map` 中.

然后就结束了, 是的, `stra_get_position` 就这么结束了.

可是我们只是记录了信号, 还没有将订单发送到交易所.

## set_positions

1.继续执行程序, 直到结束策略中的 `on_schedule`, 回到 `CtaStraContext::on_calculate`, 执行至结束, 回到 `CtaStraBaseCtx::on_schedule`, 执行至结束, 回到 `WtCtaEngine::on_schedule`.

2.执行 `ctx->enum_position`, 这里通过匿名函数方式将品种及对应目标仓位存放在 `target_pos`中. 注意, 这里会通过信号过滤器检查(不是本文重点).

3.向下执行至"//处理组合理论部位", 在这里会遍历所有策略中的 `target_pos`, 执行 `append_signal`, 进入 `WtEngine::append_signal`. 这里会将品种名称和下单量保存在 `_sig_map` 中(和前面的 `CtaStraBaseCtx` 中的 `_sig_map` 很像).然后执行结束.

4.向下执行至 `_exec_mgr.set_positions`, 进入 `WtExecuterMgr::set_positions`, 首先通过信号过滤器校验.然后开始遍历所有的执行器(每个CTA策略都有对应的执行器)

5.执行至 `executer->set_position`, 进入 `WtLocalExecuter::set_position`, 这里才真正开始准备向市场发送订单.

## WtLocalExecuter::set_position

1. 遍历全部持仓, 首先获取执行单元(每个策略都对应一个执行单元)
2. 然后检查品种下单量, 这里乘以配置中的 `_scale`
3. 将计算出来的下单量和品种名称保存至 `_target_pos`
4. 通过交易接口检查下单限制 `_trader->checkOrderLimits`
5. 调用执行单元下单 `unit->self()->set_position`


### WtMinImpactExeUnit::set_position

1.执行 `unit->self()->set_position`, 才真正进入执行单元, 这里经过一番检查, 然后走到 `do_calc`

2.`undone`: 未完成订单, `realPos`: 当前真正持仓, `newVol`: 新的目标持仓, `diffPos`: 即将要发送的持仓.

3.中间检查订单和持仓状态等做了大量判断, 这里不做详细说明, 感兴趣可以自己看看. 

4.最后的最后, 调用 `_ctx->buy`

```cpp
if (isBuy)
{
    OrderIDs ids = _ctx->buy(stdCode, buyPx, this_qty, bForceClose);
    _orders_mon.push_order(ids.data(), ids.size(), _ctx->getCurTime(), isCanCancel);
}
```

5.然后进入 `WtLocalExecuter::buy`, 调用 `_trader->buy`, 订单终于被发送到交易所. 

## 总结

流程终于走完了, 但是很多人应该已经被绕晕了, 在CTA从策略中发送一笔订单过程中, 主要经历了以下几个类.(还不包含一些数据管理, 行情和交易等类)

1. `CtaStraContext`
2. `CtaStraBaseCtx`
3. `WtCtaEngine`
4. `WtEngine`
5. `WtExecuterMgr`
6. `WtLocalExecuter`
7. `WtMinImpactExeUnit`

为方便大家理解流程, 最后再附上一张图

![](../../assets/wt/../images/wt/wt040.jpg)