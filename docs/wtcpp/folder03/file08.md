# CTA仿真完整篇3: 策略回调

source: `{{ page.path }}`

## on_tick

关于策略中 `on_tick` 触发流程参考文章 "HFT仿真", 这里直接进入 `WtCtaRtTicker.cpp`.

`WtCtaRtTicker` 被称为"生产时间步进器"(我也不知道为何叫如此拗口的名字), 它主要的功能就是处理tick事件.

### trigger_price

将断点打在 `WtCtaRtTicker::on_tick` 首行, 然后逐步运行到 `trigger_price`, 这里 `on_tick` 首先对数据时间进行了一番检查, 最后调用了 `trigger_price`, 实际上所有 `on_tick` 被触发后的动作都在 `trigger_price` 中完成.

1. 进入 `trigger_price` 后首先调用 `_engine->on_tick`
2. 进入 "WtCtaEngine.cpp", 首先执行 `WtEngine::on_tick`, 检查信号是否触发
3. 接着执行 `_data_mgr->handle_push_quote`
4. 然后判断品种是否在 `_tick_sub_map` 中, 这里的 `_tick_sub_map`, 在上篇文章策略初始化时提到过.
5. 执行 `_ctx_map.find`, 找到对应品种的上下文管理器, 然后执行 `ctx->on_tick`
6. 进入 `CtaStraBaseCtx.cpp`, 检查信号是否触发, 并更新浮动盈亏
7. 接着到这这句代码: `on_tick_updated`
8. 进入 `CtaStraContext.cpp`, 如果在 `_tick_subs` 中没有找到品种代码会直接返回(因此你明白终端为何只有 CFFEX.IH.2203的 on_tick 回调了吗)
9. tick回调流程结束

这里和 HFT 的回调流程基本一致.

## on_bar

策略文件中除了触发 `on_tick`, 最重要的就是触发 `on_bar` 和 `on_schedule`, 这两个接口是在分钟结束后才会触发

1. 将断点打在 `WtCtaRtTicker::run` 中的 `_store->onMinuteEnd` 上, 然后等待分钟闭合
2. 进入 `WtDataReader.cpp`, 会读取本地缓存数据, 之后读取成功才会执行 `_sink->on_bar`(这也是必须打开 QuoteFactory的原因)
3. 进入 `WtDtMgr.cpp`, 执行 `_bar_notifies.emplace_back`, 通知bar线闭合. 所有品种遍历后, 执行 `_sink->on_all_bar_updated`
4. 进入 `WtDtMgr::on_all_bar_updated`, 逐个遍历执行 `on_bar` 事件

这里和 HFT 的回调流程基本一致, 但是由于 CTA 只能订阅一个K线, 因此只会有一个K线闭合.

## on_schedule

`on_bar` 执行完毕, 回到 "WtCtaTicker.cpp", 执行 `_engine->on_schedule`. 这里对应 HFT 的 `on_schedule` 方法被取消了.

1. 首先检查过滤器. 然后执行 `ctx->on_schedule`
2. 进入 `CtaStraBaseCtx.cpp`, 执行 `on_calculate`
3. 进入 `CtaStraContext::on_calculate`, 回调策略的 `on_schedule`

1. 上述第1步内容较为重要, 以后会单独拿出章节讲解.
2. 这里虽然在 CTA 中 `on_bar` 和 `on_shcedule` 的触发时机都一样, 但是 `on_schedule` CTA处理策略组合中最重要的部分, 因此CTA策略逻辑基本都在 `on_schedule` 中实现