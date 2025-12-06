我先直接給你結論版 👍
下面這份就是 **TAITS_S1 — C-3：285 策略 Plugin 架構（最終版 10/10）**，
可以直接當成 `docs/C-3_Strategy_Plugin_Architecture.md` 用。

---

# 📘 TAITS_S1 — C-3

**策略 Plugin 架構（Strategy Plugin Architecture, 285 套專用）**

> 目標：
> 讓你可以用 Cursor / VS Code，**自動生成、管理、擴充 285 套策略**，
> 而且結構清楚、不會亂掉。

---

## 1️⃣ 設計目標（為什麼要做這一層？）

TAITS_S1 的策略系統，要同時滿足：

1. **可插拔（Plugin）**

   * 新增策略 = 新增一個 `.py` 檔，不用改核心程式。
2. **可描述（Metadata）**

   * 每個策略都有：類別、編號、名稱、適用盤勢、風險等。
3. **可被 AI 理解（AI Friendly）**

   * Cursor 看檔案就知道：怎麼寫、該回傳什麼格式。
4. **可批次管理（Manager）**

   * `StrategyManager` 能：載入、啟用/停用、跑全部策略。
5. **與 285 策略清單完全對齊**

   * e.g. 策略 1「SMA 突破」⇨ `SmaBreakoutStrategy`
   * 策略 275「Kronos Trend Up」⇨ `KronosTrendUpStrategy`

---

## 2️⃣ 檔案與資料夾結構（Strategies 區）

在 TAITS_S1 專案下的 `/strategies` 建議改成：

```bash
/strategies/
    __init__.py
    base_strategy.py
    registry.py
    common_types.py

    # 類別資料夾（對應你 10 大類別）
    trend/
        __init__.py
        sma_breakout.py       # 策略 1
        ema_trend.py          # 策略 2
        gmma_trend.py         # 策略 10,11,12...
    reversal/
    breakout/
    volume/
    chip/
    fundamental/
    sector/
    news/
    behavioral/
    ai/
```

---

## 3️⃣ 標準型別定義（common_types.py）

> ✅ 建議直接用這一版，之後 Cursor 會超好發揮。

```python
# strategies/common_types.py

from enum import Enum
from dataclasses import dataclass
from typing import Dict, Any, Literal, List


class SignalDirection(str, Enum):
    BUY = "BUY"
    SELL = "SELL"
    HOLD = "HOLD"
    SHORT = "SHORT"   # 若你有真正放空


class StrategyCategory(str, Enum):
    TREND = "TREND"
    REVERSAL = "REVERSAL"
    BREAKOUT = "BREAKOUT"
    VOLUME = "VOLUME"
    CHIP = "CHIP"
    FUNDAMENTAL = "FUNDAMENTAL"
    SECTOR = "SECTOR"
    NEWS = "NEWS"
    BEHAVIORAL = "BEHAVIORAL"
    AI = "AI"


class MarketRegime(str, Enum):
    BULL = "BULL"
    BEAR = "BEAR"
    RANGE = "RANGE"
    VOLATILE = "VOLATILE"
    ANY = "ANY"


@dataclass
class StrategyMetadata:
    id: int                       # 對應 1~285
    name: str
    short_code: str               # 例如 "SMA_BREAK"
    category: StrategyCategory
    tags: List[str]
    regime: MarketRegime = MarketRegime.ANY
    timeframes: List[str] = None  # e.g. ["D", "60m"]
    enabled_by_default: bool = True


@dataclass
class StrategySignal:
    strategy_id: int
    strategy_name: str
    direction: SignalDirection    # BUY/SELL/HOLD/SHORT
    confidence: float             # 0.0 ~ 1.0
    reason: str                   # 給人看 & 給 UI 顯示
    extra: Dict[str, Any] = None  # 附加資訊，例如觸發價、指標值
```

---

## 4️⃣ 基底策略類別（base_strategy.py）

> ⏩ **這一份會取代 C-2 裡面那個非常簡單的 BaseStrategy（C-2 那個是暫時版）。**
> 之後所有 285 策略都要繼承這個。

```python
# strategies/base_strategy.py

from abc import ABC, abstractmethod
from typing import Any, Dict
import pandas as pd

from .common_types import StrategyMetadata, StrategySignal, SignalDirection


class BaseStrategy(ABC):
    """
    所有策略的共同介面。
    每個策略應該是「無狀態」或「僅少量內部狀態」，可重複使用。
    """

    def __init__(self, metadata: StrategyMetadata):
        self.metadata = metadata

    @abstractmethod
    def generate_signal(self, data: pd.DataFrame) -> StrategySignal:
        """
        核心邏輯：
        輸入：已經包含所有指標欄位的 DataFrame（通常是單一標的的歷史資料）
        輸出：StrategySignal（必須永遠回傳，不要回傳 None）
        """
        raise NotImplementedError

    def _hold(self, reason: str = "No clear edge") -> StrategySignal:
        """共用方法：回傳 HOLD 訊號。"""
        return StrategySignal(
            strategy_id=self.metadata.id,
            strategy_name=self.metadata.name,
            direction=SignalDirection.HOLD,
            confidence=0.0,
            reason=reason,
            extra={}
        )
```

---

## 5️⃣ 策略註冊系統（registry.py：Decorator 版）

> 目標：**只要寫好策略 class + 用一個裝飾器註冊**，
> `StrategyManager` 就可以自動掃描全部策略。

```python
# strategies/registry.py

from typing import Dict, Type, List
from .base_strategy import BaseStrategy

# 全域註冊表
_STRATEGY_REGISTRY: Dict[int, Type[BaseStrategy]] = {}


def register_strategy(strategy_id: int):
    """
    用法：
    @register_strategy(1)
    class SmaBreakoutStrategy(BaseStrategy):
        ...
    """

    def decorator(cls: Type[BaseStrategy]):
        if strategy_id in _STRATEGY_REGISTRY:
            raise ValueError(f"Strategy ID {strategy_id} already registered.")
        _STRATEGY_REGISTRY[strategy_id] = cls
        return cls

    return decorator


def get_registered_strategies() -> Dict[int, Type[BaseStrategy]]:
    return dict(_STRATEGY_REGISTRY)
```

---

## 6️⃣ StrategyManager（新版）如何運作

> 🔄 與 C-2 連動：這裡是 **升級版的 StrategyManager**，
> 你可以直接把 C-2 的 strategy_manager.py 換成這版。

```python
# engine/strategy_manager.py

from typing import Dict, List
import pandas as pd

from strategies.registry import get_registered_strategies
from strategies.base_strategy import BaseStrategy
from strategies.common_types import StrategySignal


class StrategyManager:
    """
    功能：
    - 從 registry 載入所有策略類別
    - 產生實例
    - 對每個標的/資料跑全部策略
    """

    def __init__(self):
        self._strategy_instances: Dict[int, BaseStrategy] = {}
        self._load_all_strategies()

    def _load_all_strategies(self):
        registry = get_registered_strategies()
        # 這裡之後可以加：白名單 / 黑名單 / 只啟用某類型 等功能
        for sid, strategy_cls in registry.items():
            # 每個策略的 metadata 由 class 內部提供一個靜態方法
            metadata = strategy_cls.get_metadata()
            self._strategy_instances[sid] = strategy_cls(metadata)

    def run_all(self, df: pd.DataFrame) -> Dict[int, StrategySignal]:
        """
        給定一個標的的完整 DataFrame → 跑所有策略 → 回傳 dict
        key: strategy_id, value: StrategySignal
        """
        results: Dict[int, StrategySignal] = {}

        for sid, instance in self._strategy_instances.items():
            try:
                signal = instance.generate_signal(df)
            except Exception as e:
                # 實務上可以 log 下來
                signal = instance._hold(reason=f"Error: {e}")
            results[sid] = signal

        return results
```

---

## 7️⃣ 實作範例：策略 1 — SMA 突破（SMA Breakout）

對應你之前的定義：

> 收盤價 > SMA20 且 量能 > 5 日均量 → BUY
> 收盤價 < SMA20 → SELL

📁 `strategies/trend/sma_breakout.py`

```python
# strategies/trend/sma_breakout.py

import pandas as pd
from typing import List

from strategies.base_strategy import BaseStrategy
from strategies.common_types import (
    StrategyMetadata,
    StrategySignal,
    SignalDirection,
    StrategyCategory,
    MarketRegime,
)
from strategies.registry import register_strategy


@register_strategy(1)  # 對應「策略 1：SMA 突破」
class SmaBreakoutStrategy(BaseStrategy):

    @staticmethod
    def get_metadata() -> StrategyMetadata:
        return StrategyMetadata(
            id=1,
            name="SMA 突破策略",
            short_code="SMA_BREAKOUT",
            category=StrategyCategory.TREND,
            tags=["SMA", "TREND", "BREAKOUT"],
            regime=MarketRegime.BULL,
            timeframes=["D"],
            enabled_by_default=True,
        )

    def generate_signal(self, data: pd.DataFrame) -> StrategySignal:
        if data.empty or len(data) < 20:
            return self._hold("Not enough data")

        row = data.iloc[-1]

        # 假設 Indicator Manager 已經算好欄位：sma20, vol_ma5
        close = row.get("close")
        sma20 = row.get("sma20")
        volume = row.get("volume")
        vol_ma5 = row.get("vol_ma5")

        # 欄位不存在就 HOLD
        if any(v is None for v in [close, sma20, volume, vol_ma5]):
            return self._hold("Missing required columns")

        # BUY 條件
        if close > sma20 and volume > vol_ma5:
            return StrategySignal(
                strategy_id=self.metadata.id,
                strategy_name=self.metadata.name,
                direction=SignalDirection.BUY,
                confidence=0.8,
                reason="收盤突破 SMA20 且量能放大",
                extra={"close": float(close), "sma20": float(sma20)},
            )

        # SELL 條件
        if close < sma20:
            return StrategySignal(
                strategy_id=self.metadata.id,
                strategy_name=self.metadata.name,
                direction=SignalDirection.SELL,
                confidence=0.7,
                reason="收盤跌破 SMA20",
                extra={"close": float(close), "sma20": float(sma20)},
            )

        # 其他情況 HOLD
        return self._hold("Inconclusive for SMA breakout")
```

---

## 8️⃣ 實作範例：策略 283 — 多策略加權投票（Strategy Voting）

這個策略本身不直接讀 K 線，而是吃 **其他策略的結果**。
在架構上，建議放在 **engine 或 agents** 來實作「加權」，
但如果你堅持要做成一個策略 plugin，也可以這樣設計：

📁 `strategies/ai/strategy_voting.py`

```python
# strategies/ai/strategy_voting.py

from typing import Dict, List
import pandas as pd

from strategies.base_strategy import BaseStrategy
from strategies.common_types import (
    StrategyMetadata,
    StrategySignal,
    SignalDirection,
    StrategyCategory,
    MarketRegime,
)
from strategies.registry import register_strategy


@register_strategy(283)
class StrategyVotingWrapper(BaseStrategy):
    """
    注意：
    這個策略需要由 Orchestrator / StrategyManager 特別處理，
    把「其他策略的結果」透過 extra_data 傳進來。
    """

    @staticmethod
    def get_metadata() -> StrategyMetadata:
        return StrategyMetadata(
            id=283,
            name="多策略加權投票",
            short_code="STRAT_VOTING",
            category=StrategyCategory.AI,
            tags=["VOTING", "META"],
            regime=MarketRegime.ANY,
            timeframes=["D"],
            enabled_by_default=True,
        )

    def generate_signal(self, data: pd.DataFrame, extra_data: Dict = None) -> StrategySignal:
        # 這裡示意：extra_data["strategy_signals"] 是 Dict[int, StrategySignal]
        if not extra_data or "strategy_signals" not in extra_data:
            return self._hold("No strategy results provided")

        signals: Dict[int, StrategySignal] = extra_data["strategy_signals"]

        score = 0.0
        for sig in signals.values():
            if sig.direction == SignalDirection.BUY:
                score += sig.confidence
            elif sig.direction == SignalDirection.SELL:
                score -= sig.confidence

        if score > 0.5:
            return StrategySignal(
                strategy_id=self.metadata.id,
                strategy_name=self.metadata.name,
                direction=SignalDirection.BUY,
                confidence=min(1.0, score),
                reason="多策略投票總分偏多",
                extra={"raw_score": score},
            )
        elif score < -0.5:
            return StrategySignal(
                strategy_id=self.metadata.id,
                strategy_name=self.metadata.name,
                direction=SignalDirection.SELL,
                confidence=min(1.0, -score),
                reason="多策略投票總分偏空",
                extra={"raw_score": score},
            )

        return self._hold("Voting score neutral")
```

> 🔎 備註：
> 真實實作時，`StrategyManager` 可以保留一個特別流程：
> 先跑所有「普通策略」→ 再跑這種「meta 策略」。

---

## 9️⃣ 如何讓 Cursor 幫你「量產 285 策略」

你之後開 Cursor，可以直接對它說：

> 「請依照 `strategies/base_strategy.py`、`common_types.py`、`registry.py` 的規則，
> 幫我把策略清單（1–285）逐一實作成對應的 `.py` 檔，放在正確的資料夾（trend / reversal / volume / ai ...）。
> 每個策略都要：
>
> * 有 `@register_strategy(id)`
> * 有 `get_metadata()`
> * 有 `generate_signal()`（先寫成最小可行邏輯即可）」

這樣它就能看你既有的 C-8～C-20 策略說明，把每個都變成實際程式碼。

---

## 🔚 C-3 小結

你現在已經有：

1. **統一型別定義（common_types）**
2. **完整 BaseStrategy 介面**
3. **Decorator 式註冊機制 registry**
4. **具體 StrategyManager 實作**
5. **示範策略：#1 & #283**

👉 接下來的自然下一步是：

> **C-4：IndicatorManager × StrategyManager × TradingAgents 的實際串接流程（From DataFrame → Signals → Agents → Orchestrator）**

