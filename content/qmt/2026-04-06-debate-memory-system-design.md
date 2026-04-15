# DevLog #006: 辩论引擎记忆系统设计 — SQLite + FTS5 三阶段演进

**日期**: 2026-04-06
**状态**: 实施中
**Tags**: its, debate-engine, memory-system, sqlite, fts5

---

## 现状分析

### 当前记忆机制的缺陷

辩论引擎 (`its/debate/engine.py`) 的"记忆"仅是**进程内的 round-to-round dict**:

```python
previous_round = {
    "bull": bull_output,   # 本轮多头输出
    "bear": bear_output,   # 本轮空头输出
    "arbiter": arbiter_output,  # 本轮裁判输出
}
```

**三大问题**：
1. 进程重启后记忆消失
2. 同一股票不同时间辩论，无法参考历史判断
3. 没有事后复盘（辩论决策 vs 实际走势）

---

## 业界方案对比

| 方案 | 来源 | 存储 | 检索方式 |
|------|------|------|---------|
| ChromaDB 向量记忆 | TradingAgents-CN | ChromaDB | Embedding 相似度 |
| QMD 混合搜索 | OpenClaw/Tobi Lütke | SQLite + FTS5 + sqlite-vec | BM25 + 向量 + LLM 重排 |
| **本方案** | QMT | SQLite + FTS5 | BM25 全文搜索（可升级向量）|

### 为什么不直接用 QMD / ChromaDB

- **QMD** 是 Node.js 工具，跨语言调用引入不必要的复杂度
- **ChromaDB** 是独立数据库服务，对当前项目规模（单机交易）过重
- **SQLite 零部署、Python 内置、性能足够**（单机万级记录）

---

## 三阶段演进方案

```
Phase 1 (P0)          Phase 2 (P1)           Phase 3 (P2)
────────────────────────────────────────────────────────────
SQLite 辩论历史存储  →  反思机制 (Reflection)  →  语义检索升级
    ↓                      ↓                      ↓
持久化 + 查询        事后评估 vs 实际走势      特征向量相似度
```

### Phase 1: SQLite 辩论历史存储

**目标**: 辩论结果持久化，支持按股票/日期查询历史。

**核心 API**:

```python
class DebateMemoryStore:
    def save(self, result: DebateResult) -> int: ...
    def get_recent(self, symbol: str, limit: int = 5) -> list[DebateRecord]: ...
    def get_by_date_range(self, start: str, end: str) -> list[DebateRecord]: ...
    def search(self, query: str, limit: int = 10) -> list[DebateRecord]: ...
```

**数据模型**:

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增主键 |
| symbol | TEXT | 股票代码 |
| name | TEXT | 股票名称 |
| price | REAL | 辩论时价格 |
| action | TEXT | BUY/SELL/HOLD/WATCH |
| winner | TEXT | BULL/BEAR/DRAW |
| reasoning | TEXT | 裁判理由（FTS5 索引） |
| bull_thesis | TEXT | 多头论点（FTS5 索引） |
| bear_thesis | TEXT | 空头论点（FTS5 索引） |

### Phase 2: 反思机制 (Reflection)

**目标**: 事后评估辩论决策 vs 实际走势，存储经验教训。

```
交易信号执行后 → T+1/T+N 获取实际走势
  ↓
Reflector.evaluate(debate_record, actual_outcome)
  ↓
LLM 分析: "判断是否正确？为什么对/错？"
  ↓
ReflectionRecord 存入 SQLite
  ↓
下次辩论时注入历史经验
```

### Phase 3: 语义检索升级

**目标**: 基于行情特征相似度召回历史经验，而非仅按股票代码。

**方案演进**：
- **短期**: 用 FTS5 + 关键词匹配（MA60支撑、MACD金叉 等）
- **中期**: 引入 sqlite-vec 扩展存储 embedding 向量
- **长期**: 对接本地 GGUF embedding 模型

> ★ Insight ─────────────────────────────────────
> **SQLite + FTS5 是一个被低估的组合**。
> 对于单机量化系统，FTS5 的 BM25 全文搜索已经足够满足语义召回需求，
> 无需引入沉重的向量数据库。sqlite-vec 扩展让未来升级到向量检索成为可能，
> 但可以等到真正需要时再升级。
> ──────────────────────────────────────────────────

---

## 集成点

### DebateEngine 集成

```python
class DebateEngine:
    def __init__(self, client, config, memory_store=None):
        self.memory_store = memory_store  # 新增

    def run_debate(self, symbol, context):
        # 新增: 查询历史记忆
        history = self.memory_store.get_recent(symbol) if self.memory_store else []
        # ... 注入到 prompt_ctx ...

        result = ...  # 原有辩论逻辑

        # 新增: 保存辩论结果
        if self.memory_store:
            self.memory_store.save(result)

        return result
```

### Prompts 增强

```python
def build_bull_prompt(context, previous_round=None, history=None):
    if history:
        parts.append("\n## 历史辩论记录")
        for record in history[-3:]:
            parts.append(f"- {record.created_at}: {record.action} "
                        f"(rating={record.rating}, conf={record.confidence:.0%})")
```

---

## 文件结构

```
its/
  memory/
    __init__.py
    store.py          # Phase 1: DebateMemoryStore
    reflection.py     # Phase 2: Reflector
    search.py         # Phase 3: SemanticSearch
    models.py         # 数据模型
```

---

## 实施优先级

| Phase | 价值 | 复杂度 | 优先级 |
|-------|------|--------|--------|
| Phase 1 | 避免重复辩论，节约 API 成本 | 低 | **P0** |
| Phase 2 | 从错误中学习，提升决策质量 | 中 | P1 |
| Phase 3 | 跨股票经验迁移 | 高 | P2 |

---

*Generated from: docs/MEMORY_SYSTEM_DESIGN.md*
