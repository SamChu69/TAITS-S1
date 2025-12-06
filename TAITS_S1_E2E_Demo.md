# 📘 TAITS_S1 Minimal E2E Demo Spec（單檔 E2E 最小可行版）

> 建議檔名：`docs/TAITS_S1_E2E_Demo.md`
> 用途：
>
> * 給自己看：知道「最小版本」實際長什麼樣子
> * 給 Cursor / VS Code Copilot 看：請它**照這個規格生程式**
> * 跑出第一輪：`python main.py` → 抓資料 → 算指標 → 跑一個簡易策略 → 做一個玩具回測 → 印結果

---

## 1. Demo 目標（這一版要做到哪裡？）

這個 Demo 只做一件事：

> **針對一檔台股（例如 2330.TW），跑一個簡單 SMA 突破策略，從 Yahoo 抓資料 → 算指標 → 產生買賣訊號 → 跑簡易回測 → 印結果。**

### 功能範圍（有 / 沒有）

✅ 有：

* 單一股票（例如：`2330.TW`）
* 單一資料來源：`yfinance`
* 單一策略：**SMA20 突破策略**（你前面 Strategy #1）
* 單一 Agent：TechnicalAgent（把策略的 signal 彙總）
* 一個很簡單的 Backtest（每天收盤價決定要不要持有）
* 統一由 `Orchestrator` 串起全部流程
* 執行：`python main.py` 可以跑完整流程，**不報錯**，並在終端輸出簡單報表

❌ 沒有（這一版先不做）：

* 多股票、多策略投票
* Sandbox / PaperTrading
* 富邦 API 實盤
* Regime / 多智能體完整架構
* 完整 285 策略 & 200+ 指標
* UI / Streamlit 儀表板

> Demo 的精神：**先有一隻能走路的小貓，再長成你那隻 TradingAgents 巨龍** 🐉

---

## 2. 專案名稱與資料夾結構（TAITS_S1）

📁 專案根目錄名稱：`TAITS_S1`

在根目錄下建立以下結構（這一版是「最小可運行骨架」＋ Demo）：

```bash
TAITS_S1/
│── main.py
│── requirements.txt
│── config.yaml
│
├── data_sources/
│   └── yahoo_client.py
│
├── engine/
│   ├── orchestrator.py
│   ├── indicator_manager.py
│   ├── strategy_manager.py
│   └── agent_manager.py
│
├── indicators/
│   ├── base_indicator.py
│   └── sma.py
│
├── strategies/
│   ├── base_strategy.py
│   └── sma_breakout.py
│
├── agents/
│   └── technical_agent.py
│
└── backtest/
    └── backtester.py
```

> 之後你要擴充成完整 TAITS_S1，就在這些資料夾下面慢慢長大就好，不會亂掉。

---

## 3. 各檔案「最小功能說明」

下面是這一版 Demo，每個檔案**一定要做的事情**（不用很強，只要「會動」＋結構正確）。

---

### 3.1 `config.yaml`

簡單定義 Demo 用的參數即可：

```yaml
symbol: "2330.TW"
start_date: "2020-01-01"
end_date: "2024-12-31"

strategy:
  name: "SMA20_Breakout"
  sma_window: 20

backtest:
  initial_capital: 1000000
  fee_rate: 0.0015
```

---

### 3.2 `main.py`

**唯一入口**：執行整套流程。

```python
from engine.orchestrator import Orchestrator

def main():
    orch = Orchestrator(config_path="config.yaml")
    orch.run()

if __name__ == "__main__":
    print("TAITS_S1 Minimal E2E Demo Running...")
    main()
```

---

### 3.3 `data_sources/yahoo_client.py`

負責：從 Yahoo 抓一檔股票的歷史 K 線資料並回傳 `pandas.DataFrame`。

**最小介面：**

```python
import yfinance as yf
import pandas as pd

class YahooClient:
    def fetch_ohlcv(self, symbol: str, start: str, end: str) -> pd.DataFrame:
        data = yf.download(symbol, start=start, end=end)
        # 統一欄位名稱
        data = data.rename(columns={
            "Open": "open",
            "High": "high",
            "Low": "low",
            "Close": "close",
            "Adj Close": "adj_close",
            "Volume": "volume",
        })
        data = data.dropna()
        return data
```

---

### 3.4 `indicators/base_indicator.py`

提供一個共同基底：所有指標 class 都繼承它。Demo 版可以很簡單：

```python
import pandas as pd
from abc import ABC, abstractmethod

class BaseIndicator(ABC):
    @abstractmethod
    def compute(self, df: pd.DataFrame) -> pd.DataFrame:
        """接收 df，回傳含有指標欄位的 df。"""
        pass
```

---

### 3.5 `indicators/sma.py`

實作一個最簡單的 SMA 指標。

```python
import pandas as pd
from .base_indicator import BaseIndicator

class SMAIndicator(BaseIndicator):
    def __init__(self, window: int = 20, price_col: str = "close", output_col: str = "sma20"):
        self.window = window
        self.price_col = price_col
        self.output_col = output_col

    def compute(self, df: pd.DataFrame) -> pd.DataFrame:
        df[self.output_col] = df[self.price_col].rolling(self.window).mean()
        return df
```

---

### 3.6 `strategies/base_strategy.py`

策略基底類別：所有策略都用同樣介面。

```python
import pandas as pd
from abc import ABC, abstractmethod

class BaseStrategy(ABC):
    @abstractmethod
    def generate_signals(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        接收含有指標的 df，回傳多一欄 'signal':
        1 = buy, -1 = sell, 0 = hold
        """
        pass
```

---

### 3.7 `strategies/sma_breakout.py`

這就是你最早定義的「SMA 突破策略（Strategy #1）」的程式版。

> 規則回顧：
>
> * 收盤價 > SMA20 → buy
> * 收盤價 < SMA20 → sell
> * 其他 → hold

```python
import pandas as pd
from .base_strategy import BaseStrategy

class SMABreakoutStrategy(BaseStrategy):
    def __init__(self, sma_col: str = "sma20"):
        self.sma_col = sma_col

    def generate_signals(self, df: pd.DataFrame) -> pd.DataFrame:
        df = df.copy()
        df["signal"] = 0
        buy_cond = df["close"] > df[self.sma_col]
        sell_cond = df["close"] < df[self.sma_col]

        df.loc[buy_cond, "signal"] = 1
        df.loc[sell_cond, "signal"] = -1
        return df
```

---

### 3.8 `engine/indicator_manager.py`

負責：接收原始 K 線 df → 跑所有指標 → 回傳加入指標的 df。
Demo 版只有 SMA20。

```python
import pandas as pd
from indicators.sma import SMAIndicator

class IndicatorManager:
    def __init__(self):
        self.indicators = [
            SMAIndicator(window=20, output_col="sma20")
        ]

    def compute_all(self, df: pd.DataFrame) -> pd.DataFrame:
        for ind in self.indicators:
            df = ind.compute(df)
        return df
```

---

### 3.9 `engine/strategy_manager.py`

負責：呼叫策略，產生 signal。

```python
import pandas as pd
from strategies.sma_breakout import SMABreakoutStrategy

class StrategyManager:
    def __init__(self):
        self.strategy = SMABreakoutStrategy()

    def run(self, df: pd.DataFrame) -> pd.DataFrame:
        return self.strategy.generate_signals(df)
```

---

### 3.10 `agents/technical_agent.py`

Demo 版：只有一個策略 → 這個 Agent 只是「轉手」signal，但保留架構。

```python
import pandas as pd

class TechnicalAgent:
    def evaluate(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        Demo：直接把 strategy 的 signal 當作技術面評分，
        加一欄 tech_score（= signal）。
        """
        df = df.copy()
        if "signal" in df.columns:
            df["tech_score"] = df["signal"]
        else:
            df["tech_score"] = 0
        return df
```

---

### 3.11 `engine/agent_manager.py`

負責：呼叫所有 Agents。
Demo：只有 TechnicalAgent。

```python
import pandas as pd
from agents.technical_agent import TechnicalAgent

class AgentManager:
    def __init__(self):
        self.tech_agent = TechnicalAgent()

    def run(self, df: pd.DataFrame) -> pd.DataFrame:
        df = self.tech_agent.evaluate(df)
        # 未來可以在這裡再加 chip_agent, ai_agent 等
        return df
```

---

### 3.12 `backtest/backtester.py`

超簡化版的回測：

* 初始資金 1,000,000
* 每次「持有 or 不持有」一單位（全倉 vs 空倉）
* signal:

  * 1 → 價格漲跌都算在績效裡
  * 0 → 持有現金（不動）
* 不考慮槓桿、分批，先做玩具版就好。

```python
import pandas as pd

class Backtester:
    def __init__(self, initial_capital: float = 1_000_000):
        self.initial_capital = initial_capital

    def run(self, df: pd.DataFrame) -> dict:
        df = df.copy()
        df["position"] = df["signal"].shift(1).fillna(0)  # 昨日 signal 決定今日持有狀態
        df["return"] = df["close"].pct_change().fillna(0)
        df["strategy_return"] = df["position"] * df["return"]
        df["equity_curve"] = (1 + df["strategy_return"]).cumprod() * self.initial_capital

        final_value = df["equity_curve"].iloc[-1]
        total_return = final_value / self.initial_capital - 1
        max_dd = (df["equity_curve"].cummax() - df["equity_curve"]).max()

        return {
            "final_value": float(final_value),
            "total_return": float(total_return),
            "max_drawdown": float(max_dd),
            "equity_curve": df["equity_curve"],
        }
```

---

### 3.13 `engine/orchestrator.py`

整套 Demo 的「大腦」，把所有東西串起來。

```python
import yaml
from data_sources.yahoo_client import YahooClient
from engine.indicator_manager import IndicatorManager
from engine.strategy_manager import StrategyManager
from engine.agent_manager import AgentManager
from backtest.backtester import Backtester

class Orchestrator:
    def __init__(self, config_path: str = "config.yaml"):
        with open(config_path, "r", encoding="utf-8") as f:
            self.config = yaml.safe_load(f)

        self.yahoo = YahooClient()
        self.ind_mgr = IndicatorManager()
        self.strat_mgr = StrategyManager()
        self.agent_mgr = AgentManager()
        self.backtester = Backtester(
            initial_capital=self.config["backtest"]["initial_capital"]
        )

    def run(self):
        symbol = self.config["symbol"]
        start = self.config["start_date"]
        end = self.config["end_date"]

        print(f"[Orchestrator] Fetching data for {symbol}...")
        df = self.yahoo.fetch_ohlcv(symbol, start, end)

        print("[Orchestrator] Computing indicators...")
        df = self.ind_mgr.compute_all(df)

        print("[Orchestrator] Running strategies...")
        df = self.strat_mgr.run(df)

        print("[Orchestrator] Running agents...")
        df = self.agent_mgr.run(df)

        print("[Orchestrator] Backtesting...")
        result = self.backtester.run(df)

        print("===== Backtest Result (Demo) =====")
        print(f"Final Equity : {result['final_value']:.2f}")
        print(f"Total Return : {result['total_return'] * 100:.2f}%")
        print(f"Max Drawdown : {result['max_drawdown']:.2f}")
        print("==================================")
```

---

### 3.14 `requirements.txt`

最小依賴：

```txt
yfinance
pandas
pyyaml
```

---

## 4. 給 Cursor 用的「指令範本」

你可以開啟 Cursor（或 VS Code + Copilot），在專案根目錄 `TAITS_S1/` 下新建一個 `docs/TAITS_S1_E2E_Demo.md`，把這份規格貼進去，然後對 Cursor 下類似這段指令：

> **（可以直接複製貼上）**

```text
你現在是我的 Python 架構師與實作工程師。

請在目前的 repo 中，依照 `docs/TAITS_S1_E2E_Demo.md` 的規格，幫我產生一個「最小可執行的 Demo 系統」。

目標：

1. 專案名稱為 TAITS_S1。
2. 建立文中指定的資料夾與 .py 檔案。
3. 每個 .py 檔內要填入可以正常 import、可以執行的程式碼（不要空檔案）。
4. 我執行 `python main.py` 時，要可以：
   - 從 yfinance 抓 2330.TW 的歷史資料
   - 計算 SMA20
   - 產生策略信號
   - 由 TechnicalAgent 處理
   - 由 Backtester 跑完一圈並印出最終資金與報酬

請嚴格遵守 `TAITS_S1_E2E_Demo.md` 裡面的結構與介面，可以適度補上型別註解與簡單的錯誤處理。
```

---

## 5. 你接下來可以怎麼做？

1. **先建一個資料夾 `TAITS_S1`，把這份規格存成：**
   `TAITS_S1/docs/TAITS_S1_E2E_Demo.md`
2. 開啟 Cursor → 打開這個專案資料夾
3. 把上面那段指令貼給 Cursor，叫它**幫你生成所有 .py 檔案**
4. 打開 Terminal，執行：

```bash
pip install -r requirements.txt
python main.py
```

如果有錯誤訊息，你可以截圖或貼 log 給我，我可以陪你一起 Debug，把這隻「第一代 TAITS_S1 小貓」養成功 🐱➡🐉

如果你願意，下一步我可以幫你做：
👉 **「第二層 Demo 規格」：在這個骨架上加上第二個策略 + 今天 / 昨天的大盤 regime 判斷 + 簡單 Risk Cap。**
