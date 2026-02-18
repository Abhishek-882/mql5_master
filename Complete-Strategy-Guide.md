# MASTER TRADER EA v2.0 - COMPLETE STRATEGY GUIDE

## TABLE OF CONTENTS
1. System Architecture Overview
2. Entry Systems Explained
3. Grid Trading Mechanics
4. Exit Systems Detailed
5. Risk Management Framework
6. Strategy Performance Analysis
7. Optimization Philosophy
8. Live Trading Protocol

---

## 1. SYSTEM ARCHITECTURE OVERVIEW

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   MASTER TRADER EA v2.0                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ SIGNAL       │  │ ENTRY        │  │ EXIT         │  │
│  │ DETECTION    │→ │ SYSTEM       │→ │ SYSTEM       │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│        ↑                   ↑                   ↑        │
│   Time Bomb       Single/Grid/        Direct/Partial   │
│   +               Reversal            Portfolio        │
│   Impulse                                              │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │     POSITION MANAGEMENT & ORDER TRACKING        │  │
│  │  (Magic Numbers, Order Arrays, PnL Calc)       │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↑                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │   RISK MANAGEMENT & ACCOUNT PROTECTION         │  │
│  │  (Daily Loss, Weekly Loss, Max Lot Limits)    │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Core Concepts

**1. Signal Detection (Time Bomb + Impulse)**
```
Purpose: Identify high-probability trade opportunities
Mechanism: Two confirmations required:
  - Time Bomb: Price moves X pips in Y seconds
  - Impulse: Quiet bars broken by strong move
Result: Reduces false signals by requiring dual confirmation
```

**2. Entry System (Three Options)**
```
A. Single Entry: One order per signal
B. Reversal Entry: Hunt pullbacks after TP hit
C. Grid Entry: Accumulate positions via multiple orders
```

**3. Exit System (Three Options)**
```
A. Direct Exit: TP hit = close, SL hit = close
B. Partial Exit: Close 50% at 50% profit, rest at TP
C. Portfolio Exit: Close all when portfolio target hit
```

**4. Order Management**
```
Purpose: Track all orders with advanced features
Tracking: Magic numbers, entry price, TP, SL, lot size
Features: Partial close management, breakeven moves, PnL calc
```

---

## 2. ENTRY SYSTEMS EXPLAINED

### System 1: Single Entry (ENTRY_SINGLE_ONLY)

#### Overview
```
Most conservative entry method. One order per signal.
No averaging, no grid accumulation. Simple and clean.
```

#### Logic Flow
```
Signal Detected (BUY)
        ↓
  Current Positions?
     NO → Open 1 BUY Order
     YES → Wait for close
        ↓
   Order Properties:
   - Entry: Current Ask Price
   - TP: Entry + (TP_PIPS × PointValue)
   - SL: Entry - (SL_PIPS × PointValue)
   - Lot Size: Based on risk management rules
        ↓
   Order Opened, Exit System Applies
```

#### Pros & Cons
```
✓ Pros:
  - Simple, easy to understand
  - Lowest capital requirement
  - Predictable P&L per trade
  - Suitable for learning
  - Lowest drawdown

✗ Cons:
  - Limited profit accumulation
  - Miss averaging opportunities
  - One trade only per signal
```

#### When to Use
```
✓ Market strongly trending
✓ Want guaranteed simple execution
✓ Limited trading capital ($500-5000)
✓ Risk-averse trading style
✓ Learning phase
```

#### Example Trade
```
Signal: BUY at 2000.00
Entry Parameters:
  - TP Pips: 600
  - SL Pips: 600
  - Lot: 0.1

Trade Execution:
  Entry: 2000.00
  TP:    2006.00 (2000.00 + 600 pips)
  SL:    1994.00 (2000.00 - 600 pips)
  Lot:   0.1

Outcome:
  ✓ Profit Case: Price hits 2006.00 → +600 pips
  ✗ Loss Case: Price hits 1994.00 → -600 pips
```

---

### System 2: Reversal Entry (ENTRY_SINGLE_AND_REVERSAL)

#### Overview
```
Includes single entry PLUS reversal (pullback) hunting.
After first TP hit, hunts pullback entries before next move.
More entries, higher average profit, higher complexity.
```

#### Logic Flow
```
Phase 1: Original Signal
  BUY Signal Detected
        ↓
    Open BUY Order at 2000.00
    TP: 2006.00, SL: 1994.00
    
Phase 2: After Original TP Hit
  Price reaches 2006.00 (TP hit)
        ↓
    Pullback Detection Active
    Monitor for reversal signal
    
Phase 3: Pullback Signal
  Price pulls back (drops below recent high)
  Impulse + Time Bomb detected going DOWN
        ↓
    Open SELL Order at 2005.50
    (reversal pullback trade)
    
Phase 4: Both Orders Active
  Original BUY: +600 pips if trending
  Reversal SELL: + profit if pullback
  
Result: BOTH can profit from same trend
```

#### Reversal Strategy Detailed

**What is a Pullback?**
```
After BUY signal hits TP at $2006:
  - Price continues higher to $2008
  - Momentum fades
  - Price pulls back to $2005
  - Small Impulse Down detected
  
Action: Open SELL order for pullback trade
Expected Move: Back to $2003 (pullback target)
Result: Profit while original continues higher
```

**Why This Works**
```
1. Original Trend + TP Hit = Strong momentum
2. Pullback is natural (profit taking)
3. After pullback, original direction resumes
4. Catches the "shake out" move

Technical Basis:
- Traders take profits at TP
- Price pulls back
- New buyers enter
- Price resumes trend

Profit Opportunity:
- Short pullback swing = small profit
- Original trend continues = larger profit
- Same trend profits TWICE
```

#### Pros & Cons
```
✓ Pros:
  - 2-3x more trades per trend
  - Better trend utilization
  - Higher average account growth
  - Captures multiple swings

✗ Cons:
  - More complex management
  - Correlation risk (trend + pullback trades)
  - Higher complexity = more errors
  - Need larger capital
```

#### Example: Full Reversal Trade

```
TIMELINE: 2 hours during trend

08:00 - ORIGINAL SIGNAL
  Price: 2000.00
  Signal: BUY (Time Bomb + Impulse Up)
  Order: Buy 0.1 lot @ 2000.00
  TP: 2006.00, SL: 1994.00
  Status: OPEN

09:30 - TRENDING UP
  Price: 2003.50
  P&L: +350 pips
  Status: Order still running

10:00 - TP HIT
  Price: 2006.00
  P&L: +600 pips
  Status: Original BUY CLOSED at TP
  
  Action: Reversal detection ACTIVE
  System monitoring: Pullback beginning?

10:15 - PULLBACK STARTING
  Price: 2005.50
  Observation: Price pulling back from TP
  Status: Waiting for Impulse Down signal

10:30 - PULLBACK SIGNAL
  Price: 2005.00
  Signal: SELL (Quiet bars → Strong down close)
  Order: Sell 0.1 lot @ 2005.00
  TP: 1999.00 (pullback target)
  SL: 2011.00 (above pullback high)
  Status: REVERSAL ORDER OPENED

10:45 - REVERSAL PROFIT
  Price: 2001.50
  P&L on SELL: +350 pips
  Status: SELL order running

11:00 - REVERSAL TP HIT
  Price: 1999.00
  P&L: +600 pips
  Status: SELL CLOSED at TP

RESULT (2 hours):
  Original BUY:  +600 pips
  Reversal SELL: +600 pips
  Total:        +1200 pips on same trend!
  Compare: Single entry = +600 pips
  Reversal entry = +1200 pips (2X)
```

#### When to Use Reversal Entry
```
✓ Use When:
  - Trading strong trends
  - Willing to monitor trades
  - Have extra capital
  - Comfortable with pullback logic
  - Want higher account growth

✗ Avoid When:
  - Choppy/ranging market
  - Low volatility period
  - Limited capital
  - Prefer hands-off trading
  - During high-impact news
```

---

### System 3: Grid Entry (Integrated with Entry Systems)

*Covered in Section 3 below*

---

## 3. GRID TRADING MECHANICS

### What is Grid Trading?

```
Grid Trading = Multiple orders spaced at regular intervals
              to accumulate positions and average entry price

Concept: Instead of 1 order, open 5 orders at different prices
Result:  Larger position with better average entry price
```

### Grid Entry Types

#### Type A: Original Signal Grid

```
Original BUY Signal at 2000.00

WITHOUT Grid:
  Order 1: Buy 0.1 lot @ 2000.00 ✓

WITH Grid (5 orders, 20 pip spacing):
  Order 1: Buy 0.1 lot @ 2000.00 ← Original
  Order 2: Buy 0.1 lot @ 1998.00 ← Grid level 1 (20 pips below)
  Order 3: Buy 0.1 lot @ 1996.00 ← Grid level 2 (40 pips below)
  Order 4: Buy 0.1 lot @ 1994.00 ← Grid level 3 (60 pips below)
  Order 5: Buy 0.1 lot @ 1992.00 ← Grid level 4 (80 pips below)
  
Total Lot: 0.5 lots
Average Entry: 1996.00 (better than all at 2000)
```

#### Type B: Reversal Signal Grid

```
Same concept but for reversal pullback entries
- Opens multiple grid orders below/above the reversal entry point
- Accumulates during pullback
- Better average price when pullback is deep
```

### Grid Trading Workflow

```
Step 1: Signal Generated
  Price: 2000.00
  Signal: BUY detected
  
Step 2: Delay Timer Starts
  Wait 60 seconds (grid activation delay)
  Purpose: Confirm signal is valid, not false signal
  
Step 3: Grid Orders Placed
  After 60 seconds: Place all 5 grid orders
  Order 1 at 2000.00 (at current market)
  Order 2-5 at intervals below (20 pips apart)
  
Step 4: Price Action Determines Execution
  Scenario A: Price continues UP
    - Only Order 1 fills (@ 2000.00)
    - Rest don't trigger (price going wrong direction)
    - Result: Single order, small position
    
  Scenario B: Price pulls back
    - Order 1 fills @ 2000.00
    - Order 2 fills @ 1998.00 (pullback catches this)
    - Order 3 fills @ 1996.00 (deeper pullback)
    - Result: 3 orders, 0.3 lots, better average
    
  Scenario C: Deep pullback then reversal
    - All 5 orders fill as price drops/rises
    - Result: 0.5 lot total position
    - Average entry: Better than first order alone

Step 5: All at Same TP/SL
  All 5 orders have SAME TP and SL
  TP: 2006.00 (600 pips above highest entry)
  SL: 1994.00 (600 pips below lowest entry)
  
  Result: Larger position, same risk/reward ratio
  but with better average entry price
```

### Grid Parameters Explained

#### Parameter: Grid Step (20 pips default)

```
Grid Step = Distance between grid orders

TIGHT (10 pips):
  Grid 1: 2000.00
  Grid 2: 1999.00 ← 10 pips below
  Grid 3: 1998.00 ← 10 pips below
  Grid 4: 1997.00
  Grid 5: 1996.00
  
  Advantage: Better average price, more accumulation
  Disadvantage: More capital tied up, more filled orders

MEDIUM (20 pips) - Recommended:
  Grid 1: 2000.00
  Grid 2: 1998.00 ← 20 pips below
  Grid 3: 1996.00 ← 20 pips below
  Grid 4: 1994.00
  Grid 5: 1992.00
  
  Balance: Good spacing, reasonable capital

LOOSE (30 pips):
  Grid 1: 2000.00
  Grid 2: 1997.00 ← 30 pips below
  Grid 3: 1994.00 ← 30 pips below
  Grid 4: 1991.00
  Grid 5: 1988.00
  
  Advantage: Less capital, wider spacing
  Disadvantage: Fewer orders fill, worse averaging
```

#### Parameter: Grid Max Orders

```
Max Orders = How many grid levels to create

Setting to 3:
  Grid 1: 2000.00 (original)
  Grid 2: 1998.00
  Grid 3: 1996.00
  Total: 0.3 lots

Setting to 5 (recommended):
  Grid 1: 2000.00 (original)
  Grid 2: 1998.00
  Grid 3: 1996.00
  Grid 4: 1994.00
  Grid 5: 1992.00
  Total: 0.5 lots

Setting to 8 (aggressive):
  All of above PLUS:
  Grid 6: 1990.00
  Grid 7: 1988.00
  Grid 8: 1986.00
  Total: 0.8 lots (very large position)
```

### Grid Trading Exit

```
All grid orders have SAME TP/SL

Exit Scenario 1: Price goes UP
  - Order filled @ 2000.00
  - Price rises to 2006.00
  - ALL ORDERS (including unfilled) close at TP
  - Profit: +600 pips on 0.1 lots only

Exit Scenario 2: Price goes DOWN then UP
  - Order 1 @ 2000.00 filled
  - Order 2 @ 1998.00 filled (pullback)
  - Price falls to 1994.00 (Order 4 almost fills)
  - Price reverses, goes up
  - Reaches 2006.00 (TP level)
  - ALL orders close at TP
  - Profit: +600 pips × 0.2 lots = double profit!

Exit Scenario 3: SL Hit
  - All orders filled at various prices
  - Price falls to SL level
  - ALL orders close at SL
  - Loss: Minimal because grid avoided bad entry
```

### Capital Requirements for Grid

```
Without Grid:
  1 Order × 0.1 lot = Margin requirement X

With Grid (5 orders):
  5 Orders × 0.1 lot each = Margin requirement 5X
  Need 5X more capital!

Example (XAUUSD):
  1 lot margin requirement: $2500
  
  No Grid: 0.1 lot = $250 margin free
  Grid (5): 0.5 lots = $1250 margin needed
  
  Account $10,000:
    Available margin without grid: $9,750
    Available margin with grid: $8,750
    
Recommendation:
  - Only use grid with $10,000+ account
  - Or use smaller lot sizes per grid order
  - Never use grid with tight margin
```

### When Grid Works Best

```
✓ Best Use Cases:
  - Pullback markets (orders accumulate)
  - Choppy markets (catches multiple levels)
  - When you have capital ($10,000+)
  - Trending then reversing patterns
  - When you can monitor positions

✗ Avoid Grid When:
  - Sharp trending market (only 1 order fills)
  - Low capital account
  - High margin requirements
  - News-driven spikes (SL hits all orders)
  - Cannot monitor due to gaps
```

---

## 4. EXIT SYSTEMS DETAILED

### Exit System 1: Direct Exit (EXIT_DIRECT)

#### Concept
```
Simplest exit: TP hit = Close, SL hit = Close
No partial closes, no portfolio management.
Just pure TP/SL execution.
```

#### Workflow
```
Order Opened @ 2000.00
  TP: 2006.00
  SL: 1994.00
  
Monitor Price:
  
  IF Price ≥ 2006.00 (TP Level)
    → CLOSE at TP
    → Profit: +600 pips
    → Done
    
  IF Price ≤ 1994.00 (SL Level)
    → CLOSE at SL
    → Loss: -600 pips
    → Done
    
  OTHERWISE
    → Keep order open
    → Keep monitoring
```

#### Advantages
```
✓ Simple, automatic
✓ No complex logic
✓ Predictable outcomes
✓ No manual intervention needed
✓ Fastest execution
```

#### Disadvantages
```
✗ No profit protection
✗ No partial closes (miss early profits)
✗ All or nothing outcome
✗ Cannot adapt to market
```

#### Use Cases
```
✓ Best for:
  - Beginners learning
  - Simple, clean strategy
  - When you want fully automated
  - Scalping (small TP/SL)
  - Testing signal quality
```

---

### Exit System 2: Partial Close (EXIT_PARTIAL_CLOSE)

#### Concept
```
Close 50% of lot when reached 50% of TP profit
- Lock 50% profits early (low risk)
- Let remaining 50% run to full TP (high reward)
- Move SL to breakeven for safety
```

#### How It Works

```
INITIAL ORDER:
  Entry: 2000.00
  TP: 2006.00 (600 pips away)
  SL: 1994.00 (600 pips away)
  Lot: 0.1 lots
  
PARTIAL CLOSE TRIGGER:
  Threshold: 50% of TP = 300 pips
  Trigger Price: 2000.00 + 300 pips = 2003.00
  
WHEN PRICE HITS 2003.00:
  Profit at 2003.00: 300 pips (50% of 600)
  
  Action: Close 50% of lot
  - Close: 0.05 lots at 2003.00
  - Profit on 0.05 lots: +300 pips
  - Locked: $150 profit (assuming $0.50/pip)
  
  Remaining:
  - 0.05 lots still open
  - Entry: 2000.00
  - TP: 2006.00 (still running)
  - SL: Move to breakeven 2000.01
  
AFTER PARTIAL CLOSE:
  Closed: 0.05 @ 2003.00 = +$150
  Open: 0.05 @ 2000.00 = Risk-free (SL @ breakeven)
  
FINAL OUTCOME:
  Price reaches 2006.00 (TP)
  Close remaining 0.05 @ 2006.00 = +$150
  
  Total Profit: $150 + $150 = $300
  (Same as full TP, but with reduced risk)
```

#### Partial Close Parameters

**Partial Threshold (%)**
```
Controls: At what % of TP to close 50%

50% (Recommended):
  Close 50% when reached 50% of TP
  Example: TP=600, close at 300 pips

30%:
  Close 50% early (at 30% of TP)
  Example: TP=600, close at 180 pips
  Advantage: Lock profits sooner
  Disadvantage: Less profit locked

70%:
  Close 50% late (at 70% of TP)
  Example: TP=600, close at 420 pips
  Advantage: More profit in each partial
  Disadvantage: Risk of not hitting this level
```

**Move SL to Breakeven**
```
When Enabled (Recommended):
  After closing 50%, SL moves to 2000.01
  Result: Remaining 0.05 lot has ZERO risk
  Benefit: Can only win or break-even
  
When Disabled:
  After closing 50%, SL stays at 1994.00
  Result: Remaining 0.05 lot still has risk
  Benefit: Avoid whipsaws near breakeven
```

#### Example: Real Trade

```
GOLD XAUUSD Trade - Partial Close System

08:00 - ORDER OPENED
  Entry: 2000.50
  TP: 2006.50 (600 pips)
  SL: 1994.50 (600 pips)
  Lot: 0.2 lots
  Partial Close: 50% at 50% TP
  Move SL to BE: YES

08:30 - TRENDING UP
  Price: 2002.25
  P&L: +175 pips
  Status: Running normally

09:00 - HALF TP REACHED
  Price: 2003.50 (exactly 300 pips = 50% of 600)
  
  Action: PARTIAL CLOSE TRIGGERS
  Close: 0.1 lots @ 2003.50
  Profit locked: +300 pips = +$150
  
  Remaining: 0.1 lots
  New SL: 2000.51 (breakeven + 0.01)
  TP: Still 2006.50

10:00 - PULLBACK
  Price: 2002.00
  P&L on Remaining: -0.50 pips (near breakeven)
  Status: SL at breakeven protecting position

10:30 - RECOVERY
  Price: 2004.50
  P&L on Remaining: +400 pips
  Status: Making new profits after pullback

11:00 - FULL TP HIT
  Price: 2006.50
  Close Remaining: 0.1 lots @ 2006.50
  Profit on remaining: +600 pips = +$300
  
TOTAL RESULT:
  First 0.1 @ 2003.50: +$150
  Second 0.1 @ 2006.50: +$300
  Total: +$450 profit (1.5X vs single close)
  
BENEFIT: Captured full trend, locked early profits
```

#### Advantages of Partial Close
```
✓ Lock profits early (reduce risk)
✓ Breakeven protection (no loss possible)
✓ Let winners run (remaining part)
✓ Psychological comfort (seeing profits)
✓ Better risk/reward on remaining
✓ Average better results in choppy markets
```

#### Disadvantages
```
✗ Miss some profit if trade reverses fast
✗ More transactions (more slippage)
✗ Complex logic (potential bugs)
✗ SL moving may get stopped-out by noise
```

---

### Exit System 3: Portfolio Close (EXIT_PORTFOLIO_CLOSE)

#### Concept
```
NEW APPROACH: Close ALL orders together when sum hits target

Instead of: Close each order individually at TP/SL
Use: Close ENTIRE portfolio when total profit = target

Example:
  - 3 orders open simultaneously (grid system)
  - Each has individual SL
  - When TOTAL P&L reaches $500 → CLOSE ALL
  - Locks huge profits on portfolio
```

#### How Portfolio Close Works

```
SCENARIO: Grid trading 5 orders

Orders Opened:
  Order 1: Buy @ 2000.00
  Order 2: Buy @ 1998.00 (grid level 2)
  Order 3: Buy @ 1996.00 (grid level 3)
  Order 4: Buy @ 1994.00 (grid level 4)
  Order 5: Buy @ 1992.00 (grid level 5)
  
Individual SLs: Each has own SL
  Order 1 SL: 1994.00
  Order 2 SL: 1992.00
  Order 3 SL: 1990.00
  Order 4 SL: 1988.00
  Order 5 SL: 1986.00

Portfolio Target: 400 pips total profit

PRICE ACTION:

Scenario A: Price goes UP
  Order 1 @ 2000 → +300 pips
  Order 2 @ 1998 → +300 pips
  Order 3 @ 1996 → +300 pips
  Order 4 @ 1994 → +300 pips
  Order 5 @ 1992 → +300 pips
  
  Total: +1500 pips (way above 400 target!)
  
  Action: CLOSE ALL at this point
  Profit: 1500 pips × 5 orders = Huge!

Scenario B: Mixed prices (grid accumulation)
  Order 1 @ 2000 → +200 pips
  Order 2 @ 1998 → +350 pips
  Order 3 @ 1996 → +400 pips
  Order 4 @ 1994 → +300 pips
  Order 5 @ 1992 → (not filled)
  
  Total: +1250 pips
  
  Action: CLOSE ALL (target reached)
  Benefit: Average entry from grid gave huge profit!

Scenario C: Trend + pullback
  Order 1 @ 2000 → +100 pips
  Order 2 @ 1998 → +150 pips
  Order 3 @ 1996 → +250 pips
  Order 4 @ 1994 → +220 pips
  Order 5 @ 1992 → (not filled)
  
  Total: +720 pips (above 400 target)
  
  Action: CLOSE ALL (target reached)
  Benefit: Grid accumulation profitable!
```

#### Portfolio Target Setting

```
Parameter: InpPortfolioTargetPips

Conservative (200 pips):
  Close portfolio when sum = 200 pips
  Result: Frequent closes, smaller per trade
  Advantage: Less time in market, quicker returns

Moderate (400 pips) - Recommended:
  Close portfolio when sum = 400 pips
  Result: Good balance, captures trends

Aggressive (600+ pips):
  Close portfolio when sum = 600+ pips
  Result: Wait for large moves, bigger profits
  Advantage: Larger P&L per close
  Disadvantage: Longer hold times
```

#### Portfolio Close Methods

**Method 1: Individual SL (Default)**
```
Each order keeps its own SL
  Order 1 SL: 1994.00 (original)
  Order 2 SL: 1992.00 (original)
  Order 3 SL: 1990.00 (original)
  
Result:
  - Orders close individually if SL hit
  - Portfolio closes together if target hit
  - Mixed outcome possible
```

**Method 2: Average SL**
```
All orders move to AVERAGE SL
  Avg SL = (SL1 + SL2 + SL3 + SL4 + SL5) / 5
  Example: (1994 + 1992 + 1990 + 1988 + 1986) / 5 = 1990
  
Result:
  - All orders share same SL
  - Close together if SL hit
  - Cleaner risk management
  - All-or-nothing outcome
```

#### Partial Close at 50% in Portfolio

```
Before:
  Portfolio Target: 400 pips
  
With 50% Partial:
  At 200 pips (50% of 400):
    → Close 50% of all lots
    → Lock partial profits
    → Keep 50% running
    
After Partial:
  Remaining: 50% of lots
  New Target: Still need 400 pips total
  Result: Risk reduced, profits locked
```

#### When to Use Portfolio Close
```
✓ Best for:
  - Grid trading (multiple orders)
  - Portfolio management goals
  - When you want to close entire position together
  - Advanced traders
  - Larger accounts ($10,000+)

✗ Avoid when:
  - Single order trading (use direct/partial instead)
  - Low capital
  - Can't monitor positions
  - News-heavy trading
```

---

## 5. RISK MANAGEMENT FRAMEWORK

### Core Risk Management Rules

```
Rule 1: NEVER Risk > 2% Per Trade
Rule 2: NEVER Daily Loss > 2% of Account
Rule 3: NEVER Weekly Loss > 5% of Account
Rule 4: NEVER Max Lot > Account Risk Limit
Rule 5: NEVER Trade Without SL
```

### Lot Size Calculation

#### Method 1: Fixed Lot (Simplest)

```
InpLotMode = LOT_FIXED
InpLotSize = 0.1

Result: Always trade 0.1 lot
Advantage: Consistent, predictable
Disadvantage: Not scaled to account
```

#### Method 2: Risk-Based (Recommended)

```
InpLotMode = LOT_RISK_PERCENT
InpRiskPercent = 2.0

Calculation:
  Account Balance = $10,000
  Risk % = 2.0%
  Risk Amount = $10,000 × 2% = $200
  
  SL Pips = 600
  Per-Pip Value = SL × pointvalue × 100000
  Lot Size = Risk Amount / Per-Pip Value
  
Example:
  Risk Amount = $200
  SL = 600 pips
  Per pip value = 600 × 0.01 × 100000 = $60,000... 
  
Simplified (XAUUSD):
  Lot = (Account × Risk%) / (SL pips × 10)
  Lot = (10000 × 0.02) / (600 × 10)
  Lot = 200 / 6000 = 0.033 lots
```

#### Method 3: Equity-Based

```
InpLotMode = LOT_EQUITY_PERCENT
InpEquityPercent = 5.0

Calculation:
  Current Equity = $10,500
  Use 5% = $525
  Divide by margin per lot
  = Final lot size

Advantage: Scales with account growth
Disadvantage: More complex, changes daily
```

### Stop Loss Strategy

#### SL Sizing

```
TOO SMALL (100 pips):
  ✗ Gets stopped out by noise
  ✗ High false stop rate
  ✗ Hurts win rate

APPROPRIATE (300-800 pips):
  ✓ Captures real trends
  ✓ Reasonable drawdown
  ✓ Balanced risk/reward

TOO LARGE (1000+ pips):
  ✓ Rare stops
  ✗ Large losses when hit
  ✗ Ties up capital
```

#### SL Placement Rules

```
Rule: SL = TP (for balanced risk/reward)
Example: TP = 600 pips, SL = 600 pips

Rationale:
  - Win case: +600 pips
  - Loss case: -600 pips
  - 50% win rate = breakeven
  - 55%+ win rate = profitable

Alternative: TP > SL (wider risk/reward)
Example: TP = 800 pips, SL = 400 pips
  - Win: +800 pips
  - Loss: -400 pips
  - 1:2 risk/reward (1 risk, 2 reward)
  - Only need 33% win rate to be profitable
```

### Daily Loss Limits

```
Parameter: InpMaxDailyLossPercent

If set to 2%:
  Daily limit = 2% of account
  Example: $10,000 × 2% = $200 max loss per day
  
  When limit hit:
  - EA stops opening new orders
  - Existing orders still run
  - Protects against bad days

When to Trigger:
  ✓ After 3-4 consecutive losses
  ✓ When account drops 2%
  ✓ Bad market conditions day
  ✓ News days with high slippage
```

### Maximum Open Orders

```
Parameter: InpMaxOpenOrders

Default: 10

Meaning:
  - Maximum 10 orders can be open simultaneously
  - Prevents over-leveraging
  - Limits correlation risk

Setting Options:
  Conservative: 3 (max 3 concurrent)
  Moderate: 5-7 (balanced)
  Aggressive: 10+ (full grid capability)
```

### Calculating Maximum Lot Sizes

```
Maximum Lot = (Account / 100) × Risk%

Examples:

$1,000 account, 2% risk:
  Max lot = (1000 / 100) × 2% = 0.2 lots

$5,000 account, 2% risk:
  Max lot = (5000 / 100) × 2% = 1.0 lot

$10,000 account, 2% risk:
  Max lot = (10000 / 100) × 2% = 2.0 lots

$50,000 account, 2% risk:
  Max lot = (50000 / 100) × 2% = 10.0 lots

Parameter: InpMaxLotSize
  Use formula above as reference
  Never exceed recommended value
```

---

## 6. STRATEGY PERFORMANCE ANALYSIS

### Expected Performance Metrics

#### Win Rate Analysis

```
Industry Standard:
  Profitable systems: 40-60% win rate
  Excellent systems: 50-70% win rate
  Unrealistic: >80% (likely overfitted)

This EA Expected:
  Conservative: 42-48% win rate
  Balanced: 40-50% win rate
  Aggressive: 38-48% win rate
  
Interpretation:
  Lower win rate OK if profit factor high
  Example: 40% win rate + 2.0 PF = profitable
```

#### Profit Factor

```
Definition: Total Profit / Total Loss

Profitable Systems:
  1.2 - 1.5: Barely profitable
  1.5 - 2.0: Good system
  2.0 - 3.0: Excellent system
  > 3.0: Likely overfitted

This EA Target:
  Minimum: 1.3 (must beat this)
  Target: 1.6-2.0 (healthy range)
  Excellent: 2.0+ (exceptional)
```

#### Sharpe Ratio

```
Definition: Return Volatility / Risk Volatility
  High ratio = consistent returns
  Low ratio = erratic returns

Interpretation:
  < 0.5: Poor quality returns
  0.5 - 1.0: Below average
  1.0 - 2.0: Good quality
  > 2.0: Excellent consistency

This EA Target: 1.0+ (aim for this)
```

#### Maximum Drawdown

```
Definition: Largest drop from peak to trough

Safe Ranges:
  < 10%: Very conservative
  10-15%: Conservative
  15-20%: Moderate (RECOMMENDED)
  20-30%: Aggressive
  > 30%: Very risky (avoid)

This EA Targets:
  Conservative mode: 10-15%
  Balanced mode: 15-20%
  Aggressive mode: 20-30%
```

---

## 7. OPTIMIZATION PHILOSOPHY

### The Optimization Workflow

```
Phase 1: Baseline (1 week)
  ├─ Run EA with default settings
  ├─ Document results
  ├─ Identify weak areas
  └─ Target: Win rate 40%+, PF 1.2+

Phase 2: Signal Optimization (2 weeks)
  ├─ Optimize Time Bomb (200 → 150-250)
  ├─ Optimize Impulse Strength (3.0 → 2.5-3.5)
  ├─ Test combinations
  └─ Target: Win rate 45%+, PF 1.4+

Phase 3: Exit Optimization (1 week)
  ├─ Test TP/SL ranges (400-800)
  ├─ Test partial close thresholds
  ├─ Optimize breakeven logic
  └─ Target: PF 1.5+, DD < 20%

Phase 4: Advanced Features (1 week)
  ├─ Add grid if profitable
  ├─ Test portfolio close
  ├─ Optimize lot sizing
  └─ Target: Highest Sharpe ratio

Phase 5: Paper Trading (2-4 weeks)
  ├─ Test live (demo account)
  ├─ Verify backtest results match
  ├─ Adjust if needed
  └─ Target: Match backtest within 10%

Phase 6: Live Trading (Ongoing)
  ├─ Start with 0.01 lots
  ├─ Scale gradually
  ├─ Monitor daily
  └─ Target: 2% monthly growth
```

### Common Optimization Mistakes

```
❌ Optimizing on same data (overfitting)
   Solution: Use walk-forward testing

❌ Optimizing too many parameters
   Solution: Optimize 1-2 at a time

❌ Chasing highest profit
   Solution: Balance profit with drawdown

❌ Not testing recent data
   Solution: Always include last 3 months

❌ Ignoring market regime changes
   Solution: Test across different market types
```

---

## 8. LIVE TRADING PROTOCOL

### Pre-Launch Checklist

```
CONFIGURATION:
☐ All parameters set correctly
☐ Magic numbers assigned
☐ Lot sizes appropriate
☐ TP/SL reasonable
☐ Risk limits set

TRADING CONDITIONS:
☐ Market open (not weekend)
☐ Sufficient account balance
☐ Margin available
☐ Chart visible, EA applied
☐ AutoTrading enabled

MONITORING:
☐ Experts tab open
☐ Alert notification ON
☐ Journal tab ready
☐ Trading plan documented
☐ First signal identified on chart
```

### During Live Trading

```
EVERY 15 MINUTES:
  - Check current trades
  - Verify TP/SL prices
  - Confirm no errors

EVERY HOUR:
  - Review executed trades
  - Check partial closes
  - Monitor P&L trends

EVERY 4 HOURS (Daily):
  - Review all trades
  - Calculate daily P&L
  - Check daily loss limit
  - Log in trading journal
```

### Exit from Live Trade (if Intervention Needed)

```
STEP 1: Identify Problem
  - Check Experts tab for error
  - Review trade execution
  - Analyze price movement

STEP 2: Manual Close (if needed)
  - Right-click order in Trade tab
  - Select "Close" or "Close All"
  - Confirm execution
  - Document reason

STEP 3: Review and Adjust
  - Was it EA error or market?
  - Adjust parameters if needed
  - Resume trading

STEP 4: Log Everything
  - What went wrong?
  - When did it happen?
  - How was it fixed?
  - Will it happen again?
```

---

## FINAL RECOMMENDATIONS

### For Beginners
```
Start with Conservative Setup:
  ✓ ENTRY_SINGLE_ONLY (single orders)
  ✓ GRID_DISABLED (no grid)
  ✓ EXIT_PARTIAL_CLOSE (safety)
  ✓ TP/SL = 400 pips (balanced)
  ✓ Lot = 0.01 (minimal risk)
  
First Month Goals:
  - Learn signal behavior
  - Understand exits
  - Achieve 40%+ win rate
  - Keep daily DD < 5%
```

### For Intermediate Traders
```
Use Balanced Setup:
  ✓ ENTRY_SINGLE_AND_REVERSAL (more trades)
  ✓ GRID_ORIGINAL_SIGNAL (accumulation)
  ✓ EXIT_PARTIAL_CLOSE (profit locking)
  ✓ TP/SL = 600 pips (wider targets)
  ✓ Lot = 0.1-0.2 (medium risk)
  
Goals:
  - 45%+ win rate
  - 1.5+ profit factor
  - Growing account steadily
  - <15% monthly drawdown
```

### For Advanced Traders
```
Use Aggressive Setup:
  ✓ ENTRY_SINGLE_AND_REVERSAL
  ✓ GRID_BOTH (dual grid system)
  ✓ EXIT_PORTFOLIO_CLOSE (portfolio management)
  ✓ TP/SL = 800 pips
  ✓ Lot = 0.5-2.0 (higher risk)
  
Goals:
  - 40%+ win rate
  - 1.8+ profit factor
  - Maximized compounding
  - <20% monthly drawdown
```

---

**This EA is designed to be flexible, profitable, and manageable. Start conservative, optimize gradually, and scale your trading as profitability proves out.**
