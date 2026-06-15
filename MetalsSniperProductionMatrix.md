# Metals Sniper - Production Matrix

**Strategy Version:** Pine Script v6  
**Primary Instrument:** Silver (24/5 trading)  
**Timeframe:** Multi-timeframe (tested on 1m-15m)  
**Capital:** $10,000 minimum recommended  
**Risk Model:** Fixed 20% equity per trade  

---

## Overview

Metals Sniper is a session-based range trading strategy optimized for precious metals. It combines two distinct trading engines:

1. **Inner-Range Scalping** — Entry on rejection wicks within the established range
2. **Trend Continuation Breakouts** — Entry on breakouts above/below range extremes

The strategy uses a dynamic box quality filter to skip anomalous ranges and institutional volume confirmation to reduce false signals.

---

## How It Works

### Phase 1: Range Discovery (0000-0700 UTC)

During this window, the strategy captures the high and low to establish a trading range ("box"):

- **boxMax** = highest price during the session
- **boxMin** = lowest price during the session
- **boxHeight** = boxMax - boxMin

This range becomes the basis for all entries and exits during the active trading window.

### Phase 2: Active Trading (0700-1700 UTC)

Once the discovery window closes, the strategy monitors for two types of signals:

#### Signal Type 1: Inner-Range Scalp
Triggers when price touches the range floor/ceiling and rejects back inside:

```
Long Scalp:  Price dips to boxMin, closes above boxMin with bullish candle
Short Scalp: Price rallies to boxMax, closes below boxMax with bearish candle
```

**Confirmation Requirements:**
- Volume > 1.2x 20-bar average (institutional activity)
- Only ONE long and ONE short entry per range
- Box must pass quality filter (see Box Quality Filter section)

#### Signal Type 2: Trend Continuation Breakout
Triggers on clean breakouts:

```
Long Breakout:  Close crosses above boxMax
Short Breakout: Close crosses below boxMin
```

**Confirmation Requirements:**
- Same volume and box quality filters as scalps
- No position size limit (independent of scalp entries)

---

## Box Quality Filter

The strategy automatically skips anomalous ranges to avoid trading during high-volatility outlier sessions.

**How It Works:**
1. Stores the last 5 days of completed box heights in a rolling array
2. Calculates the average of these historical boxes
3. Rejects today's box if it's outside the acceptable range:
   - **Too Giant:** boxHeight > (avgBoxHeight × 1.5)
   - **Too Compressed:** boxHeight < (avgBoxHeight × 0.5)

**When Skipped:**
- Status dashboard shows: "Skipped: Box Too Giant" or "Skipped: Box Too Compressed"
- No entries are generated that day
- The filter automatically learns and adjusts as market volatility changes

**Can be disabled:** Toggle "Enable Box Quality Filter" in settings if you want to trade all ranges.

---

## Entry & Exit Logic

### Scalp Trades (Inner-Range)

**Entry:**
- Range long/short based on wick rejection signals
- Entry price: close of rejection candle

**Exit:**
- **Target:** boxMax - (boxHeight × 0.10) for longs | boxMin + (boxHeight × 0.10) for shorts
  - Tight target based on range extremes
- **Stop Loss:** close - (ATR × 1.8) for longs | close + (ATR × 1.8) for shorts
  - Uses 1-minute ATR for precise risk calibration
- **Circuit Breaker:** Auto-closes if price reaches emergencyExitPrice
  - Long: boxMin - (boxHeight × 1.2)
  - Short: boxMax + (boxHeight × 1.2)
  - Catches catastrophic moves/gaps below/above range

### Breakout Trades (Trend Continuation)

**Entry:**
- On crossover/crossunder of range extremes
- Entry price: close at breakout point

**Exit:**
- **Target:** None (intentional—lets winners run)
- **Stop Loss:** Trailing stop using ATR
  - Long: boxMax - (ATR × 3.0), continuously adjusted upward with current ATR
  - Short: boxMin + (ATR × 3.0), continuously adjusted downward with current ATR
  - Protects trend runners while allowing unrestricted upside/downside

---

## Risk Management

### Position Sizing
- **Fixed:** 20% of equity per trade
- **Rational:** Consistent risk across all sessions regardless of volatility
- All calculations use available equity (scales with account growth)

### Stop Loss Multipliers
- **Scalp SL (1.8x ATR):** Tight stop for range-bound trades
- **Breakout SL (3.0x ATR):** Wider stop for trend followers

### Circuit Breaker (1.2x)
- Secondary failsafe triggered only on extreme moves
- Empirically tested: 1.2x provided best ROI vs. 1.3-1.5x range
- Prevents gap slippage from destroying the trade

### ATR Calculation
- Length: 14 periods
- Uses 1-minute ATR (commented alternative: can switch to 15m)
- Frozen at entry for scalps; dynamic for breakouts

---

## Configuration Guide

### Session Time Setup
```
Range Discovery Window: 0000-0700 UTC    [adjustable]
Active Execution Window: 0700-1700 UTC   [adjustable]
```
Set these to your preferred market hours. Silver trades 24/5, so adjust for your timezone/preferred hours.

### Strategy Engines
```
Enable Inner-Range Scalping:              [true/false]
Enable Trend Continuation Breakouts:      [true/false]
```
Toggle individual engines on/off for testing or market regime changes.

### Box Quality Filter
```
Enable Box Quality Filter:                [true/false]
Historical Box Baseline (Days):           5 (default, 2-20 range)
Maximum Box Height Multiplier:            1.5 (skip giant ranges)
Minimum Box Height Multiplier:            0.5 (skip tiny ranges)
```

Tighter multipliers (e.g., 1.3/0.7) = fewer trades, higher selectivity  
Looser multipliers (e.g., 2.0/0.3) = more trades, more noise

### Volume Confirmation Engine
```
Volume MA Length:                         20 bars
Volume Spike Threshold:                   1.2x (requires 20% above average)
```

### Advanced Risk Controls
```
ATR Length:                               14 periods
Scalp Stop Multiplier:                    1.8x ATR
Breakout Trailing Multiplier:             3.0x ATR
Circuit Breaker:                          1.2x boxHeight
Scalp Target Margin Padding:              0.10 (10% of boxHeight)
```

### Visual Customization
```
Draw Clean Session Boundaries:            [true/false]
Shows boxMax (red ceiling) and boxMin (green floor) after range closes
```

---

## Visual Indicators on Chart

### Range Boundaries (After Session Close)
- **Red Line (Ceiling):** boxMax — upper range extreme
- **Green Line (Floor):** boxMin — lower range extreme

### Entry Signals
- **Green Label "SCALP LONG":** Range floor rejection, bullish setup
- **Red Label "SCALP SHORT":** Range ceiling rejection, bearish setup
- **Cyan Label "BUY BREAKOUT":** Breakout above ceiling
- **Purple Label "SELL BREAKOUT":** Breakout below floor

### Active Trade Lines
- **Red Line (Stop Loss):** Current stop loss price (updates dynamically for breakouts)
- **Green Line (Take Profit):** Target for scalp trades only (absent for breakouts)

### Status Dashboard (Top Right)
```
Today's Box:        [height value]
Strategy Status:    "Engines Hunting" | "Skipped: Box Too Giant" | "Skipped: Box Too Compressed"
```

---

## Trading Rules Summary

1. **Only enter during Active Execution Window** (0700-1700 UTC)
2. **Box must pass quality filter** (within 50%-150% of historical baseline)
3. **Volume must confirm** (>1.2x 20-bar average)
4. **Max 1 scalp entry per direction per range** (longCount, shortCount tracking)
5. **Multiple breakout entries allowed** (independent of scalp limit)
6. **Scalps use tight targets;** breakouts use trailing stops
7. **Circuit breaker acts as failsafe** for extreme moves

---

## Important Considerations

### Market Hours
- Silver trades 24/5 with brief gaps Sunday → Monday and at session transitions
- These natural gaps do **not** corrupt the strategy (rolling average adapts)
- Box filter naturally skips outlier sessions after extended closures

### Timeframe Selection
- Tested on 1m, 5m, 15m
- Lower timeframes = more trades, tighter exits
- Higher timeframes = fewer trades, wider stops
- Choose based on your risk tolerance and attention span

### Instrument Specificity
- **Built for:** Silver (XAGUSD, SI futures, etc.)
- **Not recommended for:** Equities, forex without recalibration (session windows, ATR length, multipliers need adjustment)

### Backtesting Checklist
- [ ] Test on 1 year minimum of historical data
- [ ] Verify volume spike threshold (1.2x) filters noise effectively
- [ ] Check box quality multipliers (1.5/0.5) skip only true outliers
- [ ] Validate ATR length (14) matches your chosen timeframe
- [ ] Compare Scalp vs. Breakout performance separately (toggle engines)

---

## Operational Checklist

**Before Going Live:**
- [ ] Set correct session times for your timezone
- [ ] Backtest on your broker's historical data
- [ ] Start with 1-2 contracts/shares only (demo or micro)
- [ ] Monitor first 5 trades manually for unexpected behavior
- [ ] Verify volume data is accurate on your platform
- [ ] Confirm time(timeframe.period, session) matches your chart timezone

**Daily:**
- [ ] Check status dashboard at market open
- [ ] Note if range is skipped (Too Giant/Compressed)
- [ ] Monitor stop loss and target plots during active hours
- [ ] Document unusual market conditions (gaps, halts, news)

**Weekly:**
- [ ] Review P&L by trade type (Scalp vs. Breakout)
- [ ] Check if parameters need adjustment (volatility regime change)
- [ ] Backtest any parameter tweaks before applying

---

## Support & Modifications

**Safe to Tweak:**
- Session times (your trading hours)
- Volume spike threshold (1.2x → 1.5x for stricter filtering)
- Box quality multipliers (1.5/0.5 → 1.3/0.7 for more selective trading)
- ATR length (14 → 20 for slower adaptation)

**Do Not Change Without Full Retest:**
- Stop loss multipliers (1.8, 3.0) — deeply empirical
- Circuit breaker value (1.2 = best ROI)
- Target padding (0.10) — tied to scalp exit performance
- Position sizing (20%) — tied to risk management

---

## Disclaimer

This strategy is provided as-is for educational and testing purposes. Past performance does not guarantee future results. Always trade with proper risk management, position sizing appropriate to your account, and stop losses on every trade. Silver is a leveraged instrument; substantial losses are possible.

---

**Last Updated:** June 2026  
**Status:** Production Ready
