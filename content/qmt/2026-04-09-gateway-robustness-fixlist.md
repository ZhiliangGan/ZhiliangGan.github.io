# DevLog #005: QMT Gateway 鲁棒性修复清单 — P0-P2 分级扫雷

**日期**: 2026-04-09
**状态**: 修复中
**Tags**: qmt-gateway, robustness, grpc, thread-safety

---

## 背景

目标：把 QMT Gateway 做到**"可长期稳定支撑批回测 + 准实盘（正在测）+ 实盘执行"**的工程质量。

本文偏工程健壮性（正确性、隔离性、可观测性、可恢复性），不讨论策略收益逻辑。

涉及模块：
- `qmt_gateway/*`（网关本体）
- `qdata/qdata/adapters/qmt_remote_adapter.py`（客户端适配器）
- `qss/*`（交易执行器 / 远端执行服务）

---

## P0: Must Fix Today

这些问题**不修会导致**：数据错乱、订阅串流、卡死/随机失败、交易服务不可用。

### P0-1: 实时订阅的消息隔离与正确性

**症状**：多个订阅者同时订阅时会串流/抢消息；订阅 A 可能收到订阅 B 的 tick；或漏推。

**位置**：
- `qmt_gateway/services/data_service.py`（`SubscribeQuote` 直接消费一个全局流并 yield）
- `qmt_gateway/bridge/xtdata_bridge.py`（单个 `_tick_queue` + `tick_stream()`）

**修复方向**：把 tick 分发改成 fan-out/pub-sub（每个订阅者独立队列），并按 `request.symbols` 做过滤。

### P0-2: `asyncio.Queue` 跨线程使用（线程安全问题）

**症状**：tick 回调可能在 xtdata 线程里触发，但 `asyncio.Queue.put_nowait()` 不是线程安全；会出现随机异常、丢消息、甚至卡死。

**位置**：`qmt_gateway/bridge/xtdata_bridge.py`（`on_data()` -> `_tick_queue.put_nowait()`）

**修复方向**：使用线程安全桥接（例如 `queue.Queue` + `asyncio.to_thread`/executor），或在 aio loop 上用 `loop.call_soon_threadsafe(...)` 入队。

> ★ Insight ─────────────────────────────────────
> **asyncio.Queue 本身不是线程安全的**。
> 当 xtdata 的 C 线程回调触发 Python 的 `on_data()` 时，
> 直接在那个线程调用 `asyncio.Queue.put_nowait()` 是危险的。
> 需要用 `loop.call_soon_threadsafe()` 将数据送入 asyncio 的事件循环。
> ──────────────────────────────────────────────────

### P0-3: xtdata 订阅回调未显式绑定

**症状**：`subscribe_quote()` 之后 `tick_stream()` 永远没有数据。

**位置**：`qmt_gateway/bridge/xtdata_bridge.py`（定义了 `on_data`，但没有看到把它注册给 xtdata）

**修复方向**：确认 xtdata API 的订阅签名，显式传入/注册回调。

### P0-4: gRPC 混用 `grpc.aio` + `asyncio.run`

**症状**：并发、已有 event loop、长时间运行时表现不稳定。

**位置**：`qdata/qdata/adapters/qmt_remote_adapter.py`

**修复方向**：改为同步 `grpc.insecure_channel`；或将 aio 调用统一放到后台 loop/thread。

### P0-5: 交易服务初始化参数丢失

**症状**：JSON-RPC 8080 常年起不来/或者 executor 行为不符合 account 配置。

**位置**：`qmt_gateway/bridge/xttrader_bridge.py`（`create_qmt_executor` 传入了参数但 `QMTExecutor(**kwargs)` 没吃到）

---

## P1: Should Fix Soon

这些问题**不一定立即炸，但会严重影响"稳定批跑/定位问题/快速恢复"**。

### P1-1: tick 队列无界（潜在 OOM）

**症状**：行情高频时内存持续上涨。

**位置**：`xtdata_bridge.py`（`self._tick_queue: asyncio.Queue = asyncio.Queue()`）

**修复方向**：给队列设 `maxsize`，并实现明确的丢弃策略。

### P1-2: 订阅引用计数缺少并发保护

**症状**：并发订阅/取消订阅时，引用计数可能被写坏。

**位置**：`xtdata_bridge.py`（`_sub_ref_count` 的增减没有 lock）

### P1-3: gRPC 请求超时与取消处理不完整

**症状**：客户端/服务端遇到网络抖动会挂很久；取消订阅后服务端协程可能仍阻塞。

**修复方向**：统一 timeout、重试（指数退避）、可区分的错误码。

### P1-4: 时间戳/时区语义需要统一

**症状**：bar 的 `time` 既可能来自 YYYYMMDD 也可能来自毫秒时间戳；客户端又把它当 UTC epoch。

**修复方向**：协议明确 `time` 的时区与含义（推荐 epoch_ms）。

---

## P2: Nice To Have

这些是**工程化加分项**，对长期运营和扩展很关键。

### P2-1: 鉴权与访问控制

**现状**：`insecure_channel`，局域网内任何人可访问数据/交易接口。

**方向**：在 gRPC metadata 加 token；或 mTLS。

### P2-2: 指标与追踪

**方向**：请求计数/耗时/错误码/订阅数/队列长度/丢包数。

---

## Validation Checklist

- [ ] 单订阅者实时行情：订阅后 5 秒内收到 tick
- [ ] 双订阅者隔离：A/B 订阅不同 symbol，互不串流
- [ ] 并发压测：N=10 并发 `GetMarketData` 不崩、不死锁
- [ ] 异常注入：网关断网/xtdata 崩/下载失败时快速降级

---

*Generated from: docs/GATEWAY_ROBUSTNESS_FIXLIST.md*
