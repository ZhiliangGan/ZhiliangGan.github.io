# DevLog #007: DfX 架构设计原则 — QMT 大一统系统工程纪律

**日期**: 2026-04-06
**状态**: 架构共识
**Tags**: architecture, design-principles, engineering-discipline

---

## 背景

随着 ATMS、WaveMonitor、TradingAgents 等实验性工程合体为**大一统量化基建**，代码体积正在膨胀。

为了抵御熵增，我们确立以下 **Design for X** 的系统级工程纪律。

---

## 一、Design for Maintainability — 以"减法"为核心

**痛点**：量化项目很容易留下大量"废弃的因子、没用的回测废稿、不再调用的模块"。

### 激进删除原则

> 只要是被 V7.1 Titan 或新版 qss 证明已超越的旧概念（如老版的 TA-Lib 逐行遍历、老版的提示词生成器），一旦替换，**连注释都不要留，直接物理删除文件**。

**理由**：在 Git 时代，代码库不应该做垃圾场。精简的代码库是对接替者（包括未来维护的 AI）最大的仁慈。

### 面向 Protocol 的解耦

**设计准则**：杜绝"意大利面条式"的深层类继承。

例如任何只需要历史行情的数据，只约束其入参必须符合 `typing.Protocol`：

```python
class BarProvider(Protocol):
    @property
    def open(self) -> pd.Series: ...
    @property
    def high(self) -> pd.Series: ...
    @property
    def low(self) -> pd.Series: ...
    @property
    def close(self) -> pd.Series: ...
```

而不是强迫它继承一个巨大的 `BaseDataFeed` 基类。

> ★ Insight ─────────────────────────────────────
> **Protocol vs 继承**：继承是"我是谁"，Protocol 是"我能做什么"。
> 用 Protocol 解耦的好处是：你只需要关心数据提供者有没有你需要的方法，
> 而不是关心它在类继承树中的位置。
> ──────────────────────────────────────────────────

---

## 二、Design for Reliability — 以"悲观主义"为底色

**金融系统与传统 Web 应用最大的区别**：宁可宕机停止交易，也绝不能因为状态错乱搞错仓位。

### Broker 真理同步模型

**痛点**：系统自己在 SQLite 里算盈利，一旦代码挂了重启，本地 SQLite 状态跟券商 QMT 系统里的真实持仓对不上。

**设计准则**：任何策略发出的买卖信号前，必须强制调用一次 Broker 同步接口确认真实持仓。

```
"眼见为实，系统内部账本仅作参考"
```

### Fail-fast 与护城河降级

- 绝不写无休止的 `while True: retry()` 掩盖错误
- 对于深层核心错误（如交易端口崩溃），应该抛出致命异常让主进程被操作系统干净利落地杀死并重启
- 遇到外部错误（如大模型提供商宕机），启动**物理降级**，由备用纯算力引擎接管防守

> ★ Insight ─────────────────────────────────────
> **Fail-fast  vs  Fail-safe**：在金融交易系统中，fail-fast（快速失败）往往更安全。
> 因为"带病存活"可能意味着错误的交易决策，而"重启"只是暂时的停机损失。
> ──────────────────────────────────────────────────

---

## 三、Design for Performance — 对单线程运行极限的敬畏

### 破除对象滥用

**痛点**：如果在高频 EventBus 中为全市场 5000 只股票每秒的 Tick 生成复杂的 Python 对象并传来传去，系统的 GC 会瞬间卡死主进程。

**设计准则**：
- 流经中央处理流水线的底层数据严禁包装为厚重的类实例
- 使用基础的 NamedTuple 或者纯粹的 Pandas DataFrame 进行矩阵运算
- 将 `vector_engine` 这种用 C 语言底座（Numpy）实现的向量运算优势发挥到极致

```python
# ❌ 错误：大量对象创建
class TickData:
    def __init__(self, symbol, price, volume):
        self.symbol = symbol
        self.price = price
        self.volume = volume

# ✅ 正确：向量化矩阵运算
bars_df = pd.DataFrame({"close": close_arr, "volume": volume_arr})
result = vector_engine.calculate(close_arr, volume_arr)
```

---

## 四、Design for Observability — 可观测性设计

**痛点**：一个无头（Headless）交易系统在云端默默运行，如果我们只能在它亏钱时才发现它出错了，那是架构的悲哀。

### 系统心跳脉搏

利用 OpenClaw / 飞书 webhook 通道。每天早上开盘前与收盘后，自动推送一版包含：
- 拉取条数
- 大模型平均响应延迟毫秒数
- 系统可用内存
- 抛弃的脏读数据量

让你感知到系统是**"健康且紧绷的"**。

---

## 核心启示

> 架构并非代码堆砌，而是一种**"管理复杂度的艺术"**。
>
> 目前的当务之急，不是去"写多少新代码"，而是像在园艺修剪一样，
> 把 ATMS 过去冗长杂乱的枝叶大刀阔斧地砍掉，让核心树干吸收营养。

---

*Generated from: docs/DfX_ARCHITECTURE_PRINCIPLES.md*
