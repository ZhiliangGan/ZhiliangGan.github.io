# DevLog #010: Antigravity 文档评审 — Phase 3 完成确认 + Phase 4 改进建议

**日期**: 2026-04-06
**状态**: 评审完成
**Tags**: phase3, phase4, its, debate-engine, architecture-review

---

## Phase 3: QSS 实盘策略 — 全部已完成 ✅

| 文档任务 | 实际状态 | Commit |
|---------|---------|--------|
| 实现 Titan Resonance Strategy | `qss/strategy/resonance_strategy.py` 222行，三维共振（价值30+趋势40+择时30 - 波动率惩罚） | `0865571` |
| 策略防护状态机扩容 | `qss/core/signal.py` 已有 `stop_loss_price` + `metadata` 字段 | `0865571` |
| Chandelier Exit 吊灯止损 | `qss/risk/stop_loss.py` 143行，ATR 动态乘数 + 超级趋势检测 + 硬止损 max(吊灯线, 成本*0.92) | `0865571` |
| 测试覆盖 | `tests/test_phase3_strategies.py` 429行，22个测试 | `0865571` |

**结论**: Phase 3 已在 commit `0865571` 中全部完成。文档应标记为 DONE。

---

## Phase 4: ITS Harness 维度隔离 — 方向正确，需补充

### 文档诊断正确的问题

1. **"大锅炖"确实存在**: `engine.py:_build_prompt_context()` 把整个 `DebateContext` 一锅端进 dict
2. **Bull/Bear/Arbiter 收到相同数据**: `prompts.py` 中 `build_bull_prompt()` 和 `build_bear_prompt()` 都拿到同一个 `technical_summary`
3. **LLM 容易产生附和幻觉**: 当 Bull 说"MA60 支撑"时，Bear 因为也看到 MA60，可能只是反驳但不会从基本面角度独立思考

### 文档遗漏或需要修正的问题

#### 1. `context_builder.py` 根本不存在

文档说要"修改 `its/debate/context_builder.py`"，但该文件不存在。上下文构建逻辑目前内嵌在 `engine.py:_build_prompt_context()`。

**需要新建而非修改**。

#### 2. 数据模型层缺少基本面和情绪结构

`DebateContext` 当前只有:
```python
@dataclass
class DebateContext:
    technical: TechnicalData      # 技术指标 (已有)
    position: PositionData        # 持仓 (已有)
    recent_signals: list[dict]    # 近期信号 (已有)
    # ❌ 缺少 FundamentalData
    # ❌ 缺少 SentimentData
```

**隔离不能只靠 Prompt 层剪枝，必须先补数据模型**:
- `FundamentalData`: PE/PE_TTM/PB/ROE/营收增长率/净利润率...
- `SentimentData`: 新闻情感得分/量比/换手率异常/行业资金流向...

#### 3. 异步数据层已就绪，可直接复用

Phase 4 刚完成的异步基础设施可以直接为 ITS 提供多源并发数据:
- `_retry.py`: 指数退避重试 (AKShare 网络请求)
- `_batch.py`: Semaphore 并发获取多只股票
- `AsyncDataService`: 多源竞速 (AKShare + QMT + CSV)
- `DataFeed.get_pe_history()`: PE 分位数据 (已接入 ResonanceStrategy)

---

## 新增 Agent 角色定义

当前只有 Bull/Bear/Arbiter 三个角色，全部基于同一份混合数据。隔离后应该变为:

| 角色 | 数据范围 | 盲区 |
|-----|---------|------|
| Technical Agent | K线、MA/MACD/BOLL/SKDJ、成交量 | 看不到 PE/财务/新闻 |
| Fundamental Agent | PE/PB/ROE/财报/行业对比 | 看不到价格走势图形 |
| Sentiment Agent | 新闻情感/量比/资金流向/换手率 | 看不到具体技术指标数值 |
| Arbiter | 多空观点汇总 + 价格 (仅价格) | 不直接看原始数据，只看三方观点 |

---

## 三层隔离实现策略

```
Layer 1: Data Model 隔离
  ├── FundamentalData (新)
  ├── SentimentData (新)
  └── TechnicalData (已有)

Layer 2: Context Builder 路由
  ├── build_technical_context() → 只返回技术指标
  ├── build_fundamental_context() → 只返回基本面数据
  └── build_sentiment_context() → 只返回情绪数据

Layer 3: Prompt 盲区
  ├── Technical Agent prompt → 零财务/情绪关键词
  ├── Fundamental Agent prompt → 零技术图形关键词
  └── Arbiter prompt → 只看三方观点 + 价格
```

> ★ Insight ─────────────────────────────────────
> **隔离的本质是"强制盲区"**。
> 不是告诉 Agent"你应该忽略 X"，而是干脆不把 X 传给它。
> 这样才能真正避免 Agent 在多个维度之间做加权平均，
> 而是从每个维度独立的视角出发做纯粹判断。
> ──────────────────────────────────────────────────

---

## 下一步行动

1. ✅ Phase 3 文档标记为 DONE
2. ⏳ Phase 4 按"数据模型 → Context Builder → Prompt 隔离"顺序推进
3. ⏳ 利用已有异步数据层获取基本面和情绪数据
4. ⏳ 新增穿刺测试: 高 PE 烂图形 → 应被拒绝; 好图形烂基本面 → 也应被拒绝

---

*Generated from: docs/ANTIGRAVITY_REVIEW.md*
