# QMT + xtquant 量化回测避坑指南

> 基于中证500多因子+ATR风控策略的真实回测经历总结
> 日期：2026-04-19

---

## 一、QMT 软件层面的坑

### 1. QMT Java UI 会破坏 .py 策略文件

**现象**：在 QMT Java 界面里打开、保存 .py 文件后，文件内容变成乱码或 base64 字符串。

**原因**：QMT Java 层在保存策略文件时会 base64 编码文件内容（可能是为了加密或防篡改），再次打开时解码异常导致文件损坏。

**解决方案**：
- 策略文件单独存放在 QMT 软件目录外（如 `D:\my_strategies\`）
- 用外部编辑器（VS Code）编辑策略，用命令行方式运行回测，完全绕过 QMT Java UI
- 核心原则：**QMT Java UI 只用来观察和下载数据，不要用它编辑策略文件**

---

### 2. QMT 内置回测引擎依赖交易系统配置

**现象**：尝试用 `stgentry.run_file()` 独立运行策略文件时报错 `subscribeFormula` 配置缺失。

**原因**：`stgentry.run_file()` 并不是一个纯数据API，它依赖 QMT 交易引擎的公式订阅、 benchmark 设置等交易上下文。

**解决方案**：
- 完全放弃 QMT 内置回测引擎
- 使用 **xtquant Python 包**直接连接 QMT 数据服务（mini 行情端口 58610）
- 自己实现回测循环，不依赖 stgentry

---

## 二、xtquant / xtdata 数据API的坑

### 3. `get_market_data` 的数据结构

**现象**：初学者很容易搞混返回格式。

`xtdata.get_market_data()` 返回结构：
```python
{
    "close": DataFrame(index=股票列表, columns=时间列表),   # 注意是 index=股票，columns=时间
    "volume": DataFrame(index=股票列表, columns=时间列表),
    ...
}
```
不是常见的 `index=时间, columns=股票`，而是**转置的**。

---

### 4. `fill_data=True` 不会自动下载缺失数据

**现象**：设置 `fill_data=True` 后，很多股票数据仍然是 NaN，误以为 fill_data 会联网补全。

**原因**：`fill_data=True` 只对**同一请求内**存在的 NaN 做前向填充（如停牌日），**不会**触发网络下载缺失的历史数据。

**解决方案**：
- 必须先确保本地 .DAT 缓存文件存在（QMT 会自动下载有订阅的股票数据）
- 或者用 `xtdata.download_data()` 预先下载
- 或者用 gateway 的 `xtdata_bridge.py` 里的自动下载逻辑

---

### 5. 本地 .DAT 缓存不完整

**现象**：用 `xtdata.get_market_data` 拉241只股票，203只返回全 NaN，只有38只有数据。

**原因**：QMT 的 mini 行情（mini market data）只缓存了**登录后下载过**的股票数据。如果从未在 QMT 里查看过某些股票，它们就没有 .DAT 缓存。

**解决方案**：
- 在 QMT 里手动订阅/下载全市场行情
- 或者减少候选池，只用有缓存的股票
- 或者使用 gateway 的 xtdata_bridge.py，它有自动下载逻辑

---

### 6. 成交额（amount）不是 `close × volume`

**现象**：策略里用 `close[-20:] * volume[-20:]` 计算20日平均成交额，发现所有股票都达不到5亿门槛，全被过滤掉了。

**原因**：
- `volume` 字段在 xtdata 里是**股数**（不是金额）
- 成交额（amount）= 股价 × 股数 × 100（A股1手=100股）
- 但 xtdata 的 `amount` 字段已经直接返回成交额（元），不需要再乘

**错误写法**：
```python
amount = np.mean(close_vals[-20:] * vol_vals[-20:])  # ❌ 错误，数量级差100倍
```

**正确写法**：
```python
# 方式1：用 amount 字段（推荐）
amount_df = xtdata.get_market_data(["amount"], stock_list=STOCK_LIST, ...)["amount"]
avg_amount = np.nanmean(amount_vals[-20:])

# 方式2：自己算 (volume * close * 100)
amount = np.mean(vol_vals[-20:] * close_vals[-20:]) * 100
```

---

### 7. 停牌股判断

**现象**：停牌股票价是 NaN，但选股时没有跳过，导致后续计算全部变成 NaN。

**解决方案**：
```python
if np.isnan(close_vals[-1]):  # 最新价是NaN = 停牌
    continue
```

---

## 三、pandas DataFrame 索引的坑

### 8. `df[stock][date]` 不是你想的那种访问方式

**现象**：代码 `close_df[symbol]["20200408"]` 报 KeyError，或者返回意外的数据。

**原因**：
```python
# xtdata 返回的 DataFrame 结构：
#   index = 股票列表（stock_list）
#   columns = 时间列表（timelist）

# ❌ 错误写法1：df[stock] 会先匹配 columns（日期）
df["600010.SH"]  # KeyError! 因为没有叫 "600010.SH" 的列

# ❌ 错误写法2：df[stock][date] 的 [stock] 部分失败
df["600010.SH"]["20200408"]  # KeyError!

# ✅ 正确写法：用 .loc 明确指定 index → columns
df.loc["600010.SH", "20200408"]

# ✅ 或者：
df.loc["600010.SH"]["20200408"]  # 先取行（Series），再取列
```

**核心原则**：xtdata 返回的 DataFrame 是**宽表**（index=股票，columns=时间），访问股票必须用 `.loc[index, column]`，不要用 `df[stock]`。

---

## 四、回测逻辑的坑

### 9. 调仓时用 `iloc[-1]` 取的是最后一根K线，不是当前K线

**现象**：回测从2020-02-12开始，第一天调仓（i=0）就试图买入2026年的股票（因为 `iloc[-1]` 永远取数据末尾）。

**原因**：回测循环里，`close_df` 是包含全量历史数据的 DataFrame，`iloc[-1]` 永远指向数据的最后一天。

**解决方案**：用日期字符串做列索引，不用位置索引：
```python
# ❌ 错误：永远取到数据末尾（最后一根K线）
px = float(close_df.loc[s].iloc[-1])

# ✅ 正确：用当前循环日期作为列索引
current_date = timelist[i]  # timelist[i] 是当前交易日
px = float(close_df.loc[s, current_date])
```

---

### 10. `_do_risk_control` 里用全局 positions 判断持仓

**现象**：ATR风控没有效果，没有触发止损。

**原因**：
```python
def _do_risk_control(current_date, close_df):
    held = [s for s, p in _g["positions"].items() if p["shares"] > 0]
```
`_do_risk_control` 在循环内调用，但 `_g["positions"]` 在循环开始前是空的，只有调仓后才有持仓。风控里的 held 永远是空列表（除非持仓股是当日刚买的）。

**解决方案**：在主循环里先获取当前持仓，再传入风控函数：
```python
for i, date in enumerate(timelist):
    held = [s for s, p in _g["positions"].items() if p["shares"] > 0]
    _do_risk_control(date, close_df, held)  # 把 held 传入
```

---

### 11. 每日净值记录在调仓前，不是调仓后

**现象**：回测报告中最终资产和初始资金一样，看不出增长。

**原因**：
```python
# ❌ 错误顺序：先记录，再调仓（净值记录的是调仓前的值）
_g["results"].append((current_date, total_value))  # 此时还是旧持仓
_rebalance(target, date, close_df)  # 调仓后现金/持仓已变

# ✅ 正确顺序：先调仓，再记录
_rebalance(target, date, close_df)
_g["results"].append((current_date, total_value))  # 记录调仓后的净值
```

---

### 12. 年化收益率计算错误

**现象**：年化收益率数值异常高（如400%+）。

**原因**：
```python
# ❌ 错误：分母用了总数据天数，而不是实际回测天数
annual_return = total_return / ((len(nav_series) / 242))

# ✅ 正确：应该用实际回测经过的自然年数
years = (end_date_idx - start_date_idx + 1) / 242
annual_return = total_return / years
```

---

## 五、回测结果异常的快速排查

| 症状 | 最可能的原因 |
|------|-------------|
| 所有股票"无数据" | 本地 .DAT 缓存不完整，fill_data 不会补下载 |
| 所有股票"成交额低" | 用 close×volume 而不是 amount 字段 |
| 选出的股票完全相同 | 选股用的是 `iloc[-1]`（最后一根K线），不是当前 |
| 风控完全不触发 | `_do_risk_control` 没有传入当前持仓列表 |
| 净值不变 | 净值记录在调仓之前 |
| KeyError: '600010.SH' | `df[stock]` 先匹配列名，用 `df.loc[stock]` |

---

## 六、最佳实践总结

1. **策略文件放在 QMT 目录外**，用外部编辑器 + 命令行运行
2. **用 xtquant 独立 API**，完全绕过 QMT 回测引擎
3. **用 `amount` 字段**而非自己计算成交额
4. **用日期字符串做索引**访问 DataFrame，不用 `iloc[-1]`
5. **持仓用列表传入**风控函数，不要在函数内部重新查询
6. **先调仓后记录净值**，顺序不能错
7. **NaN 价格必须跳过**，不能参与后续计算
8. **年化收益率**用实际回测的自然年数，不要用数据总长度
