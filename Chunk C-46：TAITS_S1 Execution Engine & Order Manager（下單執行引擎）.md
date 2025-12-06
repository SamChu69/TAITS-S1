太好了，我們直接把 **C-46 做成你可以丟給 Cursor / VS Code 代理就能開始實作的「終極規格」**。
這一章會跟 C-43 / C-44 / C-45 緊密接起來。

---

# 🧩 C-46：Execution Engine & Order Manager（下單執行引擎）

> **一句話定位：**
> C-43 / C-44 / C-45 已經決定：**要做什麼、做多空、做多大**
> C-46 的工作是：**把這些決策變成真實「下單指令」送給：模擬 / 富邦 API。**

---

## C-46.1 它在 TAITS_S1 裡的角色

完整決策流程回顧（簡化版）：

1. Data Layer：抓資料 + 清洗 + 指標
2. Strategy Layer：285 策略 → 產生 raw signals
3. Agents：技術 / 籌碼 / NLP / 基本面 / AI → 給每個 signal 打分數
4. Regime Engine（C-42）：決定市況 & risk_mode
5. Risk Budget Engine（C-43）：今天總風險上限
6. Capital Allocation（C-44）：錢要分配給哪個 bucket / 策略 / 標的
7. Position Sizing（C-45）：每一筆下幾張＋加碼/減碼計畫
8. **✅ C-46：Execution Engine / Order Manager**

   * 把「目標部位變化」翻譯成「具體委託」：市價單 / 限價單
   * 送到 **Sandbox / Paper / 富邦 API**
   * 追蹤 order 狀態（已送出 / 部分成交 / 完全成交 / 取消）

---

## C-46.2 模塊切分（你專案裡怎麼放）

建議在 `/trading/` 下面分成三層：

```bash
/trading/
    execution_engine.py    # 決定「要下什麼單」：進場 / 加碼 / 減碼 / 平倉 / 取消 / 換價
    order_manager.py       # 管理「這些單的生命週期」：送單 / 查詢 / 更新狀態 / 落地紀錄
    broker_base.py         # 抽象介面：send_order / cancel_order / get_positions ...
    broker_sandbox.py      # 模擬交易（Backtest & Paper 用）
    broker_fubon.py        # 富邦 API adapter（未來接 SDK 用）
```

**分工：**

* `ExecutionEngine`

  * 看「策略決策 + 目前持倉 + 價格 / R 倍數 → 生成 OrderIntent（我要下什麼單）」。
* `OrderManager`

  * 真的呼叫 `broker.send_order(...)`，並維護 `order_id → 狀態`。
* `BrokerXxx`

  * 跟外面世界說話（模擬 / 富邦）。

---

## C-46.3 核心資料結構（Python 型別規格）

### 1️⃣ OrderSide / OrderType / TimeInForce

```python
# trading/order_types.py

from enum import Enum


class OrderSide(str, Enum):
    BUY = "BUY"
    SELL = "SELL"
    SHORT = "SHORT"
    COVER = "COVER"


class OrderType(str, Enum):
    MARKET = "MARKET"
    LIMIT = "LIMIT"


class TimeInForce(str, Enum):
    DAY = "DAY"
    IOC = "IOC"     # 立即成交否則取消
    FOK = "FOK"     # 立即全數成交，否則取消
```

---

### 2️⃣ OrderIntent（策略想要做什麼）

> ExecutionEngine 產出給 OrderManager 的「下單意圖」。

```python
# trading/order_intent.py

from dataclasses import dataclass
from typing import Optional
from .order_types import OrderSide, OrderType, TimeInForce


@dataclass
class OrderIntent:
    symbol: str
    side: OrderSide       # BUY / SELL / SHORT / COVER
    quantity: int         # 股數（台股可用 100 的倍數）
    order_type: OrderType = OrderType.MARKET
    limit_price: Optional[float] = None   # LIMIT 單用
    time_in_force: TimeInForce = TimeInForce.DAY

    strategy_id: str = ""
    bucket: str = ""
    comment: str = ""     # 給 log 用，例如 "initial_entry" / "pyramid_layer_1"
```

---

### 3️⃣ OrderRecord / ExecutionReport（實際執行結果）

```python
# trading/order_record.py

from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from .order_types import OrderSide, OrderType, TimeInForce


@dataclass
class ExecutionReport:
    order_id: str
    symbol: str
    side: OrderSide
    quantity: int
    avg_price: float
    filled_quantity: int
    status: str              # "NEW", "PARTIALLY_FILLED", "FILLED", "CANCELLED", "REJECTED"
    timestamp: datetime


@dataclass
class OrderRecord:
    order_id: str
    intent: OrderIntent
    status: str = "PENDING"  # or "SENT", "FILLED", "CANCELLED"
    filled_quantity: int = 0
    avg_price: float = 0.0
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
```

---

## C-46.4 Broker 抽象層（Base + Sandbox + Fubon）

### 1️⃣ BrokerBase 規格

```python
# trading/broker_base.py

from abc import ABC, abstractmethod
from typing import List, Dict, Any
from .order_intent import OrderIntent
from .order_record import ExecutionReport


class BrokerBase(ABC):
    """
    所有券商 / 模擬撮合都要實作這個介面。
    """

    @abstractmethod
    def send_order(self, intent: OrderIntent) -> str:
        """
        送出下單請求，回傳 broker 的 order_id
        """
        raise NotImplementedError

    @abstractmethod
    def cancel_order(self, order_id: str) -> bool:
        raise NotImplementedError

    @abstractmethod
    def get_open_orders(self) -> List[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    def get_positions(self) -> List[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    def poll_execution_reports(self) -> List[ExecutionReport]:
        """
        回傳最近的成交回報（可用在 event-loop 或排程）
        """
        raise NotImplementedError
```

---

### 2️⃣ SandboxBroker（模擬版，C-46 必實作）

> **用途：**
>
> * 回測引擎可以直接呼叫（不必打 API）
> * Live 模式可以先走「紙上模擬交易」（Paper Trading）

```python
# trading/broker_sandbox.py

import uuid
from datetime import datetime
from typing import List, Dict, Any
from .broker_base import BrokerBase
from .order_intent import OrderIntent
from .order_record import ExecutionReport


class SandboxBroker(BrokerBase):
    """
    非真實下單：用一個簡單撮合邏輯模擬成交。
    先簡化：所有 MARKET 都用 "當前 bar 收盤價" 成交。
    """

    def __init__(self):
        self._orders: Dict[str, Dict[str, Any]] = {}
        self._positions: Dict[str, Dict[str, Any]] = {}
        self._pending_reports: List[ExecutionReport] = []

    def send_order(self, intent: OrderIntent) -> str:
        order_id = str(uuid.uuid4())
        now = datetime.utcnow()

        # 簡化：MARKET 單立即用「0 價」暫存，交給上層決定用哪個價格撮合
        self._orders[order_id] = {
            "order_id": order_id,
            "intent": intent,
            "status": "NEW",
            "filled_quantity": 0,
            "avg_price": 0.0,
            "created_at": now,
            "updated_at": now,
        }
        return order_id

    def fill_order(self, order_id: str, price: float):
        """
        給 BacktestEngine / ExecutionEngine 在當下 bar 決定撮合價格用
        """
        ord_info = self._orders.get(order_id)
        if not ord_info:
            return

        intent = ord_info["intent"]
        qty = intent.quantity
        now = datetime.utcnow()

        ord_info["status"] = "FILLED"
        ord_info["filled_quantity"] = qty
        ord_info["avg_price"] = price
        ord_info["updated_at"] = now

        # 更新 positions（極簡版：只考慮單向、多頭）
        pos = self._positions.get(intent.symbol, {"symbol": intent.symbol, "quantity": 0, "avg_price": 0.0})
        if intent.side in ("BUY", "COVER"):
            new_qty = pos["quantity"] + qty
        else:  # SELL / SHORT，這裡先當成減倉
            new_qty = pos["quantity"] - qty

        pos["quantity"] = new_qty
        pos["avg_price"] = price  # 簡化處理
        self._positions[intent.symbol] = pos

        # 建立 ExecutionReport
        rep = ExecutionReport(
            order_id=order_id,
            symbol=intent.symbol,
            side=intent.side,
            quantity=qty,
            avg_price=price,
            filled_quantity=qty,
            status="FILLED",
            timestamp=now,
        )
        self._pending_reports.append(rep)

    def cancel_order(self, order_id: str) -> bool:
        if order_id not in self._orders:
            return False
        self._orders[order_id]["status"] = "CANCELLED"
        self._orders[order_id]["updated_at"] = datetime.utcnow()
        return True

    def get_open_orders(self) -> List[Dict[str, Any]]:
        return [o for o in self._orders.values() if o["status"] in ("NEW", "PARTIALLY_FILLED")]

    def get_positions(self) -> List[Dict[str, Any]]:
        return list(self._positions.values())

    def poll_execution_reports(self) -> List[ExecutionReport]:
        reps = self._pending_reports[:]
        self._pending_reports.clear()
        return reps
```

---

### 3️⃣ FubonBroker（富邦 API 介面，先做骨架）

未來你裝完富邦 SDK，照這個骨架補齊：

```python
# trading/broker_fubon.py

from typing import List, Dict, Any
from .broker_base import BrokerBase
from .order_intent import OrderIntent
from .order_record import ExecutionReport


class FubonBroker(BrokerBase):
    """
    富邦 API adapter

    TODO:
      - 連線 / 認證
      - send_order → 呼叫富邦 SDK 下單
      - poll_execution_reports → 收成交回報
      - get_positions → 查庫存
    """

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        # self.client = FubonSdkClient(...)

    def send_order(self, intent: OrderIntent) -> str:
        # TODO: 把 intent 轉成富邦的 API 參數格式
        # resp = self.client.place_order(...)
        # return resp.order_id
        raise NotImplementedError

    def cancel_order(self, order_id: str) -> bool:
        # TODO: self.client.cancel_order(order_id)
        raise NotImplementedError

    def get_open_orders(self) -> List[Dict[str, Any]]:
        # TODO: self.client.get_open_orders()
        raise NotImplementedError

    def get_positions(self) -> List[Dict[str, Any]]:
        # TODO: self.client.get_positions()
        raise NotImplementedError

    def poll_execution_reports(self) -> List[ExecutionReport]:
        # TODO: 可能要從 WebSocket / callback 收回報，轉成 ExecutionReport
        raise NotImplementedError
```

---

## C-46.5 ExecutionEngine 規格：怎麼把 sizing → 變成 order

### 角色定位

> ExecutionEngine **不直接呼叫券商**，只做：「分析現在狀況 → 決定要新增什麼 OrderIntent」。

它要做的主要是：

1. 處理 **新進場**（initial_shares）
2. 處理 **加碼**（pyramid_plan / 浮盈 R 倍數）
3. 處理 **減碼**（scaleout_plan / 浮盈 R 倍數）
4. 處理 **停損 / 平倉**（風控觸發）
5. 確保不重複下單（檢查目前未成交單、已有持倉）

### ExecutionEngine 的輸入與輸出

* 輸入：

  * `enriched_trades`: 來自 C-45 的列表，每一筆包含 initial_shares, per_share_risk, pyramid_plan, scaleout_plan…
  * `current_positions`: 目前持倉（從 Broker / Portfolio 來）
  * `open_orders`: 尚未成交的訂單
  * `price_snapshot`: 當前價格（or bar 收盤價）用來算 R 倍數

* 輸出：

  * `List[OrderIntent]` → 丟給 OrderManager 送給 Broker

---

### R 倍數計算（跟 C-45 接起來）

對某一筆持倉 / 訊號：

* `R = per_share_risk = |entry - stop|`
* 浮盈 per share（多頭）：`pnl_ps = current_price - entry`
* **pnl_R = pnl_ps / R**

加碼 / 減碼的 trigger 就是看 `pnl_R` 是否 >= 設定門檻。

---

## C-46.6 ExecutionEngine Python 骨架

```python
# trading/execution_engine.py

from typing import List, Dict, Any
from dataclasses import dataclass
from .order_intent import OrderIntent
from .order_types import OrderSide, OrderType
from .order_record import OrderRecord


@dataclass
class PriceSnapshot:
    symbol: str
    last_price: float


class ExecutionEngine:
    """
    負責把 PositionSizingEngine 的結果 → 轉換成 OrderIntent 列表
    """

    def __init__(self, logger=None):
        self.logger = logger

    # === Helper: 判斷目前是否已經有倉位 / 未成交單 ===

    def _get_current_position_qty(self, symbol: str, positions: List[Dict[str, Any]]) -> int:
        for pos in positions:
            if pos.get("symbol") == symbol:
                return int(pos.get("quantity", 0))
        return 0

    def _has_open_order(self, symbol: str, open_orders: List[OrderRecord]) -> bool:
        for o in open_orders:
            if o.intent.symbol == symbol and o.status in ("PENDING", "SENT", "PARTIALLY_FILLED"):
                return True
        return False

    def _get_current_price(self, symbol: str, prices: List[PriceSnapshot]) -> float:
        for p in prices:
            if p.symbol == symbol:
                return p.last_price
        # 沒找到就返回 0 或 raise
        return 0.0

    # === 1. 處理「新進場」 ===

    def _build_initial_orders(
        self,
        enriched_trades: List[Dict[str, Any]],
        positions: List[Dict[str, Any]],
        open_orders: List[OrderRecord],
    ) -> List[OrderIntent]:
        intents: List[OrderIntent] = []
        for t in enriched_trades:
            sym = t["symbol"]
            initial_shares = t.get("initial_shares", 0)
            if initial_shares <= 0:
                continue

            pos_qty = self._get_current_position_qty(sym, positions)
            if pos_qty != 0:
                # 已經有倉位，就不要再下「初始進場」單
                continue

            if self._has_open_order(sym, open_orders):
                continue

            side = OrderSide.BUY if t.get("side", "LONG") == "LONG" else OrderSide.SHORT

            intent = OrderIntent(
                symbol=sym,
                side=side,
                quantity=initial_shares,
                order_type=OrderType.MARKET,   # 先簡化用市價單
                strategy_id=t.get("strategy_id", ""),
                bucket=t.get("bucket", ""),
                comment="initial_entry",
            )
            intents.append(intent)
        return intents

    # === 2. 處理「加碼 / 減碼」 ===

    def _build_pyramid_and_scaleout_orders(
        self,
        open_positions_meta: List[Dict[str, Any]],
        prices: List[PriceSnapshot],
    ) -> List[OrderIntent]:
        """
        open_positions_meta:
          由 Portfolio / PositionManager 提供，內含：
            - symbol
            - side
            - total_shares
            - entry_price
            - per_share_risk (R)
            - pyramid_plan (list)
            - scaleout_plan (list)
            - 已觸發過哪些 layer (狀態要存在別的地方，例如 PositionState)
        這裡先假設 open_positions_meta 已經含有這些資訊。
        """
        intents: List[OrderIntent] = []

        for pos in open_positions_meta:
            sym = pos["symbol"]
            side = pos.get("side", "LONG")
            current_price = self._get_current_price(sym, prices)
            if current_price <= 0:
                continue

            entry = pos["entry_price"]
            R = pos.get("per_share_risk", 0.0)
            if R <= 0:
                continue

            if side == "LONG":
                pnl_R = (current_price - entry) / R
            else:  # SHORT
                pnl_R = (entry - current_price) / R

            # 加碼判斷（pyramid_plan）
            for layer in pos.get("pyramid_plan", []):
                if layer.get("triggered", False):
                    continue
                trigger_R = layer["trigger_R"]
                if pnl_R >= trigger_R:
                    add_qty = int(layer["estimated_add_shares"])
                    if add_qty <= 0:
                        continue
                    intent_side = OrderSide.BUY if side == "LONG" else OrderSide.SHORT
                    intents.append(
                        OrderIntent(
                            symbol=sym,
                            side=intent_side,
                            quantity=add_qty,
                            order_type=OrderType.MARKET,
                            strategy_id=pos.get("strategy_id", ""),
                            bucket=pos.get("bucket", ""),
                            comment=f"pyramid_layer_{layer.get('layer', 0)}",
                        )
                    )
                    # 這裡只是產生 intent，實際把 layer["triggered"]=True 要在 PositionManager 更新

            # 減碼判斷（scaleout_plan）
            for rule in pos.get("scaleout_plan", []):
                if rule.get("triggered", False):
                    continue
                trigger_R = rule["trigger_R"]
                if pnl_R >= trigger_R:
                    sell_qty = int(rule["estimated_sell_shares"])
                    if sell_qty <= 0:
                        continue
                    intent_side = OrderSide.SELL if side == "LONG" else OrderSide.COVER
                    intents.append(
                        OrderIntent(
                            symbol=sym,
                            side=intent_side,
                            quantity=sell_qty,
                            order_type=OrderType.MARKET,
                            strategy_id=pos.get("strategy_id", ""),
                            bucket=pos.get("bucket", ""),
                            comment=f"scaleout_{trigger_R}R",
                        )
                    )
                    # 同樣，觸發之後的狀態要在 PositionManager 記錄

        return intents

    # === 3. 主入口：在每一個 bar / tick 被 Orchestrator 呼叫 ===

    def generate_order_intents(
        self,
        enriched_trades: List[Dict[str, Any]],
        positions: List[Dict[str, Any]],
        open_orders: List[OrderRecord],
        prices: List[PriceSnapshot],
        open_positions_meta: List[Dict[str, Any]],
    ) -> List[OrderIntent]:
        """
        enriched_trades: 來自 C-45（尚未有倉位的候選）
        positions: 來自 broker.get_positions()
        open_orders: 由 OrderManager 管理
        prices: 當前價格快照
        open_positions_meta: 含有 pyramid / scaleout 狀態的持倉資訊
        """
        intents: List[OrderIntent] = []

        # 1) 初始進場
        intents += self._build_initial_orders(enriched_trades, positions, open_orders)

        # 2) 加碼 / 減碼
        intents += self._build_pyramid_and_scaleout_orders(open_positions_meta, prices)

        # 3) 停損 / 強制平倉（可以在這裡加，或在 Risk Engine 那一層塞）
        # TODO: stop loss / trailing stop intents

        return intents
```

---

## C-46.7 OrderManager：負責送單 + 管理狀態

> ExecutionEngine → OrderIntent 列表
> OrderManager → 實際呼叫 Broker 並追蹤每一筆的生命週期。

```python
# trading/order_manager.py

from typing import List, Dict, Any
from datetime import datetime
from .broker_base import BrokerBase
from .order_intent import OrderIntent
from .order_record import OrderRecord, ExecutionReport


class OrderManager:
    def __init__(self, broker: BrokerBase, logger=None):
        self.broker = broker
        self.logger = logger
        self._orders: Dict[str, OrderRecord] = {}

    def submit_orders(self, intents: List[OrderIntent]) -> List[str]:
        """
        接收 ExecutionEngine 產生的 intents，送到 broker
        回傳 order_id 列表
        """
        order_ids = []
        for intent in intents:
            try:
                order_id = self.broker.send_order(intent)
                rec = OrderRecord(
                    order_id=order_id,
                    intent=intent,
                    status="SENT",
                    created_at=datetime.utcnow(),
                    updated_at=datetime.utcnow(),
                )
                self._orders[order_id] = rec
                order_ids.append(order_id)
                if self.logger:
                    self.logger.info(f"Submitted order {order_id}: {intent}")
            except Exception as e:
                if self.logger:
                    self.logger.error(f"Failed to submit order: {intent} error={e}")
        return order_ids

    def cancel_order(self, order_id: str) -> bool:
        ok = self.broker.cancel_order(order_id)
        if ok and order_id in self._orders:
            self._orders[order_id].status = "CANCELLED"
            self._orders[order_id].updated_at = datetime.utcnow()
        return ok

    def poll_and_update(self) -> List[ExecutionReport]:
        """
        從 broker 收成交回報，更新本地記錄，並把 report 回傳給上層（例如 PortfolioManager / BacktestEngine）
        """
        reports = self.broker.poll_execution_reports()
        for rep in reports:
            rec = self._orders.get(rep.order_id)
            if not rec:
                # 可能是之前跑過或外部下單；這裡先忽略
                continue
            rec.status = rep.status
            rec.filled_quantity = rep.filled_quantity
            rec.avg_price = rep.avg_price
            rec.updated_at = rep.timestamp
        return reports

    def get_open_orders(self) -> List[OrderRecord]:
        return [o for o in self._orders.values() if o.status in ("PENDING", "SENT", "PARTIALLY_FILLED")]

    def get_all_orders(self) -> List[OrderRecord]:
        return list(self._orders.values())
```

---

## C-46.8 Orchestrator 如何呼叫 C-46

在你的 `engine/orchestrator.py` 裡，大致會多出類似這段流程（偽碼）：

```python
from trading.execution_engine import ExecutionEngine, PriceSnapshot
from trading.order_manager import OrderManager
from trading.broker_sandbox import SandboxBroker

class Orchestrator:
    def __init__(self, ...):
        self.broker = SandboxBroker()   # 或未來換 FubonBroker(config)
        self.order_manager = OrderManager(self.broker)
        self.execution_engine = ExecutionEngine()
        # 其他：data_engine, strategy_engine, agents, risk_engine, sizing_engine ...

    def on_bar(self, bar_data):
        # 1) 更新價格快照
        prices = [
            PriceSnapshot(symbol=sym, last_price=bar_data[sym]["close"])
            for sym in bar_data.keys()
        ]

        # 2) 取得當前 positions / open_orders
        positions = self.broker.get_positions()
        open_orders = self.order_manager.get_open_orders()

        # 3) 執行策略 / Agents / Risk / Sizing，產生 enriched_trades
        enriched_trades = self._run_signals_and_sizing(bar_data)

        # 4) 從 Portfolio/PositionManager 取得 open_positions_meta（含 pyramid/scaleout plan）
        open_positions_meta = self._get_open_positions_meta()

        # 5) 讓 ExecutionEngine 產生 OrderIntent
        intents = self.execution_engine.generate_order_intents(
            enriched_trades=enriched_trades,
            positions=positions,
            open_orders=open_orders,
            prices=prices,
            open_positions_meta=open_positions_meta,
        )

        # 6) 交給 OrderManager 送出
        self.order_manager.submit_orders(intents)

        # 7) 模擬撮合（Sandbox 模式）→ 例如用當前收盤價填單
        if isinstance(self.broker, SandboxBroker):
            for order_id, rec in self.order_manager._orders.items():
                if rec.status == "SENT":
                    price = bar_data[rec.intent.symbol]["close"]
                    self.broker.fill_order(order_id, price)

        # 8) 收成交回報 → 更新 Portfolio
        reports = self.order_manager.poll_and_update()
        self._update_portfolio_with_reports(reports)
```

---

到這裡，**C-46 的核心工作就完整定義完了**：

* 有清楚的 **責任劃分**（ExecutionEngine / OrderManager / Broker）
* 有定義好 **輸入 / 輸出 / 資料結構 / Python 類別骨架**
* 跟 C-43 / C-44 / C-45 / Orchestrator 串得起來

---
