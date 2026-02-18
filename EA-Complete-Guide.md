# ADVANCED MULTI-SYSTEM FOREX EA - COMPLETE DOCUMENTATION

## TABLE OF CONTENTS
1. [Strategy Overview](#strategy-overview)
2. [Entry Systems](#entry-systems)
3. [Exit Systems](#exit-systems)
4. [System Architecture](#system-architecture)
5. [Configuration Guide](#configuration-guide)
6. [Risk Management](#risk-management)
7. [Performance Metrics](#performance-metrics)

---

## STRATEGY OVERVIEW

### Core Concept
This EA implements a sophisticated multi-entry, multi-exit trading system designed for XAUUSD (Gold) with the following characteristics:

- **Signal Detection**: Time Bomb (momentum spike) + Impulse (trend confirmation)
- **Dual Entry Modes**: Single order OR Pullback reversal entry
- **Dual Grid Modes**: Grid-based accumulation (original trend + reversal trend)
- **Triple Exit Modes**: Direct hits, Partial closes, or Average profit closes
- **Dynamic Risk Management**: Virtual stops, partial position management, breakeven automation

### Market Conditions
- **Optimal**: Ranging to trending markets (4H+ timeframes)
- **Currency Pair**: XAUUSD recommended (can adapt to other forex pairs)
- **Leverage**: 1:100 minimum, 1:500 recommended for grid trading
- **Spread Tolerance**: Up to 0.3 pips for XAUUSD

---

## ENTRY SYSTEMS

### SYSTEM 1: SINGLE vs REVERSAL ENTRIES

#### Entry Type 1A: Single Order Entry (Original Signal)
```
TRIGGER: Time Bomb Up + Impulse Up → BUY ORDER
         Time Bomb Down + Impulse Down → SELL ORDER

MECHANICS:
- Time Bomb: Price moves X pips in Y seconds
- Impulse: Quiet bars (low volatility) followed by strong candle close
- Opens 1 market order at current market price
- Full TP/SL applied immediately

BEST FOR: Quick scalps, trending moves, low drawdown trading
```

#### Entry Type 1B: Pullback/Reversal Entry
```
TRIGGER: Original Signal TP Hit + Reversal Detection
         (Price breaks TP, reverses back for pullback entry)

MECHANICS:
- After original order TP is hit, system monitors for pullback
- Pullback = New impulse in opposite direction with fading momentum
- Enters on pullback signal opposite to original entry
- Captures brief reversals before trend resumes

BEST FOR: Mean reversion, range-bound periods
RISK: Higher false signals during strong trending
```

### SYSTEM 2: GRID ENTRIES (ACCUMULATION STRATEGY)

#### Grid Type 2A: Original Signal Grid
```
MECHANICS:
- Opens first order at original signal price
- If TP not hit, opens additional orders at GRID_STEP intervals
- Each order gets same TP/SL (proportional to grid level)
- Maximum GRID_ORDERS_MAX = default 5 orders

GRID SPACING: 
- 1st Order: Entry signal
- 2nd Order: Entry + GRID_STEP pips
- 3rd Order: Entry + (GRID_STEP × 2) pips
- etc.

EXAMPLE (BUY with 20 pip grid):
Price 2000.00 - Order 1 (0.1 lot)
Price 2000.20 - Order 2 (0.1 lot)
Price 2000.40 - Order 3 (0.1 lot)
```

#### Grid Type 2B: Reversal Signal Grid
```
MECHANICS:
- After original grid reaches TP, system hunts pullback
- When pullback impulse detected (opposite direction)
- Opens grid orders in REVERSE direction
- Second grid accumulates counter-trend position

DUAL GRID SCENARIO:
Entry 1 (BUY grid): 5 orders averaging 2000.00
  ↓ (all close at TP or exit system)
Entry 2 (SELL grid): 5 orders averaging 2000.50 (pullback zone)
  ↓ Both grids close at respective TPs
```

---

## EXIT SYSTEMS

### SYSTEM 1: DIRECT TP/SL EXIT
```
MECHANICS:
- Each order has individual TP and SL
- Order closes when price hits TP or SL independently
- Simplest, most predictable exit
- No position management required

WHEN TO USE: Single entry mode, risk-averse trading
PROS: Clear risk/reward, easy to backtest
CONS: No opportunity for optimization, misses partial exit opportunities
```

### SYSTEM 2: PARTIAL CLOSE + BREAKEVEN SYSTEM

#### Subtype 2.1: Profit-Based Partial Close (Profit Only)
```
MECHANICS:
- Profit Threshold: When profitable order reaches 50% of TP
- Action: Close 50% of the order lot size
- Remaining 50%: Stays open for full TP hit
- Breakeven Option: Move SL to entry price (0.01 pips to ensure close)

EXAMPLE (500 pips TP):
- Entry: 2000.00, 0.1 lot, TP 2005.00 (500 pips), SL 1995.00 (500 pips)
- Trigger: Profit reaches 250 pips (50% of TP)
- Action 1: Close 0.05 lot at profit (lock 50% gains)
- Action 2: If "Move SL to Breakeven" enabled → SL moves to 2000.01
- Remaining: 0.05 lot runs to 2005.00

BEST FOR: Locking quick profits, reducing drawdown
ACTIVATION: Automatically when order profit > (TP_PIPS × 50%)
```

#### Subtype 2.2: Profit & Loss Partial Close (Bidirectional)
```
MECHANICS:
- Two conditions checked simultaneously:
  A) PROFIT TRIGGER: Order profit > 50% of TP → Close 50% at profit
  B) LOSS TRIGGER: Order loss > 50% of SL → Close 50% at loss
- Breakeven Move: On profit side only (after 50% close)

EXAMPLE - PROFIT SIDE:
- Profit > 250 pips → Close 0.05 lot with profit
- Move SL to breakeven for remaining 0.05 lot

EXAMPLE - LOSS SIDE:
- Loss > 250 pips → Close 0.05 lot with loss
- Keep remaining 0.05 lot running full SL

BEST FOR: Aggressive grid trading, limiting drawdown on losers
USE CASE: Grid accumulation with large 500+ pip SL
```

### SYSTEM 3: AVERAGE PROFIT CLOSING (Portfolio-Level Exit)

#### Subtype 3.1: Individual SL + Sum-Based TP
```
MECHANICS:
- Each order maintains individual SL (can't be hit)
- Portfolio Exit: Sum of ALL open orders' P&L reaches target profit
- When Target Profit Reached: Close ALL orders (both running + pending)
- Partial Close (50%): When portfolio reaches 50% of target

EXAMPLE (Scenario):
- 3 Orders open: 
  Order 1: +150 pips profit
  Order 2: -50 pips loss
  Order 3: +100 pips profit
- Portfolio Total: +200 pips (150-50+100)
- Target Profit: 400 pips
- At +400 pips: ALL orders close immediately

PARTIAL EXIT:
- When portfolio reaches +200 pips (50% of 400):
  Action: Close 50% lot from ALL possible orders
  (Remaining 50% runs until full TP or SL)

BREAKEVEN ACTIVATION:
- After 50% partial close, move SL to breakeven on closed positions
- Only affects already-closed portions (no effect)
- Affects remaining open portions

BEST FOR: Large portfolio management, hedge trading
RISK MANAGEMENT: Excellent for limiting max drawdown
```

#### Subtype 3.2: Average SL + Sum-Based TP (Portfolio-Level)
```
MECHANICS:
- All orders' SLs are calculated AVERAGES (not individual)
- Example: Orders at 2000, 2001, 2002 with 500 pip SL each
  Average SL = (1500 + 1501 + 1502) / 3 = 1501 (approx)
- Portfolio Loss Trigger: When sum of losses > 50% portfolio SL
- Portfolio Profit Trigger: When sum of profits > target

PORTFOLIO LOSS SCENARIO:
- 3 Orders losing on average 250 pips each
- Total loss: 750 pips (exceeds 50% of avg 500 pip SL)
- Action: Close 50% of all orders at loss
- Remaining: Continue to individual SLs or portfolio TP

BEST FOR: Maximum capital optimization, large grid positions
COMPLEX: Requires careful monitoring during implementation
```

---

## SYSTEM ARCHITECTURE

### Core Components

```
┌─────────────────────────────────────────────────────────┐
│           OnTick() Main Event Loop                      │
└────────┬────────────────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────────────────────┐
    │  1. Signal Detection (Time Bomb + Impulse)          │
    ├─────────────────────────────────────────────────────┤
    │  - TimeBombUp: Momentum up detection                │
    │  - ImpulseUp: Trend confirmation                    │
    │  - TimeBombDown: Momentum down detection            │
    │  - ImpulseDown: Trend confirmation                  │
    └────┬────────────────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────────────────────┐
    │  2. Entry System Selection                          │
    ├─────────────────────────────────────────────────────┤
    │  ┌──SYSTEM_1 (Single/Reversal)──┐                  │
    │  │ Single: Open 1 order          │                  │
    │  │ Reversal: Hunt pullback       │                  │
    │  └───────────────────────────────┘                  │
    │  ┌──SYSTEM_2 (Grid)──────────────┐                  │
    │  │ Grid Type A: Original          │                  │
    │  │ Grid Type B: Reversal          │                  │
    │  └───────────────────────────────┘                  │
    └────┬────────────────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────────────────────┐
    │  3. Position Management                             │
    ├─────────────────────────────────────────────────────┤
    │  - Order creation with TP/SL                        │
    │  - Position tracking                                │
    │  - Lot size calculation                             │
    └────┬────────────────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────────────────────┐
    │  4. Exit System Selection                           │
    ├─────────────────────────────────────────────────────┤
    │  ┌──SYSTEM_1 (Direct)────────┐                      │
    │  │ Hit TP → Close             │                      │
    │  │ Hit SL → Close             │                      │
    │  └────────────────────────────┘                      │
    │  ┌──SYSTEM_2 (Partial)───────┐                      │
    │  │ 50% close at profit        │                      │
    │  │ Move SL to breakeven       │                      │
    │  └────────────────────────────┘                      │
    │  ┌──SYSTEM_3 (Portfolio)─────┐                      │
    │  │ Sum profit trigger         │                      │
    │  │ Close all on target        │                      │
    │  └────────────────────────────┘                      │
    └────┬────────────────────────────────────────────────┘
         │
    ┌────▼─────────────────────────────────────────────────┐
    │  5. Order Management (Close, Modify)                │
    └─────────────────────────────────────────────────────┘
```

### Order Grouping by Magic Number
```
Magic Number Structure:
BASE_MAGIC = 10000 (configurable)

Magic_Single_Buy     = 10001
Magic_Single_Sell    = 10002
Magic_Reversal_Buy   = 10003
Magic_Reversal_Sell  = 10004
Magic_Grid_Buy       = 10005
Magic_Grid_Sell      = 10006
Magic_Grid_Rev_Buy   = 10007
Magic_Grid_Rev_Sell  = 10008

Benefit: Easy identification, quick portfolio filtering
```

---

## CONFIGURATION GUIDE

### Input Parameters (Organized by System)

#### SIGNAL PARAMETERS
```
//--- Time Bomb Detection
INPUT_TIMEBOMB_UP_PIPS = 200.0          // Pips to rise in time window
INPUT_TIMEBOMB_UP_TIME = 3.0            // Seconds window
INPUT_TIMEBOMB_DOWN_PIPS = 200.0        // Pips to fall in time window
INPUT_TIMEBOMB_DOWN_TIME = 3.0          // Seconds window

//--- Impulse Detection
INPUT_IMPULSE_QUIET_BARS = 3            // Number of quiet bars
INPUT_IMPULSE_QUIET_MIN_RANGE = 5.0     // Minimum range for "quiet" bars
INPUT_IMPULSE_STRENGTH = 3.0            // Impulse strength multiplier
```

#### ENTRY SYSTEM PARAMETERS
```
//--- System Selection (only ONE can be true at a time)
ENABLE_SYSTEM_1_SINGLE_ENTRY = true     // Single entry mode
ENABLE_SYSTEM_1_REVERSAL_ENTRY = false  // Reversal hunting mode
ENABLE_SYSTEM_2_GRID_ORIGINAL = false   // Grid for original signal
ENABLE_SYSTEM_2_GRID_REVERSAL = false   // Grid for reversal signal

//--- Grid Parameters (if System 2 enabled)
GRID_STEP_PIPS = 20.0                   // Distance between grid orders
GRID_ORDERS_MAX = 5                     // Maximum grid orders
GRID_ACTIVATION_DELAY = 60              // Seconds before opening grid orders
```

#### LOT SIZE PARAMETERS
```
//--- Volume Management
INPUT_LOT_SIZE = 0.1                    // Base lot size (fixed mode)
INPUT_RISK_PERCENT = 2.0                // Risk % per trade
INPUT_LOT_MODE = "fixed"                // Options: "fixed", "risk", "equity"
INPUT_MAX_LOT_SIZE = 5.0                // Maximum lot size limit
```

#### STOP/PROFIT PARAMETERS
```
//--- Risk Management
INPUT_TP_PIPS = 600.0                   // Take Profit in pips
INPUT_SL_PIPS = 600.0                   // Stop Loss in pips
INPUT_TRAILING_STOP = false             // Enable trailing stops
INPUT_TRAILING_STOP_PIPS = 100.0        // Trailing stop distance

//--- Breakeven Automation
INPUT_MOVE_SL_TO_BREAKEVEN = true       // Auto move SL to entry after partial
INPUT_BREAKEVEN_OFFSET = 0.01           // Tiny offset to ensure close
```

#### EXIT SYSTEM PARAMETERS
```
//--- Exit System Selection
EXIT_SYSTEM_MODE = 1                    // 1=Direct, 2=Partial, 3=Portfolio

//--- If EXIT_SYSTEM = 2 (Partial Close)
EXIT_PARTIAL_PROFIT_ONLY = true         // true=profit only, false=both
EXIT_PARTIAL_THRESHOLD_PERCENT = 50.0   // % of TP to trigger close
EXIT_PARTIAL_CLOSE_PERCENT = 50.0       // % of lot to close
EXIT_MOVE_SL_BREAKEVEN = true           // Move SL after partial close

//--- If EXIT_SYSTEM = 3 (Portfolio Close)
EXIT_PORTFOLIO_TARGET_PIPS = 400.0      // Sum profit target
EXIT_PORTFOLIO_SL_TYPE = "individual"   // "individual" or "average"
EXIT_PORTFOLIO_PARTIAL_CLOSE = true     // Close 50% at 50% profit
EXIT_PORTFOLIO_BREAKEVEN = true         // Move SL to breakeven after partial
```

---

## RISK MANAGEMENT

### Position Sizing Calculation
```
FIXED MODE:
Lot Size = INPUT_LOT_SIZE (e.g., 0.1 lot)

RISK MODE:
Risk Amount = Account_Balance × INPUT_RISK_PERCENT / 100
Pips at Risk = INPUT_SL_PIPS
Lot Size = Risk_Amount / (Pips_at_Risk × Point_Value)

EQUITY MODE:
Free Equity = Account_Equity × (Max_Risk_Percent)
Margin Required per Lot = SymbolInfoDouble(...SYMBOL_MARGIN_INITIAL)
Lot Size = Free_Equity / Margin_Required
```

### Drawdown Limits
```
MAX_DAILY_LOSS = 2.0%           // Stop trading after 2% daily loss
MAX_WEEKLY_LOSS = 5.0%          // Stop trading after 5% weekly loss
MAX_MONTHLY_LOSS = 10.0%        // Stop trading after 10% monthly loss
```

### Trade Limits
```
MAX_CONCURRENT_ORDERS = 10      // Max open orders at once
MAX_ORDERS_PER_DAY = 20         // Max orders per 24 hours
```

---

## PERFORMANCE METRICS

### Key Performance Indicators
```
Win Rate: Profitable trades / Total trades (Target: 45%+)
Profit Factor: Gross profit / Gross loss (Target: 1.5+)
Recovery Factor: Net profit / Max drawdown (Target: 2.0+)
Sharpe Ratio: Average return / Std deviation (Target: 1.0+)
Max Consecutive Losses: Maximum losing streak (Limit: 5-7)
```

### Optimization Targets
```
Objective: Maximize Sharpe Ratio while maintaining:
- Win rate > 40%
- Profit factor > 1.2
- Max drawdown < 20%

Optimize Parameters:
1. TP_PIPS (400-800 range)
2. SL_PIPS (300-800 range)
3. GRID_STEP_PIPS (10-50 range)
4. Entry confirmation bars (2-5 range)
```

---

## TROUBLESHOOTING

### Common Issues

**Issue**: Orders not opening
- Check: Magic numbers, order limits
- Verify: Account balance > margin requirement
- Review: Broker trade hours restrictions

**Issue**: High drawdown
- Increase: SL_PIPS or reduce LOT_SIZE
- Enable: Partial close system
- Reduce: GRID_STEP_PIPS

**Issue**: Too many false signals
- Increase: IMPULSE_STRENGTH value
- Increase: QUIET_BARS count
- Adjust: Signal confirmation timeframe

**Issue**: Orders not closing
- Check: TP/SL prices are valid (respects spread)
- Verify: Breakeven logic not conflicting
- Review: Partial close percentages add up to 100%

---

## ADVANCED FEATURES

### Grid Recovery Strategy
- Grid opens 5 orders on strong signal
- If initial orders lose, grid DOESN'T increase (anti-martingale)
- Instead, reversal signal opens counter-grid
- Counter-grid captures pullback, offsets losses

### Breakeven Automation
- After 50% close at profit, SL moves to entry (or +0.01 pips)
- Ensures remaining position is "risk-free"
- Remaining position can still hit TP for full profit

### Portfolio Synchronization
- All orders tracked by entry time and signal type
- Portfolio close triggers simultaneous closure
- Avoids orphaned orders

---

## BEST PRACTICES

1. **Start Small**: Begin with 0.01 lot sizes in live trading
2. **Test Thoroughly**: Backtest 2+ years of data
3. **Monitor**: Use daily logs to track performance
4. **Adjust**: Change ONE parameter at a time
5. **Document**: Keep trade journal for analysis
6. **Risk First**: Never risk > 2% per trade

---

## VERSION HISTORY

**v2.0** (Current)
- Multi-system entry/exit implementation
- Portfolio-level profit management
- Partial close with breakeven
- Advanced grid strategy

**v1.0** (Original)
- Basic Time Bomb + Impulse signals
- Single order entry
- Direct TP/SL exit

---

**Last Updated**: February 2026  
**Tested Broker**: MetaTrader 5 (Standard)  
**Recommended Timeframe**: H4 (4-hour)  
**Asset**: XAUUSD (Gold)
