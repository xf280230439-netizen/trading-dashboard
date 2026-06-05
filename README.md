# TDX Daily Backtest

一个面向中国 A 股日线回测的最小可用框架，重点是让你快速试通达信风格指标，而不是先陷进复杂框架里。

当前版本的定位：

- 日线级别
- 单标的
- 多头单向
- 收盘出信号，下一交易日开盘撮合
- 支持 T+1、100 股整数倍、手续费、印花税、滑点
- 支持近似的涨跌停和停牌过滤
- 用 Python 写策略，但指标函数尽量贴近通达信写法

## 1. 安装

```powershell
uv sync --python 3.12
```

如果你想直接用 `AKShare` 下载 A 股日线，再装扩展依赖：

```powershell
uv sync --python 3.12 --extra akshare
```

## 2. 准备数据

仓库里已经放了一份可直接跑通流程的演示数据，注意它是合成样本，不是真实历史行情：

- [data/sample_synthetic_600519.csv](/E:/WorkBuddyProjects/trading/data/sample_synthetic_600519.csv)

### 方式 A：直接下载 A 股日线

```powershell
uv run tdx-bt download-akshare `
  --symbol 600519 `
  --start 2020-01-01 `
  --end 2026-04-19 `
  --adjust qfq `
  --output data\600519.csv
```

### 方式 B：用你自己的 CSV

CSV 至少要有这些列，英文或中文列名都可以：

- `date` / `日期`
- `open` / `开盘`
- `high` / `最高`
- `low` / `最低`
- `close` / `收盘`
- `volume` / `成交量`

可选列：

- `amount` / `成交额`
- `symbol` / `代码`
- `is_suspended`
- `up_limit`
- `down_limit`

## 3. 运行示例策略

均线金叉示例：

```powershell
uv run tdx-bt backtest `
  --csv data\600519.csv `
  --strategy strategies\ma_cross_demo.py:create_strategy `
  --symbol 600519 `
  --limit-pct auto `
  --output-dir runs\ma_cross_demo
```

生命线粘合示例：

```powershell
uv run tdx-bt backtest `
  --csv data\600519.csv `
  --strategy strategies\lifeline_confluence_demo.py:create_strategy `
  --symbol 600519 `
  --limit-pct auto `
  --output-dir runs\lifeline_confluence_demo
```

运行后会输出：

- `metrics.json`
- `trades.csv`
- `equity.csv`
- `daily_signals.csv`

## 4. 写你自己的通达信风格策略

复制 [strategies/template_strategy.py](/E:/WorkBuddyProjects/trading/strategies/template_strategy.py) 开始改。核心写法像这样：

```python
from tdx_daily_backtest.formula import CROSS, EMA, MA, REF
from tdx_daily_backtest.strategy import StrategySignals


def create_strategy(data):
    c = data["close"]

    fast = MA(c, 10)
    slow = MA(c, 30)
    trend = EMA(c, 60)

    buy = CROSS(fast, slow) & c.gt(trend)
    sell = CROSS(slow, fast) | c.lt(REF(trend, 1))

    return StrategySignals(
        name="my_strategy",
        buy=buy,
        sell=sell,
        columns={
            "FAST": fast,
            "SLOW": slow,
            "TREND": trend,
        },
    )
```

## 5. 已支持的通达信风格函数

- `MA`
- `EMA`
- `SMA`
- `REF`
- `HHV`
- `LLV`
- `SUM`
- `ABS`
- `MAX`
- `MIN`
- `STD`
- `COUNT`
- `EVERY`
- `EXIST`
- `CROSS`
- `BARSLAST`
- `FILTER`
- `IF`
- `VALUEWHEN`

## 6. 当前限制

- 目前是单标的，不是全市场多股票轮动。
- 不做分钟级和盘口级撮合。
- 涨跌停处理是日线近似，不是一字板级别的精细成交模拟。
- `limit-pct auto` 只按代码前缀粗分 10% / 20% / 30%，不会自动识别 ST 的 5%。
- 没有做复权正确性校验，数据质量仍取决于你的来源。

对“先验证指标逻辑，再不断改买卖规则”这个阶段，这套框架已经够快了。
