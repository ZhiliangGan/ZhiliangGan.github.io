# DevLog #011: SPS OpenClaw Skill 部署架构 — 本地引用 vs HTTP API

**日期**: 2026-04-08
**状态**: 已建立
**Tags**: sps, openclaw, skill, deployment

---

## 当前模式: 本地引用

```
~/.agents/skills/sps/
  ├── SKILL.md                 ← 能力声明
  ├── scripts/run_sps.py       ← 入口路由器
  ├── config/*.yaml            ← 只读拷贝
  ├── data/.watchlist_a.json   ← 只读拷贝
  └── .qmt_root                ← 指向 /Users/james/code/qmt/qmt
                                  ↑ 运行时动态 import sps/ 下全部代码
```

### 关键设计

**不是独立部署**。`run_sps.py` 通过 `.qmt_root` 找到项目根，把 `sps/` 加入 `sys.path`，直接 import 原始模块。

改了代码**自动生效**，只有 config/watchlist 变了才需重跑 `sync_sps_skill.sh`。

---

## 依赖链

```
run_sps.py (入口)
  ├── wave_monitor.py ← db_manager, data_adapter, vector_engine
  ├── market_radar.py ← baostock
  ├── backtest.py ← db_manager, vector_engine, strategy_b
  └── hk_us_monitor.py

vector_engine.py ← strategy_b
db_manager.py ← data_adapter
data_adapter.py ← qdata / xtquant / akshare (QMT数据源)
```

---

## 未来计划: 统一 HTTP API 部署

**Why**: 当前 `data_adapter.py` 硬依赖 QMT 客户端（qdata/xtquant），打包到别的机器上数据源就断了。

**Solution**: 开发 HTTP API 替换。

**届时**: SPS + qdata + QSS 一起部署，`data_adapter.py` 换成调 HTTP API，其余代码不动。打包范围 = 整个 qmt 项目。

---

## qmt_gateway 目录归属架构决策

**结论**: `qmt_gateway/` 不应放在项目根目录（与 qdata/、sps/、qss/ 并列）。

**Why**: qmt 只是众多量化数据源之一。qmt_gateway 是 QMT 客户端（xtquant）的 gRPC/HTTP 桥接服务，本质上属于 **qdata 的一个数据源基础设施**，不是独立子系统。

### 已执行结构

```
qdata/
  gateway/
    qmt/              ← qmt_gateway 内容移入此处
      bridge/
      services/
      protos/
      gateway.py
      ...
  qdata/
    adapters/
      qmt_remote_adapter.py  ← 调用 gateway/qmt/
      akshare_adapter.py
      ...
```

> ★ Insight ─────────────────────────────────────
> **模块归属的决策原则**：看谁依赖谁，谁为谁服务。
> qmt_gateway 表面上为整个 QMT 系统提供交易功能，
> 但实际上它是 qdata 层对接 QMT 终端的适配器，
> 所以归属 qdata 是更合理的分类。
> ──────────────────────────────────────────────────

---

*Generated from: 记忆文件 project_sps_deployment_arch.md*
