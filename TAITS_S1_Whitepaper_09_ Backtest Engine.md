# 📘 **CHAPTER 9 — STRATEGY CATALOG PART II

（Strategies 21–40 · RSI / KD / ATR / SuperTrend / PSAR / 通道）**
**TAITS S1 OFFICIAL WHITEPAPER — CHAPTER 9**

# 🌐 **CHAPTER 9 前言**

本章涵蓋 TAITS S1 中：

✔ **動能模型（Momentum）**
✔ **震盪模型（RSI / KD）**
✔ **波動模型（ATR）**
✔ **趨勢反轉（SuperTrend / PSAR）**
✔ **通道系統（BB / KC / Donchian）**

這些策略屬於 **策略全集中使用率最高的一群**，
也是 *Technical Agent* 的主力策略來源。

---

# ———— 🎯 **STRATEGY 21–40 展開開始** ————

---

# **21. RSI 超賣反轉（RSI Oversold Reversal）**

**分類：** 動能 / 反轉（Reversal）
**適用市場：** 盤整、急跌後反彈

### 進場條件

* RSI < 30
* 出現反轉 K（Hammer / Engulfing）

### 出場

* RSI > 50

```python
if rsi < 30 and reversal_k:
    signal = 1
elif rsi > 50:
    signal = -1
```

---

# **22. RSI 高檔鈍化（RSI Bull Range Shift）**

**分類：** 強勢動能
**特徵：** 強勢股 RSI 會維持 60–80 區間

### 進場

* RSI > 70 持續至少 3 根

### 出場

* RSI < 60

```python
if rsi > 70 for 3 bars:
    signal = 1
elif rsi < 60:
    signal = -1
```

---

# **23. KD 黃金交叉（Stochastic Golden Cross）**

**分類：** 動能

```python
if k_cross_up_d:
    signal = 1
elif k_cross_down_d:
    signal = -1
```

---

# **24. KD 高檔鈍化（KD Overbought Sustain）**

**分類：** 動能強勢延伸

```python
if k > 80 and d > 80:
    signal = 1
elif k < 70:
    signal = -1
```

---

# **25. KD 低檔背離（KD Bullish Divergence）**

```python
if price_new_low and kd_higher_low:
    signal = 1
```

---

# **26. ATR 突破（ATR Expansion Breakout）**

**分類：** 波動突破（Volatility Expansion）

```python
if range > 1.5 * atr:
    signal = 1
elif range < atr:
    signal = -1
```

---

# **27. ATR 波動收斂（ATR Squeeze）**

**分類：** 波動壓縮 → 等突破

```python
if atr_falling and bb_squeeze:
    prepare_breakout = True
```

---

# **28. ATR 假突破偵測（ATR False Breakout）**

**分類：** 偽突破（Failed Breakout）

```python
if breakout and atr decreasing:
    signal = -1
```

---

# **29. ATR 停損（ATR Stop Loss Model）**

**分類：** 風控模型

```python
stop = entry_price - 2 * atr
```

---

# **30. Normalized ATR 強弱（ATR% Strength）**

**分類：** 波動強度量化

```python
if atr / close > 0.03:
    signal = 1
```

---

# **31. SuperTrend 趨勢反轉（SuperTrend Flip）**

**分類：** 趨勢 / 反轉

```python
if close > supertrend:
    signal = 1
elif close < supertrend:
    signal = -1
```

---

# **32. SuperTrend + EMA 共振（ST × EMA Confluence）**

```python
if st_long and ema20 > ema60:
    signal = 1
```

---

# **33. PSAR 多空反轉（Parabolic SAR Flip）**

```python
if close > psar:
    signal = 1
elif close < psar:
    signal = -1
```

---

# **34. 布林通道下緣反彈（BB Lower Bounce）**

**分類：** 均值回歸 / 反轉

```python
if low <= bb_lower and reversal_k:
    signal = 1
```

---

# **35. 布林中線回補（BB Mid Reversion）**

```python
if close < bb_lower * 0.98:
    signal = 1
elif close >= bb_mid:
    signal = 0
```

---

# **36. 布林收斂突破（BB Squeeze Breakout）**

```python
if bb_squeeze and close > bb_upper:
    signal = 1
```

---

# **37. Keltner 趨勢突破（Keltner Channel Trend）**

```python
if close > kc_upper:
    signal = 1
elif close < kc_mid:
    signal = -1
```

---

# **38. Keltner 通道假突破（KC Fake Breakout）**

```python
if close > kc_upper and volume < vol_ma5:
    signal = -1
```

---

# **39. Donchian 20 日突破（Donchian Breakout 20）**

```python
if close > donchian_high20:
    signal = 1
elif close < donchian_low20:
    signal = -1
```

---

# **40. Donchian 假突破（Donchian Fakeout）**

```python
if breakout and next_day_close < prev_high:
    signal = -1
```

---
