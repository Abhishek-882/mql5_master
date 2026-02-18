# `mastercodemql5new.mq5` Comprehensive Problem & Fix Plan

## Scope reviewed
- `mastercodemql5new.mq5`
- `EA-Complete-Guide.md`
- `Complete-Strategy-Guide.md`
- `Problems-Analysis.md`
- `Master-Trader.txt`

---

## Executive summary
The EA is substantially more complete than the legacy fxDreema export in `Master-Trader.txt`, but there are still **high-impact correctness and strategy-conformance gaps**. Most issues are in:
1. pending/grid order validation,
2. partial-close execution in netting accounts,
3. state-machine + signal throttling behavior,
4. portfolio SL/average-SL semantics,
5. mismatch between documented strategy requirements and implementation details.

---

## Detailed problems and required fixes

## 1) Pending order success check is too strict
**Problem**
`PlacePendingOrder()` treats success only as `TRADE_RETCODE_DONE`. For pending placement, brokers often return `TRADE_RETCODE_PLACED`, so valid orders can be treated/logged as failures.  
Code: `mastercodemql5new.mq5` lines 774-785.

**Why it matters**
Grid/reversal pending logic may silently degrade because successful placements are considered failed.

**Fix needed**
Accept both `TRADE_RETCODE_DONE` and `TRADE_RETCODE_PLACED` as success for pending orders.

---

## 2) No pending-distance validation vs stops/freeze levels
**Problem**
Pending orders are opened without validating that pending price, SL, and TP are at broker-allowed minimum distances.  
Code: lines 753-786 and grid callers at 979-993 / 1035-1048.

**Why it matters**
Frequent broker rejections in live trading; strategy docs emphasize robust order management and broker compliance.

**Fix needed**
Before `OrderOpen`, validate:
- pending price distance from current Bid/Ask (respect `SYMBOL_TRADE_STOPS_LEVEL`),
- SL/TP distances from pending open price,
- freeze-level constraints for modification-sensitive scenarios.

---

## 3) Grid limit/stop direction logic is incorrect
**Problem**
Grid price calculation always places BUY grid below and SELL grid above (`orderPrice = basePrice ± step`) regardless of `GRID_Limits` vs `GRID_Stops`.  
Code: lines 984-987 and 1039-1042.

**Why it matters**
For stop grids, this is inverted: BUY STOP should generally be above market and SELL STOP below market.

**Fix needed**
Branch price generation by order type:
- limits: buy below / sell above,
- stops: buy above / sell below.

---

## 4) Grid lot-cap check ignores existing exposure
**Problem**
`totalLots` starts at 0 for each new placement pass and does not include already-open + pending grid lots.  
Code: lines 954, 968-969, 1010, 1025-1026.

**Why it matters**
`InpGrid_MaxTotalLots` can be exceeded across multiple replenishment cycles.

**Fix needed**
Compute existing grid lots (positions + pendings with same grid magic) first, then enforce cap on `existing + new` lots.

---

## 5) Reversal “BothRequired” mode duplicates “BeyondTP” logic
**Problem**
`REV_BothRequired` path uses the same condition as `REV_AfterBeyondTPByPips`; no second requirement is enforced.  
Code: lines 845-854.

**Why it matters**
This violates user-facing configuration semantics.

**Fix needed**
Define and enforce a second condition (e.g., TP-hit flag + beyond distance + optional pullback confirmation stage) so mode behavior is distinct.

---

## 6) Pullback-from-peak calculation uses absolute distance
**Problem**
Pullback in `PB_PipsFromPeak` is computed with `MathAbs(currentPrice - g_peakAfterTP)`.  
Code: line 872.

**Why it matters**
Absolute distance can mis-detect continuation as pullback depending on direction; pullback should be directional from extreme.

**Fix needed**
Compute directional pullback:
- if original was BUY: `g_peakAfterTP - currentPrice`,
- if original was SELL: `currentPrice - g_peakAfterTP`,
then compare against threshold.

---

## 7) Min-seconds-between-entries only enforced for System 1
**Problem**
`InpSys1_MinSecBetween` is checked only in single-entry logic. Grid entry paths do not have equivalent throttling.  
Code: line 805 vs grid logic lines 929+.

**Why it matters**
Can violate expected cadence controls during grid replenishment.

**Fix needed**
Add a generic inter-entry throttle for grid placements (or separate `InpSys2_MinSecBetween`).

---

## 8) Opposite-direction control is not enforced for grid mode
**Problem**
`InpAllowOppositeDirection` check exists in `PlaceOriginalEntries()` only.  
Code: lines 808-812.

**Why it matters**
Grid mode can open directionally conflicting books despite user intent.

**Fix needed**
Apply same direction policy to grid original and reversal order creation paths.

---

## 9) Partial close implementation is unsafe on netting accounts
**Problem**
`ClosePositionPartial()` sends opposite market orders (`g_trade.Sell`/`g_trade.Buy`) instead of using `PositionClosePartial`, and may not target the exact ticket under netting behavior.  
Code: lines 1108-1113.

**Why it matters**
Can produce unexpected netting behavior, wrong magic attribution, or partial-close failure.

**Fix needed**
Use ticket-aware partial close API where available (`CTrade::PositionClosePartial`) and add account mode checks (`ACCOUNT_MARGIN_MODE_*`).

---

## 10) Partial close lot rounding can violate minimum remainder constraints
**Problem**
The routine normalizes closed volume but does not robustly guarantee that both closed and remaining volumes satisfy symbol step/min constraints after rounding.  
Code: lines 1095-1100.

**Why it matters**
Broker rejects partial closes intermittently.

**Fix needed**
Quantize both close and remainder to step, ensure:
- `closeVol >= minLot`,
- `remainVol == 0 || remainVol >= minLot`,
- both aligned to `SYMBOL_VOLUME_STEP`.

---

## 11) Once-per-bar logic can suppress all same-bar entries forever after first tick
**Problem**
When `InpOneTradePerBar` is true and state is idle, second+ ticks in same bar return early regardless of whether a trade occurred.  
Code: lines 1653-1660.

**Why it matters**
This is effectively “one signal evaluation per bar,” not “one trade per bar.” It can miss valid late-bar signals.

**Fix needed**
Track **last trade bar time** separately; only block when a trade was actually opened this bar.

---

## 12) State transition from signal-triggered to idle can race for pending-only grids
**Problem**
After `PlaceGridOrdersOriginal(signal)`, state checks only `CountAllEAPositions()>0` to remain running. If first cycle creates only pendings (or placement delays), it may reset idle.  
Code: lines 1534-1538.

**Why it matters**
Can destabilize grid lifecycle and signal handling.

**Fix needed**
Use `CountAllEAPositions() + CountPendings()` for state transition when grid mode is active.

---

## 13) Portfolio average-SL definition input is ignored
**Problem**
`InpExit3_AvgSLDef` exists (weighted/simple enum) but only weighted calculation is implemented; simple average mode is not honored.  
Code: input at lines 214-215, calc at 1313-1342.

**Why it matters**
Configuration mismatch with documented strategy options.

**Fix needed**
Implement both:
- weighted average SL,
- simple arithmetic average SL,
and select by `InpExit3_AvgSLDef`.

---

## 14) Portfolio SL mode semantics are incomplete
**Problem**
`E3_IndividualSL` mode is declared, but portfolio loss checks/partial-loss handling run only under `E3_AverageSL`.  
Code: lines 1413-1442.

**Why it matters**
In individual-SL mode, configurable portfolio SLValue/partial-loss behavior is effectively bypassed.

**Fix needed**
Clarify intended behavior and implement explicitly:
- if IndividualSL: rely only on per-order SLs (and disable portfolio SL params), or
- support optional portfolio loss threshold in both modes.

---

## 15) Portfolio partial trigger flags are not reset on all lifecycle paths
**Problem**
Flags reset in cooldown and TP-close path, but not consistently when manual/forced closures occur mid-cycle without state reaching cooldown promptly.  
Code: lines 1384-1385, 1608-1609.

**Why it matters**
Can suppress future partial events unexpectedly.

**Fix needed**
Centralize lifecycle reset into a single function called on:
- full cycle completion,
- emergency close-all,
- external/manual flat detection.

---

## 16) OnTradeTransaction TP detection may miss events without history selection safeguards
**Problem**
`HistoryDealGet*` is used directly in transaction callback without ensuring history window selection context.  
Code: lines 1458-1479.

**Why it matters**
Depending on terminal state/broker, history access can fail intermittently.

**Fix needed**
Add defensive checks for deal ticket validity and fallback selection handling before reading deal fields.

---

## 17) Trade filling policy is set twice and can conflict
**Problem**
`SetTypeFilling(ORDER_FILLING_FOK)` then `SetTypeFillingBySymbol(g_symbol)` are both called in init.  
Code: lines 328-329.

**Why it matters**
First setting is redundant/confusing and can be wrong for brokers not supporting FOK.

**Fix needed**
Use only `SetTypeFillingBySymbol(g_symbol)` unless user explicitly configures filling policy.

---

## 18) Input naming/behavior mismatch against strategy docs
**Problem**
Docs describe exact behaviors (e.g., grid semantics, average-SL variants, one-trade-per-bar intent), but implementation differs in several places above.  
Docs: `EA-Complete-Guide.md`, `Complete-Strategy-Guide.md`.

**Why it matters**
Operational expectation mismatch and optimization confusion.

**Fix needed**
Update either code to match docs, or docs to match code. Prefer code fixes for strategy-critical items.

---

## 19) Legacy analysis file is now outdated and misleading
**Problem**
`Problems-Analysis.md` claims numerous missing functions/classes that are no longer relevant to `mastercodemql5new.mq5` (it appears to describe the old fxDreema file).  
Files: `Problems-Analysis.md` vs `Master-Trader.txt` and current EA.

**Why it matters**
Can mislead development priorities and bug triage.

**Fix needed**
Replace with a current gap analysis focused on `mastercodemql5new.mq5`, or clearly label it as legacy reference.

---

## 20) Missing validation of key input ranges
**Problem**
No hard validation for many user inputs (percent bounds, non-negative pips, grid max >=1, etc.).  
Code: `OnInit()` currently does not enforce parameter sanity.

**Why it matters**
Invalid params can create undefined behavior or silent no-op logic.

**Fix needed**
Add centralized `ValidateInputs()` called from `OnInit()`; fail init (`INIT_PARAMETERS_INCORRECT`) on invalid ranges.

---

## Priority fix order
1. **Execution correctness**: items 1,2,3,9,10,11,12.
2. **Strategy conformance**: items 5,6,8,13,14,18.
3. **Robustness/maintainability**: items 4,15,16,17,19,20.

---

## Recommended acceptance tests after fixes
- Pending placement success on broker returning `PLACED`.
- Grid limit and stop modes produce correct side-relative price ladders.
- Netting vs hedging partial close behavior validated.
- One-trade-per-bar allows multiple signal checks but max one open action per bar.
- Portfolio close mode tests:
  - money target,
  - pips target,
  - simple vs weighted avg SL,
  - partial profit/loss triggers.
