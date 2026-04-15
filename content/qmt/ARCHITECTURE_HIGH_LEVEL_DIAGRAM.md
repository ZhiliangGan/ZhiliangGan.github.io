# QMT 高阶架构图

下图为 QMT 项目的高阶架构视图（Mermaid）：

```mermaid
%%{init: {"theme":"neutral"}}%%
flowchart LR
  %% Data Sources
  subgraph DataSources [外部数据源]
    XT[QMT xtdata]
    AK[AKShare / CSV]
    API[第三方 API]
  end

  %% Ingestion Layer
  subgraph Ingest [数据接入层]
    StockQ[stock-qdata
(统一数据网关)]
    DataFeed[DataFeed
(qss/core/data_feed)]
  end

  %% Core Platform
  subgraph Core [核心平台]
    Event[EventBus
(异步事件总线)]
    Strategy[策略引擎
(策略生命周期、回放)]
    Executor[执行器
(Simulated / Real / Remote)]
    Monitor[wavemonitor
(信号监控)]
  end

  %% Storage & Observability
  subgraph Storage [存储与观测]
    DB[SQLite / Parquet]
    Metrics[Prometheus / Grafana]
    Logs[集中日志]
  end

  %% Config & Ops
  subgraph Ops [配置与运维]
    Config[quant_config
(配置中心/YAML)]
    CI[CI/CD]
    Notif[Feishu Webhook
(告警/通知)]
  end

  %% Flows
  XT -->|kline/tick| StockQ
  AK -->|kline/csv| StockQ
  API -->|quotes| StockQ

  StockQ -->|normalized data| DataFeed
  DataFeed -->|DataResult| Event

  Event -->|tick/bar| Strategy
  Strategy -->|order req| Executor
  Executor -->|order/trade events| Event

  DataFeed -->|persist| DB
  Strategy -->|results| DB
  Executor -->|fills| DB

  Event -->|alerts| Monitor
  Monitor -->|alerts| Notif
  Executor -->|trade alerts| Notif

  Config --> Strategy
  Config --> Executor
  CI -->|tests/build| Repo[(code repo)]

  Metrics -->|dashboards| Ops
  Logs -->|search| Ops

  style DataSources fill:#f8f9fa,stroke:#333,stroke-width:1px
  style Ingest fill:#eef6ff,stroke:#3b82f6
  style Core fill:#fff7ed,stroke:#f59e0b
  style Storage fill:#f0fdf4,stroke:#10b981
  style Ops fill:#f3f4f6,stroke:#6b7280
  classDef smallText font-size:12px;
  class StockQ,DataFeed,Event,Strategy,Executor,DB,Config smallText;
```

保存路径： `docs/ARCHITECTURE_HIGH_LEVEL_DIAGRAM.md`

如果需要我可以：
- 输出 PNG/SVG 导出版本
- 在 README 中嵌入该图
- 根据你的反馈调整节点或连线细节
