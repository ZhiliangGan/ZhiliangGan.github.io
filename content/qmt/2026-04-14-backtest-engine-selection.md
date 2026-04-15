# DevLog #001: 回测引擎选型辩论 — Qlib vs VectorBT vs 自研

**日期**: 2026-04-14
**状态**: 已定稿
**Tags**: architecture, backtest-engine, qlib, duckdb

---

## 背景

在 SAF 平台开发过程中，我们面临一个关键架构决策：**回测引擎层应该如何演进？**

已有的组件：
- `TimeSeriesExecutionEngine` — 时序执行框架
- `PanelExecutionEngine` — 截面执行框架
- `VnPyAdapterEngine` — vn.py CTA 适配器
- `BarStore (SQLite)` — K 线持久化
- `legacy/evaluate/engine.py` — 历史包袱

候选引入：
- **Qlib** — 截面评价 + IC/IR + 参数扫描
- **VectorBT** — 参数扫描
- **DuckDB** — 多数据源 ETL Hub

---

## 讨论焦点

### 1. VectorBT 的去留

**初始提案**：用 VectorBT 做参数扫描，替代手写参数枚举。

**辩论过程**：
- 支持引入方：VectorBT 的参数扫描 API 很成熟，可以节省大量开发时间
- 反对引入方：Qlib 已经覆盖了参数扫描能力，引入 VectorBT 会增加额外依赖

**最终结论**：**不引入 VectorBT**，理由是 Qlib 已覆盖其核心能力。

> ★ Insight ─────────────────────────────────────
> **依赖最小化原则**：在架构选型时，如果引入新依赖无法带来显著差异化价值，优先利用现有组件。
> Qlib 同时提供了 IC/IR 分析和参数扫描，一套依赖解决两个问题。
> ──────────────────────────────────────────────────

### 2. Qlib 的定位

**结论**：引入 Qlib 作为截面评价层。

**引入点**：

| Qlib 组件 | 用途 |
|-----------|------|
| `DataloaderCSV` | 读取 DuckDB 导出的 CSV |
| `BacktestTest` | 分组收益回测 |
| `Analyzer` | IC/IR/RankIC 计算 |
| `HPO` | 参数扫描（替代 VectorBT） |

### 3. DuckDB 的按需引入

**触发条件**：当 WaveMonitor / 007 等外部特征数据源需要 JOIN 对齐时。

```python
# 典型用法
hub = DuckDBHub()
hub.register("qmt", "qmt_bars.parquet")
hub.register("wave", "wave_features.parquet")
hub.register("sentiment", "sentiment.parquet")

df = hub.join_features(
    base_table="qmt",
    join_tables=["wave", "sentiment"],
    on=["date", "symbol"],
)
```

### 4. Legacy 引擎的废弃

**决策**：逐步废弃 `strategy_factory/legacy/evaluate/engine.py`，迁移到 `core/saf/engine/` 统一框架。

**过渡策略**：
- `run_evaluate.py --engine` 默认值从 `"legacy"` 改为 `"saf"`
- 保留 `--engine legacy` 选项，仅用于对比验证
- 不删除 legacy 文件，标注废弃

---

## 最终架构图

```
数据层
├── BarStore (SQLite)          ← QMT K线持久化（保留）
├── DuckDB (ETL Hub)           ← 多数据源 JOIN 对齐（新增，按需）
│   └── QMT / WaveMonitor / 007 → CSV
│
回测层
├── SAF Engine Layer (统一框架)
│   ├── TimeSeriesExecutionEngine  ← 时序信号（现状）
│   ├── PanelExecutionEngine       ← 截面执行（现状）
│   └── VnPyAdapterEngine         ← vn.py CTA（现状）
└── Qlib                          ← 截面评价 + IC/IR + 参数扫描（引入）
```

---

## 重构计划

| 阶段 | 任务 | 验收条件 |
|------|------|---------|
| Phase 0 | 废弃 legacy 引擎路由 | `run_evaluate.py --limit 10` 走新框架 |
| Phase 1 | 引入 Qlib，新增 `QlibEngine` | IC/IR 结果正确性验证 |
| Phase 2 | 实现 `calculate_panel_metrics` | 与 Qlib Analyzer 输出对比 |
| Phase 3 | DuckDB ETL Hub（按需） | WaveMonitor + 007 数据 JOIN 正确 |

---

## 关键决策记录

| 决策 | 理由 |
|------|------|
| 不引入 VectorBT | Qlib 已覆盖参数扫描能力 |
| 引入 Qlib | IC/IR/RankIC 截面评价 + 参数扫描，一站式解决 |
| 废弃 legacy | 统一到 `core/saf/engine/` 框架 |
| DuckDB 按需引入 | 只在多数据源 JOIN 场景触发 |

---

*Generated from: SAF/discuss_backtest_engines.md*
