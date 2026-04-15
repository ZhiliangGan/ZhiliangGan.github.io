# QMT Detailed Runtime Flow

> 时间: 2026-04-07
> 目的: 画清楚 `ITS / QTS / SPS / TSS / qdata / scheduler` 的详细运行流程
> 说明: 本文分为两部分
> 1. 当前代码真实运行流程
> 2. 推荐演进后的分层池消费流程

## 1. 当前代码真实运行流程

### 1.1 运行说明

当前仓库里，最稳定、最清晰的主流程是:

1. 夜间由 `scheduler.py` 拉起 `nightly_screener.py`
2. `nightly_screener.py` 从种子池读取股票列表
3. `ITS` 用技术过滤 + AI 辩论筛出可关注标的
4. 结果写入 `config/golden_pool.yaml`
5. 次日 `scheduler.py` 拉起 `qss/main.py`
6. `QTS` 加载配置、数据源、执行器、风控和策略后进入交易时段运行
7. `SPS` 当前是相对独立的盘中监控链路，更多消费自己的 watchlist，而不是统一消费 `ITS` 分层池

### 1.2 详细运行流程图

```mermaid
flowchart TD
    subgraph Time0["夜间研究与洗池"]
        A1["scheduler.py<br/>01:00 定时触发"] --> A2["job_run_nightly_screener()"]
        A2 --> A3["scripts/jobs/nightly_screener.py"]
        A3 --> A4["读取 config/pools/zxg_a_share_pool.csv<br/>种子股票池"]
        A3 --> A5["ConfigManager.load()<br/>日志 / 配置初始化"]
        A3 --> A6["DataFeed(settings)<br/>经 qdata / xtdata / CSV 获取日线"]
        A3 --> A7["AIClient<br/>LLM 客户端"]
        A4 --> A8["遍历股票"]
        A6 --> A8
        A7 --> A8
        A8 --> A9["ITS.StockScreener.screen_symbol()"]
        A9 --> A10["IndicatorEngine<br/>趋势过滤 / 择时过滤"]
        A10 --> A11{"技术过滤通过?"}
        A11 -- 否 --> A12["skip_reason<br/>淘汰"]
        A11 -- 是 --> A13["DebateContextBuilder<br/>组装技术/价格/持仓上下文"]
        A13 --> A14["DebateEngine.run_debate()"]
        A14 --> A15["debate_to_signal()<br/>生成 BUY / HOLD / SELL 等信号"]
        A15 --> A16{"action in BUY/STRONG_BUY?"}
        A16 -- 否 --> A17["不入池"]
        A16 -- 是 --> A18["写入 golden_symbols[]<br/>symbol / name / confidence / risk / logic"]
        A18 --> A19["save_golden_pool()"]
        A19 --> A20["config/golden_pool.yaml"]
    end

    subgraph Time1["盘前调度"]
        B1["scheduler.py<br/>09:25 定时触发"] --> B2["job_start_qss_engine()"]
        B2 --> B3["subprocess 启动 qss/main.py"]
    end

    subgraph Time2["QTS 交易主链路"]
        C1["qss/main.py"] --> C2["ConfigManager.load()"]
        C2 --> C3["DataFeed(settings)<br/>可接 qdata / xtdata"]
        C2 --> C4["create_executor()<br/>QMTExecutor 或 SimulatedExecutor"]
        C4 --> C5["ExecutorPortfolioAdapter"]
        C5 --> C6["RiskManager"]
        C4 --> C7["SignalProcessor.start()"]
        C2 --> C8["create_strategy()<br/>按 settings.strategies 装载策略"]
        C8 --> C9["strategy.start()"]
        C2 --> C10["NotificationBridge.start()"]
        C2 --> C11["FastAPI init_state / uvicorn"]
        C1 --> C12["进入阻塞运行 while True"]
        C3 --> C8
        C6 --> C7
        C8 --> C7
        C7 --> C13["风控检查 / 订单路由 / 执行"]
        C13 --> C14["QMT 实盘下单 或 模拟执行"]
    end

    subgraph Time3["SPS 独立盘中监控链路"]
        D1["sps/wave_monitor.py"] --> D2["读取 .watchlist_a.json"]
        D1 --> D3["QuoteFetcher<br/>QMT xtdata 或 Sina"]
        D1 --> D4["HistoryDataManager"]
        D3 --> D5["实时行情"]
        D4 --> D6["历史K线"]
        D5 --> D7["WaveComputeEngine<br/>V7.1 Titan 向量计算"]
        D6 --> D7
        D7 --> D8["状态机 / 变化检测 / 告警格式化"]
        D8 --> D9["控制台 / 持久化 / 推送"]
    end

    subgraph Time4["TSS 研究与回测链路"]
        E1["TSS runner / backtest"] --> E2["更宽研究池 / 自定义 universe"]
        E2 --> E3["qdata / CSV / yfinance / vn.py"]
        E3 --> E4["回测结果 / 对比报告"]
    end

    A20 -. 次日作为候选输入 .-> C8
    A20 -. 当前也可供其他模块读取 .-> D2
    C3 -->|"统一数据入口"| E3
```

## 2. 当前流程的关键结论

- 当前最明确的“日切换”主线是 `scheduler -> nightly_screener -> golden_pool.yaml -> qss/main.py`
- `ITS` 已经承担了夜间评审和洗池职责
- `QTS` 是唯一真正承担执行闭环的系统
- `SPS` 目前更像并行监控服务，还没有完全切到统一的 `ITS` 输出契约
- `TSS` 保留更宽研究池是合理的，不必被 `ITS` 完全约束

## 3. 推荐演进后的详细运行流程

### 3.1 演进目标

你前面提的方向我认可，推荐把 `ITS` 定义为“分层池生产者”，而不是“强耦合驱动所有下游”的控制中心。

也就是说:

- `ITS` 负责评审和分层
- `QTS` 负责执行
- `SPS` 负责监控和推送
- `TSS` 负责研究和回测

真正需要统一的是“池对象”和“消费契约”。

### 3.2 推荐分层池消费流程图

```mermaid
flowchart TD
    F1["种子池 / 外部候选来源<br/>watchlist / 自选池 / 主题池 / 事件池"] --> F2["ITS 评审入口"]
    F2 --> F3["技术过滤<br/>趋势 / 择时 / 风险前置筛选"]
    F3 --> F4["多视角辩论<br/>technical / fundamental / sentiment / arbiter"]
    F4 --> F5["ITS Memory<br/>历史评审结果 / lesson / 命中率反馈"]
    F5 --> F6["分层池生成器"]

    F6 --> G1["research_universe<br/>研究全集"]
    F6 --> G2["reviewed_pool<br/>已评审候选池"]
    F6 --> G3["monitor_pool<br/>重点监控池"]
    F6 --> G4["investable_top20<br/>可投资池"]
    F6 --> G5["execution_top5<br/>执行池"]

    G1 --> H1["TSS 消费<br/>策略搜索 / 样本外回测 / 参数比较"]
    G2 --> H2["人工复核 / 投研复盘"]
    G3 --> H3["SPS 消费<br/>盘中波段监控 / 告警推送"]
    G4 --> H4["QTS 候选输入<br/>策略可见 universe"]
    G5 --> H5["QTS 高优先级执行池"]

    H4 --> I1["QTS 策略层"]
    H5 --> I1
    I1 --> I2["SignalProcessor"]
    I2 --> I3["RiskManager"]
    I3 --> I4["Executor<br/>QMT / Simulated"]

    H3 --> J1["实时行情 + 历史K线"]
    J1 --> J2["WaveComputeEngine"]
    J2 --> J3["状态变化 / 告警 / 推送"]

    G1 -. 可大于 ITS 最终投资池 .-> H1
    G4 -. 可作为执行候选，不等于必买 .-> I1
    G5 -. 允许成为更强约束的最终执行池 .-> I1
```

## 4. 推荐解读口径

如果要在文档或汇报里一句话解释这张图，我建议这样说:

`ITS 负责把更大的候选股票宇宙评审成分层股票池，TSS 负责研究验证，SPS 负责盘中监控，QTS 只消费适合进入交易闭环的那一部分池子。`

## 5. 相关代码入口

- [nightly_screener.py](/Users/james/code/qmt/qmt/scripts/jobs/nightly_screener.py)
- [scheduler.py](/Users/james/code/qmt/qmt/scripts/scheduler.py)
- [main.py](/Users/james/code/qmt/qmt/qss/main.py)
- [wave_monitor.py](/Users/james/code/qmt/qmt/sps/wave_monitor.py)
- [screener.py](/Users/james/code/qmt/qmt/its/screener.py)
- [golden_pool.py](/Users/james/code/qmt/qmt/its/golden_pool.py)
