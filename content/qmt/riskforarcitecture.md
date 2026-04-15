# QMT Architecture Risk Details

> 基于当前代码仓库现状整理，聚焦此前架构文档里提到的前三类风险。
> 时间: 2026-04-07
> 范围: `qmt/` 当前主仓代码、历史文档与运行脚本

## 1. 文档命名与实际目录漂移

### 1.1 现象

仓库已经明显从历史命名体系迁移到新的模块化命名体系，但大量旧文档仍使用老名字，导致“文档里的系统”和“代码里的系统”不是同一套口径。

当前对外推荐口径:

| 当前口径 | 代码目录 | 历史口径 |
| --- | --- | --- |
| `QTS` | `qss/` | `qmt_system/` |
| `SPS` | `sps/` | `wavemonitor/` |
| `TSS` | `tss/` | `search_essential/` |
| `qdata` | `qdata/` | `stock-qdata/` |
| `config` | `config/` | `quant_config/` |

### 1.2 具体问题

1. 根级架构文档仍以旧目录为主，容易让阅读者误以为仓库目录尚未迁移。
2. 子系统文档中混用了“旧目录名”和“新简称”，导致同一个模块在不同材料里有两到三个名字。
3. 一些蓝图文档已经开始做新旧映射，但没有成为统一事实来源，阅读时仍需要人工脑补转换。
4. 历史输出物中仍保留“`qmt_system/skills/wavemonitor/`”这类已经不符合当前结构的表述。

### 1.3 代码与文档证据

- [ARCHITECTURE.md](/Users/james/code/qmt/qmt/ARCHITECTURE.md:13) 仍写 `qmt_system / wavemonitor / stock-qdata / search_essential / quant_config`
- [ARCHITECTURE.md](/Users/james/code/qmt/qmt/ARCHITECTURE.md:96) 之后的大段目录树仍基于旧结构
- [Architecture.md](/Users/james/code/qmt/qmt/qss/docs/Architecture.md:11) 仍以 `qmt_system / search_essential / wavemonitor / stock-qdata` 为主名
- [Architecture.md](/Users/james/code/qmt/qmt/tss/Architecture.md:13) 仍把 `search_essential / wavemonitor / qmt_system` 当作主要模块名
- [ARCHITECTURE_HIGH_LEVEL_DIAGRAM.md](/Users/james/code/qmt/qmt/docs/ARCHITECTURE_HIGH_LEVEL_DIAGRAM.md:17) 图中仍使用 `stock-qdata / wavemonitor / quant_config`
- [integration-blueprint.md](/Users/james/code/qmt/qmt/docs/integration-blueprint.md:42) 已开始建立 `旧目录 -> 新简称` 映射，但这份映射尚未完全反向同步到其他文档

### 1.4 架构层影响

- 沟通成本上升: 老成员说旧名，新成员看新目录，讨论容易错位
- 汇报口径不稳定: 对外材料、PPT、代码目录三套话语并存
- 重命名评估容易失真: 无法快速分辨“文档已迁移”还是“代码已迁移”
- 影响自动化治理: 后续如果做目录扫描、依赖图、ADR 归档，命名不统一会让结果失真

### 1.5 建议

1. 指定一份主口径文档，推荐继续以 [QMT_CODE_ARCHITECTURE_20260406.md](/Users/james/code/qmt/qmt/docs/architecture/QMT_CODE_ARCHITECTURE_20260406.md) 作为对外事实来源
2. 在旧文档顶部统一加“历史文档 / 已迁移命名对照”提示，而不是默默保留旧名
3. 对外统一使用 `QTS / SPS / TSS / ITS / qdata`
4. 对内在过渡期保留说明: `QTS` 对应当前代码目录 `qss/`

## 2. `sys.path` 注入与隐式导入较多，包边界未完全硬化

### 2.1 现象

当前多个核心模块和运行脚本仍通过手工修改 `sys.path` 来完成跨目录导入。这说明仓库虽已开始形成模块边界，但尚未完全进入“稳定包 + 明确入口”的状态。

### 2.2 具体问题

1. 启动脚本依赖路径注入，自解释性较弱，换目录运行或被外部调度时更容易出问题。
2. 一些核心运行文件也在运行时修改 `sys.path`，说明边界问题不只存在于测试和一次性脚本。
3. `xtquant` 这类特殊依赖还要求使用 `append()` 而不是 `insert(0)`，否则会引发 `numpy` 版本覆盖问题，导入顺序已经影响实际运行结果。
4. 测试也通过修改 `sys.path` 才能通过，意味着当前测试环境没有完全模拟“真实包安装后”的使用方式。
5. 有些导入通过隐式根路径生效，短期灵活，但长期不利于拆包、发布和 CI 稳定化。

### 2.3 代码证据

核心运行链路中的显式路径注入:

- [main.py](/Users/james/code/qmt/qmt/qss/main.py:17)
- [backtest.py](/Users/james/code/qmt/qmt/qss/backtest.py:17)
- [wave_monitor.py](/Users/james/code/qmt/qmt/sps/wave_monitor.py:34)
- [scheduler.py](/Users/james/code/qmt/qmt/scripts/scheduler.py:28)
- [nightly_screener.py](/Users/james/code/qmt/qmt/scripts/jobs/nightly_screener.py:25)
- [run_zxg_debate.py](/Users/james/code/qmt/qmt/scripts/run_zxg_debate.py:27)

依赖 `xtquant` 时的运行时路径拼接:

- [data_feed.py](/Users/james/code/qmt/qmt/qss/core/data_feed.py:53)
- [trader.py](/Users/james/code/qmt/qmt/qss/core/trader.py:18)
- [qmt_executor.py](/Users/james/code/qmt/qmt/qss/executor/qmt_executor.py:18)
- [data_adapter.py](/Users/james/code/qmt/qmt/sps/data_adapter.py:95)

测试中的路径注入:

- [conftest.py](/Users/james/code/qmt/qmt/tests/conftest.py:1)
- [test_phase3_strategies.py](/Users/james/code/qmt/qmt/tests/test_phase3_strategies.py:13)
- [test_pe_adapter.py](/Users/james/code/qmt/qmt/tests/test_pe_adapter.py:17)
- [test_e2e_piercing.py](/Users/james/code/qmt/qmt/tests/test_e2e_piercing.py:22)

开发文档已直接说明导入顺序会影响运行:

- [developer_guide.md](/Users/james/code/qmt/qmt/qss/docs/developer_guide.md:476)

### 2.4 架构层影响

- 导入结果依赖启动方式: 同一份代码在不同 cwd、不同 shell、不同调度器下可能行为不同
- 包发布困难: 一旦想把 `qdata / qss / sps / tss` 变成可安装包，就要先清理这些隐式路径假设
- CI 与本地差异增大: 本地能跑不等于在干净环境里可复现
- 隐性命名冲突风险更高: 路径前后顺序不同会改变实际 import 到的模块
- 平台治理困难: 很难准确定义“官方入口”与“受支持运行方式”

### 2.5 建议

1. 先区分三类文件: 核心运行入口、一次性脚本、测试
2. 优先清理核心运行入口中的 `sys.path` 注入，测试和临时脚本可后置
3. 为 `qdata / qss / sps / tss` 明确对外 import 根和启动入口
4. 把 `xtquant` 相关路径处理收敛到单点适配层，不要在多个模块各自拼接
5. 在 CI 中增加“从干净环境按官方入口启动”的验证，而不是仅验证当前工作区可跑

## 3. 可选依赖较多，不同机器上的可运行能力不一致

### 3.1 现象

系统当前采用“能力优先、环境适配”的策略，同一功能会根据环境自动切换到不同依赖和不同数据源。这让系统很灵活，但也意味着不同机器上实际获得的是不同子集。

### 3.2 具体问题

1. Windows + QMT 环境可使用 `xtquant / xtdata / xttrader`，接近实盘能力。
2. Mac/Linux 研究环境更多依赖 `akshare / yfinance / csv / sina / aiohttp` 等替代链路。
3. 同一个“取数据”动作，可能来自 QMT、AKShare、本地 CSV、Sina，数据粒度、时效性、字段质量和稳定性都不完全一致。
4. 一些子系统在缺失依赖时会优雅降级，但降级后的行为差异并不总是被统一显式标记。
5. 当前更像“环境能力矩阵”，还不是“单一发行版”。

### 3.3 代码证据

环境检查脚本已把多项依赖视为 optional:

- [verify_env.py](/Users/james/code/qmt/qmt/scripts/verify_env.py:41)
- [verify_env.py](/Users/james/code/qmt/qmt/scripts/verify_env.py:100)
- [verify_env.py](/Users/james/code/qmt/qmt/scripts/verify_env.py:155)

统一数据层本身就是多源自动切换:

- [data_service.py](/Users/james/code/qmt/qmt/qdata/qdata/service/data_service.py:28)
- [pyproject.toml](/Users/james/code/qmt/qmt/qdata/pyproject.toml:11)

SPS 在运行时会从 QMT 退化到 Sina，再退到 CSV 或 AKShare:

- [data_adapter.py](/Users/james/code/qmt/qmt/sps/data_adapter.py:76)
- [data_adapter.py](/Users/james/code/qmt/qmt/sps/data_adapter.py:238)
- [data_adapter.py](/Users/james/code/qmt/qmt/sps/data_adapter.py:290)

TSS 的研究/回测路径依赖额外生态:

- [data_loader.py](/Users/james/code/qmt/qmt/tss/data_loader.py:22)
- [vnpy_backtest_runner.py](/Users/james/code/qmt/qmt/tss/vnpy_backtest_runner.py:110)
- [vnpy_wave_runner.py](/Users/james/code/qmt/qmt/tss/vnpy_wave_runner.py:91)
- [run_strategy_comparison.py](/Users/james/code/qmt/qmt/tss/run_strategy_comparison.py:59)

QTS 和调度层也会因依赖是否存在而切换形态:

- [main.py](/Users/james/code/qmt/qmt/qss/main.py:100)
- [scheduler.py](/Users/james/code/qmt/qmt/scripts/scheduler.py:39)
- [async_fetcher.py](/Users/james/code/qmt/qmt/sps/async_fetcher.py:12)

### 3.4 架构层影响

- 同名功能的行为并不完全等价，跨机器复现结果要打问号
- 回测、监控、执行看到的行情口径可能不同，给策略比较和问题排查带来噪声
- 上线前验证复杂: “本机可跑”并不等于“目标环境能力齐全”
- 运维文档难写: 需要描述的不只是安装步骤，还有能力分层和降级路径
- 测试矩阵扩大: 至少要区分研究环境、监控环境、QMT 执行环境

### 3.5 建议

1. 明确官方支持的三种运行形态: `Research`, `Monitor`, `Execution`
2. 每种形态给出最小依赖集、可选增强依赖和不支持能力
3. 在启动日志里显式打印当前激活的数据源与执行能力，不只在异常时降级
4. 对关键功能补“能力声明”，例如 `supports_live_trade`, `supports_realtime_quote`, `supports_research_backtest`
5. 将研究结果、监控结果、执行结果分别标记数据来源，减少对齐成本

## 4. 关于第四条风险的补充判断

此前表述“`ITS -> QTS` 集成偏弱耦合”容易引起误解。更准确的说法应该是:

- `ITS` 与 `QTS / SPS` 保持弱耦合，本身是合理架构选择
- `ITS` 负责筛选、评审、分层输出
- `QTS` 负责执行
- `SPS` 负责监控和推送
- `TSS` 在研究与回测场景下可以使用更宽的股票池，不必被 `ITS` 完全约束

真正值得关注的不是“耦合太弱”，而是下游如何消费 `ITS` 输出的契约尚未完全标准化，例如:

- `TOP5 / TOP20 / Watchlist / Blacklist` 是否成为统一对象
- `QTS` 是消费“最终投资池”还是“评审后的候选池”
- `SPS` 是监控 `ITS` 全池，还是只监控 `QTS` 执行池
- 回测场景何时允许使用超出 `ITS` 的更大全市场样本

如果后续 `ITS` 评审质量足够高，那么由 `ITS` 输出分层股票池，再由 `QTS / SPS` 按场景消费，是很自然的演进方向。
