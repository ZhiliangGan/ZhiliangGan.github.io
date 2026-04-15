# QMT 高层架构图

> 日期: 2026-04-06
> 说明: 本文档基于当前代码结构绘制，不沿用旧命名。
> 命名说明: 对外简称采用 **QTS**；当前代码目录与 import 路径仍为 `qss/`

## 1. 模块缩写与全称

| 缩写 | 英文全称 | 中文全称 |
|---|---|---|
| **QTS** | **Quantitative Trading System** | **量化交易系统** |
| **SPS** | **Strategy Push System** | **策略监控推送系统** |
| **TSS** | **Trading Strategy Search** | **交易策略搜索系统** |
| **ITS** | **Investment Target Selection** | **投资标的选择系统** |

## 2. 原始定位补充

- **QTS**: 交易执行主系统，负责真正的策略执行、风控与交易闭环
- **SPS**: 监控推送系统，负责低频监控和消息推送，**只推不交易**
- **TSS**: 策略搜索与回测系统，负责研究、验证和比较
- **ITS**: 投资标的筛选与评审系统，负责候选股筛选与评审，**看和评，不直接做**

## 3. 系统上下文图

```mermaid
flowchart LR
    User[交易员 / 研究员 / 调度脚本] --> QMT[QMT Quant Workspace]
    Market[QMT xtdata / AKShare / CSV / Sina / News] --> QMT
    QMT --> Notify[Feishu / Dashboard / 报表]
    QMT --> Broker[QMT 交易终端 / 模拟执行]
```

## 4. 当前代码高层架构图

```mermaid
flowchart TB
    subgraph Config[统一配置层]
        CFG1[config/base.yaml]
        CFG2[accounts / pools / strategies]
        CFG3[loader.py]
    end

    subgraph Data[数据与存储层]
        QD[qdata.service.DataService]
        ADP[QMTAdapter / AKShareAdapter / CSVAdapter]
        ASYNC[AsyncDataService]
        BAR[BarStore SQLite]
    end

    subgraph Trading[交易执行层]
        DF[DataFeed]
        QTS[QTS\nQuantitative Trading System\n量化交易系统]
        BUS[EventBus]
        PROC[SignalProcessor]
        RISK[RiskManager]
        EXE[QMTExecutor / SimulatedExecutor / RemoteExecutor]
        API[FastAPI + Dashboard]
        NOTI[NotificationBridge]
    end

    subgraph Monitor[盘中监控层]
        SPS[SPS\nStrategy Push System\n策略监控推送系统]
        WM[wave_monitor]
        QF[QuoteFetcher]
        VE[vector_engine]
        HDB[HistoryDataManager]
    end

    subgraph Research[策略研究层]
        TSS[TSS\nTrading Strategy Search\n交易策略搜索系统]
        DL[data_loader]
        VBT[vnpy_backtest_runner]
        COMP[comparison / report scripts]
    end

    subgraph Intelligence[智能分析层]
        ITS[ITS\nInvestment Target Selection\n投资标的选择系统]
        DCB[DebateContextBuilder]
        DEB[DebateEngine]
        AIC[AIClient]
        MEM[DebateMemoryStore]
        GP[GoldenPoolManager]
        NEWS[news + sentiment]
    end

    subgraph Ops[编排运维层]
        SCH[scheduler.py]
        NIGHT[nightly_screener.py]
    end

    CFG1 --> QD
    CFG2 --> QD
    CFG3 --> QD
    CFG3 --> Trading
    CFG3 --> Research

    ADP --> QD --> BAR
    QD --> ASYNC

    QD --> DF
    QD --> SPS
    QD --> TSS
    QD --> ITS
    DF --> QTS

    QTS --> BUS --> PROC --> RISK --> EXE
    BUS --> NOTI
    BUS --> API

    QF --> WM
    VE --> WM
    HDB --> WM
    SPS --> WM

    TSS --> DL
    DL --> VBT
    COMP --> VBT

    ITS --> DCB
    DCB --> DEB --> AIC
    DEB --> MEM
    DEB --> GP
    NEWS --> DCB
    GP -. 候选池 / 事件 .-> QTS

    SCH --> NIGHT
    SCH --> QTS
    NIGHT --> ITS
```

## 5. 推荐部署图

```mermaid
flowchart LR
    subgraph StrategySide[策略与研究侧 Mac / Linux]
        TSS[TSS\nTrading Strategy Search]
        SPS[SPS\nStrategy Push System]
        ITS[ITS\nInvestment Target Selection]
        QTSL[QTS Logic\nQuantitative Trading System]
        QDATA[qdata]
    end

    subgraph ExecSide[执行侧 Windows + QMT]
        REMOTE[RemoteExecutorServer]
        QMTX[xtdata / xttrader / QMTExecutor]
        TERMINAL[QMT Client]
    end

    TSS --> QDATA
    SPS --> QDATA
    ITS --> QDATA
    QTSL --> QDATA
    QTSL -->|JSON-RPC/TCP| REMOTE --> QMTX --> TERMINAL
```

## 6. 阅读说明

- `qdata` 是全系统共享底座
- **QTS** 是最核心的在线执行域，也是唯一承担交易执行闭环的主系统
- **SPS** 与 **TSS** 分别承担监控和研究职责，其中 **SPS** 只推不交易
- **ITS** 负责智能分析、候选池沉淀和评审结论输出
- `scripts` 负责日常编排，不直接承载业务逻辑
