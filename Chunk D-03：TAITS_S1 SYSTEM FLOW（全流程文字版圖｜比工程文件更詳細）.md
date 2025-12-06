好的，來把 **D-3：SYSTEM FLOW（全流程文字版圖）** 做成你可以「一眼看懂整個 TAITS_S1 在運作什麼」的版本。

---

# 📘 Chunk D-3 — TAITS_S1 SYSTEM FLOW（全流程文字版圖｜最終版）

> 這一份是 **「系統實際跑起來時，一步一步到底做了什麼」** 的說明。
> 你可以把它當成：
> 👉 給工程師的「白箱流程圖」
> 👉 給未來自己 Debug、擴充時的「操作說明書」

---

## 0. 系統模式總覽：三種運作模式

TAITS_S1 的 Orchestrator 支援 3 種模式：

1. **Backtest 模式**：用歷史資料回測
2. **Sandbox 模式**：即時模擬下單（不真正送券商）
3. **Live 模式**：實盤，透過富邦 API 真實下單

大方向架構：

```text
[User] → main.py → Orchestrator(run_mode)
   → Data Pipeline
   → Indicator Pipeline
   → Strategy Engine
   → Agents Engine
   → Decision Engine
   → Execution Layer(Backtest / Sandbox / Live)
   → UI Dashboard
```

---

# 1. 系統啟動流程（從 main.py 開始）

### 1.1 main.py 啟動步驟

1. 讀取 `config.yaml` & `.env`
2. 建立 Orchestrator 物件
3. 呼叫 `orch.run()`，進入主流程

**概念流程：**

```text
main.py
 └─ load_config()
 └─ orch = Orchestrator(config)
 └─ orch.run()
```

---

# 2. Orchestrator.run() 總流程

這是整個系統的「主劇本」。

```text
Orchestrator.run()

1. 初始化環境（Logging, Config, Mode）
2. 準備標的列表（symbols）
3. 資料準備（Data Pipeline）
4. 指標運算（Indicator Pipeline）
5. 策略執行（Strategy Pipeline）
6. Agents 評分（Agents Pipeline）
7. 信號彙整（Decision Engine）
8. 根據模式：
   - Backtest → Backtester
   - Sandbox → SandboxEngine
   - Live    → LiveTradingEngine
9. 更新 UI Dashboard
10. 結束 / 或進入下一個週期
```

---

# 3. Data Pipeline（資料流程）

### 3.1 單一標的的資料處理流程

以 `symbol = "2330"` 為例：

```text
Orchestrator
  ↓
FallbackDataLoader.get_price("2330", start, end)
  ↓
[YahooLoader] 嘗試 → 若失敗 → [TWSELoader] → 若失敗 → [FinMindLoader]
  ↓
DataValidator 檢查 & 補欄位
  ↓
CacheManager 儲存成 parquet
  ↓
回傳標準化 DataFrame
(index = Datetime, columns = open/high/low/close/volume/...)
```

### 3.2 資料標準化規則

* Index：`DatetimeIndex`（台灣時間）
* 價格欄位：`open, high, low, close, volume`
* 其他欄位（如法人、融資券、財報…）用統一命名：

  * `foreign_buy`, `inv_trust_buy`, `dealer_buy` …
  * `eps`, `revenue_yoy`, `gross_margin` …

---

# 4. Indicator Pipeline（指標流程）

### 4.1 全流程概念

```text
原始 df
  ↓ IndicatorManager.compute_all(df)
  ↓ 逐個 Indicator 計算後回填欄位
  ↓ df_with_indicators （已包含 100+ 欄指標）
```

### 4.2 指標運算順序（簡化版）

1. **Trend 類**

   * MA / EMA / GMMA / PSAR / SuperTrend…
2. **Momentum 類**

   * RSI / MACD / KD / CCI / ROC…
3. **Volatility 類**

   * ATR / ATR% / BBands / Keltner…
4. **Volume 類**

   * OBV / Volume MA / Volume Spike…
5. **Pattern / Candles**

   * K 線型態 flag（True/False）
6. **Chip / Fundamental / NLP / AI**

   * 由 Data Layer 的籌碼、財報、新聞來源整合後產生

> ✅ **Indicator 層只做「計算欄位」，不做買賣決策。**

---

# 5. Strategy Pipeline（策略運算流程）

### 5.1 單一策略的標準流程

以「策略 1：SMA 突破」為例：

```text
df_with_indicators
  ↓ 傳入 SMABreakoutStrategy.run(df)
  ↓ 內部根據欄位（close, sma20, volume_ma5 ...）判斷
  ↓ 回傳 StrategyOutput：
       - signal: Series{-1, 0, 1}
       - score : Series[-1, 1]
       - meta  : dict（觸發原因、註解）
```

### 5.2 多策略運算順序

1. Orchestrator 告訴 StrategyManager：

   * 本輪要啟動哪些策略（例如：Top 50 核心策略）
2. StrategyManager 對每個策略：

   * 建立策略實例 `strategy = StrategyClass(config)`
   * 呼叫 `strategy.run(df)`
   * 收集所有 `StrategyOutput`
3. 最後得到：

```python
all_strategy_outputs = {
    "SMA_Breakout": StrategyOutput(...),
    "EMA_Trend": StrategyOutput(...),
    ...
}
```

> ✅ 策略層依舊 **只產生各自的 signal & score，不決定下單**。

---

# 6. Agents Pipeline（10 大智能體流程）

### 6.1 Agent 吃什麼？（Input）

每個 Agent 都收到一個 `context`，包含：

```python
context = {
  "symbol": symbol,
  "df": df_with_indicators,
  "strategies": all_strategy_outputs,
  "meta": {... 各種設定/市場資訊 ...}
}
```

### 6.2 每個 Agent 做什麼？（Process）

舉例：

* **TechnicalAgent**

  * 讀取：趨勢策略、動能策略、突破策略的 score
  * 依權重加總 → 得到技術面分數 `tech_score`
* **ChipAgent**

  * 讀取：外資 / 投信 / 融資券 / 主力集中度 指標 & 策略
  * 產生 `chip_score`
* **AIAgent**

  * 讀取：AI 模型輸出的未來漲跌機率
  * 轉成 `ai_score`
* **RiskAgent**

  * 讀取：波動度 / VIX / 回撤 / 持倉 / 市場 regime
  * 產生 `risk_adjustment`（可能降低整體槓桿）

每個 Agent 輸出一個 `AgentOutput`：

```python
AgentOutput = {
  "score": float,         # -1 ~ +1
  "confidence": float,    # 0 ~ 1
  "reason": str,          # 簡易說明
  "details": dict         # 需要追蹤的細節
}
```

---

### 6.3 Summary / Orchestrator Agent（總結智能體）

當所有 Agent 都跑完後：

```text
AgentManager.run_all(context)
  ↓
得到 agent_outputs = {
  "technical": AgentOutput,
  "chip": AgentOutput,
  "fundamental": ...,
  "news": ...,
  "sentiment": ...,
  "macro": ...,
  "ai": ...,
  "risk": ...,
  "pattern": ...,
  ...
}
  ↓
SignalAggregator.aggregate(agent_outputs)
  ↓
得到：
   final_score ( -1 ~ +1 )
   final_confidence ( 0 ~ 1 )
   final_reason
```

**最後 Output：**

```python
Decision = {
  "score": final_score,           # >0 多頭偏多；<0 偏空
  "confidence": final_confidence, # 越高越敢下手
  "action": "BUY" / "SELL" / "HOLD",
  "meta": { "agents": agent_outputs, ... }
}
```

---

# 7. Execution Layer Flow（三種模式的具體流程）

---

## 7.1 Backtest 模式流程

```text
for 每個 symbol:
  準備 df (Data Pipeline)
  計算指標 & 策略 (Indicator + Strategy)
  for 每一根 K 線時間 t:
      - 準備該時間點前的 df[:t]
      - Agents 評估 → Decision_t
      - Backtester.apply_decision(Decision_t)
  完成後 → 計算績效 / 指標 / 風險
  報表輸出 → csv / html / dashboard
```

Backtester 核心：

1. 接收 `Decision_t`
2. 根據：

   * 目前部位
   * Decision.action
   * 風控（RiskAgent）
     決定：
   * 是否建倉 / 加碼 / 減碼 / 出場
3. 記錄：

   * 交易紀錄
   * 每日資產曲線
   * Max Drawdown
   * Win Rate 等

---

## 7.2 Sandbox 模式流程（模擬即時交易）

```text
固定每 X 秒 或每一根 1m K 線：
  1. 下載最新資料（只補尾端）
  2. 更新 df & 指標 & 策略
  3. Agents 重新評分 → Decision
  4. SandboxEngine 模擬撮合：
     - 價格用 close 或 mid price
     - 更新虛擬持倉與損益
  5. 更新 UI Dashboard
```

特點：

* 不會真的送出委託
* 用來測試決策流程 / 停損 / 加碼邏輯

---

## 7.3 Live 模式流程（富邦 API）

```text
固定排程（例如 每 1 分鐘）：

  1. 從富邦 API 取得最新報價 / 持倉 / 下單狀態
  2. 更新 df & 指標 & 策略
  3. Agents → Decision
  4. RiskAgent 再次檢查：
       - 今日停損是否達成？
       - 單股風險是否超標？
  5. 通過風控 → 產生 OrderRequest：
       - symbol, side, qty, price, type
  6. Broker_Fubon.place_order(...)
  7. 寫入 TradeLog
  8. UI 實時顯示：
       - 持股
       - 損益
       - 未成交委託
```

> ※ Live 模式實作時，**所有 API 錯誤都要 Fail-Safe 處理**：寧可不下單，也不要亂下單。

---

# 8. Error & Retry Flow（錯誤與重試流程）

---

## 8.1 資料錯誤

**情境：** Yahoo 取價失敗

```text
YahooLoader.fetch() raise Exception
  ↓
FallbackDataLoader 捕捉錯誤 → log.warning
  ↓
改用 TWSELoader.fetch()
  ↓
若 TWSE 也失敗 → 改用 FinMindLoader
  ↓
全失敗 → 嘗試從 CacheManager.load()
  ↓
Cache 也沒有 → raise DataSourceError
           → Orchestrator 記錄錯誤，該 symbol 略過
```

---

## 8.2 策略錯誤

**情境：** 某策略 code bug

```text
StrategyManager.run_all():
  try:
     strategy.run(df)
  except Exception:
     log.error(...)
     標記該策略為 failed，不中斷整體流程
```

Decision Engine 在加總時：

* 忽略 `failed` 策略的 score
* 仍然可以產生 Decision

---

## 8.3 Agent 錯誤

同理，Agent 出錯時：

```text
AgentManager.run_all()
  try:
      agent.run(context)
  except Exception:
      log.error(...)
      該 Agent 輸出 score = 0, confidence = 0
```

不讓整個系統當掉。

---

## 8.4 下單錯誤（Live）

如果 `Broker_Fubon.place_order()` 失敗：

* 回傳錯誤碼
* OrderManager：

  * 寫入錯誤 log
  * 把該筆 Decision 標記為 `not_executed`
  * 絕不自動重複亂下單（避免重複下單風險）

---

# 9. UI Flow（儀表板流程）

UI 目前定位：
**顯示結果、不控制交易邏輯。**

---

## 9.1 Backtest 報表顯示

1. Backtester 完成 → 產出：

   * `equity_curve.csv`
   * `trades.csv`
   * `metrics.json`
2. UI `dashboard.py` 讀取這些檔案：

   * 顯示績效曲線
   * 顯示每筆交易
   * 顯示各策略貢獻度

---

## 9.2 Realtime 顯示（Sandbox / Live）

1. Execution Layer 每次執行完 Decision：

   * 把最新狀態 dump 成 `state.json`
2. `dashboard.py` 每隔 X 秒讀一次：

   * 顯示：

     * 目前所有持股
     * 總資產
     * 今日損益
     * 各 Agent score
     * 最終 Decision（BUY/SELL/HOLD）

---

# 10. 未來擴充 Flow 建議（S2/S3）

這份 D-3 也預留了未來 S2/S3：

* **多標的 Portfolio Flow**：

  * 現在是單一 symbol 流程
  * 之後可在 Orchestrator 層：

    * 先對多個 symbol 各自執行一輪 Pipeline
    * 再由 PortfolioAgent 做資金分配
* **多時間週期 Flow**：

  * 現在是單一時間級別（例如日線或 30m）
  * 未來可加入：

    * Daily Pipeline
    * Intraday Pipeline
  * 最後由 MultiTimeframeAgent 統合

---

# 🎯 總結一句話

> **D-2 告訴你「有哪一些模組、長什麼樣」**
> **D-3 告訴你「這些模組，從早到晚實際是怎麼跑的」**

—
