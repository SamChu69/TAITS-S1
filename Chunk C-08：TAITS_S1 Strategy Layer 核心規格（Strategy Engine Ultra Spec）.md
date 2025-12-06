我會延續前面 C-1～C-7 的風格，讓 Cursor / VS Code 看到就能接著寫程式。

---

# 📘 C-8 — Strategy Layer 核心規格（Strategy Engine Ultra Spec）

> 這一章是在定義：
> **「策略怎麼寫？怎麼被載入？怎麼輸出 BUY / SELL / HOLD？怎麼給 Agents 和 Orchestrator 用？」**

你之前 285 策略是「內容」。
**C-8 是「架構 ＋ 介面 ＋ 規格」。**

---

## C-8.1 Strategy Layer 的定位

在 TAITS_S1 裡，策略層是這樣的位置：

```text
Data (C-3/C-4)
  ↓
Validator (C-5)
  ↓
Indicators (C-7)
  ↓
⚡ Strategy Layer（本章 C-8）
  ↓
Agents（D-2 技術、籌碼、AI…）
  ↓
Orchestrator（D-4）
  ↓
Backtest / Sandbox / Live
```

**Strategy Layer 的工作只有一件事：**

> 給定：
>
> * 「某一檔股票 / 時間」的 **資料 + 指標 + 狀態**
>   → 回傳：該策略對這個標的的 **信號（多空 / 停看）＋ 信心分數＋理由**

---

## C-8.2 專案目錄定位（與前面章節對齊）

專案結構裡，策略區塊：

```text
TAITS_S1/
│
├── engine/
│   ├── strategy_manager.py   # 本章定義
│   └── ...
│
└── strategies/
    ├── base_strategy.py      # 本章定義
    ├── registry.py           # 策略註冊系統
    ├── sma_breakout.py       # 範例策略
    ├── ema_trend.py          # 範例策略
    ├── rsi_reversal.py       # 範例策略
    └── ...  之後會放 285 個策略
```

---

## C-8.3 統一策略輸入 / 輸出規格（超重要）

### 1️⃣ 輸入（Strategy Input Context）

每一個策略在執行時，都會收到一個標準 context，包含：

* `symbol`：股票代碼
* `as_of`：當前這一根 K 線的時間（date / datetime）
* `df`：這檔股票的 DataFrame（含指標欄位）
* `extra`：額外資訊（大盤、類股、regime、AI 預測…）

設計成一個小物件，比較好擴充：

```python
from dataclasses import dataclass
import pandas as pd
from typing import Dict, Any

@dataclass
class StrategyContext:
    symbol: str
    as_of: pd.Timestamp
    df: pd.DataFrame        # 含 O/H/L/C/V + 指標欄位
    extra: Dict[str, Any]   # regime, sector, ai, macro...
```

---

### 2️⃣ 輸出（Strategy Output）

所有策略統一輸出一個 dict：

```python
{
    "name": "EMA_Trend",
    "symbol": "2330",
    "as_of": Timestamp(...),
    "signal": "BUY",       # BUY / SELL / HOLD / SHORT
    "confidence": 0.0~1.0, # 浮點數
    "score": 0.73,         # 可用於加權的分數（可與 confidence 相同）
    "reason": "EMA20 > EMA60 & price above EMA20",
    "meta": { ... }        # 額外資訊，給 Debug / UI 顯示
}
```

這樣 Orchestrator / Agents 就可以標準化處理。

---

## C-8.4 BaseStrategy 抽象基底類別

📄 檔案：`strategies/base_strategy.py`

```python
from abc import ABC, abstractmethod
from typing import Dict, Any
from .context import StrategyContext  # 你可以放在 base_strategy.py 同一檔

class BaseStrategy(ABC):
    """
    所有策略的共同父類別。
    """

    #: 策略英文代碼（檔案級常數）
    name: str = "BaseStrategy"
    #: 所屬類別（trend / breakout / volume / ai / ...）
    category: str = "base"
    #: 描述文字（給 UI 用）
    description: str = "Abstract base strategy."

    def __init__(self, config: Dict[str, Any] | None = None):
        self.config = config or {}

    @abstractmethod
    def run(self, ctx: StrategyContext) -> Dict[str, Any]:
        """
        主邏輯：
        Input:
            ctx: StrategyContext
        Output:
            dict: 包含 signal, confidence, reason, meta 等欄位
        """
        raise NotImplementedError

    def _default_output(
        self,
        ctx: StrategyContext,
        signal: str = "HOLD",
        confidence: float = 0.0,
        score: float | None = None,
        reason: str = "",
        meta: Dict[str, Any] | None = None,
    ) -> Dict[str, Any]:
        """
        幫助子類快速建立標準輸出格式。
        """
        return {
            "name": self.name,
            "category": self.category,
            "symbol": ctx.symbol,
            "as_of": ctx.as_of,
            "signal": signal,
            "confidence": float(confidence),
            "score": float(score if score is not None else confidence),
            "reason": reason,
            "meta": meta or {},
        }
```

---

## C-8.5 策略註冊系統（Strategy Registry）

📄 檔案：`strategies/registry.py`

```python
from typing import Dict, Type
from .base_strategy import BaseStrategy

STRATEGY_REGISTRY: Dict[str, Type[BaseStrategy]] = {}

def register_strategy(name: str):
    """
    用 decorator 註冊策略類別：
    @register_strategy("EMA_Trend")
    class EMATrendStrategy(BaseStrategy):
        ...
    """
    def decorator(cls: Type[BaseStrategy]):
        STRATEGY_REGISTRY[name] = cls
        cls.name = name     # 順便寫回類別屬性
        return cls
    return decorator
```

這個跟 C-7 的 Indicator Registry 是對稱設計的。
之後你有 285 策略，全都用 `@register_strategy` 加進來即可。

---

## C-8.6 StrategyManager（策略管理器核心）

📄 檔案：`engine/strategy_manager.py`

```python
from typing import List, Dict, Any
import pandas as pd

from strategies.registry import STRATEGY_REGISTRY
from strategies.base_strategy import BaseStrategy, StrategyContext


class StrategyManager:

    def __init__(self, enabled: List[str] | None = None, config: Dict[str, Any] | None = None):
        """
        enabled: 要啟用的策略名稱清單（對應註冊名稱）
        config: 每個策略的個別設定，例如:
            {
              "SMA_Breakout": {"min_volume": 1_000_000},
              "EMA_Trend": {"min_confidence": 0.6}
            }
        """
        self.config = config or {}
        self.enabled_names = enabled or list(STRATEGY_REGISTRY.keys())
        self.strategies: List[BaseStrategy] = []

        self._init_strategies()

    def _init_strategies(self):
        for name in self.enabled_names:
            cls = STRATEGY_REGISTRY.get(name)
            if not cls:
                continue
            cfg = self.config.get(name, {})
            self.strategies.append(cls(cfg))

    def run_for_symbol(
        self,
        symbol: str,
        df: pd.DataFrame,
        as_of: pd.Timestamp,
        extra: Dict[str, Any] | None = None,
    ) -> List[Dict[str, Any]]:
        """
        對某一檔股票 / 某一時間點，執行所有策略。
        """
        if extra is None:
            extra = {}

        ctx = StrategyContext(
            symbol=symbol,
            as_of=as_of,
            df=df,
            extra=extra
        )

        outputs: List[Dict[str, Any]] = []
        for strat in self.strategies:
            try:
                result = strat.run(ctx)
                outputs.append(result)
            except Exception as e:
                # 真實專案可加 logging
                outputs.append({
                    "name": strat.name,
                    "category": strat.category,
                    "symbol": symbol,
                    "as_of": as_of,
                    "signal": "ERROR",
                    "confidence": 0.0,
                    "score": 0.0,
                    "reason": f"Exception: {e}",
                    "meta": {}
                })
        return outputs
```

之後 Orchestrator / Agents 會拿 `outputs` 去做：

* 多數決
* 加權投票
* score 統計

---

## C-8.7 三個示範策略（讓整套流程真的跑得起來）

這三個策略對應你之前 1～20 號策略中的某幾個簡化版。

### 1️⃣ SMA 突破策略（Strategy #1 簡化版）

📄 檔案：`strategies/sma_breakout.py`

```python
import pandas as pd
from .base_strategy import BaseStrategy, StrategyContext
from .registry import register_strategy


@register_strategy("SMA_Breakout")
class SMABreakoutStrategy(BaseStrategy):
    category = "trend"
    description = "Close breaks above SMA, volume filter."

    def run(self, ctx: StrategyContext):
        df = ctx.df
        if len(df) < 21:
            return self._default_output(ctx, "HOLD", 0.0, reason="insufficient data")

        window = self.config.get("window", 20)
        vol_window = self.config.get("vol_window", 5)

        sma_col = f"ma_{window}"
        if sma_col not in df.columns:
            # 若尚未有 MA，這裡簡單動態計算（正式系統會由 Indicator Layer 處理）
            df[sma_col] = df["close"].rolling(window).mean()

        latest = df.iloc[-1]
        vol_ma = df["volume"].rolling(vol_window).mean().iloc[-1]

        cond_price = latest["close"] > latest[sma_col]
        cond_volume = latest["volume"] > vol_ma

        if cond_price and cond_volume:
            return self._default_output(
                ctx,
                signal="BUY",
                confidence=0.7,
                reason=f"Close({latest['close']:.2f}) > SMA{window} & Volume spike",
                meta={"sma": float(latest[sma_col])}
            )
        else:
            return self._default_output(
                ctx,
                signal="HOLD",
                confidence=0.2,
                reason="No breakout",
                meta={"sma": float(latest[sma_col])}
            )
```

---

### 2️⃣ EMA 趨勢策略（Strategy #2 簡化版）

📄 檔案：`strategies/ema_trend.py`

```python
import pandas as pd
from .base_strategy import BaseStrategy, StrategyContext
from .registry import register_strategy


@register_strategy("EMA_Trend")
class EMATrendStrategy(BaseStrategy):
    category = "trend"
    description = "EMA20 > EMA60 & price above EMA20."

    def run(self, ctx: StrategyContext):
        df = ctx.df
        if len(df) < 61:
            return self._default_output(ctx, "HOLD", 0.0, reason="insufficient data")

        fast = self.config.get("fast", 20)
        slow = self.config.get("slow", 60)

        ema_fast_col = f"ema_{fast}"
        ema_slow_col = f"ema_{slow}"

        if ema_fast_col not in df.columns:
            df[ema_fast_col] = df["close"].ewm(span=fast).mean()
        if ema_slow_col not in df.columns:
            df[ema_slow_col] = df["close"].ewm(span=slow).mean()

        latest = df.iloc[-1]

        cond_trend = latest[ema_fast_col] > latest[ema_slow_col]
        cond_price = latest["close"] > latest[ema_fast_col]

        if cond_trend and cond_price:
            return self._default_output(
                ctx,
                signal="BUY",
                confidence=0.8,
                reason=f"Uptrend EMA{fast}>{slow} and price above EMA{fast}",
                meta={
                    "ema_fast": float(latest[ema_fast_col]),
                    "ema_slow": float(latest[ema_slow_col])
                }
            )

        if not cond_trend:
            return self._default_output(
                ctx,
                signal="SELL",
                confidence=0.7,
                reason=f"Trend broken: EMA{fast}<{slow}",
                meta={
                    "ema_fast": float(latest[ema_fast_col]),
                    "ema_slow": float(latest[ema_slow_col])
                }
            )

        return self._default_output(
            ctx,
            signal="HOLD",
            confidence=0.3,
            reason="Trend ok but price below EMA fast",
            meta={
                "ema_fast": float(latest[ema_fast_col]),
                "ema_slow": float(latest[ema_slow_col])
            }
        )
```

---

### 3️⃣ RSI 反轉策略（Strategy #21 簡化版）

📄 檔案：`strategies/rsi_reversal.py`

```python
import pandas as pd
from .base_strategy import BaseStrategy, StrategyContext
from .registry import register_strategy


@register_strategy("RSI_Reversal")
class RSIReversalStrategy(BaseStrategy):
    category = "reversal"
    description = "RSI oversold/overbought reversal."

    def run(self, ctx: StrategyContext):
        df = ctx.df
        period = self.config.get("period", 14)
        rsi_col = f"rsi_{period}"

        if rsi_col not in df.columns:
            # 簡易 RSI 計算（正式版本已在 Indicator Layer）
            delta = df["close"].diff()
            gain = delta.clip(lower=0).rolling(period).mean()
            loss = (-delta.clip(upper=0)).rolling(period).mean()
            rs = gain / loss
            df[rsi_col] = 100 - (100 / (1 + rs))

        latest = df.iloc[-1]
        rsi = latest[rsi_col]

        if rsi < 30:
            return self._default_output(
                ctx,
                signal="BUY",
                confidence=0.75,
                reason=f"RSI oversold ({rsi:.2f})",
                meta={"rsi": float(rsi)}
            )

        if rsi > 70:
            return self._default_output(
                ctx,
                signal="SELL",
                confidence=0.75,
                reason=f"RSI overbought ({rsi:.2f})",
                meta={"rsi": float(rsi)}
            )

        return self._default_output(
            ctx,
            signal="HOLD",
            confidence=0.2,
            reason=f"RSI neutral ({rsi:.2f})",
            meta={"rsi": float(rsi)}
        )
```

---

## C-8.8 Orchestrator / Agent 怎麼用 StrategyManager？

之後在 D-2 / D-4（Agents / Orchestrator）會大概長這樣：

```python
# engine/orchestrator.py 裡面的一段 pseudo
def run_for_symbol(self, symbol: str, df, as_of, extra):
    strategy_outputs = self.strategy_manager.run_for_symbol(
        symbol=symbol,
        df=df,
        as_of=as_of,
        extra=extra
    )

    # 之後交給 AgentManager/SignalAggregator 做加權、投票
    # 再轉成 final_signal / final_confidence
```

---

## ✅ C-8 小結（已達 10/10）

你現在擁有：

* ✅ **Strategy Layer 的完整架構**
* ✅ BaseStrategy 抽象類別
* ✅ 統一的 input（StrategyContext） / output 格式
* ✅ Strategy Registry + StrategyManager 實作
* ✅ 三個可直接實作的示範策略（SMA / EMA / RSI）
* ✅ 與 C-7（Indicator Layer）與 D-系列（Agents / Orchestrator）自然接軌

這一章已經是 **工程師拿去就能在 Cursor 裡開工的等級**。

---

