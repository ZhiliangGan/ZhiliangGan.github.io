# SAF 重构迁移方案

**日期**: 2026-04-13
**目标**: 将 SAF 提升到与 StrategeFactoryFlow/TSS 并列层次，废弃两者中与 SAF 重叠的部分
**范围**: `StrategeFactoryFlow`, `TSS`, `SAF` 三个项目的回测/数据/评估层

---

## 1. 当前架构状态

### 1.1 三个项目现状

```
qmt/
├── SAF/                          # 新共享内核（重构骨架）
│   ├── src/saf/
│   │   ├── capability/           # ✅ 策略能力检测
│   │   ├── data/                # ⚠️ 框架完成，实现缺失
│   │   ├── engine/               # ⚠️ 框架完成，时序实现待补全
│   │   ├── evaluation/           # ⚠️ 框架完成，评分实现缺失
│   │   ├── models/               # ✅ 强类型对象模型
│   │   ├── orchestrator/         # ✅ 编排层框架
│   │   ├── storage/              # ✅ 接口定义
│   │   └── strategy/             # ✅ 策略解析器
│   └── tests/
│
├── StrategeFactoryFlow/          # 策略工厂（待迁移/废弃）
│   ├── evaluate/
│   │   ├── engine.py            # ❌ 与 SAF engine 重复 → 废弃
│   │   ├── fast_screen_policy.py # ✅ 保留（SAF 暂无对应）
│   │   └── scorer.py            # ⚠️ 与 SAF evaluation 重复 → 迁移/废弃
│   ├── run_evaluate.py          # ✅ 保留（批量编排）
│   └── pipeline/                 # ⚠️ 部分迁移
│
└── tss/                          # 策略运营（待迁移/废弃）
    ├── data_loader.py            # ❌ 与 SAF data 重复 → 废弃
    ├── run_*_comparison.py       # ⚠️ 保留（多策略对比）
    ├── vnpy_backtest_runner.py   # ⚠️ SAF vnpy_adapter 替代
    └── ...
```

### 1.2 SAF 能力 vs 现有实现 (2026-04-13 更新)

| 模块 | SAF 现状 | StrategeFactoryFlow | TSS | 结论 |
|------|----------|---------------------|-----|------|
| **策略拓扑检测** | ✅ 完整 | ✅ 完整 | ❌ 无 | SAF 优，废弃 SFF |
| **FailureCode** | ✅ 完整 | ❌ 散落字符串 | ❌ 无 | SAF 优，废弃 SFF |
| **对象模型** | ✅ 完整 | ❌ pd.DataFrame | ❌ 无 | SAF 优 |
| **时序回测引擎** | ✅ 完整 | ✅ 完整 | ⚠️ 简单实现 | SAF 已可替代 SFF |
| **截面回测引擎** | ✅ 完整 | ✅ 完整 | ❌ 无 | SAF 已可替代 SFF |
| **配对回测引擎** | ⚠️ 骨架定义 | ❌ 无 | ❌ 无 | SAF 框架已定义，实现待补 |
| **vnpy 适配器** | ⚠️ 骨架定义 | ❌ 无 | ✅ 完整 | 迁移到 SAF vnpy_adapter |
| **数据加载** | ⚠️ 协议定义 | ❌ 直接用 DataService | ✅ 完整 | 迁移 data_loader 到 SAF |
| **评分系统** | ⚠️ 简化版 | ✅ 6维评分 | ❌ 无 | 迁移 6维到 SAF evaluation |
| **编排/批量** | ✅ 框架完整 | ✅ 完整 | ⚠️ 脚本 | SAF orchestrator 已可替代 |
| **快筛重试** | ❌ 无 | ✅ 完整 | ❌ 无 | SFF 保留或迁移到 SAF |

---

## 2. 迁移原则

### 2.1 三不原则

1. **不删除代码**：所有待废弃代码移入 `legacy/` 目录，保留历史
2. **不中断开发**：迁移期间现有流水线照常运行
3. **不破坏接口**：SAF 接口先稳定再对接

### 2.2 迁移顺序

```
Phase 1: 稳固 SAF 内核（共享层先完成）
Phase 2: 废弃 StrategeFactoryFlow 中对应模块
Phase 3: 废弃 TSS 中对应模块
Phase 4: 清理 legacy/ 目录
```

---

## 3. 完整迁移方案

### 3.1 Phase 1: 稳固 SAF 内核 ✅ 已完成 (2026-04-13)

**目标**: 补全 SAF 中缺失的关键实现，使其能够承接废弃模块

#### 已完成 ✅

| # | 任务 | 实现位置 | 状态 |
|---|------|----------|------|
| 1.1 | TimeSeriesExecutionEngine（含 equity curve） | `saf/engine/time_series.py` | ✅ 已完成 |
| 1.2 | PanelExecutionEngine（含 DataFrame 信号支持） | `saf/engine/panel.py` | ✅ 已完成 |
| 1.3 | Sandbox（超时、异常处理、DataFrame 支持） | `saf/engine/sandbox.py` | ✅ 已完成 |
| 1.4 | BacktestConfig + CostModel | `saf/engine/config.py` | ✅ 已完成 |
| 1.5 | CapabilityResolver + StrategyTopologyDetector | `saf/capability/resolver.py`, `detector.py` | ✅ 已完成 |
| - | CapabilityResult dataclass + SupportMatrix | `saf/capability/support_matrix.py` | ✅ 已完成 |
| - | JobExecutorImpl + BatchExecutor | `saf/orchestrator/orchestrator.py`, `batch.py` | ✅ 已完成 |
| - | RetryPolicy + RetryableError | `saf/orchestrator/retry.py` | ✅ 已完成 |

**Architecture**: `StrategyKind` (业务分类) → `CapabilityResolver` → `StrategyTopology` (执行路由) → `ExecutionEngine`

```
StrategyKind.TIME_SERIES     → StrategyTopology.TIME_SERIES     → TimeSeriesExecutionEngine
StrategyKind.CROSS_SECTIONAL → StrategyTopology.CROSS_SECTIONAL → PanelExecutionEngine
StrategyKind.PAIRS           → StrategyTopology.PAIR           → PairExecutionEngine (待实现)
```

#### 待完成 ⚠️

| # | 任务 | 实现位置 | 状态 | 优先级 |
|---|------|----------|------|--------|
| 1.6 | 迁移 6 维评分 | `saf/evaluation/scorer.py` | ⚠️ 待实现 | P1 |
| 1.7 | 迁移切片评分 | `saf/evaluation/slices.py` | ⚠️ 待实现 | P2 |
| 1.8 | PairBundle + PairExecutionEngine | `saf/engine/pair.py` | ⚠️ 骨架待实现 | P3 |

#### 废弃模块对照表（更新）

| 待废弃（SFF） | 替代（SAF） | 状态 |
|---------------|-------------|------|
| `evaluate/engine.py` 时序 | `engine/time_series.py` | ✅ SAF 已完成 |
| `evaluate/engine.py` 截面 | `engine/panel.py` | ✅ SAF 已完成 |
| `evaluate/scorer.py` | `evaluation/scorer.py` | ⚠️ SAF 简化版已有，6维待迁移 |
| `evaluate/fast_screen_policy.py` | 新建 `evaluation/fast_screen.py` | ❌ 待新建 |
| `pipeline/` | `orchestrator/` | ⚠️ 部分重叠 |

| 待废弃（TSS） | 替代（SAF） | 状态 |
|---------------|-------------|------|
| `data_loader.py` | `data/providers.py` | ⚠️ 协议定义已有，实现待迁移 |
| `vnpy_backtest_runner.py` | `engine/vnpy_adapter.py` | ⚠️ 骨架已有，实现待对齐 |

**剩余工作量估算**: 2-3 人天（原计划 5 人天，已完成大部分）

---

### 3.2 Phase 2: 迁移 StrategeFactoryFlow（2-3 天）

#### 2.1 目录重组

```
StrategeFactoryFlow/
├── SAF/                    # 保持（作为独立包）
├── SAF_new/                # 新建：指向 qmt/SAF/src/saf
│   └── (symlink or pyproject path)
├── legacy/                 # 新建：废弃代码归档
│   ├── evaluate/
│   │   ├── engine.py      # 移入（废弃）
│   │   └── scorer.py     # 移入（废弃）
│   └── pipeline/          # 移入（废弃）
├── evaluate/               # 重写：对 SAF 的 thin wrapper
│   ├── engine_adapter.py   # 新建：SAF engine 适配器
│   ├── scorer_adapter.py  # 新建：SAF scorer 适配器
│   └── fast_screen.py     # 保留（SAF 暂无）
├── run_evaluate.py        # 修改：调用 SAF_new
└── ...
```

#### 2.2 迁移步骤

**Step 1**: 创建 `legacy/` 目录，移入废弃代码

```bash
mkdir -p StrategeFactoryFlow/legacy/evaluate
mv StrategeFactoryFlow/evaluate/engine.py legacy/evaluate/
mv StrategeFactoryFlow/evaluate/scorer.py legacy/evaluate/
mv StrategeFactoryFlow/pipeline legacy/pipeline
```

**Step 2**: 创建 `evaluate/` 新实现（thin wrapper）

```python
# evaluate/engine_adapter.py
from saf.engine import TimeSeriesExecutionEngine, PanelExecutionEngine
from saf.engine.protocols import BacktestRequest, BacktestBundle

class SAFTimeSeriesAdapter:
    """SFF wrapper around SAF TimeSeriesExecutionEngine."""
    ...

class SAFPanelAdapter:
    """SFF wrapper around SAF PanelExecutionEngine."""
    ...
```

**Step 3**: 更新 `run_evaluate.py` 使用 SAF

**Step 4**: 保留以下模块（不迁移到 SAF）:
- `fast_screen_policy.py` → `evaluate/fast_screen.py`
- `run_pipeline.py` → 评估后处理，保留
- `pipeline/backfill_claims.py` → 迁移到 SAF 或保留

---

### 3.3 Phase 3: 迁移 TSS（1-2 天）

#### 3.1 目录重组

```
tss/
├── legacy/                 # 新建：废弃代码归档
│   ├── data_loader.py
│   ├── run_*_comparison.py  # 保留（多策略对比，非回测核心）
│   └── vnpy_*.py
├── SAF/                    # 新建：指向 qmt/SAF/src/saf
├── data/                   # 重写：SAF data provider 封装
│   └── provider_adapter.py
└── ...
```

#### 3.2 迁移步骤

**Step 1**: 创建 `legacy/` 目录

```bash
mkdir -p tss/legacy
mv tss/data_loader.py tss/legacy/
```

**Step 2**: 实现 `tss/data/` 基于 SAF 协议

**Step 3**: 废弃 `vnpy_backtest_runner.py`，使用 `SAF/engine/vnpy_adapter.py`

---

### 3.4 Phase 4: 清理与验证（1 天）

#### 4.1 最终目录结构

```
qmt/
├── SAF/                              # 统一共享内核
│   └── src/saf/
│       ├── capability/                # ✅ 完整
│       ├── data/                     # ✅ 完整（迁移后）
│       ├── engine/                   # ✅ 完整（迁移后）
│       ├── evaluation/                # ✅ 完整（迁移后）
│       ├── models/                   # ✅ 完整
│       ├── orchestrator/             # ✅ 完整（迁移后）
│       ├── strategy/                  # ✅ 完整
│       └── storage/                   # ✅ 完整
│
├── StrategeFactoryFlow/              # 策略工厂应用
│   ├── legacy/                       # 归档（可删除）
│   ├── evaluate/                     # SAF thin wrapper
│   │   ├── engine_adapter.py
│   │   ├── scorer_adapter.py
│   │   └── fast_screen.py
│   ├── run_evaluate.py               # 批量评估入口
│   └── ...
│
├── tss/                              # 策略运营应用
│   ├── legacy/                       # 归档（可删除）
│   ├── data/                         # SAF data wrapper
│   ├── strategies/                   # 策略池
│   └── ...
│
└── qdata/                            # 数据服务（保留，供 SAF 调用）
```

#### 4.2 验证清单

- [ ] SAF 时序引擎输出与 SFF 旧引擎一致（误差 < 0.01%）
- [ ] SAF 截面引擎输出与 SFF 旧引擎一致
- [ ] SAF 评分系统输出与 SFF 旧系统一致
- [ ] `run_evaluate.py` 端到端运行成功
- [ ] TSS 数据加载使用 SAF 协议
- [ ] 所有测试通过

---

## 4. 详细任务分解

### 4.1 Phase 1 任务（SAF 内核补全）✅ 已完成

| # | 任务 | 状态 | 优先级 |
|---|------|------|--------|
| 1.1 | 迁移 SFF `SignalBacktester` 逻辑到 SAF `TimeSeriesExecutionEngine` | ✅ 已完成 | P0 |
| 1.2 | 迁移 SFF `CrossSectionalBacktester` 逻辑到 SAF `PanelExecutionEngine` | ✅ 已完成 | P0 |
| 1.3 | 实现 `execute_signal_function` 超时与异常处理 | ✅ 已完成 | P0 |
| 1.4 | 迁移 `CostModel` 和成本计算到 SAF `engine/config.py` | ✅ 已完成 | P1 |
| 1.5 | CapabilityResolver + StrategyTopologyDetector | ✅ 已完成 | P0 |
| 1.6 | 迁移 SFF 6维评分到 SAF `evaluation/scorer.py` | ⚠️ 待实现 | P1 |
| 1.7 | 迁移切片评分到 SAF `evaluation/slices.py` | ⚠️ 待实现 | P2 |
| 1.8 | 实现 `PairBundle` 和 `PairExecutionEngine` 骨架 | ⚠️ 待实现 | P3 |

### 4.2 Phase 2 任务（SFF 迁移）

| # | 任务 | 负责人 | 优先级 | 依赖 |
|---|------|--------|--------|------|
| 2.1 | 创建 `legacy/` 并移入废弃代码 | - | P0 | 1.1, 1.2, 1.3 |
| 2.2 | 实现 `evaluate/engine_adapter.py` (SAF wrapper) | - | P0 | 1.1, 1.2 |
| 2.3 | 实现 `evaluate/scorer_adapter.py` (SAF wrapper) | - | P0 | 1.6 |
| 2.4 | 更新 `run_evaluate.py` 使用 SAF adapter | - | P0 | 2.2, 2.3 |
| 2.5 | 保留 `fast_screen_policy.py` 为 `evaluate/fast_screen.py` | - | P1 | 无 |
| 2.6 | 端到端验证 | - | P0 | 2.4 |

### 4.3 Phase 3 任务（TSS 迁移）

| # | 任务 | 负责人 | 优先级 | 依赖 |
|---|------|--------|--------|------|
| 3.1 | 创建 `tss/legacy/` 并移入废弃代码 | - | P0 | 1.5 |
| 3.2 | 实现 `tss/data/provider_adapter.py` | - | P0 | 1.5 |
| 3.3 | 废弃 `vnpy_backtest_runner.py`，使用 SAF adapter | - | P1 | 1.1, 1.2 |
| 3.4 | 端到端验证 | - | P0 | 3.2 |

### 4.4 Phase 4 任务（清理）

| # | 任务 | 负责人 | 优先级 | 依赖 |
|---|------|--------|--------|------|
| 4.1 | 删除 `legacy/` 目录（或保留备查） | - | P2 | Phase 2, 3 完成 |
| 4.2 | 更新文档 | - | P1 | Phase 2, 3 完成 |
| 4.3 | 全面测试验证 | - | P0 | Phase 4 完成 |

---

## 5. 影响评估

### 5.1 功能影响

| 废弃项 | 影响 | 缓解措施 |
|--------|------|----------|
| SFF `engine.py` | 时序/截面回测依赖 | SAF 补全前不删除 |
| SFF `scorer.py` | 评分系统依赖 | SAF 补全前不删除 |
| TSS `data_loader.py` | 数据加载依赖 | SAF 补全前不删除 |
| SFF `fast_screen_policy.py` | 快筛重试依赖 | 保留为独立模块 |

### 5.2 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| SAF 引擎输出与旧引擎不一致 | 策略排名变化 | 严格对比测试，误差 > 0.01% 则回退 |
| 迁移期间开发中断 | 影响日常工作 | Phase 1 可并行，不影响 SFF/TSS 运行 |
| SAF data provider 性能下降 | 回测速度变慢 | 验证性能无显著下降 |

### 5.3 依赖关系

```
Phase 1: SAF 内核补全（可 5 人并行）
├── 1.1 TimeSeriesEngine ← 无依赖，立即可启动
├── 1.3 Sandbox ← 无依赖，立即可启动
├── 1.5 DataProvider ← 无依赖，立即可启动
├── 1.8 PairEngine ← 无依赖，立即可启动
├── 1.2 PanelEngine ← 无依赖，立即可启动（可与1.1并行）
├── 1.4 CostModel ← 依赖 1.1
├── 1.6 Scoring ← 依赖 1.1, 1.2
└── 1.7 Slices ← 依赖 1.6

Phase 2: SFF 迁移 ← 依赖 1.1, 1.2, 1.3
Phase 3: TSS 迁移 ← 依赖 1.5
Phase 4: 清理 ← 依赖 Phase 2, 3 完成
```

---

## 6. 推荐执行顺序

### 第一步（今天即可开始）

创建 SAF 分支并补全 Phase 1.1-1.3（时序引擎核心）

### 第二步（明天）

完成 Phase 1.4-1.6（数据+评分）

### 第三步

Phase 2 SFF 迁移，废弃 `legacy/evaluate/engine.py`

### 第四步

Phase 3 TSS 迁移，废弃 `legacy/data_loader.py`

### 第五步

Phase 4 清理与验证

---

## 7. 一句话总结

> **以 SAF 为共享内核，废弃 `evaluate/engine.py`、`scorer.py`、`data_loader.py`，其余保留在应用层。**

---

## 8. 并行执行方案（更新 2026-04-13）

### 8.1 依赖关系图（已更新）

```
Phase 1: SAF 内核补全 ✅ 已完成
│
├── 1.1 TimeSeriesEngine ──────────┐ ✅ 已完成
├── 1.2 PanelEngine ───────────────┤ ✅ 已完成
├── 1.3 Sandbox ───────────────────┤ ✅ 已完成
├── 1.4 CostModel ─────────────────┘ ✅ 已完成
├── 1.5 CapabilityResolver ───────── ✅ 已完成
│
│    ⚠️ 剩余工作
│
├── 1.6 Scoring ───────────────────┐ ⚠️ 待实现
└── 1.7 Slices ───────────────────┘ ⚠️ 待实现（依赖1.6）
         │
         │    Phase 2: SFF 迁移 (依赖1.6)
         │    Phase 3: TSS 迁移 (依赖1.6)
         │         ↓
         │    可并行执行
```

### 8.2 Phase 1 并行任务分配（更新）

| # | 任务 | 状态 |
|---|------|------|
| **1.1** TimeSeriesEngine | ✅ 已完成 |
| **1.2** PanelEngine | ✅ 已完成 |
| **1.3** Sandbox | ✅ 已完成 |
| **1.4** CostModel | ✅ 已完成 |
| **1.5** CapabilityResolver | ✅ 已完成 |
| **1.6** Scoring | ⚠️ 待实现 |
| **1.7** Slices | ⚠️ 待实现 |
| **1.8** PairEngine | ⚠️ 待实现 |

| 任务 | 依赖 | 可并行 |
|------|------|--------|
| **1.1** TimeSeriesEngine | 无 | ✅ |
| **1.2** PanelEngine | 无 | ✅ |
| **1.3** Sandbox | 无 | ✅ |
| **1.5** DataProvider | 无 | ✅ |
| **1.8** PairEngine | 无 | ✅ |
| **1.4** CostModel | 1.1 | ⏳ 等 1.1 |
| **1.6** Scoring | 1.1, 1.2 | ⏳ 等 1.1+1.2 |
| **1.7** Slices | 1.6 | ⏳ 等 1.6 |

### 8.3 Phase 2/3 并行（更新）

| 人员 | 任务 | Phase 1 依赖 |
|------|------|--------------|
| **人 1** | Phase 2: SFF wrapper 适配 | 1.6 Scoring 完成后 |
| **人 2** | Phase 3: TSS wrapper 适配 | 1.6 Scoring 完成后 |

> 注：Engine 部分 (1.1-1.5) 已完成，Phase 2/3 可在 1.6 完成后立即开始。

### 8.4 推荐分工方案（更新 2026-04-13）

> Phase 1 核心已完成，剩余工作大幅减少

#### 简化分工（1-2 人即可）

```
人 1 (主):
├── Phase 1.6 Scoring (P1) ← 最高优先级
├── Phase 1.7 Slices (P2)
└── Phase 2: SFF wrapper 适配

人 2 (可选):
├── Phase 1.8 PairEngine (P3)
├── Phase 3: TSS wrapper 适配
└── DataProvider 实现对齐
```

**原 5 人并行方案已不需要** - Phase 1 核心引擎已由 SAF 重构完成。

### 8.5 任务启动条件（更新 2026-04-13）

| 任务 | 启动条件 | 状态 |
|------|----------|------|
| 1.1, 1.2, 1.3, 1.4, 1.5 | ✅ 已完成 | - |
| 1.6 6维评分 | **立即可启动** | ⚠️ 待实现 |
| 1.7 切片评分 | 1.6 完成后 | ⚠️ 待实现 |
| 1.8 PairEngine | 可并行 | ⚠️ 待实现 |
| Phase 2 SFF迁移 | 1.6 完成后可开始 engine 部分 | 等待 1.6 |
| Phase 3 TSS迁移 | 1.6 完成后可开始 | 等待 1.6 |

### 8.6 关键路径（更新 2026-04-13）

```
关键路径: 1.6 → 1.7 → Phase 2/3 → Phase 4
最短工期估算（已更新）:
- Phase 1: 1.1,1.2,1.3,1.4,1.5 ✅ 已完成
- Phase 1.6 (6维评分): 约 0.5 天
- Phase 1.7 (切片评分): 约 0.5 天
- Phase 2,3 (SFF/TSS wrapper): 1-2 天 (并行)
- Phase 4 (清理验证): 0.5 天
总计剩余: 2-3 个工作日（原估算 5 天，已大幅缩短）
```

### 8.7 风险控制（更新）

| 风险 | 缓解 |
|------|------|
| Phase 1 已完成 | ✅ 核心引擎已验证通过 |
| 1.6/1.7 延期 | 阻塞 Phase 2 的 scorer 部分，但 engine 部分可先做 |
| 人力不足 | 优先保证 1.6 (评分)，1.7/1.8 可后续补充 |
