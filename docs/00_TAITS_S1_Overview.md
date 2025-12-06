# # **TAITS S1 — 台股多智能體量化交易系統（總覽 Overview）**

> **TAITS S1（Taiwan AI Trading System — Stage 1）**
> 是一套專為台股打造的下一代量化交易系統，
> 特色是 **多智能體架構（TradingAgents）＋ 285 套策略 + 200 指標 + AI 模型 + 回測引擎 + 富邦 API 實盤**
> 由完整架構、資料層、策略層、AI 層、交易層、UI層組成。

---

# ## 🧩 **0. 專案用途（What this project is for）**

TAITS S1 可以做到：

### ✔ 建構自己的台股量化策略工廠

### ✔ 用多智能體（技術、籌碼、AI、情緒、宏觀…）做交易決策

### ✔ 用 Backtest 引擎做策略回測

### ✔ 用 Sandbox 模擬交易

### ✔ 用富邦 API 直接實盤下單

### ✔ 用 Streamlit 打造儀表板 UI

你之後會用它來打造：

* 你的 AI 交易模型
* 自動化交易策略
* 多策略投票系統
* 多資料來源 fallback 系統

---

# ## 🧱 **1. 系統架構（Architecture Overview）**

整個架構可分成六大層：

```
┌────────────────────────────┐
│ 1. Data Layer（多資料來源） │
├────────────────────────────┤
│ 2. Indicator Layer（200 指標） │
├────────────────────────────┤
│ 3. Strategy Layer（285 策略） │
├────────────────────────────┤
│ 4. Agent Layer（10 個智能體） │
├────────────────────────────┤
│ 5. Engine Layer（回測 / Orchestrator）│
├────────────────────────────┤
│ 6. Trading Layer（模擬 + 實盤）   │
├────────────────────────────┤
│ 7. UI Layer（Streamlit）        │
└────────────────────────────┘
```

---

# ## 🔗 **2. 多資料來源（Data Sources）**

TAITS S1 支援 **3 層 fallback（自動切換）**：

1. **Yahoo Finance**（最快）
2. **TWSE OpenAPI**（官方資料）
3. **FinMind**（籌碼、融資券、財報）

若三層都失敗，會回到：

* **本地 Cache**
* 或錯誤提示

---

# ## 📊 **3. 指標層（Indicator Layer）**

共 **200+ 指標模組化設計**：

* 趨勢（MA、EMA、GMMA、MACD…）
* 動能（RSI、KD、ROC…）
* 波動（ATR、HV、VIX proxy…）
* 量能（OBV、量增量縮、換手率…）
* K 線型態（18 種）
* 籌碼（外資、投信、自營、融資券、大戶集中度）
* AI 指標（Prob、Kronos、LSTM、Transformer）

---

# ## 🎯 **4. 策略層（Strategy Layer）**

共 **285 套策略**：

### 類型包含：

* 趨勢
* 反轉
* 動能
* 均值回歸
* 量價
* K 線
* 籌碼
* 類股
* 市場結構
* 纏論（Chan Theory）
* AI 多模型策略
* 多策略投票策略

每個策略包含：

* 進場邏輯
* 出場邏輯
* 適用市場
* Pseudocode

---

# ## 🤖 **5. Agent 層（TradingAgents）**

TAITS S1 內建 **10 個智能體（Agents）**：

| Agent 名稱          | 功能                         |
| ----------------- | -------------------------- |
| Technical Agent   | 分析技術面指標、技術策略               |
| Chip Agent        | 外資、投信、自營、籌碼指標              |
| Fundamental Agent | 基本面、財報、估值                  |
| News Agent        | 新聞 NLP、重大事件                |
| Sentiment Agent   | 散戶情緒、Fear/Greed            |
| Macro Agent       | 美股、匯率、VIX 模擬               |
| Pattern Agent     | K線型態、型態偵測                  |
| Chan Agent        | 分型、筆、中樞、買賣點                |
| AI Agent          | LSTM、Transformer、Kronos 模型 |
| Risk Agent        | 風控、停損、資金管理                 |

每個 Agent 都會輸出一個 Score (-1~+1)。

---

# ## ⚙️ **6. Orchestrator（主控流程）**

Orchestrator 是 TAITS 的大腦。

完整流程：

```
Load Config
↓
Load Data（Yahoo → TWSE → FinMind → Cache）
↓
Compute Indicators
↓
Run 285 Strategies
↓
Activate 10 Agents
↓
Aggregate all scores（多策略多智能體投票系統）
↓
Decision Engine（Buy / Sell / Hold）
↓
Backtest / Sandbox / Live
```

---

# ## 📈 **7. Backtest（回測引擎）**

功能：

* 事件驅動模式 event-driven
* 支援多策略同時回測
* 每日/每分鐘回測
* 交易成本、滑點、停損
* 輸出完整報告（報酬率、夏普、回撤、勝率）

---

# ## 💰 **8. Live Trading（模擬 & 實盤）**

TAITS S1 支援：

### ① Sandbox（本地模擬下單）

### ② 富邦證券 API（官方 SDK）

功能：

* 自動下單
* 自動停損
* 單筆 / 多筆交易
* 回報監控

---

# ## 📊 **9. UI（Streamlit Dashboard）**

可顯示：

* 指標
* 策略訊號
* Agent Score
* 整體決策分數（Final Confidence）
* 回測結果

---

# ## 📘 **10. 本專案包含的所有文件**

你會在 GitHub 上看到：

```
/docs
│── 00_TAITS_S1_Overview.md     ← 就是現在這份
│── 01_Chapter_01_to_18.md
│── 02_C_Notes_C01_C46.md
│── 03_D_Specs_Master.md
│── ...
```

---

# ## 📌 這份 Overview 的目的

讓任何人（包括未來的你）看到此專案後：

* 一次看懂整個 TAITS 系統
* 系統為何而生
* 系統包含哪些部分
* 如何執行
* 如何擴充

---
