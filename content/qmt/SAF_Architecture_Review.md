# SAF Architecture Review
> 策略自动化工厂 — 架构深度评审（修订版）
> 更新时间: 2026-04-14
> 评审范围: SAF / Strategy Factory / 各子系统联动

---

## 1. 定位与愿景

```
QMT 工作区
│
├── qdata/          统一数据层
├── qss/            量化策略系统
├── its/            舆情/情绪分析
├── sps/            盘中信号监控
├── tss/            策略研发回测
│
└── SAF/            策略自动化工厂
    ├── core/saf/   saf Python 包（核心引擎）
    └── strategy_factory/  策略工厂（批量评估）
```

---

## 2. 实际架构：双轨并行

```
strategy_factory/
│
├── run_evaluate.py        ← 批量评估入口（当前生产路径）
│   │
│   ├─ legacy/evaluate/engine.py     CrossSectionalBacktester ← 实际运行的回测引擎
│   ├─ legacy/evaluate/scorer.py    6D 评分 + 场景切片
│   └─ evaluate/fast_screen_policy.py  快筛策略
│
├── run_pipeline.py        ← 策略数据提纯入口（LLM 清洗 raw→JSON）
│   └── pipeline/runner.py
│
├── evaluate/
│   ├── engine_adapter.py  ⚠️ 存在但未接入 — SAF 引擎适配器
│   └── fast_screen.py
│
└── backtest_*.py         ← 专项回测工具（独立使用）
    ├── backtest_regime.py         Regime 三策略对比 (374L) ✅
    ├── backtest_regime_aware.py   Regime-Aware 回测 (499L) ✅
    ├── backtest_skills_v71.py     Skills V7.1 回测 (342L) ✅
    ├── backtest_sps.py            SPS 回测 (360L) ✅
    ├── backtest_sps_v2.py         SPS V2 回测 (325L) ✅
    ├── backtest_v7_vs_v8.py       V7 vs V8 对比 (289L) ✅
    └── test_tdx_api.py             TDX API 测试 (297L)
```

---

## 3. 双轨详解

### 3.1 Legacy 轨（✅ 当前生产路径）

```
run_evaluate.py
  │
  ├─ CrossSectionalBacktester   (legacy/evaluate/engine.py)
  │   ├─ load_data_yfinance()        ✅ Mac 主路径
  │   ├─ load_data_csv()             ✅ CSV fallback
  │   ├─ load_data_gateway()         ✅ QMT gateway
  │   ├─ _normalize_ohlcv()         ✅ 列名统一
  │   └─ signal_func(df) → equity   ✅ 截面回测
  │
  ├─ fast_screen_policy.py
  │   ├─ infer_profile()             ✅ 策略画像推断
  │   ├─ should_retry_rejected()     ✅ 重试策略
  │   └─ classify_*()                ✅ 分类逻辑
  │
  └─ legacy/evaluate/scorer.py
      ├─ score_strategy()             ✅ 6D 评分
      ├─ score_scenario_slices()     ✅ 场景切片评分
      └─ generate_report()            ✅ JSON/MD 输出
```

**特点**: 直接操作 pandas DataFrame，不依赖 `saf.data` 包，零抽象开销。

**策略路由与专用池**：Legacy 轨支持 futures/etf/cb 专用池路由（`plan_strategy_route`），拒绝不足以 symbol 的专用池标的，不再 fallback 到 equity 池。

### 3.2 SAF Engine 轨（✅ 已接入 run_evaluate.py）

```
run_evaluate.py --engine saf
  │
  └─ SAFEngineAdapter (evaluate/engine_adapter.py)
       │
       ├─ strategy_topology=time_series
       │    → run_time_series_backtest()  逐标的调用 TimeSeriesBundle → SAF 引擎
       └─ strategy_topology=cross_sectional
            → run_panel_backtest()         批量调用 PanelBundle → SAF 引擎

SAF/core/saf/engine/
  ├─ TimeSeriesExecutionEngine    ✅ 264行，时序单标的回测
  ├─ PanelExecutionEngine        ✅ 354行，截面多标的回测（含再平衡逻辑）
  ├─ PairsExecutionEngine        ✅ 411行，配对交易回测
  ├─ sandbox.py                 ✅ execute_signal_function()，30s 超时
  ├─ config.py                  ✅ CostModel (commission/slippage/stamp_duty)
  └─ protocols.py               ✅ BacktestRequest / BacktestResult

✅ 共同依赖: saf.data.models
   (StandardBar / TimeSeriesBundle / PanelBundle / PairBundle / Frequency)
   ✅ 已实现（见 core/saf/data/models/）
```

### 3.3 两轨对比 ✅

| 维度 | Legacy 轨 | SAF Engine 轨 |
|------|-----------|--------------|
| 状态 | ✅ 生产可用 | ✅ 代码完整 |
| 数据抽象 | 直接用 pandas | ✅ `saf.data` 包已实现 |
| 截面回测 | ✅ CrossSectionalBacktester | ✅ PanelExecutionEngine |
| 时序回测 | ❌ 无单独时序引擎 | ✅ TimeSeriesExecutionEngine |
| 配对回测 | ❌ 无 | ✅ PairsExecutionEngine |
| 连接方式 | 直接调用 | ✅ SAFEngineAdapter（--engine saf 接入） |

---

## 4. SAF/core/saf/ — saf Python 包详情

```
SAF/core/saf/          (包名: saf)
│
├── models/              StrategySpec / BacktestArtifact / enums         ✅
├── capability/          CapabilityResolver / ExecutionPlanner            ✅
│
├── engine/
│   ├── time_series.py   TimeSeriesExecutionEngine   ✅ 264行
│   ├── panel.py         PanelExecutionEngine       ✅ 354行
│   ├── pair.py          PairsExecutionEngine       ✅ 411行
│   ├── sandbox.py       execute_signal_function()  ✅ 148行
│   ├── config.py        CostModel                  ✅
│   ├── protocols.py     BacktestRequest/Result      ✅
│   └── vnpy_adapter.py VnPyAdapterEngine           ✅
│
├── evaluation/
│   ├── metrics.py       基础 + 扩展指标            ✅
│   ├── scorer.py        消除门 + 6D 评分           ✅
│   ├── slices.py        年度/状态切片 + 报告        ✅
│   ├── diagnostics.py   诊断                        ✅
│   └── failure_codes.py 失败码                      ✅
│
├── orchestrator/        BatchExecutor / Scheduler / JobState   ✅
├── strategy/            Parser / Compiler / StaticChecker      ✅
├── storage/            ArtifactStore / ManifestStore / CacheStore ✅
└── api/               FastAPI routes                        ✅

✅ 已实现（core/saf/data/models/）— Frequency / StandardBar / TimeSeriesBundle / PanelBundle / PairBundle / UniverseSnapshot
```

---

## 5. 核心缺失项

### P0 ✅ — `saf.data` 包已实现

```
core/saf/data/
├── __init__.py          # 公共导出
└── models/
    ├── __init__.py       # 子模块统一导出
    ├── frequency.py     # Frequency 枚举
    ├── standard_bar.py  # StandardBar
    ├── universe.py       # UniverseSnapshot
    ├── time_series.py    # TimeSeriesBundle
    ├── panel.py          # PanelBundle
    └── pair.py           # PairBundle
```

**验证**: 三引擎 import 链路已通过 smoke test：

```
TimeSeriesExecutionEngine ✓  ← TimeSeriesBundle
PanelExecutionEngine     ✓  ← PanelBundle + StandardBar
PairExecutionEngine      ✓  ← PairBundle + StandardBar
BacktestBundle           ✓  ← 三者联合类型
```

**注意**: `qdata/qdata/adapters/` 是 qdata 自身的数据适配层（akshare / tdx / qmt_remote / xtpython_futures / tdx_futures / qmt_etf / tdx_etf / tdx_cb / csv），**不属于** `saf.data`，是两个独立的适配器体系。

#### qdata 专用池适配器（2026-04-14 新增）

```
qdata/qdata/adapters/
├── xtpython_futures_adapter.py   # XTPython 期货主力合约 + 历史K线
├── tdx_futures_adapter.py        # TDX 期货 K线（无主力合约接口，用默认列表）
├── qmt_etf_adapter.py            # QMT ETF 下载 + 本地缓存
├── tdx_etf_adapter.py            # TDX ETF 筛选 + K线
└── tdx_cb_adapter.py             # TDX 可转债
```

对应专用池配置：
```
strategy_factory/config/pools/
├── futures_pool.csv   # 38 个期货品种（RB/HC/I/J/AU/AG...）
├── etf_pool.csv       # 28 个 ETF（510050/510300/159915...）
└── cb_pool.csv       # 24 个可转债
```

### P1 ✅ — SAFEngineAdapter 已接入 run_evaluate.py

```
evaluate/engine_adapter.py 存在，已通过 --engine flag 接入 run_evaluate.py。
run_evaluate.py --engine legacy (默认) → Legacy 引擎
run_evaluate.py --engine saf          → SAF 引擎

SAF 引擎路径:
  strategy_topology=time_series  → SAFEngineAdapter.run_time_series_backtest() (逐标的)
  strategy_topology=cross_sectional → SAFEngineAdapter.run_panel_backtest() (批量截面)

两种路径输出格式相同，统一经由 _aggregate_results + score_strategy 评分。
```

### P2 — 专项回测工具未集成

```
backtest_regime.py         独立工具，未接入 run_evaluate.py
backtest_regime_aware.py   同上
backtest_skills_v71.py     同上
backtest_sps.py           同上
backtest_sps_v2.py        同上
backtest_v7_vs_v8.py      同上
```

---

## 6. run_evaluate.py CLI 参数（生产可用）

```
run_evaluate.py [OPTIONS]

标的与时间:
  --symbols              标的代码 (默认: 510300.SSE)
  --start / --end       日期范围

股票池:
  --pool-file           股票池 CSV 路径
  --pool-limit          最大标的数量

专用池 (2026-04-14 新增):
  --futures-pool-file   期货池 CSV 路径
  --etf-pool-file       ETF 池 CSV 路径
  --cb-pool-file        可转债池 CSV 路径
  --enable-futures-routing  启用期货路由

策略限制:
  --limit               最大处理策略数

数据源:
  --data-source         csv | gateway
  --csv-dir             CSV 目录
  --gateway-strict-remote

回测参数:
  --engine             回测引擎: legacy (默认) | saf
  --cash                初始资金 (默认: 1000000)
  --commission          手续费率 (默认: 0.001)
  --slippage            滑点 bps (默认: 0.2%)
  --signal-timeout-sec  信号超时 (默认: 20s)

快筛:
  --fast-screen-size    快筛数量 (默认: 24)
  --fast-screen-min-valid-symbols

缓存与报告:
  --strategy-cache-dir  策略缓存目录 (默认: .cache/evaluate_strategy)
  --report-dir          报告输出目录 (默认: .cache/evaluate_results)
```

---

## 7. 6D 评分系统 ✅ 已完整实现

```
消除门 (任一触发 → 直接淘汰):
  MaxDD > 40% / Sharpe < 0.3 / CAGR < 0 / trades < 6 / win_rate < 20%

6D:
  Return Score (35%)    CAGR(40%) + Sharpe(30%) + ProfitFactor(30%)
  Robust Score (45%)   MaxDD(30%) + Calmar(25%) + RollingSharpeStd(25%) + Sortino(20%)
  Implement Score (20%) WinRate(30%) + PayoffRatio(30%) + HoldingDays(20%) + Turnover(20%)

Tier: S≥85 / A≥70 / B≥55 / C<55
```

---

## 8. 已实现组件总览

| 组件 | 文件 | 状态 |
|------|------|------|
| 截面回测引擎（含 cancel_futures=True 超时修复） | legacy/evaluate/engine.py | ✅ 生产可用 |
| 6D 评分 | legacy/evaluate/scorer.py | ✅ 生产可用 |
| TimeSeriesExecutionEngine | core/saf/engine/time_series.py | ✅ 代码完整 |
| PanelExecutionEngine | core/saf/engine/panel.py | ✅ 代码完整 |
| PairsExecutionEngine | core/saf/engine/pair.py | ✅ 代码完整 |
| SAFEngineAdapter | evaluate/engine_adapter.py | ✅ 已接入 run_evaluate.py |
| Regime 回测对比 | backtest_regime.py | ✅ 独立工具 |
| Regime-Aware 回测 | backtest_regime_aware.py | ✅ 独立工具 |
| Skills V7.1 回测 | backtest_skills_v71.py | ✅ 独立工具 |
| SPS 回测 | backtest_sps.py | ✅ 独立工具 |
| SPS V2 回测 | backtest_sps_v2.py | ✅ 独立工具 |
| V7 vs V8 对比 | backtest_v7_vs_v8.py | ✅ 独立工具 |
| TDX API 测试 | test_tdx_api.py | ✅ |
| 策略提纯流水线 | run_pipeline.py | ✅ LLM 清洗入口 |
| `saf.data` 抽象层 | core/saf/data/ | ✅ 已实现 |
| 期货池 CSV | strategy_factory/config/pools/futures_pool.csv | ✅ 38 品种 |
| ETF 池 CSV | strategy_factory/config/pools/etf_pool.csv | ✅ 28 ETF |
| 可转债池 CSV | strategy_factory/config/pools/cb_pool.csv | ✅ 24 可转债 |
| XTPython 期货适配器 | qdata/qdata/adapters/xtpython_futures_adapter.py | ✅ |
| TDX 期货适配器 | qdata/qdata/adapters/tdx_futures_adapter.py | ✅ |
| QMT ETF 适配器 | qdata/qdata/adapters/qmt_etf_adapter.py | ✅ |
| TDX ETF 适配器 | qdata/qdata/adapters/tdx_etf_adapter.py | ✅ |
| TDX 可转债适配器 | qdata/qdata/adapters/tdx_cb_adapter.py | ✅ |

---

## 9. 建议补齐路径

```
第一步: 专项回测工具集成 (P1)
  │
  └─ backtest_regime/regime_aware/sps 等接入统一流水线
```

---

## 10. 相关文档

| 文档 | 位置 |
|------|------|
| SAF 特性树 | SAF/docs/SAF_FEATURE_TREE.md |
| SAF 系统需求 | SAF/docs/SAF_SYSTEM_REQUIREMENTS.md |
| 策略工厂特性树 | SAF/strategy_factory/docs/strategy_factory_FEATURE_TREE.md |
| 统一特性树 | docs/FeatureTree.md |
| FeatureTree HTML | docs/FeatureTree.html |

---

*本评审基于代码逐行分析，各文件行数均已核实。如有出入请指出。*
