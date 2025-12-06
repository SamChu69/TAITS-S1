太好了，我們來把 **C-44：Capital Allocation Engine（資金分配引擎）** 一次寫到「能直接實作」的等級。

> 如果說 C-43 是「**你最多可以冒多少風險**」，
> 那 C-44 就是「**這些錢要分給誰、分多少**」。

---

# 🧩 C-44：Capital Allocation Engine（資金分配引擎）

## 🎯 C-44.1 這個模組在 TAITS_S1 裡到底負責什麼？

一句話：

> **在「風險預算已經決定」的前提下，幫你決定：
> 哪些策略 + 哪些股票 + 各自拿多少資金 / 部位。**

角色分工：

* C-42：Regime Engine → 判斷現在是牛、熊、盤整、崩盤
* C-43：Risk Budget Engine → 全部最多可以輸多少（風險上限）
* **C-44：Capital Allocation Engine → 在這個風險框架下，誰拿錢、拿多少**

---

## 🧱 C-44.2 資金分配的 5 個層級

Capital Allocation 分成 5 層：

1. **Portfolio Level（整體資金層）**

   * 現在要多少比例進市場？多少保持現金？
   * 是否要保留一部分給「防禦策略 / Hedge」？

2. **Bucket Level（策略類別層）**

   * Trend / Mean Reversion / AI / News / Hedge / Cash
   * 每個 Bucket 有一個「目標比例」，會依照 Regime 調整

3. **Strategy Level（單策略層）**

   * 同一 Bucket 裡有很多策略（例如 46 個趨勢策略）
   * 怎麼分錢給 Sharpe 高/低、最近績效好/壞的策略？

4. **Symbol Level（標的層）**

   * 同一策略可能挑出 3 檔：2330、2454、2603
   * 要怎麼分？平均？照強弱？照風險？

5. **Trade Level（單筆交易層）**

   * 結合 C-43 的單筆 Risk，轉成最終部位 / 張數
   * 若超過資金上限或風險上限，做縮放

---

## 📥 C-44.3 Capital Allocation Engine 的輸入（Inputs）

這個模組不會自己亂決定，它會從其他模組拿資料：

### 1️⃣ 資金與風險狀態（來自帳戶 + C-43）

* `account_equity`（總資產）
* `cash_available`（可用現金）
* `risk_budget_state`（包含：

  * 今日可用風險
  * 各 symbol / strategy 已用風險
  * 單筆風險上限）

### 2️⃣ 市場 Regime（來自 C-42）

* `regime`: BULL, BEAR, SIDEWAY, VOLATILE…
* `risk_mode`: AGGRESSIVE / NORMAL / DEFENSIVE / CAPITAL_PROTECTION

### 3️⃣ 策略訊號清單（來自 Strategy Manager / Orchestrator）

每一筆候選交易（candidate）至少包含：

```python
{
  "symbol": "2330",
  "side": "LONG",                 # LONG / SHORT
  "strategy_id": "trend_gmma",
  "bucket": "TREND",              # TREND / MEANREV / AI / HEDGE...
  "confidence": 0.82,             # 策略信心 0~1
  "score": 0.76,                  # 綜合分數（指標/AI/etc）
  "entry_price": 850.0,
  "stop_loss_price": 820.0,
  "atr_pct": 0.025,               # ATR% or 波動度
  "symbol_liquidity_score": 0.9,  # 流動性分數
  "symbol_rs_score": 0.88         # 相對強弱分數
}
```

### 4️⃣ 策略與 Bucket 的 Meta 資訊

* 每個策略的：

  * `long_term_sharpe`
  * `max_drawdown`
  * `recent_performance`（最近 30 天回測或實盤）
  * `enabled` / `disabled`

* 每個 Bucket 的 Target 比例設定（config 裡）

---

## 📤 C-44.4 Capital Allocation Engine 的輸出（Outputs）

最後要產生「可以直接送去給 RiskBudget + OrderManager 的東西」：

對每個 candidate trade：

```python
{
  "symbol": "2330",
  "strategy_id": "trend_gmma",
  "side": "LONG",
  "capital_allocated": 250000,  # 分配到這筆交易的資金額
  "target_risk_amount": 5000,   # 預期最大可承受虧損（給 C-43 用）
  "max_shares": 200,            # 初步計算的張數（再由 C-43 做最終風險修正）
  "priority": 0.91              # 排序優先權
}
```

Orchestrator 之後流程：

1. C-44 先決定資金分配 + 預期風險
2. C-43 再檢查「這樣會不會爆風險上限？要不要縮小？」
3. 通過後 → OrderManager 真正下單

---

## 🧠 C-44.5 Bucket（策略類別）資金分配邏輯

### 1️⃣ Bucket 定義（可放在 config）

預設建議：

```yaml
capital_buckets:
  TREND:
    base_weight: 0.35
  MEANREV:
    base_weight: 0.15
  AI:
    base_weight: 0.20
  NEWS:
    base_weight: 0.10
  HEDGE:
    base_weight: 0.05
  CASH:
    base_weight: 0.15
```

### 2️⃣ Regime 調整表

| Regime              | TREND | MEANREV | AI   | NEWS | HEDGE | CASH |
| ------------------- | ----- | ------- | ---- | ---- | ----- | ---- |
| Steady Bull         | 0.45  | 0.10    | 0.20 | 0.10 | 0.05  | 0.10 |
| Strong Bull         | 0.50  | 0.05    | 0.20 | 0.05 | 0.05  | 0.15 |
| Sideway Low Vol     | 0.20  | 0.30    | 0.15 | 0.15 | 0.05  | 0.15 |
| Sideway High Vol    | 0.15  | 0.25    | 0.15 | 0.15 | 0.10  | 0.20 |
| Weak Bear           | 0.10  | 0.20    | 0.10 | 0.15 | 0.20  | 0.25 |
| Strong Bear / Crash | 0.05  | 0.10    | 0.10 | 0.10 | 0.25  | 0.40 |

> 這張表可以放在 `config/regime_allocation.py` 或 `settings.py` 裡。

### 3️⃣ 計算每個 Bucket 當日可用資金

舉例：
`account_equity = 1,000,000`, Regime = Steady Bull

* TREND bucket 資金 ≈ 450,000
* MEANREV ≈ 100,000
* AI ≈ 200,000
* …依照權重計算

---

## 📊 C-44.6 策略層（Strategy Level）分配邏輯

在 TREND bucket 裡假設有 10 個策略：

我們希望：

* 長期 Sharpe 高的策略 → 權重高
* 最近表現大虧的策略 → 暫時降權或禁用
* 信心（confidence）高的訊號 → 更優先拿資金

### 1️⃣ 定義一個策略分數（Strategy Score）

對每個 `strategy_id` 定義：

```text
strategy_score = 
    w1 * normalized_sharpe
  + w2 * recent_performance_score
  - w3 * drawdown_penalty
  + w4 * stability_score
```

簡化版（可以寫在 Spec 裡給 Cursor）：

```python
score = 0.4 * sharpe_norm + 0.3 * recent_norm - 0.2 * dd_norm + 0.1 * stability
```

然後做 softmax / normalize，得到每個策略在該 Bucket 的相對權重：

```python
weight_i = score_i / sum(score_j)
bucket_capital_for_strategy = bucket_capital * weight_i
```

---

## 📈 C-44.7 標的層（Symbol Level）分配邏輯

一個策略可能同時選到：

* 2330（台積電）
* 2454（聯發科）
* 2303（聯電）

我們考慮：

1. **相對強弱（RS）**
2. **流動性（成交量 / 市值）**
3. **分散度（不要 All-in 一檔）**

簡單版權重：

```text
symbol_score = 
    0.5 * rs_score
  + 0.3 * liquidity_score
  + 0.2 * confidence
```

再 normalize：

```python
weight_symbol_i = symbol_score_i / sum(symbol_score_j)
capital_for_symbol_i = capital_for_strategy * weight_symbol_i
```

---

## ⚙️ C-44.8 Trade Level：轉成「建議資金」「建議股數」

對於每一筆 candidate：

1. 已知：

```python
capital_for_symbol_i      # C-44 算出來
entry_price
stop_loss_price
```

2. 粗略計算最大股數（之後還要丟給 C-43 再過一輪）：

```python
max_possible_shares = int(capital_for_symbol_i / entry_price)
```

3. 同時推一個「目標風險金額」，給 C-43：

```python
risk_per_share = abs(entry_price - stop_loss_price)
target_risk_amount = max_possible_shares * risk_per_share
```

4. C-44 的 output（之後會被 C-43 修正）：

```python
{
  "symbol": symbol,
  "strategy_id": strategy_id,
  "side": side,
  "capital_allocated": capital_for_symbol_i,
  "prelim_shares": max_possible_shares,
  "target_risk_amount": target_risk_amount,
  "confidence": confidence,
  "priority": overall_priority
}
```

---

## 🧩 C-44.9 Python 類別骨架（可以直接丟給 Cursor 實作）

```python
# engine/capital_allocation_engine.py

from dataclasses import dataclass
from typing import List, Dict, Any

@dataclass
class BucketConfig:
    name: str
    base_weight: float


@dataclass
class StrategyMeta:
    strategy_id: str
    bucket: str
    sharpe: float
    recent_perf: float     # 近 30 天或 N 策略週期表現
    max_drawdown: float
    stability: float       # 訊號穩定度 0~1
    enabled: bool = True


class CapitalAllocationEngine:

    def __init__(self, bucket_config: Dict[str, BucketConfig], regime_allocation_table: Dict[str, Dict[str, float]], logger=None):
        """
        bucket_config: 例如 {"TREND": BucketConfig(name="TREND", base_weight=0.35), ...}
        regime_allocation_table: Regime -> {bucket_name -> weight}
        """
        self.bucket_config = bucket_config
        self.regime_alloc = regime_allocation_table
        self.logger = logger

    # ---- Portfolio / Bucket 層 ----
    def allocate_to_buckets(self, account_equity: float, regime: str) -> Dict[str, float]:
        """
        回傳每個 bucket 的資金額度，例如：
        {"TREND": 450000, "MEANREV": 100000, ...}
        """
        if regime in self.regime_alloc:
            weights = self.regime_alloc[regime]
        else:
            # fallback: 用 base_weight 正規化
            total = sum(b.base_weight for b in self.bucket_config.values())
            weights = {name: cfg.base_weight / total for name, cfg in self.bucket_config.items()}

        bucket_capital = {}
        for bucket_name, w in weights.items():
            bucket_capital[bucket_name] = account_equity * w

        return bucket_capital

    # ---- Strategy 層 ----
    def compute_strategy_weights(self, strategies: List[StrategyMeta]) -> Dict[str, float]:
        """
        輸入：某一 Bucket 裡所有策略 meta
        輸出：該 bucket 內部策略的相對權重
        """
        scores = {}
        for sm in strategies:
            if not sm.enabled:
                continue
            sharpe_norm = max(sm.sharpe, -1.0)  # 簡單截斷避免爆
            dd_norm = min(sm.max_drawdown, 1.0)
            recent_norm = sm.recent_perf

            score = (
                0.4 * sharpe_norm
                + 0.3 * recent_norm
                - 0.2 * dd_norm
                + 0.1 * sm.stability
            )
            scores[sm.strategy_id] = max(score, 0.0)

        total = sum(scores.values())
        if total <= 0:
            # fallback: 均分
            n = len(scores)
            return {sid: 1.0 / n for sid in scores.keys()} if n > 0 else {}

        return {sid: v / total for sid, v in scores.items()}

    # ---- Symbol / Trade 層 ----
    def allocate_for_candidates(
        self,
        account_equity: float,
        regime: str,
        strategy_meta_map: Dict[str, StrategyMeta],
        candidates: List[Dict[str, Any]]
    ) -> List[Dict[str, Any]]:
        """
        主要入口：
        - 根據 Regime → 決定各 Bucket 資金
        - Bucket 裡面再依策略權重、標的分數分配
        - 回傳每個 candidate 建議的 capital / prelim_shares / target_risk_amount
        """

        # 1. 計算 bucket 資金配額
        bucket_capital = self.allocate_to_buckets(account_equity, regime)

        # 2. 依 bucket + strategy 先 group 起來
        # bucket -> strategy -> [candidates...]
        grouped: Dict[str, Dict[str, List[Dict[str, Any]]]] = {}
        for c in candidates:
            bucket = c.get("bucket", "TREND")
            sid = c["strategy_id"]
            grouped.setdefault(bucket, {}).setdefault(sid, []).append(c)

        results = []

        # 3. 每個 bucket 內部處理
        for bucket, strat_dict in grouped.items():
            bucket_cap = bucket_capital.get(bucket, 0.0)
            if bucket_cap <= 0:
                continue

            # 3a. 準備這個 bucket 內的策略 meta
            metas = []
            for sid in strat_dict.keys():
                meta = strategy_meta_map.get(sid)
                if meta and meta.enabled:
                    metas.append(meta)

            if not metas:
                continue

            strat_weights = self.compute_strategy_weights(metas)

            # 3b. 每個策略再分配資金
            for sid, cand_list in strat_dict.items():
                if sid not in strat_weights:
                    continue
                strat_cap = bucket_cap * strat_weights[sid]
                if strat_cap <= 0 or not cand_list:
                    continue

                # 4. 該策略裡面：依 symbol_score 分配
                # symbol_score = 0.5 * rs + 0.3 * liquidity + 0.2 * confidence
                symbol_scores = []
                for c in cand_list:
                    rs = c.get("symbol_rs_score", 0.5)
                    liq = c.get("symbol_liquidity_score", 0.5)
                    conf = c.get("confidence", 0.5)
                    score = 0.5 * rs + 0.3 * liq + 0.2 * conf
                    symbol_scores.append(score)

                total_symbol_score = sum(symbol_scores)
                if total_symbol_score <= 0:
                    # 平均分
                    per_cap = strat_cap / len(cand_list)
                    for c in cand_list:
                        entry = c["entry_price"]
                        cap = per_cap
                        prelim_shares = int(cap / entry) if entry > 0 else 0
                        risk_per_share = abs(entry - c["stop_loss_price"])
                        target_risk_amount = prelim_shares * risk_per_share
                        results.append({
                            **c,
                            "capital_allocated": cap,
                            "prelim_shares": prelim_shares,
                            "target_risk_amount": target_risk_amount,
                            "priority": c.get("confidence", 0.5)
                        })
                else:
                    # 依照分數比例分配
                    for c, s_score in zip(cand_list, symbol_scores):
                        weight = s_score / total_symbol_score
                        cap = strat_cap * weight
                        entry = c["entry_price"]
                        prelim_shares = int(cap / entry) if entry > 0 else 0
                        risk_per_share = abs(entry - c["stop_loss_price"])
                        target_risk_amount = prelim_shares * risk_per_share
                        overall_priority = 0.5 * c.get("confidence", 0.5) + 0.5 * s_score
                        results.append({
                            **c,
                            "capital_allocated": cap,
                            "prelim_shares": prelim_shares,
                            "target_risk_amount": target_risk_amount,
                            "priority": overall_priority
                        })

        return results
```

> 之後你可以直接對 Cursor 說：
> **「請依據 C-43、C-44 Spec，建立 `engine/risk_budget_engine.py` 和 `engine/capital_allocation_engine.py`，並在 Orchestrator 裡串接。」**

---

## 🔗 C-44.10 在 TAITS_S1 總流程的放置位置

整個執行順序可以這樣想：

1. **Data Layer**：抓資料、算指標

2. **Strategies**：跑 285 策略 → 產生一堆 candidates

3. **Agents & Orchestrator**：

   * 綜合多智能體 → 過濾 / 打分數
   * 產生 **Trade Plan List**

4. **Regime Engine（C-42）**：判斷當前市場狀態

5. **Risk Budget Engine（C-43）**：

   * 計算今日可用風險
   * 控制最大風險水位

6. **Capital Allocation Engine（C-44）**：

   * 在風險框架下，把錢分給 Bucket / 策略 / 標的 / 單筆
   * 輸出 `capital_allocated`, `prelim_shares`, `target_risk_amount`

7. **再次丟回 C-43 校正**：

   * 若某筆 target_risk_amount 超過 Risk Budget → 縮小或拒絕

8. **Order Manager**：真正下單 / Sandbox 模擬

9. **UI**：顯示：

   * 每個 Bucket / 策略 / 標的的資金使用率
   * 當前 Regime & Risk Mode
   * 哪些單被資金限制擋掉

---

要繼續就跟我說：
**「開始 C-45」**
