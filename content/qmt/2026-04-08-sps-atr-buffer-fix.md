# DevLog #009: SPS ATR 缓冲修复 — 均线滞后性的边界弹性设计

**日期**: 2026-04-08
**状态**: 已修复
**Tags**: sps, vector-engine, atr, trend-detection

---

## 问题描述

**现象**: `market_radar.py` 和 `vector_engine.py` 使用硬 MA 阈值判断趋势。大涨后价格仍略低于 MA20/MA60 时，系统错误判定为 BEAR/GREEN_WAVE。

**用户痛点**: 均线是滞后指标，一天的大涨不足以让 MA 穿越，但用户体感"明显不是空头"。

---

## 修复方案

### market_radar.py 修复

加 ATR(14) × 0.5 缓冲，价格贴线时 **BEAR → CHOPPY**

```python
# 修复前
if price < ma20: BEAR

# 修复后
if price < ma20 - atr * 0.5: BEAR
else: CHOPPY  # 贴线区域不轻易下定论
```

### vector_engine.py 修复

加 ATR × 0.15 缓冲，MA5 略低于 MA21 但在缓冲区内时标 **GRAY_ZONE** 而非 GREEN_WAVE

```python
# 修复前
if ma5 < ma21: GREEN_WAVE

# 修复后
if ma5 < ma21 - atr_15 * 0.15: GREEN_WAVE
else: GRAY_ZONE  # 在缓冲区内，观望
```

---

## 修复原理

> ★ Insight ─────────────────────────────────────
> **ATR 作为市场波动率的代理变量**，可以给均线的静态阈值增加动态弹性。
>
> 当市场波动大时，ATR 值变大，缓冲区间自动扩大；
> 当市场平稳时，ATR 值变小，缓冲区间自动收窄。
>
> 这比固定百分比的缓冲更合理，因为它反映了"当前市场正常波动范围"。
> ──────────────────────────────────────────────────

---

## -commit 记录

修复 commit: `f855b61`

相关文件：
- `market_radar.py`
- `vector_engine.py`

---

*Generated from: 记忆文件 + 代码分析*
