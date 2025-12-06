下面這一份就是你要的 **TAITS_S1 MASTER SPEC（D-1 超詳細規格書）**，你可以整份存成：

> `docs/TAITS_S1_MASTER_SPEC.md`
> 或直接當成專案的主 README（再加個簡短版 TL;DR 也行）

---

# 🧬 TAITS_S1 MASTER SPEC（超詳細總規格）

> **TAITS_S1 = Taiwan AI Trading System, Stage 1**
> 目標：完成一套「有完整架構、可逐步擴充到實盤」的台股多智能體交易系統。

---

## 0. 系統總目標與範圍

### 0.1 目標

1. **台股專用** 的量化交易系統，支援：

   * 歷史資料下載
   * 指標計算
   * 多策略訊號
   * 多智能體（Agents）決策
   * 回測 / Sandbox
   * 未來接富邦 API 實盤

2. **架構優先、功能可漸進**

   * 先完成 **最小可運行骨架（MVP）**
   * 再逐步補齊：策略全集、AI 模型、UI 儀表板、實盤接口。

3. **模組化 / Plugin 化**

   * 指標、策略、Agent 都是獨立模組，可**新增 / 關閉 / 替換**。
   * 回測引擎與交易引擎解耦，之後可連其他券商 API。

---

## 1. 專案總體架構

### 1.1 專案目錄

專案根名稱：`TAITS_S1`

```text
TAITS_S1/
│── main.py
│── requirements.txt
│── config.yaml            # 全域設定檔（資料來源、回測參數、交易模式等）
│
├── config/
│   └── settings.py        # 讀取 config.yaml，提供程式層級的 Config 物件
│
├── data_sources/
│   ├── base_loader.py     # 資料載入抽象基底
│   ├── yahoo_loader.py    # Yahoo Finance 台股 / 美股 / 指數
│   ├── twse_loader.py     # TWSE 官方 API
│   ├── finmind_loader.py  # FinMind API
│   ├── fallback_manager.py# 三層資料來源自動切換
│   └── cache_manager.py   # 本地快取與檔案結構
│
├── engine/
│   ├── orchestrator.py    # 系統主控（讀資料→算指標→跑策略→Agents→回測/交易）
│   ├── indicator_manager.py
│   ├── strategy_manager.py
│   ├── agent_manager.py
│   ├── signal_aggregator.py
│   └── data_validator.py
│
├── indicators/
│   ├── base_indicator.py
│   ├── trend/             # MA, EMA, GMMA, MACD...
│   ├── momentum/          # RSI, Stoch, CCI...
│   ├── volatility/        # ATR, HV, 波動率模型...
│   ├── volume/            # OBV, Volume Spike...
│   ├── candle/            # 各種 K 線形態偵測
│   ├── chip/              # 法人 / 融資券 / 大戶
│   └── ai/                # Kronos / LSTM / Transformer 特徵
│
├── strategies/
│   ├── base_strategy.py   # 策略介面 & 共用工具
│   ├── registry.py        # 285 策略註冊表
│   ├── ta/                # 技術面策略群
│   ├── chip/              # 籌碼策略群
│   ├── fundamental/       # 基本面策略群
│   ├── sector/            # 類股輪動策略群
│   ├── news/              # NLP / 新聞事件
│   ├── behavior/          # 行為金融 / 心理
│   └── ai/                # AI 預測型策略
│
├── agents/
│   ├── base_agent.py
│   ├── technical_agent.py
│   ├── chip_agent.py
│   ├── fundamental_agent.py
│   ├── news_agent.py
│   ├── sentiment_agent.py
│   ├── macro_agent.py
│   ├── pattern_agent.py   # K 線 / 結構
│   ├── ai_agent.py
│   └── risk_agent.py
│
├── backtest/
│   ├── backtester.py      # 單品種事件驅動 Backtest（已有最小版）
│   ├── position_manager.py# 多標的、組合部位管理（進階版再補）
│   └── report.py          # 報表與績效統計
│
├── trading/
│   ├── sandbox.py         # 模擬撮合（用來測 OrderManager + RiskManager）
│   ├── order_manager.py   # 下單紀錄/路由
│   ├── risk_manager.py    # 風控規則
│   └── broker_fubon.py    # 富邦 API（預留接口，之後接 SDK）
│
└── ui/
    ├── dashboard.py       # Streamlit 儀表板（已有最小版）
    ├── charts.py          # 畫價量/指標/部位
    └── components/        # UI 組件（篩選器 / 表格 / 卡片等）
```

---

## 2. 資料層（Data Layer）

### 2.1 資料來源（與之前 Data Layer chunk 對齊）

**台股 K 線（日/週/月/分鐘）**

* 來源：

  * Yahoo Finance：價格 / 指數 / 國際市場
  * TWSE：官方日 K
  * FinMind：完整日 K + 籌碼 + 財報

關鍵欄位：

```text
date (DatetimeIndex)
open, high, low, close
volume
turnover            # 成交金額，可選
adj_close           # 若有
```

**法人籌碼（三大法人）**

* 來源：FinMind
* 欄位（對單一股票／全市場）：

```text
date
foreign_buy, foreign_sell
investment_trust_buy, investment_trust_sell
dealer_buy, dealer_sell
three_institutions_sum
```

**融資券**

```text
date
margin_buy, margin_sell, margin_balance
short_sale_buy, short_sale_sell, short_sale_balance
```

**財報**

```text
quarter
EPS
revenue
revenue_yoy
gross_margin
operating_margin
net_margin
ROE
PE
market_cap
```

**新聞 / NLP**

```text
datetime
source
title
content
sentiment_score  # -1 ~ 1
impact_level     # 0~1 或 low/medium/high
tagged_events    # ex: ["法說會", "併購", "產業利多"]
```

**類股指數 / 市場寬度**

```text
date
sector_name
close
change_percent
volume
strength_score   # 0~1, 由指標計算出來
```

**國際市場 / 匯率 / VIX**

```text
date
close
change_percent
```

**AI 模型輸出（Kronos / LSTM / Transformer）**

```text
date
symbol
future_up_prob
future_down_prob
future_side_prob
predicted_trend   # "U" / "D" / "S"
model_name        # kronos / lstm / transformer
```

---

### 2.2 標準 DataFrame 格式

所有時間序列資料在系統內部統一為：

```python
index: DatetimeIndex（tz-naive, local time）
columns: 多欄組合，例如：
    [
        "open", "high", "low", "close", "volume",
        "foreign_buy", "margin_balance", ...,
        "rsi_14", "ema_20", ...,
        "ai_kronos_up_prob", ...
    ]
```

* **欄位命名規則：**

  * 指標：`<indicator_name>_<window>`
    例：`ema_20`, `sma_60`, `rsi_14`, `atr_14`
  * 籌碼：`chip_<source>_<field>`
    例：`chip_foreign_net`, `chip_margin_balance`
  * AI：`ai_<model>_<feature>`
    例：`ai_kronos_up_prob`, `ai_lstm_trend`

---

### 2.3 資料快取 & 檔案結構

本地資料根目錄預設：`./data/`

```text
data/
├── price/
│   └── {symbol}.parquet       # 台股個股日 K (含衍生欄位)
├── chip/
│   └── {symbol}_chip.parquet
├── margin/
│   └── {symbol}_margin.parquet
├── news/
│   └── news_{YYYY}.parquet
├── sector/
│   └── sector_index.parquet
├── macro/
│   └── {index}.parquet        # SOX, SPX, VIX, USD_TWD...
└── ai/
    └── ai_signals.parquet
```

`cache_manager.py` 提供 API：

```python
class CacheManager:
    def load(symbol: str, dtype: str) -> pd.DataFrame | None
    def save(symbol: str, dtype: str, df: pd.DataFrame) -> None
```

---

### 2.4 Fallback Manager

**優先順序：**

```text
Yahoo → TWSE → FinMind → Cache → Error
```

`fallback_manager.py` 對外暴露：

```python
class FallbackDataLoader:
    def get_price(self, symbol: str, start: str, end: str) -> pd.DataFrame:
        ...

    def get_chip(self, symbol: str, start: str, end: str) -> pd.DataFrame:
        ...

    # 之後可以擴充 get_news / get_sector / get_macro ...
```

內部流程：

1. 先查 Cache，有資料且覆蓋區間 → 直接回傳。
2. 沒有 → 依優先順序輪詢資料源。
3. 拉回資料 → 透過 `data_validator` 清洗 → 存 Cache。
4. 若全部失敗 → raise 明確的 `DataSourceError`。

---

## 3. 指標層（Indicator Layer）

### 3.1 指標總量與分類

* 技術指標：93
* 籌碼指標：18
* K 線型態：18
* 波動度：8
* 基本面特徵：12
* 新聞與事件：8
* 類股 / 市場結構：14
* 行為金融：20
* AI 特徵：10

> **總計 167+ 指標**（前面已完整列出，這裡不重複逐一貼）

### 3.2 實作模式

`indicators/base_indicator.py`：

```python
from abc import ABC, abstractmethod
import pandas as pd

class BaseIndicator(ABC):
    name: str

    @abstractmethod
    def compute(self, df: pd.DataFrame) -> pd.DataFrame:
        """接收價格 DataFrame，回傳新增欄位後的 DataFrame"""
        ...
```

範例：`indicators/ema.py`：

```python
class EMAIndicator(BaseIndicator):
    def __init__(self, window: int = 20, col: str = "close"):
        self.window = window
        self.col = col
        self.name = f"ema_{window}"

    def compute(self, df: pd.DataFrame) -> pd.DataFrame:
        df[self.name] = df[self.col].ewm(span=self.window).mean()
        return df
```

### 3.3 指標管理器

`indicator_manager.py` 負責：

* 從設定檔讀取**要啟用哪些指標**。
* 依順序執行 `indicator.compute(df)`。
* 確保**不重複計算**（快取結果）。

介面：

```python
class IndicatorManager:
    def __init__(self, config):
        ...

    def register_indicator(self, indicator: BaseIndicator) -> None:
        ...

    def compute_all(self, df: pd.DataFrame) -> pd.DataFrame:
        ...
```

---

## 4. 策略層（Strategy Layer）

### 4.1 285 策略總覽

你已經有完整 **285 策略列表 + 1~285 詳細邏輯 & pseudo-code**。
在規格書裡只需定義：

* **統一界面**
* **分類 + Meta 資訊**
* **Plugin 機制**

### 4.2 策略抽象介面

`strategies/base_strategy.py`：

```python
from abc import ABC, abstractmethod
from typing import Dict, Any
import pandas as pd

class StrategyOutput(Dict[str, Any]):
    """
    統一欄位：
    - signal_series: pd.Series，值 ∈ {-1, 0, 1}
    - score_series : pd.Series，連續分數
    - meta         : Dict，放附加資訊
    """

class BaseStrategy(ABC):
    id: int
    name: str
    category: str  # "TA", "Chip", "Fundamental", ...

    @abstractmethod
    def run(self, df: pd.DataFrame) -> StrategyOutput:
        ...
```

### 4.3 策略註冊表

`strategies/registry.py`：

```python
STRATEGY_REGISTRY: dict[int, type[BaseStrategy]] = {}

def register_strategy(cls):
    STRATEGY_REGISTRY[cls.id] = cls
    return cls
```

每個策略檔案內：

```python
@register_strategy
class SmaBreakoutStrategy(BaseStrategy):
    id = 1
    name = "SMA 突破"
    category = "TA"

    def run(self, df):
        # 依你之前的 pseudo-code 實作
        ...
```

### 4.4 StrategyManager

`engine/strategy_manager.py`：

```python
class StrategyManager:
    def __init__(self, enabled_ids: list[int] | None = None):
        self.enabled_ids = enabled_ids or list(STRATEGY_REGISTRY.keys())

    def run_all(self, df: pd.DataFrame) -> dict[int, StrategyOutput]:
        results = {}
        for sid in self.enabled_ids:
            strategy_cls = STRATEGY_REGISTRY[sid]
            strat = strategy_cls()
            results[sid] = strat.run(df)
        return results
```

---

## 5. Agent 層（TradingAgents）

### 5.1 10 大智能體

1. `TechnicalAgent`
2. `ChipAgent`
3. `FundamentalAgent`
4. `NewsAgent`
5. `SentimentAgent`
6. `MacroAgent`
7. `PatternAgent`（K 線 / 結構 / 纏論）
8. `ChanAgent`（如果拆開也可獨立，或併入 PatternAgent）
9. `AIAgent`
10. `RiskAgent`

### 5.2 Agent 輸入 / 輸出

共同介面：`agents/base_agent.py`：

```python
from abc import ABC, abstractmethod
from typing import Dict, Any

class AgentOutput(Dict[str, Any]):
    """
    統一欄位：
    - score      : float  (-1 ~ 1)
    - confidence : float  (0 ~ 1)
    - reason     : str
    - details    : Dict[str, Any]
    """

class BaseAgent(ABC):
    name: str

    @abstractmethod
    def run(self, symbol: str, context: Dict[str, Any]) -> AgentOutput:
        ...
```

**context** 內容由 `Orchestrator` 提供，可能包含：

```python
{
    "df":                價格 + 指標 DataFrame,
    "strategies":        {strategy_id: StrategyOutput, ...},
    "chip_data":         pd.DataFrame,
    "fundamental_data":  pd.DataFrame,
    "news_events":       List[NewsEvent],
    "macro_data":        pd.DataFrame,
    "ai_signals":        pd.DataFrame,
    "regime":            "bull" / "bear" / "sideways" ...
}
```

### 5.3 AgentManager 與 SignalAggregator

`agent_manager.py`：

```python
class AgentManager:
    def __init__(self, config):
        self.agents = [...]   # 根據設定建立各 agent

    def run_all(self, symbol: str, context: Dict[str, Any]) -> Dict[str, AgentOutput]:
        outputs = {}
        for agent in self.agents:
            outputs[agent.name] = agent.run(symbol, context)
        # summary 由 Aggregator 計算
        outputs["summary"] = SignalAggregator.aggregate(outputs)
        return outputs
```

`signal_aggregator.py`：

```python
class SignalAggregator:
    @staticmethod
    def aggregate(agent_outputs: Dict[str, AgentOutput]) -> AgentOutput:
        # 例：加權平均分數
        scores = []
        weights = []
        for name, out in agent_outputs.items():
            if name == "summary":
                continue
            w = out.get("confidence", 0.5)
            scores.append(out.get("score", 0.0) * w)
            weights.append(w)

        final_score = sum(scores) / (sum(weights) or 1)
        return AgentOutput(
            score=final_score,
            confidence=min(1.0, sum(weights) / (len(weights) or 1)),
            reason="Weighted average of all Agents",
            details={"raw": agent_outputs},
        )
```

---

## 6. Orchestrator（系統主控）

### 6.1 主要責任

1. 讀取設定 `config.yaml`
2. 初始化：

   * `FallbackDataLoader`
   * `IndicatorManager`
   * `StrategyManager`
   * `AgentManager`
   * `Backtester`
3. 依模式執行：

   * backtest / sandbox / live（之後）
4. 組裝 context 給 Agents

### 6.2 Backtest 流程（目前已有最小版）

簡化版流程：

```python
def run_backtest_single_symbol(self, symbol: str) -> BacktestResult:
    df_price = self.fallback_loader.get_price(symbol, start, end)
    df_price = self.indicator_mgr.compute_all(df_price)

    strategy_results = self.strategy_mgr.run_all(df_price)

    context = {
        "df": df_price,
        "strategy_results": strategy_results,
        # chip_data, fundamental_data, news, ... 之後加入
    }

    agent_outputs = self.agent_mgr.run_all(symbol, context)
    summary = agent_outputs["summary"]

    result = self.backtester.run(
        symbol=symbol,
        df=df_price,
        agent_summary=summary,
    )
    return result
```

---

## 7. 回測層（Backtest Layer）

### 7.1 現在的 MVP（已實作）

* 單一標的
* 只支援「做多 / 平倉」
* 使用 summary score > 0 作為是否持有
* 以 close 當成交價
* 計算：

  * Equity Curve
  * Total Return
  * Max Drawdown
* 回傳 `BacktestResult`（含 trades list）

### 7.2 未來擴充

在 `backtest/` 內再加：

* 事件驅動／多標的版 Backtester
* `PositionManager`

  * 每日更新：現金 / 部位 / 槓桿 / 保證金
* `report.py`

  * 年化報酬、Sharpe、Sortino、勝率、平均盈虧等
* 匯出結果：

  * CSV / Parquet
  * HTML 報表
  * 給 UI 繪圖用的 JSON

---

## 8. 交易層（Trading Layer）

### 8.1 Sandbox（模擬撮合）

`trading/sandbox.py`（之後實作）：

* 接收 `OrderManager` 中的 orders
* 以**最新行情**或**上一根 close** 模擬成交
* 更新：

  * 現金
  * 部位
  * 平均成本
* 把結果寫回 Portfolio 狀態（可以存到 `backtest/position_manager.py` 或獨立 `portfolio.py`）

### 8.2 Broker_Fubon（預留）

`trading/broker_fubon.py`：

* 只定義 **介面**，實際呼叫 SDK 之後再補：

```python
class FubonBroker:
    def __init__(self, config):
        ...

    def place_order(self, symbol: str, side: str, size: int, price: float | None):
        """下單到富邦 API"""

    def get_positions(self): ...
    def get_balance(self): ...
```

### 8.3 風控（RiskManager）

你已經有最小版，未來可以加入：

* 單日最大虧損、連續虧損次數
* 單檔停損 / 停利
* 組合層級 VaR / 波動限制
* 黑名單產業 / 股票

---

## 9. UI 層（Dashboard）

### 9.1 現有 MVP

`ui/dashboard.py`：

* 按一顆按鈕 → 呼叫 Orchestrator 假資料回測
* 顯示：

  * 總報酬
  * 最大回撤
  * 初始資金 / 期末資金
  * Equity Curve
  * 交易明細 DataFrame

### 9.2 未來視圖規劃

1. **Symbol 選單 + 日期區間選擇**
2. **策略 / Agent 開關**
3. **回測結果視圖**

   * Equity Curve
   * 回撤曲線
   * 每日盈虧 Bar Chart
4. **策略分解視圖**

   * 各策略 signal / score time-series
5. **Agent 分解視圖**

   * Technical / Chip / AI 的 score 與理由
6. **Live Monitor**

   * 實盤模式時顯示：即時行情 + position + 未平倉損益

---

## 10. 設定檔（Config）

### 10.1 `config.yaml` 大致結構

```yaml
mode: backtest   # backtest / sandbox / live

data:
  root_dir: "./data"
  sources_priority: ["yahoo", "twse", "finmind"]
  start_date: "2015-01-01"
  end_date: "2025-12-31"

indicators:
  enabled:
    - ema_20
    - ema_60
    - rsi_14
    - macd
    - atr_14
    # ...

strategies:
  enabled_ids: [1, 2, 3, 16, 21, 41, 144, 184, 235, 275, 283, 285]

agents:
  technical:
    weight: 1.0
  chip:
    weight: 1.0
  ai:
    weight: 1.5
  risk:
    weight: 2.0

backtest:
  initial_capital: 1000000
  risk_per_trade: 0.1

trading:
  broker: "sandbox"   # sandbox / fubon
  fubon:
    api_key: ""
    api_secret: ""
```

`config/settings.py` 讀取 YAML 並提供一個 `Settings` 物件給整個系統使用。

---

## 11. 非功能性需求（NFR）

1. **可讀性優先**

   * 小白也能看懂模組分工
   * 每個檔案頂部有 docstring 說明用途

2. **可測試性**

   * 重要模組（indicator, strategy, agent）之後都要可以在 notebook / 測試檔裡單獨運行。

3. **擴充性**

   * 新增策略 → 只要寫一個 class + `@register_strategy` 即可
   * 新增 Agent / 指標 → 不需要動到其他核心檔案

4. **安全性（未來實盤）**

   * 風控規則獨立在 RiskManager
   * LIVE mode 一定要有 double-confirm 或 dry-run 開關

---

## 12. 接下來適合的實作順序（給未來的你看）

1. ✅（已做）最小骨架 + Orchestrator + Dummy Backtest + UI MVP
2. ▶ 把 FallbackDataLoader 真的接上 Yahoo / TWSE / FinMind 其中一個（先 Yahoo）
3. ▶ 指標層：先完成最常用的一批（EMA / SMA / MACD / RSI / ATR / Volume）
4. ▶ 策略層：先實作 10～20 個高權重策略，透過 StrategyManager 跑起來
5. ▶ Agents：先做 TechnicalAgent + AIAgent + RiskAgent 基本版
6. ▶ Backtest 擴充到「真實歷史資料」
7. ▶ UI 加入 symbol 選單 + 指標/策略結果視圖
8. ▶ 再來才是富邦 API / AI 模型 etc.

---

如果你願意，下一步我可以幫你把這份 **MASTER SPEC** 切成幾個獨立 `.md` 文件（例如 `docs/01_master_spec.md`, `02_engineering_doc.md`），或是直接幫你設計一個 `docs/` 目錄結構，讓以後用 VSCode / Cursor 打開看起來更舒服。
