# TAITS-S1
Taiwan Alpha Intelligent Trading System — Multi-Agent + AI + Quant Trading Framework
# 🏆 **TAITS_S1 — 台灣阿爾法智能交易系統《終極規格整合版》（Ultra Final Master Specification）**

# 🚀 **《TAITS_S1 — 終極規格整合版》開始**

---

# ⭐ **第 1 章｜系統總覽（System Overview）**

TAITS（Taiwan Alpha Intelligence Trading System）

> **台灣股市專用的：
> 多策略 × 多智能體 × 多資料源 × AI 預測 × 回測 × 模擬 × 實盤 × 監控 × UI 控制台 的全自動智慧交易系統。**

系統最重要的方程式：

```
Final Decision  
=  Data → Indicators → Strategies → Agents → Aggregator → Confidence → Action
```

產出：

```
BUY / SELL / HOLD  
+ 信心分數（0–1）  
+ 理由（多欄位）
```

---

# ⭐ **第 2 章｜最終系統架構（Final Global Architecture）**

```
TAITS_S1
│── main.py
│── config.yaml
│── requirements.txt
│── README.md

├── config/
│   └── settings.py

├── data_sources/
│   ├── base_loader.py
│   ├── yahoo_loader.py
│   ├── twse_loader.py
│   ├── finmind_loader.py
│   ├── fallback_manager.py
│   └── cache_manager.py

├── engine/
│   ├── orchestrator.py
│   ├── indicator_manager.py
│   ├── strategy_manager.py
│   ├── agent_manager.py
│   ├── signal_aggregator.py
│   └── data_validator.py

├── indicators/
│   ├── trend/
│   ├── momentum/
│   ├── volatility/
│   ├── volume/
│   ├── candle/
│   ├── chip/
│   ├── ai/
│   └── cbl/   ← 顧比倒數線 Count Back Line

├── strategies/
│   ├── base_strategy.py
│   ├── cbl_strategy.py     ← 新增顧比倒數線策略
│   ├── trend/
│   ├── breakout/
│   ├── reversal/
│   ├── volume/
│   ├── chip/
│   ├── fundamental/
│   ├── sector/
│   ├── chan/
│   ├── ai/
│   └── all_285_strategies.py (自動註冊)

├── agents/
│   ├── technical_agent.py
│   ├── chip_agent.py
│   ├── fundamental_agent.py
│   ├── news_agent.py
│   ├── sentiment_agent.py
│   ├── macro_agent.py
│   ├── pattern_agent.py
│   ├── chan_agent.py
│   ├── ai_agent.py
│   └── risk_agent.py

├── backtest/
│   ├── backtester.py
│   ├── position_manager.py
│   └── report.py

├── trading/
│   ├── sandbox.py
│   ├── order_manager.py
│   ├── risk_manager.py
│   └── broker_fubon.py   ← 富邦 API 模組

└── ui/
    ├── dashboard.py
    ├── strategy_switch.py  ← 策略開關 UI
    ├── charts.py
    └── components/
```

---

# ⭐ **第 3 章｜最終資料流程（Data Pipeline — Ultra Version）**

```
(1) 資料來源
Yahoo → TWSE → FinMind → Cache

(2) 資料驗證
缺值 → 補值 → 對齊 → 標準化

(3) 指標層（167 指標）
MA / EMA / GMMA / CBL / MACD / ADX / RSI / ATR / BB…

(4) 策略層（285 策略）
趨勢 / 突破 / 反轉 / 量價 / 籌碼 / 基本面 / NLP / AI / CBL…

(5) Agents（10）
每個 Agent 做獨立分析輸出 score 與 signal

(6) Aggregator
多智能體加權投票（可改模型或 AI ensemble）

(7) Orchestrator
產出 Final Decision：BUY/SELL/HOLD

(8) Backtest / Sandbox / Live
三層安全流程保護

(9) UI
策略開關、各 Agent 分數、K 線、回測等視覺化
```

---

# ⭐ **第 4 章｜資料來源（Data Sources — 3 層 Fallback）**

### **Yahoo Finance（主資料源）**

* 股票
* 匯率（USD/TWD）
* 類股指數 (SOX, NASDAQ)
* 盤後資料最快

### **TWSE**

* 官方資料
* 無 SSL 錯誤
* 多種股票資訊

### **FinMind**

* 三大法人
* 融資券
* 財報
* 產業資料

### **Cache（避免 API 重複抓取）**

---

# ⭐ **第 5 章｜指標層（Indicators — 167 指標，最終列表）**

分類如下：

## ① 趨勢指標（40）

SMA, EMA, WMA, HMA, GMMA, MACD, Zero-lag MACD, DMI, ADX, Ichimoku…

## ② 動能指標（20）

RSI, Stoch, CCI, ROC, KDJ, Ultimate Oscillator…

## ③ 波動度（15）

ATR, NATR, HV, GK, YZ, Parkinson…

## ④ 成交量（20）

OBV, A/D, Volume Spike, Volume Ratio…

## ⑤ K 線（18）

Hammer, Engulfing, Three White Soldiers…

## ⑥ 籌碼（18）

外資、投信、自營、集中度、融資券…

## ⑦ 基本面（12）

EPS YoY、毛利率等

## ⑧ NLP（8）

情緒分數、事件強度、新聞量…

## ⑨ AI（10）

LSTM、Transformer、Kronos 預測

## ⑩ 結構（6）

Pivot、Swing High/Low、Trend Line

## ⑪ CBL（2）

* CBL Uptrend Support Line
* CBL Downtrend Resistance Line

---

# ⭐ **第 6 章｜策略層（Strategies — 285 套最終分類）**

完整分類如下：

| 類別    | 策略數量 |
| ----- | ---- |
| 趨勢    | 93   |
| K 線   | 18   |
| 市場結構  | 18   |
| 成交量   | 14   |
| 籌碼    | 40   |
| 基本面   | 40   |
| 類股輪動  | 14   |
| NLP   | 20   |
| 行為心理  | 20   |
| AI 策略 | 20   |
| CBL   | 2    |

所有策略皆使用 plug-in：

```
/strategies/*.py → 自動註冊
```

---

# ⭐ **第 7 章｜智能體層（Agents — 10 大最終版）**

| Agent            | 功能                        |
| ---------------- | ------------------------- |
| TechnicalAgent   | 技術面全部策略                   |
| ChipAgent        | 籌碼面                       |
| FundamentalAgent | 財報                        |
| NewsAgent        | 新聞分類 NLP                  |
| SentimentAgent   | 市場情緒                      |
| MacroAgent       | SOX、NASDAQ、匯率、VIX         |
| PatternAgent     | K 線形態                     |
| ChanAgent        | 纏論                        |
| AIAgent          | LSTM, Transformer, Kronos |
| RiskAgent        | 風控、最大回撤、倉位管理              |

每個 Agent 輸出：

```
{
   "signal": BUY/SELL/HOLD,
   "score": 0~1,
   "reason": [...]
}
```

---

# ⭐ **第 8 章｜Orchestrator（總控大腦）**

功能：

* 整合所有 Agents
* 加權投票機制
* 跨策略一致性檢查
* 最終決策輸出

演算法：

```
final_score = Σ(agent_score × weight)
```

決策：

```
BUY  if score > 0.65
SELL if score < -0.65
HOLD otherwise
```

---

# ⭐ **第 9 章｜回測與模擬（Backtest / Sandbox）**

### Backtest Engine

* event-driven
* 單策略、多策略
* 分析盈虧、勝率、MDD、Sharpe

### Sandbox（策略隔離區）

* 新策略需連跑 21 天穩定度
* 避免 junk strategy 直接進 Live

---

# ⭐ **第 10 章｜Live Trading（富邦 API）**

包含：

* 登入
* 下單（整股/零股）
* 市價/限價
* 停損停利
* 委託回報
* 錯誤自動重試
* 風控

程式模組：

```
/trading/broker_fubon.py
```

---

# ⭐ **第 11 章｜UI（Streamlit Dashboard）**

功能：

* 策略開關（Trend / Breakout / Reversal / Chip / AI…）
* 指標顯示
* Smart Agent 雷達圖
* CBL 支撐線與 K 線疊圖
* 回測結果
* Live 訊號
* 系統健康指標

---

# ⭐ **第 12 章｜參考資料（Reference — Final Master List）**

## 🔹 指標與技術分析

* TA-Lib 官方文檔
* Steve Nison《Japanese Candlestick Charting Techniques》
* Bulkowski Pattern Study

## 🔹 纏論（Chan Theory）

* 通達信 Chanlun
* Chan 教學文檔

## 🔹 策略框架

* QuantConnect Lean
* Backtrader
* Zipline

## 🔹 多智能體

* TradingAgents-AI
* AutoGPT

## 🔹 AI 模型

* Kronos K-Line Model
* LSTM 時序模型
* Transformer Time Series

## 🔹 台股資料源

* TWSE Open API
* FinMind
* Yahoo Finance

---

# ⭐ **第 13 章｜參考程式（Reference Code）**

## 1. Data loader 類別

* YahooLoader（yfinance）
* TWSELoader（requests）
* FinMindLoader（API Token模式）

## 2. Indicators Template

```python
class BaseIndicator:
    def compute(self, df):
        pass
```

## 3. Strategy Template

```python
class BaseStrategy:
    def run(self, df):
        return {"signal": "HOLD", "confidence": 0.0, "reason": ""}
```

## 4. AI Agent Example

```python
class AIAgent:
    def analyze(self, df):
        prob = self.model.predict(df)
        return {"signal": "BUY" if prob>0.6 else "HOLD", "score": prob}
```

## 5. Orchestrator Template

```python
class Orchestrator:
    def run(self):
        self.load_data()
        self.run_indicators()
        self.run_strategies()
        self.run_agents()
        return self.aggregate()
```

---

# 🎉 **《TAITS_S1 — 終極規格整合版》**
