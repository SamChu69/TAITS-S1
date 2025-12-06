# # 🌐 **TAITS — Taiwan Alpha Intelligence Trading System**

### **台灣阿爾法智能交易系統（TAITSS1 Edition）**

![TAITS Banner](https://img.shields.io/badge/TAITS-S1%20Edition-blueviolet?style=for-the-badge\&logo=python\&logoColor=white)

---

# ## 🧠 Overview

**TAITS**（Taiwan Alpha Intelligence Trading System）是一套專為 **台股市場打造**，
結合 **AI 模型、量化策略、多智能體（TradingAgents）、實盤 API、自動回測、雲端部署** 的 **世界級交易系統架構**。

本專案 S1 版本（Stage 1）包含：

* 285 策略量化模型
* 10 大智能體（Technical, Chip, Fundamental, AI, Macro, etc.）
* 多時間框架資料流（Yahoo / TWSE / FinMind）
* Event-driven Backtest Engine
* Fubon API Live Trading（富邦證券自動下單）
* Streamlit Dashboard（可視化操作介面）
* 完整工程級文件：Chapters（架構）、C 系列（技術）、D 系列（工程）

---

# ## 🏛️ 系統特色（World-Class Features）

| 模組                              | 說明                                  |
| ------------------------------- | ----------------------------------- |
| **多資料源（多層 Fallback）**          | Yahoo → TWSE → FinMind 自動切換         |
| **285 策略引擎**                    | 趨勢 / 反轉 / 籌碼 / AI / 動能 / 均值 / 缠论    |
| **TradingAgents 多智能體架構**        | 10 種 Agent 產生多視角判斷                  |
| **AI 引擎**                       | LSTM / Transformer / Kronos         |
| **回測系統（Backtest Engine）**       | 事件驅動、Portfolio、報表                   |
| **富邦 API 實盤系統**                 | 下單、回報、QE 等完整支持                      |
| **風控引擎（Risk Engine）**           | Exposure, Drawdown, Position Sizing |
| **策略投票（Multi-Strategy Voting）** | 加權整合所有策略                            |
| **決策引擎（Orchestrator）**          | 全系統中樞，完成交易決策                        |
| **UI 面板**                       | Streamlit Dashboard：策略/績效/下單可視化     |

---

# ## 🚀 快速開始（Quick Start）

### **1. Clone 專案**

```
git clone https://github.com/你的帳號/TAITS_S1.git
cd TAITS_S1
```

---

### **2. 安裝依賴**

```
pip install -r requirements.txt
```

---

### **3. 執行最小可運行版本**

```
python main.py
```

成功後你會看到：

```
TAITS S1 — Minimal System Running ✔
```

---

# ## 🧬 專案架構（S1 Ultra FINAL）

```
TAITS_S1/
│── main.py
│── requirements.txt
│── README.md（本檔案）
│── /docs
│     ├── CHAPTER_01_System_Overview.md
│     ├── CHAPTER_02_Global_Architecture.md
│     ├── CHAPTER_03_Data_Pipeline.md
│     ├── CHAPTER_04_Indicator_Layer.md
│     ├── CHAPTER_05_Strategy_Layer.md
│     ├── ...
│     ├── C-01 ~ C-46 （所有技術分析與程式結構文件）
│     └── D-01 ~ D-04 （工程技術指南）
│
├── data_sources/
├── engine/
├── indicators/
├── strategies/
├── agents/
├── backtest/
├── trading/
└── ui/
```

---

# ## 📊 專案章節文件（完整 18 章）

以下僅展示章節標題（內容已在 docs/）：

| 章節         | 名稱                         |
| ---------- | -------------------------- |
| CHAPTER 01 | 系統總覽（System Overview）      |
| CHAPTER 02 | 全域架構（Global Architecture）  |
| CHAPTER 03 | 資料流設計（Data Pipeline）       |
| CHAPTER 04 | 技術指標層                      |
| CHAPTER 05 | 策略層（285 策略全集）              |
| CHAPTER 06 | 多智能體（Agent Layer）          |
| CHAPTER 07 | Orchestrator 決策引擎          |
| CHAPTER 08 | 指標系統（Indicator Layer）      |
| CHAPTER 09 | 策略結構（Strategies Framework） |
| CHAPTER 10 | TradingAgents 架構           |
| CHAPTER 11 | 資料處理（Processing Layer）     |
| CHAPTER 12 | AI 引擎（AI Engine）           |
| CHAPTER 13 | 回測系統（Backtest Engine）      |
| CHAPTER 14 | 實盤下單（Fubon API）            |
| CHAPTER 15 | UI（Streamlit Dashboard）    |
| CHAPTER 16 | Logging / Error Handling   |
| CHAPTER 17 | Deployment                 |
| CHAPTER 18 | 完整文件索引                     |

---

# ## 🧩 C 系列（技術核心文件）共 46 篇

用於實作每個模組的程式要求、資料格式、介面定義。
例如：
🔰 核心架構群
| 編號   | 標題                                        |
| ---- | ----------------------------------------- |
| C-01 | 完整專案目錄＋責任定義                               |
| C-02 | 最小可運行骨架（Python 專案）                        |
| C-03 | 285 策略 Plugin 架構                          |
| C-04 | 指標管理器 × 策略管理器 × Agents × Orchestrator 關聯圖 |
| C-05 | 資料流設計（Data Flow + Validator）              |
| C-06 | 資料快取層 Cache System                        |
📊 指標層（Indicators Layer）
| 編號   | 標題                            |
| ---- | ----------------------------- |
| C-07 | 指標層完整規格                       |
| C-08 | 技術指標核心規格（Indicator Core Spec） |
| C-09 | 技術指標：數學接口標準（IIndicator）       |
| C-10 | 技術指標：GMMA、MACD、RSI、ATR… 參數規格  |
📈 策略層（Strategies Layer）
| 編號   | 標題                                |
| ---- | --------------------------------- |
| C-11 | 策略層核心接口（IStrategy Spec）           |
| C-12 | 策略訊號整合層（Voting Engine）            |
| C-13 | 策略資料管線（Strategy Data Pipeline）    |
| C-14 | TradingAgents × Strategies 完整互動規格 |
| C-15 | 策略層 ULTRA FINAL（全部 285 策略分類）      |
🤖 智能體層（Agents Layer）
| 編號   | 標題                        |
| ---- | ------------------------- |
| C-16 | Agents Core（Agent 介面規格）   |
| C-17 | Agents 全列表：10 大智能體規格完整定義  |
| C-18 | Signal Aggregator（訊號整合引擎） |
| C-19 | Decision Engine（決策處理管線）   |
🏦 交易層（Execution, Portfolio, Risk）
| 編號   | 標題                       |
| ---- | ------------------------ |
| C-20 | Portfolio Engine（投資組合模型） |
| C-21 | Backtest Engine（事件驅動）    |
| C-22 | 模擬交易 Sandbox Engine      |
| C-23 | Backtest & 回測報告模組        |
| C-24 | Live Trading（富邦 API 全規格） |
| C-25 | Risk Control（風險控制核心）     |
| C-26 | Position Sizing（部位管理）    |
📝 系統工具（System Utility & Logging）
| 編號   | 標題                                   |
| ---- | ------------------------------------ |
| C-27 | Logging + Audit Trail（交易審計與系統紀錄）     |
| C-28 | Config & Settings（模式管理與環境參數）         |
| C-29 | Monitoring（監控 / 警報 / 健康檢測）           |
| C-30 | 測試與驗證框架（Unit Test & Regression Test） |
🧬 其他進階章節
| 編號   | 標題                                          |
| ---- | ------------------------------------------- |
| C-31 | Backtest Report System（KPI、統計、策略績效細節）       |
| C-32 | 模組互動圖（Everything Interaction Map）           |
| C-33 | Paper Trading 模型（虛擬券商）                      |
| C-34 | Live Trading 2.0（Event-based broker spec）   |
| C-35 | 策略評分引擎（Strategy Scoring Engine）             |
| C-36 | AI Model Layer（LSTM & Transformer & Kronos） |
| C-37 | 多策略投票（Multi-Strategy Voting Engine）         |
| C-38 | Position Risk Engine（最大跌損、自動調整）             |
| C-39 | Execution Engine（下單、撤單、改單）                  |
| C-40 | Portfolio Engine 2.0（資金分配＋再平衡）              |
| C-41 | Alert & Event System                        |
| C-42 | Regime Engine（市場 regime 判定）                 |
| C-43 | Budget Engine（頭寸限制）                         |
| C-44 | Capital Allocation Engine                   |
| C-45 | Sizing Engine 2.0（多層級部位計算）                  |
| C-46 | Execution Manager（流動性、高頻細節）                 |

---

# ## 🧩 D 系列（工程指南）

* **D-01：TAITS_S1 MASTER SPEC**（總規格書）
* **D-02：Engineering Tech Doc**（工程設計文檔）
* **D-03：System Flow**（全系統流程）
* **D-04：Full Reference**（參考資料全書）

---

# ## 📈 UI 預覽（Dashboard）

（你之後如果想做 Logo / Banner，我可以幫你畫）

```
/ui/dashboard.py
```

可顯示：

* K 線
* 指標
* 策略訊號
* 回測績效
* 證券下單面板

---

# ## 📡 Live Trading（富邦 API 介接）

```
/trading/broker_fubon.py
```

具備：

* 登入 / 持倉查詢
* 下單 / 回補 / 委託回報
* 錯誤處理
* 重試機制

如果你完成 Fubon SDK 安裝，我可以幫你寫完整可用版本。

---

# ## 🧾 License

MIT 或你想使用的授權方式皆可。

---

# ## ⭐ 如果你希望，我可以進一步幫你：

### ✔ 自動生成 GitHub Wiki

### ✔ 自動生成 TAITS Logo

### ✔ 自動生成多圖版本 README

### ✔ 自動生成整個 TAITS 官網（用 GitHub Pages）

### ✔ 自動生成開發 Roadmap、Milestone 與 Issue 模板

---
