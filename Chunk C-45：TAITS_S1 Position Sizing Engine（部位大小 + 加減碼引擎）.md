超讚，我們來把 **C-45：Position Sizing Engine（部位大小 + 加減碼引擎）** 做成你之後可以直接丟給 Cursor / VS Code 代理實作的「完整規格」。

> C-43：我今天最多能承擔多少風險？
> C-44：這些錢要分給誰（哪個 bucket / 策略 / 標的）？
> **C-45：每一筆實際要下多少張？要怎麼加碼、怎麼減碼？**

---

# 🧩 C-45：Position Sizing Engine（部位大小 + 加減碼引擎）

## C-45.1 它在 TAITS_S1 當中的角色是什麼？

一句話：

> **在「資金已分配（C-44）＋風險上限（C-43）」的前提下，
> 決定每一筆訂單「實際下多少部位」，以及「何時加碼 / 減碼」。**

它要解決三件事：

1. **初始部位（Initial Position Size）**

   * 一開始這筆進場要下幾張 / 幾股？

2. **加碼（Pyramiding / Scale-in）**

   * 如果走對了，要不要再加？加幾次？每次加多少？

3. **減碼（Scale-out / Take Profit）**

   * 走對了要不要分批獲利了結？
   * or 走錯了是否要分批減碼降低傷害？

---

## C-45.2 Position Sizing Engine 的輸入（Inputs）

這個模組接收的資料主要來自：

### 1️⃣ 來自 Capital Allocation Engine（C-44）的資訊

每一筆候選交易（candidate_trade）會已經被 C-44 處理過，包含：

```python
{
  "symbol": "2330",
  "side": "LONG",                 # LONG / SHORT
  "strategy_id": "trend_gmma",
  "bucket": "TREND",
  "confidence": 0.82,
  "priority": 0.90,
  "entry_price": 850.0,
  "stop_loss_price": 820.0,
  "capital_allocated": 250000,    # C-44 分配到的資金
  "target_risk_amount": 5000.0,   # C-44 粗算的最大虧損額
  "prelim_shares": 200,          # C-44 用錢/價初步算的股數
  "atr": 20.0,                    # 當日 ATR（或來自指標引擎）
  "atr_pct": 0.025,              # ATR% = ATR / price
  "symbol_liquidity_score": 0.9,
  "symbol_rs_score": 0.88
}
```

### 2️⃣ 來自 Risk Budget Engine（C-43）的風險邊界

* `max_risk_per_trade`（單筆最大風險金額，例如 1R = 1% Equity）
* `remaining_risk_today`（今天還能用多少風險餘額）
* `max_leverage`（最大槓桿倍數）
* `per_symbol_risk_limit`（每個標的最多占總風險多少 %）
* `per_strategy_risk_limit`（每個策略最多多少 %）

### 3️⃣ 來自 Regime Engine（C-42）的市況風險模式

* `regime`: BULL / BEAR / SIDEWAY / CRASH …
* `risk_mode`: AGGRESSIVE / NORMAL / DEFENSIVE / CAPITAL_PROTECTION

> Risk mode 會影響：
> ✅ 初始部位倍率（例如熊市打 0.5 倍）
> ✅ 允許加碼層數（熊市可能禁止加碼）

---

## C-45.3 Position Sizing Engine 的輸出（Outputs）

對每一筆「決定要執行的交易」，輸出：

```python
{
  "symbol": "2330",
  "strategy_id": "trend_gmma",
  "side": "LONG",

  # 初始部位
  "entry_price": 850.0,
  "initial_shares": 120,            # 實際初始要下的股數（風險/資金都控制過）

  # 風險資料
  "per_share_risk": 30.0,           # entry - stop
  "initial_risk_amount": 3600.0,    # 120 * 30
  "max_risk_allowed": 5000.0,       # 由 C-43 下來的上限（or 這筆決策計算）

  # 加碼計畫（可以給 Order Manager 使用）
  "pyramid_plan": [
      {"trigger_R": 1.0, "add_fraction": 0.5},   # 漲到 +1R 加 0.5 倍
      {"trigger_R": 2.0, "add_fraction": 0.5},   # 漲到 +2R 再加 0.5 倍
  ],

  # 減碼計畫
  "scaleout_plan": [
      {"trigger_R": 1.5, "sell_fraction": 0.3},
      {"trigger_R": 3.0, "sell_fraction": 0.7}
  ],

  # 方便後續追蹤
  "regime": "STEADY_BULL",
  "risk_mode": "NORMAL"
}
```

之後：

* Backtest Engine / Live Trading Engine 可以拿這些資訊，根據浮動盈虧（以 R 倍數表示），觸發加減碼。

---

## C-45.4 TAITS_S1 支援哪些 Position Sizing 模式？

你未來可以擴充很多種，但 **V1 建議實作核心三種**：

1. **固定金額模式（Fixed Capital per Trade）**

   * 每筆最多不超過該策略對應的 `capital_allocated`
   * 不看 stop、只看資金，適合很早期的簡化版 / 測試用

2. **固定風險比例模式（Fixed Fractional Risk / %Risk）✅ 主力**

   * 每筆風險 = `account_equity * risk_per_trade_pct`
   * 利用「距離停損價」計算可容許股數
   * 這是專業交易最常用的模式之一（Van Tharp / Turtle 都類似）

3. **ATR 單位模式（Volatility Unit / N-Unit）✅ 建議搭配**

   * 每一個 ATR 視為 1 個「風險單位」
   * 初始部位可能是 1N 或 2N，之後每漲 1N 加一單
   * 非常適合趨勢策略

你現在的系統設計上，我們可以規定：

> **TAITS_S1 初版：
> 預設採用「固定風險比例（Fixed Fractional）＋ ATR 防呆」，
> ATR-Unit 當成選配（特定策略才開）。**

---

## C-45.5 初始部位計算——完整公式與流程

### Step 0：基本定義

對某筆交易：

* `E = entry_price`
* `S = stop_loss_price`
* `Δ = per_share_risk = |E - S|`
* `C_alloc = capital_allocated`（C-44 給的）
* `R_max_trade = max_risk_per_trade`（C-43 給的單筆風險上限）
* `R_remain = remaining_risk_today`（今天還能用多少風險）
* `R_trade = min(R_max_trade, target_risk_amount_from_C44, R_remain)`
  → 這一筆可承受的風險金額

### Step 1：風險角度能下多少股（Risk-based Shares）

```text
shares_by_risk = floor( R_trade / Δ )
```

如果 Δ = 30 元、R_trade = 5000 → 最大 166 股（下單考慮100股一張）。

### Step 2：資金角度能下多少股（Capital-based Shares）

```text
shares_by_capital = floor( C_alloc / E )
```

例如 C_alloc = 250000, E = 850
→ 約 294 股。

### Step 3：取兩者中比較小的（安全保守）

```text
raw_shares = min(shares_by_risk, shares_by_capital)
```

### Step 4：套用 Regime / Risk Mode 修正倍率

例：

| risk_mode          | size_multiplier |
| ------------------ | --------------- |
| AGGRESSIVE         | 1.2             |
| NORMAL             | 1.0             |
| DEFENSIVE          | 0.5             |
| CAPITAL_PROTECTION | 0.25 或 0        |

```text
adj_shares = floor( raw_shares * size_multiplier )
```

### Step 5：考慮「張數 / 最小交易單位」

台股範例：

```text
final_shares = floor( adj_shares / 100 ) * 100
```

若 final_shares < 100 → 可以規則決定：

* 要嘛不下這筆（標記為 `too_small`）
* 要嘛進一張「最小測試單」（例如只下 100 股，風險很小）

### Step 6：回寫實際風險金額

```text
final_risk_amount = final_shares * Δ
```

若 `final_risk_amount` 遠低於 `R_trade`，代表這筆距離停損較近、或資金剩餘不多，風險其實更保守。

---

## C-45.6 加碼（Pyramiding）規格

### 核心原則

1. **只在浮動獲利時加碼**（不在虧損中加碼攤平）
2. **最多 N 層**（避免無限瘋狂加）
3. **每一層加碼都會重新計算「總風險」**，不可爆掉 C-43 的風險預算
4. **加碼位置用「R 倍數」或「ATR 單位」定義**

### 推薦預設（可放 config）

```yaml
pyramiding:
  enabled: true
  max_layers: 2              # 最多 2 次加碼（共 3 層：初始 + 2 次）
  trigger_R: [1.0, 2.0]      # 盈虧達 +1R 加一次、+2R 再加一次
  add_fraction: [0.5, 0.5]   # 每次加 0.5 倍初始部位
  only_for_buckets: ["TREND", "AI"]  # 只給趨勢/AI 策略用
  disable_in_regimes: ["CRASH", "DEEP_BEAR"]
```

### R 的定義

> 1R = 初始 per_share_risk = Δ

* 若 Long：
  浮盈（per_share）= current_price - entry_price
  當：`浮盈 / Δ >= trigger_R[i]` → 觸發加碼 i

### 加碼股數計算

假設初始部位 `N0`，第 i 次加碼比例 `f_i`：

```text
N_add_i = floor( N0 * f_i )
```

當加碼時：

1. 檢查：`新增風險` 是否會讓「該標的＋該策略＋整體」突破 C-43 上限
2. 若超過 → 這一層加碼作廢 or 縮小
3. 每加一次，可以選擇：

   * 重新計算平均成本與新的「共同停損」
   * 或針對每一層設自己停損（進階版）

> 初版建議：**共用一個動態停損線**，由 Risk Engine / Trailing Stop 另外負責。

---

## C-45.7 減碼（Scale-out）規格

減碼的目的：

* 鎖住部分獲利
* 降低回吐時的心理壓力
* 留一小部分讓大波段繼續跑

### 推薦預設（也放 config）

```yaml
scaleout:
  enabled: true
  rules:
    - trigger_R: 1.5
      sell_fraction: 0.3     # 到 +1.5R 時，先賣出 30%
    - trigger_R: 3.0
      sell_fraction: 0.7     # 到 +3R 時，賣掉剩餘 70%
  min_position_to_keep: 0    # 若希望保留一點倉位，可設 e.g. 0.1
```

實務處理：

* 每當浮盈 / Δ ≥ 某個 trigger_R，就根據 `sell_fraction` 計算要減多少股
* 注意要避免重複觸發（可在 order_state 裡記錄已觸發過哪些階段）

---

## C-45.8 Python 類別骨架（PositionSizingEngine）

這是你可以直接丟給 Cursor 的「骨架」，它跟 C-43 / C-44 可以自然銜接：

```python
# engine/position_sizing_engine.py

from dataclasses import dataclass
from typing import List, Dict, Any, Optional


@dataclass
class PyramidingConfig:
    enabled: bool = True
    max_layers: int = 2
    trigger_R: List[float] = None
    add_fraction: List[float] = None
    only_for_buckets: List[str] = None
    disable_in_regimes: List[str] = None


@dataclass
class ScaleoutConfig:
    enabled: bool = True
    rules: List[Dict[str, float]] = None   # [{"trigger_R": 1.5, "sell_fraction": 0.3}, ...]
    min_position_to_keep: float = 0.0


@dataclass
class RiskContext:
    account_equity: float
    max_risk_per_trade: float
    remaining_risk_today: float
    regime: str
    risk_mode: str   # "AGGRESSIVE", "NORMAL", "DEFENSIVE", "CAPITAL_PROTECTION"


class PositionSizingEngine:

    def __init__(
        self,
        pyramiding_cfg: Optional[PyramidingConfig] = None,
        scaleout_cfg: Optional[ScaleoutConfig] = None,
        logger=None,
    ):
        self.pyramiding_cfg = pyramiding_cfg or PyramidingConfig(
            enabled=True,
            max_layers=2,
            trigger_R=[1.0, 2.0],
            add_fraction=[0.5, 0.5],
            only_for_buckets=["TREND", "AI"],
            disable_in_regimes=["CRASH", "DEEP_BEAR"],
        )
        self.scaleout_cfg = scaleout_cfg or ScaleoutConfig(
            enabled=True,
            rules=[
                {"trigger_R": 1.5, "sell_fraction": 0.3},
                {"trigger_R": 3.0, "sell_fraction": 0.7},
            ],
            min_position_to_keep=0.0,
        )
        self.logger = logger

    # 風險模式 → 部位倍率
    def _risk_mode_multiplier(self, risk_mode: str) -> float:
        if risk_mode == "AGGRESSIVE":
            return 1.2
        if risk_mode == "DEFENSIVE":
            return 0.5
        if risk_mode == "CAPITAL_PROTECTION":
            return 0.25
        return 1.0  # NORMAL

    def compute_initial_position(
        self,
        candidate: Dict[str, Any],
        risk_ctx: RiskContext,
    ) -> Dict[str, Any]:
        """
        輸入：
          - candidate: C-44 給的單筆候選交易（已包含 capital_allocated, target_risk_amount...）
          - risk_ctx: 來自 C-43 / Regime Engine 的風險環境
        輸出：
          - 加上 initial_shares / initial_risk_amount 等欄位的 dict
        """
        entry = candidate["entry_price"]
        stop = candidate["stop_loss_price"]
        capital_allocated = candidate.get("capital_allocated", 0.0)
        target_risk_amount = candidate.get("target_risk_amount", risk_ctx.max_risk_per_trade)

        per_share_risk = abs(entry - stop)
        if per_share_risk <= 0:
            # 不能算風險，直接丟棄或給 0
            if self.logger:
                self.logger.warning(f"Per-share risk <= 0 for {candidate['symbol']}")
            return {**candidate, "initial_shares": 0, "initial_risk_amount": 0.0}

        # 單筆允許使用的風險金額
        R_trade = min(
            risk_ctx.max_risk_per_trade,
            target_risk_amount,
            risk_ctx.remaining_risk_today,
        )
        if R_trade <= 0 or capital_allocated <= 0:
            return {**candidate, "initial_shares": 0, "initial_risk_amount": 0.0}

        # 風險角度可承受的股數
        shares_by_risk = int(R_trade // per_share_risk)
        # 資金角度可買的股數
        shares_by_cap = int(capital_allocated // entry)

        raw_shares = min(shares_by_risk, shares_by_cap)

        # 風險模式倍率
        mult = self._risk_mode_multiplier(risk_ctx.risk_mode)
        adj_shares = int(raw_shares * mult)

        # 台股調整成 100 股一張
        final_shares = (adj_shares // 100) * 100

        if final_shares <= 0:
            return {**candidate, "initial_shares": 0, "initial_risk_amount": 0.0}

        final_risk_amount = final_shares * per_share_risk

        return {
            **candidate,
            "per_share_risk": per_share_risk,
            "initial_shares": final_shares,
            "initial_risk_amount": final_risk_amount,
            "max_risk_allowed": R_trade,
        }

    def build_pyramiding_plan(self, candidate: Dict[str, Any], risk_ctx: RiskContext) -> List[Dict[str, Any]]:
        """
        產生這筆交易的加碼計畫（用 R 倍數定義）
        """
        if not self.pyramiding_cfg.enabled:
            return []

        bucket = candidate.get("bucket", "TREND")
        if self.pyramiding_cfg.only_for_buckets and bucket not in self.pyramiding_cfg.only_for_buckets:
            return []

        if risk_ctx.regime in (self.pyramiding_cfg.disable_in_regimes or []):
            return []

        # 初始股數基準
        N0 = candidate.get("initial_shares", 0)
        if N0 <= 0:
            return []

        trig = self.pyramiding_cfg.trigger_R or []
        frac = self.pyramiding_cfg.add_fraction or []

        plan = []
        for i, R_trig in enumerate(trig[: self.pyramiding_cfg.max_layers]):
            f = frac[i] if i < len(frac) else 0.0
            if f <= 0:
                continue
            plan.append({
                "layer": i + 1,
                "trigger_R": R_trig,
                "add_fraction": f,
                "estimated_add_shares": int(N0 * f),
            })
        return plan

    def build_scaleout_plan(self, candidate: Dict[str, Any]) -> List[Dict[str, Any]]:
        if not self.scaleout_cfg.enabled:
            return []
        N0 = candidate.get("initial_shares", 0)
        if N0 <= 0:
            return []
        plan = []
        for rule in self.scaleout_cfg.rules or []:
            plan.append({
                "trigger_R": rule["trigger_R"],
                "sell_fraction": rule["sell_fraction"],
                "estimated_sell_shares": int(N0 * rule["sell_fraction"]),
            })
        return plan

    def process_candidates(
        self,
        candidates: List[Dict[str, Any]],
        risk_ctx: RiskContext,
    ) -> List[Dict[str, Any]]:
        """
        對所有候選交易：
        - 計算 initial position size
        - 建立 pyramiding / scaleout 計畫
        """
        results = []
        for c in candidates:
            enriched = self.compute_initial_position(c, risk_ctx)
            if enriched.get("initial_shares", 0) <= 0:
                continue  # 部位太小或不適合交易

            pyr_plan = self.build_pyramiding_plan(enriched, risk_ctx)
            so_plan = self.build_scaleout_plan(enriched)

            enriched["pyramid_plan"] = pyr_plan
            enriched["scaleout_plan"] = so_plan
            enriched["regime"] = risk_ctx.regime
            enriched["risk_mode"] = risk_ctx.risk_mode

            results.append(enriched)

        return results
```

---

## C-45.9 在整體 Orchestrator 流程中的位置

你目前的整體順序可以整理成：

1. **Data Layer**：抓資料、清洗、算指標

2. **Strategy Layer**：285 策略 → 產生候選訊號（candidates_raw）

3. **Agent Layer**：

   * 技術 / 籌碼 / 基本面 / 新聞 / 情緒 / AI
   * 過濾掉垃圾訊號，給每筆打分數（confidence, score）

4. **Regime Engine（C-42）**：

   * 判斷市況 + 風險模式 → risk_mode

5. **Risk Budget Engine（C-43）**：

   * 計算今日可用風險 / 單筆風險上限
   * 產生 `RiskContext`

6. **Capital Allocation Engine（C-44）**：

   * 把 Portfolio 資金分配到 Bucket / 策略 / 標的
   * 產生含 `capital_allocated` + `target_risk_amount` 的 candidates

7. **Position Sizing Engine（C-45）✅**：

   * 根據風險和資金 → 算出初始部位 + 加碼/減碼計畫

8. **Risk Engine 最終校正**（可視為 C-43 的子流程）：

   * 檢查所有新開部位加起來是否超過 Portfolio Risk
   * 若超過 → 砍掉優先度最低的 or 做比例縮放

9. **Order Manager / Broker API**：

   * 根據 `initial_shares` 等資訊真正下單
   * 實盤 or sandbox

---
