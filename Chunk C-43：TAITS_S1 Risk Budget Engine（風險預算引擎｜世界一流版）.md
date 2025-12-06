# 🛡 TAITS_S1 — C-43：Risk Budget Engine（風險預算引擎｜世界一流 10/10 版）

> 目標：**讓整個系統「不會因為幾筆錯誤交易就爆炸」，而是穩定、可控地承擔風險。**
> C-42 在決定「現在是什麼市場環境」，C-43 負責：「在這種環境下，我最多可以冒多大的險？」

---

## 🎯 C-43.1 角色定位：Risk Budget Engine 做什麼？

**一句話：**

> 它不管「買哪一檔」，它只決定「最多買多少、加多少碼、承受多少風險」。

在 TAITS_S1 中，它負責：

1. **決定整體風險水位**（今天要 aggressive 還是 defensive）
2. **分配風險到：資產 / 策略 / 單筆交易**
3. **限制最大虧損與回撤**（避免帳戶爆掉）
4. **將 Orchestrator 的「方向」轉成「安全可執行的部位大小」**

---

## 🧱 C-43.2 風險預算的層級（4 層架構）

Risk Budget Engine 的決策分成 4 個層次：

1. **Portfolio Level（整體資金層）**

   * 今天總體最多可冒風險幾 %？
   * 單日最大虧損、單週最大虧損是多少？
2. **Strategy Level（策略層）**

   * 每個策略最多可以用多少風險？（趨勢 vs 反轉 vs AI）
3. **Asset Level（標的層）**

   * 單一股票（如 2330）最多可用多少資金 / 多少風險？
4. **Trade Level（單筆交易層）**

   * 此筆進場：

     * 買幾張？
     * 停損多少？
     * 這筆虧光時佔帳戶幾 % 損失？

你可以理解成：

> 「先決定今天整個遊戲最多輸多少，再決定每個策略、每檔股票、每一筆單可以輸多少。」

---

## 📥 C-43.3 Risk Budget Engine 的輸入（Inputs）

Risk Budget Engine 不會自己亂猜，它會讀這些：

1. **帳戶/資金狀況**

   * `account_equity`（總權益）
   * `cash_available`（可動用現金）
   * `current_drawdown`（目前回撤）
   * `max_drawdown_30d`（30 日內最大回撤）

2. **市場環境（來自 C-42 Regime Engine）**

   * `regime`：BULL / BEAR / SIDEWAY / VOLATILE
   * `regime_score`：-1 ~ +1
   * `risk_mode`：AGGRESSIVE / NORMAL / DEFENSIVE / CAPITAL_PROTECTION

3. **風控設定（config）**

   * `max_daily_loss_pct`（每日最大虧損%）
   * `max_trade_risk_pct`（單筆最大風險%）
   * `max_asset_risk_pct`（單一標的最大風險%）
   * `max_strategy_risk_pct`（每個策略最大風險%）
   * `regime_factor`（根據 Regime 調整風險）

4. **標的波動資訊**

   * `ATR` 或 `ATR%`
   * `price`
   * `lot_size`（台股一張 = 100 股）

5. **策略與信號資訊**

   * `strategy_id`
   * `signal_confidence`（0~1）
   * `side`（LONG / SHORT）

---

## 📤 C-43.4 Risk Budget Engine 的輸出（Outputs）

最後 Risk Budget Engine 要給系統：

1. **Portfolio Risk Limit**

   * `max_portfolio_risk_today`
   * `remaining_risk_budget_today`

2. **Strategy Risk Quota**

   * `strategy_risk_limit[strategy_id]`
   * `strategy_risk_used[strategy_id]`

3. **Asset Risk Quota**

   * `asset_risk_limit[symbol]`
   * `asset_risk_used[symbol]`

4. **每筆交易建議**

   * `position_size`（張數）
   * `entry_price`
   * `stop_loss_price`
   * `risk_amount`（這筆最多虧多少）
   * `accepted`（True/False：這筆單是否允許進場）

這些輸出會由：

* **Order Manager（下單）**
* **Risk Agent（風控智能體）**
* **Orchestrator（總控制）**

直接使用。

---

## 📐 C-43.5 核心原則（Risk Budget 政策）

### 1️⃣ 單筆交易風險：R（Risk per Trade）

一般專業交易會設定：

* 單筆最大風險 = 帳戶總資金的 **0.25% ~ 1%**
* 例如：100 萬本金，1% = **1 萬** → 一筆最多虧 1 萬

> TAITS_S1 建議預設：
>
> * **Normal 模式：0.5% / trade**
> * **Aggressive：1.0% / trade**
> * **Defensive：0.25% / trade**

---

### 2️⃣ 風險 = 部位大小 × 停損距離

> **R = position_size × (entry_price – stop_loss_price)**

例如：

* 單筆風險允許：10,000
* entry = 100 元
* stop = 95 元
* 單股風險 = 5 元

→ 可承受 10,000 / 5 = **2,000 股 → 20 張**

---

### 3️⃣ 不只看「單筆」，還要看「總體」

風險預算要避免這些：

* 同一類股（例如 AI 股）全部滿倉
* 同一方向（ALL LONG）在 Strong Bear 裡被屠殺
* 多策略同時對同一標的下單（槓上加槓）

所以 C-43 要做到：

* 單標的最大總風險（所有策略加總）
* 單策略最大總風險
* 整體帳戶當日最大風險

---

## ⚙️ C-43.6 Regime × Risk Mode 對應表

C-42 給的 Regime，會決定 Risk Mode：

| Regime           | Risk Mode          | 單筆風險      | 總曝險上限     |
| ---------------- | ------------------ | --------- | --------- |
| Steady Bull      | AGGRESSIVE         | 1.0%      | 120%（含融資） |
| Strong Bull      | AGGRESSIVE         | 1.0%      | 130%      |
| Toppy Bull       | NORMAL             | 0.5%      | 80%       |
| Sideway Low Vol  | NORMAL             | 0.5%      | 70%       |
| Sideway High Vol | DEFENSIVE          | 0.25%     | 50%       |
| Weak Bear        | DEFENSIVE          | 0.25%     | 40%       |
| Strong Bear      | CAPITAL_PROTECTION | 0.1%      | 20%       |
| Capitulation     | CAPITAL_PROTECTION | 0.1% or 0 | 0~10%     |

---

## 🔄 C-43.7 完整流程（從信號到風險允許）

1. **Orchestrator 產生交易計畫：**

```python
plan = {
    "symbol": "2330",
    "side": "LONG",
    "strategy_id": "trend_gmma",
    "confidence": 0.78,
    "entry_price": 850.0,
    "stop_loss_price": 820.0
}
```

2. **Regime Engine 提供市場狀態：**

```python
regime = "BULL"
risk_mode = "NORMAL"
```

3. **Risk Budget Engine：**

   * 讀取 `account_equity = 1,000,000`
   * Normal 模式 → 單筆風險 = 0.5% = 5,000
   * 風險 / 股 = 850 - 820 = 30
   * 可承受股數 = 5,000 / 30 ≒ 166 股 → 1 張（台股 100 股一張）
   * 再檢查：

     * 這檔目前已用風險？（ex: 已有 3000 風險）
     * 該策略目前已用風險？
     * 今日累積虧損是否接近 daily max loss？

4. **如果一切 OK → 回傳：**

```python
{
  "accepted": True,
  "max_shares": 100,
  "risk_amount": 3000,
  "reason": "within risk budget"
}
```

5. **如果超出風險 → 回傳：**

```python
{
  "accepted": False,
  "max_shares": 0,
  "risk_amount": 0,
  "reason": "daily risk limit exceeded"
}
```

---

## 🧩 C-43.8 Python 類別骨架（可以直接給 Cursor 實作）

```python
# engine/risk_budget_engine.py

from dataclasses import dataclass

@dataclass
class RiskConfig:
    max_daily_loss_pct: float = 0.02        # 每日 -2% 上限
    max_trade_risk_pct_normal: float = 0.005  # 0.5% / trade
    max_trade_risk_pct_defensive: float = 0.0025
    max_trade_risk_pct_aggressive: float = 0.01
    max_symbol_risk_pct: float = 0.02      # 單一標的總風險上限
    max_strategy_risk_pct: float = 0.05    # 單策略總風險上限

class RiskBudgetEngine:

    def __init__(self, risk_config: RiskConfig, logger=None):
        self.cfg = risk_config
        self.logger = logger
        # 這些可以用來記錄已用風險
        self.symbol_risk_used = {}
        self.strategy_risk_used = {}
        self.daily_loss = 0.0

    def reset_daily(self):
        """每天開盤前重置."""
        self.symbol_risk_used.clear()
        self.strategy_risk_used.clear()
        self.daily_loss = 0.0

    def update_daily_pl(self, realized_pl: float):
        """更新目前已虧損 / 盈利."""
        self.daily_loss += min(realized_pl, 0)  # 只計虧損部分

    def _get_trade_risk_pct(self, risk_mode: str) -> float:
        if risk_mode == "AGGRESSIVE":
            return self.cfg.max_trade_risk_pct_aggressive
        elif risk_mode == "DEFENSIVE" or risk_mode == "CAPITAL_PROTECTION":
            return self.cfg.max_trade_risk_pct_defensive
        else:
            return self.cfg.max_trade_risk_pct_normal

    def compute_position_size(
        self,
        account_equity: float,
        symbol: str,
        strategy_id: str,
        entry_price: float,
        stop_loss_price: float,
        regime: str,
        risk_mode: str,
    ):
        """給一筆交易，計算可以承擔的最大部位大小."""

        # 1. 檢查每日虧損上限
        max_daily_loss = -self.cfg.max_daily_loss_pct * account_equity
        if self.daily_loss <= max_daily_loss:
            return {
                "accepted": False,
                "max_shares": 0,
                "reason": "DAILY_LOSS_LIMIT_REACHED",
            }

        # 2. 單筆風險上限
        trade_risk_pct = self._get_trade_risk_pct(risk_mode)
        max_trade_risk_amount = trade_risk_pct * account_equity

        risk_per_share = abs(entry_price - stop_loss_price)
        if risk_per_share <= 0:
            return {
                "accepted": False,
                "max_shares": 0,
                "reason": "INVALID_STOP_DISTANCE",
            }

        max_shares_by_trade = int(max_trade_risk_amount / risk_per_share)

        # 3. 單標的風險上限
        used_symbol_risk = self.symbol_risk_used.get(symbol, 0.0)
        max_symbol_risk = self.cfg.max_symbol_risk_pct * account_equity
        remaining_symbol_risk = max(0.0, max_symbol_risk - used_symbol_risk)
        max_shares_by_symbol = int(remaining_symbol_risk / risk_per_share)

        # 4. 單策略風險上限
        used_strategy_risk = self.strategy_risk_used.get(strategy_id, 0.0)
        max_strategy_risk = self.cfg.max_strategy_risk_pct * account_equity
        remaining_strategy_risk = max(0.0, max_strategy_risk - used_strategy_risk)
        max_shares_by_strategy = int(remaining_strategy_risk / risk_per_share)

        # 5. 綜合限制
        max_shares = min(
            max_shares_by_trade,
            max_shares_by_symbol,
            max_shares_by_strategy,
        )

        if max_shares <= 0:
            return {
                "accepted": False,
                "max_shares": 0,
                "reason": "NO_RISK_BUDGET_LEFT",
            }

        # 6. 回寫已用風險（假設之後會真的下這個部位）
        risk_amount = max_shares * risk_per_share
        self.symbol_risk_used[symbol] = used_symbol_risk + risk_amount
        self.strategy_risk_used[strategy_id] = used_strategy_risk + risk_amount

        return {
            "accepted": True,
            "max_shares": max_shares,
            "risk_amount": risk_amount,
            "reason": "OK",
        }
```

> 你之後可以直接跟 Cursor 說：
> **「請依照這個 RiskBudgetEngine 的 Spec 實作並整合到 TAITS_S1 專案。」**

---

## 🔗 C-43.9 與整個 TAITS_S1 的整合位置

**1️⃣ Orchestrator（C-37）**

* 在產生 Trade Plan 後，呼叫 `RiskBudgetEngine.compute_position_size`
* 若 `accepted = False` → 直接丟掉這筆交易或記錄原因

**2️⃣ Agents**

* Risk Agent 會使用 Risk Budget Engine 的資料判斷

  * 是否降低某策略的信心
  * 是否進入「暫停交易模式」

**3️⃣ Execution Engine / Order Manager（C-39）**

* 只根據 `max_shares` 下單
* 不直接計算風險，交給 RiskBudgetEngine 集中管理

**4️⃣ UI / Dashboard**

* 顯示：

  * 今日已用風險 / 上限
  * 各策略已用風險
  * 各標的已用風險
  * 目前 Risk Mode：AGGRESSIVE / NORMAL / DEFENSIVE / CAPITAL_PROTECTION

---

## ✅ 小結：C-43 你現在擁有的東西

* 一個可以**直接實作**的 Risk Budget 核心設計
* 完整層級：Portfolio / Strategy / Asset / Trade
* 清楚可轉成程式的計算流程
* 與 Regime Engine、Orchestrator、Order Manager 完全貼合

---
