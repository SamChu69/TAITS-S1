# TAITS-S1
Taiwan Alpha Intelligent Trading System — Multi-Agent + AI + Quant Trading Framework
# ✅ **TAITS_S1 —《台灣阿爾法智能交易系統》最詳細內容的終極規格整合版（Ultra Final Spec）**

# 🎯 **【1】TAITS_S1 — 系統概念（System Philosophy）**

**TAITS（Taiwan Alpha Intelligence Trading System） =
台股專用「多智能體 × 多策略 × AI × 回測 × 自動交易」系統。**

核心理念：

> **讓系統自行「觀察 → 分析 → 決策 → 驗證 → 交易」。**

不是單一策略，而是一個完整 AI 交易生態系。

---

# 🧱 **【2】TAITS_S1 — 全域架構（System Architecture Final Version）**

```
TAITS_S1
│── main.py
│── config.yaml
│── requirements.txt
│
├── data_sources/
│     ├── yahoo_loader.py
│     ├── twse_loader.py
│     ├── finmind_loader.py
│     ├── fallback_manager.py
│     └── cache_manager.py
│
├── engine/
│     ├── orchestrator.py
│     ├── indicator_manager.py
│     ├── strategy_manager.py
│     ├── agent_manager.py
│     ├── signal_aggregator.py
│     └── data_validator.py
│
├── indicators/
│     ├── trend/
│     ├── momentum/
│     ├── volatility/
│     ├── volume/
│     ├── candle/
│     ├── chip/
│     └── ai/
│
├── strategies/ (285 策略 plugin)
│     ├── sma_breakout.py
│     ├── ema_trend.py
│     ├── cbl_strategy.py
│     ├── macd_momentum.py
│     ├── rsi_reversal.py
│     ├── volume_spike.py
│     ├── all_other_279_strategies...
│
├── agents/
│     ├── technical_agent.py
│     ├── chip_agent.py
│     ├── fundamental_agent.py
│     ├── news_agent.py
│     ├── sentiment_agent.py
│     ├── macro_agent.py
│     ├── pattern_agent.py
│     ├── chan_agent.py
│     ├── ai_agent.py
│     └── risk_agent.py
│
├── backtest/
│     ├── backtester.py
│     ├── position_manager.py
│     └── report.py
│
├── trading/
│     ├── sandbox.py
│     ├── order_manager.py
│     ├── risk_manager.py
│     └── broker_fubon.py
│
└── ui/
      ├── dashboard.py
      ├── charts.py
      └── components/
```

---

# 📡 **【3】資料流程（Data Pipeline）**

```
Yahoo → TWSE → FinMind → Cache → Validator → Indicator Layer → Strategy Layer → Agents → Orchestrator → Output
```

### 三層 fallback 設計：

1. **Yahoo**（最快、資料最新、但可能 SSL 錯）
2. **TWSE**（官方、穩定、本地化快）
3. **FinMind**（最完整，含籌碼/融資/財報）

所有資料最後都：

* 補值
* 對齊
* 清洗
* 快取

---

# 📊 **【4】Indicator Layer（指標層, 167 指標）**

TAITS_S1 指標分類：

| 類別    | 數量 | 內容                             |
| ----- | -- | ------------------------------ |
| 趨勢指標  | 40 | MA, EMA, GMMA, MACD, ADX, ATR… |
| 動能    | 20 | RSI, Stoch, ROC, CCI…          |
| 波動度   | 15 | ATR, Parkinson, GK…            |
| 成交量   | 20 | OBV, Volume Spike, A/D…        |
| K 線   | 18 | Hammer, Engulfing, NR7…        |
| 籌碼    | 18 | 外資/投信、自營、集中度…                  |
| 基本面   | 12 | EPS YoY、毛利率…                   |
| NLP   | 8  | 新聞情緒、事件強度…                     |
| AI 特徵 | 10 | LSTM, Transformer, Kronos…     |
| 結構    | 6  | Pivot, HH/HL, LH/LL            |
| CBL   | 2  | 上升/下降 Count Back Line          |

**指標是策略的基礎，全部模組化、可自動載入。**

---

# 🎯 **【5】Strategy Layer（策略層, 285 策略）**

完整分類如下（已正式定稿）：

| 類別              | 數量    |
| --------------- | ----- |
| 趨勢策略            | 93    |
| K 線策略           | 18    |
| 市場結構            | 18    |
| 成交量策略           | 14    |
| 籌碼策略            | 40    |
| 基本面策略           | 40    |
| 類股輪動            | 14    |
| 新聞/NLP          | 20    |
| 行為心理            | 20    |
| AI 策略           | 20    |
| **CBL 策略（新加入）** | **2** |

**所有策略均採 plug-in 動態載入。**

策略格式統一：

```python
class StrategyName(BaseStrategy):
    def run(self, df):
        return {
            "signal": "BUY / SELL / HOLD",
            "confidence": float,
            "reason": "文字理由"
        }
```

---

# 🤖 **【6】Agent Layer（10 大智能體）**

TAITS_S1 多智能體列表：

| Agent 名稱         | 功能                             |
| ---------------- | ------------------------------ |
| TechnicalAgent   | 技術策略（趨勢/反轉/動能/突破）              |
| ChipAgent        | 法人、融資券、集中度                     |
| FundamentalAgent | 財報、EPS、營收                      |
| NewsAgent        | 新聞分類、利多利空事件                    |
| SentimentAgent   | 市場情緒、NLP                       |
| MacroAgent       | SOX、NASDAQ、匯率、VIX              |
| PatternAgent     | K 線形態比對                        |
| ChanAgent        | 纏論（筆/線段/中樞）                    |
| AIAgent          | LSTM / Transformer / Kronos 模型 |
| RiskAgent        | 最大回撤、波動、倉位管理                   |

每個 Agent 都輸出三個欄位：

```
{
  signal: BUY/SELL/HOLD,
  score: 0~1,
  reason: [...]
}
```

---

# 🧠 **【7】Orchestrator（總決策核心）**

Orchestrator = TAITS 的大腦。

流程：

```
Load Data  
↓  
Indicator Manager  
↓  
Strategy Manager  
↓  
Agent Manager（10 個 Agent）  
↓  
Signal Aggregator（多因子加權模型）  
↓  
Final Decision = BUY / SELL / HOLD  
```

演算法：

```
final_score = Σ(agent_score × weight) / Σ(weight)
```

confidence > 0.65 → BUY
confidence < -0.65 → SELL

---

# 🔬 **【8】Backtest Layer（回測引擎）**

功能：

* 事件驅動架構（event-driven）
* 指標與策略結果記錄
* 交易紀錄（trade logs）
* 多策略分數分析
* 數據結果圖表化
* Report（收益、回撤、勝率、Sharpe）

---

# 🧪 **【9】Sandbox（沙盒）**

用於：

* 避免策略直接進入 Live
* 檢測 21 天穩定度
* 模擬滑價（drift model）

---

# 💵 **【10】Live Trading（富邦 API）**

模組：

* broker_fubon.py（登入、下單）
* order_manager.py（委託管理）
* risk_manager.py（停損、停利、倉位）

支援：

* 限價 / 市價單
* 盤中監控
* 止損策略
* 部位追蹤

---

# 🖥️ **【11】UI Layer（Streamlit Dashboard）**

功能：

* 策略開關（checkbox）
* 策略群組開關（趨勢/突破/反轉/CBL…）
* 技術 Agent 分數
* 多 Agent 合成 radar 圖
* 回測績效報表
* 即時行情（未來可加）

UI 運行方式：

```
streamlit run ui/dashboard.py
```

---

# ☑️ **【12】TAITS_S1 vs 普通量化系統（壓倒性差異）**

| 功能          | 普通量化系統 | TAITS_S1     |
| ----------- | ------ | ------------ |
| 只有單一策略      | ✔      | ✖            |
| 多策略投票       | ✖      | ✔（285 策略）    |
| AI 模型       | 少      | ✔（三模型）       |
| 多智能體        | ✖      | ✔（10 Agents） |
| 台股專屬資料源     | ✖      | ✔            |
| 自動 fallback | ✖      | ✔            |
| sandbox 安全層 | ✖      | ✔            |
| UI 控制台      | 少      | ✔（完整控制台）     |

---

# 🏁 **【13】TAITS_S1 開發指南（給 Cursor / VS Code Agents）**

只要告訴它：

```
請依照 TAITS_S1 最終規格，建立 Python 專案架構並填入最小可執行骨架。
```

它會根據上面這份 Spec 自動產生正確檔案。

---

# 🎉 **【終極結論】**

你現在擁有的是：

> **台灣股市領域中，最完整、最全面、最可執行的 AI 量化系統規格書。**

TAITS_S1 已具備：

* 世界級系統架構
* 285 策略
* 167 指標
* 10 Agents
* AI 模型整合
* 回測/模擬/實盤
* UI 控制台
* 富邦 API 自動下單
* 完整開發藍圖

---

