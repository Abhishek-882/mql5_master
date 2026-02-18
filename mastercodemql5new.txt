//+------------------------------------------------------------------+
//|                                           MasterTrader_V2.mq5    |
//|                        Master Trader EA V2.0 - Multi-System EA   |
//|                  2 Entry Systems | 3 Exit Systems | State Machine |
//+------------------------------------------------------------------+
#property copyright "Master Trader V2.0"
#property link      ""
#property version   "2.00"
#property strict
#property description "Production-Ready Multi-System EA"
#property description "2 Mutually Exclusive Entry Systems, 3 Exit Systems"
#property description "State Machine, Grid, Reversal, Portfolio Close"

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>
#include <Trade\OrderInfo.mqh>

//+------------------------------------------------------------------+
//|                         ENUMERATIONS                              |
//+------------------------------------------------------------------+
enum ENUM_ENTRY_SYSTEM
{
   ENTRY_System1_SingleReversal = 0, // System1: Single + Reversal
   ENTRY_System2_GridReversal   = 1  // System2: Grid + Reversal Grid
};

enum ENUM_EXIT_SYSTEM
{
   EXIT_DirectTPSL       = 0, // Exit1: Direct TP/SL Only
   EXIT_DirectWithPartial = 1, // Exit2: Direct + Partial Close
   EXIT_PortfolioClose    = 2  // Exit3: Portfolio Close
};

enum ENUM_LOT_MODE
{
   LOT_Fixed        = 0, // Fixed Lot
   LOT_RiskPercent  = 1, // Risk % Per Trade
   LOT_EquityPercent = 2  // Equity % For Lots
};

enum ENUM_REVERSAL_TRIGGER
{
   REV_AfterTPHitOnly       = 0, // After Original TP Hit Only
   REV_AfterBeyondTPByPips  = 1, // After Price Beyond TP by X Pips
   REV_BothRequired         = 2  // Both Conditions Required
};

enum ENUM_PULLBACK_MODE
{
   PB_PipsFromPeak   = 0, // Pullback X Pips From Peak
   PB_OppositeSignal = 1, // Opposite Mini-Signal
   PB_CandlePattern  = 2  // Candle Pattern Reversal
};

enum ENUM_GRID_MODE
{
   GRID_MarketPlusPending = 0, // Market + Pending Grid
   GRID_PendingOnly       = 1  // Pending Only
};

enum ENUM_GRID_ORDER_TYPE
{
   GRID_Limits = 0, // Limit Orders
   GRID_Stops  = 1  // Stop Orders
};

enum ENUM_GRID_LOT_DIST
{
   GRID_EqualLots       = 0, // Equal Lots
   GRID_Martingale      = 1, // Martingale
   GRID_CustomMultiplier = 2  // Custom Multiplier
};

enum ENUM_EXIT2_MODE
{
   E2_ProfitOnly       = 0, // Partial On Profit Only
   E2_ProfitAndLoss    = 1  // Partial On Profit And Loss
};

enum ENUM_EXIT3_PROFIT_MODE
{
   E3_Money   = 0, // Money Target
   E3_PipsSum = 1  // Pips Sum Target
};

enum ENUM_EXIT3_SL_MODE
{
   E3_IndividualSL = 0, // Individual SL Per Order
   E3_AverageSL    = 1  // Average SL (Basket)
};

enum ENUM_EXIT3_AVGSL_DEF
{
   E3_WeightedAvg = 0, // Weighted By Lots Average
   E3_SimpleAvg   = 1  // Simple Average Of SL Prices
};

enum ENUM_SIGNAL_TYPE
{
   SIG_Continuous = 0, // Continuous (fires every tick)
   SIG_OncePerBar = 1  // Once Per Bar
};

//+------------------------------------------------------------------+
//|                     STATE MACHINE ENUM                            |
//+------------------------------------------------------------------+
enum ENUM_EA_STATE
{
   STATE_Idle                    = 0,
   STATE_OriginalSignalTriggered = 1,
   STATE_OriginalPositionsRunning = 2,
   STATE_OriginalTPHit           = 3,
   STATE_ReversalEligibleWindow  = 4,
   STATE_ReversalPositionsRunning = 5,
   STATE_CoolingDown             = 6
};

//+------------------------------------------------------------------+
//|                       INPUT PARAMETERS                            |
//+------------------------------------------------------------------+

//--- GENERAL
input string   InpSymbol                = "";          // Symbol (empty=current)
input ENUM_TIMEFRAMES InpTimeframe      = PERIOD_CURRENT; // Timeframe
input long     InpMagicBase             = 10000;       // Magic Number Base
input string   InpTradeCommentPrefix    = "MTV2";      // Trade Comment Prefix
input bool     InpMaxSpreadFilterEnabled = false;      // Enable Max Spread Filter (default OFF)
input int      InpMaxSpreadPoints       = 50;          // Max Spread Points (if filter ON)
input int      InpSlippagePoints        = 10;          // Slippage/Deviation Points
input bool     InpOneTradePerBar        = true;        // One Trade Per Bar
input int      InpCooldownSecondsAfterTrade = 60;      // Cooldown Seconds After Trade
input bool     InpAllowOppositeDirection = true;       // Allow Opposite Direction Positions

//--- SIGNAL (Ported from original EA)
input double   InpTimeBombUpPips        = 200.0;       // TimeBomb Up: Pips Threshold
input double   InpTimeBombUpSeconds     = 3.0;         // TimeBomb Up: Time Window (seconds)
input double   InpTimeBombDownPips      = 200.0;       // TimeBomb Down: Pips Threshold
input double   InpTimeBombDownSeconds   = 3.0;         // TimeBomb Down: Time Window (seconds)
input ENUM_SIGNAL_TYPE InpSignalType    = SIG_Continuous; // Signal Type
input int      InpImpulseQuietBars      = 3;           // Impulse: Quiet Bars Count
input double   InpImpulseQuietMinRange  = 5.0;         // Impulse: Quiet Bars Min Range (pips)
input double   InpImpulseStrength       = 3.0;         // Impulse: Strength Multiplier
input int      InpImpulseShift          = 0;           // Impulse: Bar Shift

//--- ENTRY SYSTEM MASTER SWITCH
input ENUM_ENTRY_SYSTEM InpEntrySystemMode = ENTRY_System1_SingleReversal; // Entry System Mode

//--- SYSTEM 1 (Single + Reversal)
input bool     InpSys1_EnableOriginal   = true;        // Sys1: Enable Original Single Entry
input bool     InpSys1_EnableReversal   = false;       // Sys1: Enable Reversal Entry
input int      InpSys1_MaxPositions     = 3;           // Sys1: Max Simultaneous Positions
input int      InpSys1_MinSecBetween    = 30;          // Sys1: Min Seconds Between Entries

//--- REVERSAL (Pullback) LOGIC
input bool     InpReversal_Enable       = false;       // Reversal: Master Enable
input ENUM_REVERSAL_TRIGGER InpReversal_TriggerMode = REV_AfterTPHitOnly; // Reversal: Trigger Mode
input double   InpReversal_BeyondTP_Pips = 50.0;       // Reversal: Beyond TP Pips
input ENUM_PULLBACK_MODE InpReversal_PullbackMode = PB_PipsFromPeak; // Reversal: Pullback Detect Mode
input double   InpReversal_PullbackPips = 30.0;        // Reversal: Pullback Pips From Peak
input int      InpReversal_MaxSecsAfterTP = 300;       // Reversal: Max Seconds After TP
input bool     InpReversal_OnlyIfFading = false;       // Reversal: Only If Trend Fading
input double   InpReversal_FadingMomDrop = 50.0;       // Reversal: Fading Momentum Drop %
input int      InpReversal_FadingBars   = 3;           // Reversal: Fading Check Bars
input double   InpReversal_TP_Pips      = 400.0;       // Reversal: TP Pips
input double   InpReversal_SL_Pips      = 400.0;       // Reversal: SL Pips

//--- SYSTEM 2 (Grid)
input bool     InpSys2_EnableOriginalGrid = true;      // Sys2: Enable Original Grid
input bool     InpSys2_EnableReversalGrid = false;     // Sys2: Enable Reversal Grid
input int      InpGrid_MaxOrders        = 5;           // Grid: Max Orders
input double   InpGrid_StepPips         = 20.0;        // Grid: Step Pips
input ENUM_GRID_MODE InpGrid_Mode       = GRID_MarketPlusPending; // Grid: Mode
input ENUM_GRID_ORDER_TYPE InpGrid_OrderType = GRID_Limits; // Grid: Order Type
input int      InpGrid_ActivationDelay  = 60;          // Grid: Activation Delay Seconds
input ENUM_GRID_LOT_DIST InpGrid_LotDist = GRID_EqualLots; // Grid: Lot Distribution
input double   InpGrid_LotMultiplier    = 1.5;         // Grid: Lot Multiplier (Martingale/Custom)
input double   InpGrid_MaxTotalLots     = 5.0;         // Grid: Max Total Lots
input bool     InpGrid_CancelOnExit     = true;        // Grid: Cancel Pendings On Exit
input bool     InpGrid_ReplaceMissing   = true;        // Grid: Replace Missing Pendings

//--- RISK / LOT SIZING
input ENUM_LOT_MODE InpLotMode          = LOT_Fixed;   // Lot Mode
input double   InpFixedLot              = 0.1;         // Fixed Lot Size
input double   InpRiskPercentPerTrade   = 2.0;         // Risk % Per Trade
input double   InpEquityPercentForLots  = 5.0;         // Equity % For Lots
input double   InpMaxLot                = 10.0;        // Max Lot Size
input double   InpMinLot                = 0.0;         // Min Lot Override (0=use symbol min)
input int      InpMaxPositionsTotal     = 10;          // Max Positions Total
input int      InpMaxPendingsTotal      = 20;          // Max Pendings Total
input bool     InpDailyLossLimitEnabled = false;       // Enable Daily Loss Limit
input double   InpDailyLossPercent      = 2.0;         // Daily Loss Limit %
input bool     InpWeeklyLossLimitEnabled = false;      // Enable Weekly Loss Limit
input double   InpWeeklyLossPercent     = 5.0;         // Weekly Loss Limit %

//--- BASE TP/SL
input double   InpOriginal_TP_Pips      = 600.0;       // Original TP Pips
input double   InpOriginal_SL_Pips      = 600.0;       // Original SL Pips

//--- EXIT SYSTEM
input ENUM_EXIT_SYSTEM InpExitSystemMode = EXIT_DirectTPSL; // Exit System Mode

//--- EXIT SYSTEM 2: Partial Close
input bool     InpExit2_EnablePartial   = true;        // Exit2: Enable Partial Close
input double   InpExit2_PartialClosePct = 50.0;        // Exit2: Partial Close %
input double   InpExit2_ActivPctOfTP    = 50.0;        // Exit2: Activation % of TP (Profit Side)
input ENUM_EXIT2_MODE InpExit2_Mode     = E2_ProfitOnly; // Exit2: Mode
input double   InpExit2_ActivPctOfSL    = 50.0;        // Exit2: Activation % of SL (Loss Side)
input bool     InpExit2_MoveSLtoBE      = true;        // Exit2: Move SL to Breakeven After Partial
input int      InpExit2_BEOffsetPoints  = 1;           // Exit2: Breakeven Offset Points

//--- EXIT SYSTEM 3: Portfolio Close
input ENUM_EXIT3_PROFIT_MODE InpExit3_ProfitMode = E3_Money; // Exit3: Target Profit Mode
input double   InpExit3_TargetValue     = 100.0;       // Exit3: Target Profit Value
input ENUM_EXIT3_SL_MODE InpExit3_SLMode = E3_IndividualSL; // Exit3: SL Mode
input ENUM_EXIT3_AVGSL_DEF InpExit3_AvgSLDef = E3_WeightedAvg; // Exit3: Average SL Definition
input double   InpExit3_SLValue         = 500.0;       // Exit3: SL Value (Pips or Money)
input bool     InpExit3_EnablePartialProfit = false;   // Exit3: Enable Portfolio Partial Profit
input double   InpExit3_PartialProfitTrigPct = 50.0;   // Exit3: Partial Profit Trigger %
input double   InpExit3_PartialProfitClosePct = 50.0;  // Exit3: Partial Profit Close %
input bool     InpExit3_MoveSLtoBEAfterPartial = true; // Exit3: Move SL to BE After Partial
input int      InpExit3_BEOffsetPoints  = 1;           // Exit3: BE Offset Points
input bool     InpExit3_EnablePartialLoss = false;     // Exit3: Enable Portfolio Partial Loss
input double   InpExit3_PartialLossTrigPct = 50.0;     // Exit3: Partial Loss Trigger %
input double   InpExit3_PartialLossClosePct = 50.0;    // Exit3: Partial Loss Close %

//+------------------------------------------------------------------+
//|                      MAGIC NUMBER OFFSETS                          |
//+------------------------------------------------------------------+
#define MAGIC_ORIGINAL_SINGLE_BUY   1001
#define MAGIC_ORIGINAL_SINGLE_SELL  1002
#define MAGIC_REVERSAL_SINGLE_BUY   1003
#define MAGIC_REVERSAL_SINGLE_SELL  1004
#define MAGIC_GRID_ORIG_BUY         2001
#define MAGIC_GRID_ORIG_SELL        2002
#define MAGIC_GRID_REV_BUY          2003
#define MAGIC_GRID_REV_SELL         2004

//+------------------------------------------------------------------+
//|                     GLOBAL VARIABLES                               |
//+------------------------------------------------------------------+
CTrade         g_trade;
CPositionInfo  g_posInfo;
COrderInfo     g_ordInfo;

string         g_symbol;
ENUM_TIMEFRAMES g_timeframe;
int            g_digits;
double         g_point;
double         g_pipValue;     // points per pip
double         g_tickSize;
double         g_lotStep;
double         g_lotMin;
double         g_lotMax;
long           g_stopsLevel;
long           g_freezeLevel;

// State Machine
ENUM_EA_STATE  g_state = STATE_Idle;
datetime       g_lastTradeTime = 0;
datetime       g_lastBarTime = 0;
datetime       g_cooldownUntil = 0;

// TP-Hit detection for reversal
datetime       g_lastTPHitTime = 0;
double         g_lastTPHitPrice = 0.0;
int            g_lastTPHitDirection = 0; // 1=was BUY, -1=was SELL
double         g_peakAfterTP = 0.0;

// TimeBomb state
double         g_tbUp_lastPrice = 0;
datetime       g_tbUp_lastTime = 0;
double         g_tbDn_lastPrice = 0;
datetime       g_tbDn_lastTime = 0;

// Impulse state
double         g_impUp_QBrange = -1;
datetime       g_impUp_barTime = 0;
datetime       g_impUp_barTime2 = 0;
double         g_impUp_open = 0;
double         g_impDn_QBrange = -1;
datetime       g_impDn_barTime = 0;
datetime       g_impDn_barTime2 = 0;
double         g_impDn_open = 0;

// Grid state
datetime       g_gridActivationTime = 0;
bool           g_gridActivated = false;
int            g_gridDirection = 0; // 1=BUY grid, -1=SELL grid

// Partial close tracking (per-position flags stored by ticket)
long           g_partialClosedTickets[];
int            g_partialClosedCount = 0;

// Portfolio partial tracking
bool           g_portfolioPartialProfitDone = false;
bool           g_portfolioPartialLossDone = false;

// Daily/Weekly loss tracking
double         g_dailyStartBalance = 0;
double         g_weeklyStartBalance = 0;
datetime       g_dailyResetTime = 0;
datetime       g_weeklyResetTime = 0;

//+------------------------------------------------------------------+
//|                        INITIALIZATION                              |
//+------------------------------------------------------------------+
int OnInit()
{
   // Resolve symbol
   g_symbol = (InpSymbol == "" || InpSymbol == "current") ? _Symbol : InpSymbol;
   g_timeframe = (InpTimeframe == PERIOD_CURRENT) ? (ENUM_TIMEFRAMES)_Period : InpTimeframe;

   // Cache symbol properties
   g_digits      = (int)SymbolInfoInteger(g_symbol, SYMBOL_DIGITS);
   g_point       = SymbolInfoDouble(g_symbol, SYMBOL_POINT);
   g_tickSize    = SymbolInfoDouble(g_symbol, SYMBOL_TRADE_TICK_SIZE);
   g_lotStep     = SymbolInfoDouble(g_symbol, SYMBOL_VOLUME_STEP);
   g_lotMin      = SymbolInfoDouble(g_symbol, SYMBOL_VOLUME_MIN);
   g_lotMax      = SymbolInfoDouble(g_symbol, SYMBOL_VOLUME_MAX);
   g_stopsLevel  = SymbolInfoInteger(g_symbol, SYMBOL_TRADE_STOPS_LEVEL);
   g_freezeLevel = SymbolInfoInteger(g_symbol, SYMBOL_TRADE_FREEZE_LEVEL);
   g_pipValue    = GetPipValuePoints();

   // Setup trade object
   g_trade.SetExpertMagicNumber((ulong)InpMagicBase);
   g_trade.SetDeviationInPoints(InpSlippagePoints);
   g_trade.SetTypeFilling(ORDER_FILLING_FOK);
   g_trade.SetTypeFillingBySymbol(g_symbol);

   // Initialize loss tracking
   g_dailyStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   g_weeklyStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   g_dailyResetTime = iTime(g_symbol, PERIOD_D1, 0);
   g_weeklyResetTime = iTime(g_symbol, PERIOD_W1, 0);

   Print("=== Master Trader V2.0 Initialized ===");
   Print("Symbol: ", g_symbol, " TF: ", EnumToString(g_timeframe));
   Print("Entry System: ", EnumToString(InpEntrySystemMode));
   Print("Exit System: ", EnumToString(InpExitSystemMode));
   Print("Pip Value Points: ", g_pipValue);
   Print("======================================");

   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   Print("Master Trader V2.0 Deinitialized. Reason: ", reason);
}

//+------------------------------------------------------------------+
//|                    PIP VALUE CALCULATION                           |
//+------------------------------------------------------------------+
double GetPipValuePoints()
{
   // Handle 2/3/4/5 digits + metals
   int d = g_digits;
   if(d == 5 || d == 3) return 10.0;
   if(d == 4 || d == 2) return 1.0;
   if(d == 1) return 1.0;  // metals like XAUUSD with 1 digit (rare)
   if(d == 6) return 10.0;
   return 10.0; // safe default for 5-digit
}

double PipsToPrice(double pips)
{
   return NormalizeDouble(pips * g_pipValue * g_point, g_digits);
}

double PriceToPips(double priceDistance)
{
   if(g_pipValue * g_point == 0) return 0;
   return priceDistance / (g_pipValue * g_point);
}

//+------------------------------------------------------------------+
//|                    NORMALIZE FUNCTIONS                              |
//+------------------------------------------------------------------+
double NormalizePrice(double price)
{
   if(g_tickSize > 0)
      price = MathRound(price / g_tickSize) * g_tickSize;
   return NormalizeDouble(price, g_digits);
}

double NormalizeLots(double lots)
{
   double minLot = (InpMinLot > 0) ? InpMinLot : g_lotMin;
   double maxLot = MathMin(InpMaxLot, g_lotMax);

   if(g_lotStep > 0)
      lots = MathFloor(lots / g_lotStep) * g_lotStep;

   if(lots < minLot) lots = minLot;
   if(lots > maxLot) lots = maxLot;

   return NormalizeDouble(lots, 8);
}

//+------------------------------------------------------------------+
//|                   POSITION / ORDER COUNTING                        |
//+------------------------------------------------------------------+
int CountPositions(long magicOffset = -1, int direction = 0)
{
   int count = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(magicOffset >= 0 && posMagic != InpMagicBase + magicOffset) continue;
      if(magicOffset < 0)
      {
         // Count any position belonging to our EA
         if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;
      }

      if(direction > 0 && g_posInfo.PositionType() != POSITION_TYPE_BUY) continue;
      if(direction < 0 && g_posInfo.PositionType() != POSITION_TYPE_SELL) continue;

      count++;
   }
   return count;
}

int CountAllEAPositions()
{
   return CountPositions(-1, 0);
}

int CountPendings(long magicOffset = -1)
{
   int count = 0;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!g_ordInfo.SelectByIndex(i)) continue;
      if(g_ordInfo.Symbol() != g_symbol) continue;

      long ordMagic = g_ordInfo.Magic();
      if(magicOffset >= 0 && ordMagic != InpMagicBase + magicOffset) continue;
      if(magicOffset < 0)
      {
         if(ordMagic < InpMagicBase || ordMagic > InpMagicBase + 9999) continue;
      }
      count++;
   }
   return count;
}

//+------------------------------------------------------------------+
//|                   LOT SIZE CALCULATION                              |
//+------------------------------------------------------------------+
double CalculateLotSize(double slPips = 0)
{
   double lots = InpFixedLot;

   if(InpLotMode == LOT_RiskPercent && slPips > 0)
   {
      double balance = AccountInfoDouble(ACCOUNT_BALANCE);
      double riskMoney = balance * InpRiskPercentPerTrade / 100.0;
      double tickValue = SymbolInfoDouble(g_symbol, SYMBOL_TRADE_TICK_VALUE);
      double tickSize  = SymbolInfoDouble(g_symbol, SYMBOL_TRADE_TICK_SIZE);

      if(tickValue > 0 && tickSize > 0)
      {
         double slPrice = slPips * g_pipValue * g_point;
         double lossPerLot = (slPrice / tickSize) * tickValue;
         if(lossPerLot > 0)
            lots = riskMoney / lossPerLot;
      }
   }
   else if(InpLotMode == LOT_EquityPercent)
   {
      double equity = AccountInfoDouble(ACCOUNT_EQUITY);
      double marginPerLot = 0;
      if(OrderCalcMargin(ORDER_TYPE_BUY, g_symbol, 1.0, SymbolInfoDouble(g_symbol, SYMBOL_ASK), marginPerLot))
      {
         if(marginPerLot > 0)
            lots = (equity * InpEquityPercentForLots / 100.0) / marginPerLot;
      }
   }

   return NormalizeLots(lots);
}

//+------------------------------------------------------------------+
//|                  DAILY/WEEKLY LOSS CHECK                            |
//+------------------------------------------------------------------+
bool IsDailyLossLimitHit()
{
   if(!InpDailyLossLimitEnabled) return false;

   datetime dayStart = iTime(g_symbol, PERIOD_D1, 0);
   if(dayStart != g_dailyResetTime)
   {
      g_dailyResetTime = dayStart;
      g_dailyStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   }

   double currentBal = AccountInfoDouble(ACCOUNT_BALANCE);
   double lossPercent = (g_dailyStartBalance - currentBal) / g_dailyStartBalance * 100.0;
   return (lossPercent >= InpDailyLossPercent);
}

bool IsWeeklyLossLimitHit()
{
   if(!InpWeeklyLossLimitEnabled) return false;

   datetime weekStart = iTime(g_symbol, PERIOD_W1, 0);
   if(weekStart != g_weeklyResetTime)
   {
      g_weeklyResetTime = weekStart;
      g_weeklyStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   }

   double currentBal = AccountInfoDouble(ACCOUNT_BALANCE);
   double lossPercent = (g_weeklyStartBalance - currentBal) / g_weeklyStartBalance * 100.0;
   return (lossPercent >= InpWeeklyLossPercent);
}

//+------------------------------------------------------------------+
//|            SIGNAL DETECTION (Ported from Original EA)              |
//+------------------------------------------------------------------+
bool DetectTimeBombUp()
{
   datetime timeNow = (datetime)TimeCurrent();
   double priceNow  = SymbolInfoDouble(g_symbol, SYMBOL_ASK);
   bool fired = false;

   if(g_tbUp_lastPrice > 0)
   {
      double pipsDiff = PriceToPips(priceNow - g_tbUp_lastPrice);
      int timeDiff    = (int)(timeNow - g_tbUp_lastTime);

      if(pipsDiff >= InpTimeBombUpPips && timeDiff <= (int)InpTimeBombUpSeconds)
      {
         if(priceNow > g_tbUp_lastPrice)
            fired = true;
         g_tbUp_lastPrice = priceNow;
         g_tbUp_lastTime  = timeNow;
      }
      else if(timeDiff >= (int)InpTimeBombUpSeconds)
      {
         g_tbUp_lastPrice = priceNow;
         g_tbUp_lastTime  = timeNow;
      }
   }
   else
   {
      g_tbUp_lastPrice = priceNow;
      g_tbUp_lastTime  = timeNow;
   }

   return fired;
}

bool DetectTimeBombDown()
{
   datetime timeNow = (datetime)TimeCurrent();
   double priceNow  = SymbolInfoDouble(g_symbol, SYMBOL_BID);
   bool fired = false;

   if(g_tbDn_lastPrice > 0)
   {
      double pipsDiff = PriceToPips(g_tbDn_lastPrice - priceNow);
      int timeDiff    = (int)(timeNow - g_tbDn_lastTime);

      if(pipsDiff >= InpTimeBombDownPips && timeDiff <= (int)InpTimeBombDownSeconds)
      {
         if(priceNow < g_tbDn_lastPrice)
            fired = true;
         g_tbDn_lastPrice = priceNow;
         g_tbDn_lastTime  = timeNow;
      }
      else if(timeDiff >= (int)InpTimeBombDownSeconds)
      {
         g_tbDn_lastPrice = priceNow;
         g_tbDn_lastTime  = timeNow;
      }
   }
   else
   {
      g_tbDn_lastPrice = priceNow;
      g_tbDn_lastTime  = timeNow;
   }

   return fired;
}

bool DetectImpulseUp()
{
   bool newBar = false;
   bool pass = false;
   string sigType = (InpSignalType == SIG_Continuous) ? "continuous" : "once";

   datetime barT = iTime(g_symbol, g_timeframe, 1);
   if(g_impUp_barTime < barT || g_impUp_QBrange < 0)
   {
      g_impUp_barTime = barT;
      g_impUp_QBrange = 0;

      for(int i = 0; i < InpImpulseQuietBars; i++)
      {
         int shift = i + 1 + InpImpulseShift;
         double barRange = iHigh(g_symbol, g_timeframe, shift) - iLow(g_symbol, g_timeframe, shift);
         if(barRange > g_impUp_QBrange)
            g_impUp_QBrange = barRange;
      }

      g_impUp_open = iOpen(g_symbol, g_timeframe, InpImpulseShift);
      newBar = true;
   }

   double minRange = PipsToPrice(InpImpulseQuietMinRange);

   if((sigType == "continuous" || g_impUp_barTime2 < barT)
      && ((InpImpulseShift == 0 && g_impUp_open > 0) || newBar)
      && g_impUp_QBrange > minRange)
   {
      double impSize = iClose(g_symbol, g_timeframe, InpImpulseShift) - g_impUp_open;
      if(impSize >= g_impUp_QBrange * InpImpulseStrength)
      {
         if(sigType != "continuous")
            g_impUp_barTime2 = barT;
         pass = true;
      }
   }

   return pass;
}

bool DetectImpulseDown()
{
   bool newBar = false;
   bool pass = false;
   string sigType = (InpSignalType == SIG_Continuous) ? "continuous" : "once";

   datetime barT = iTime(g_symbol, g_timeframe, 1);
   if(g_impDn_barTime < barT || g_impDn_QBrange < 0)
   {
      g_impDn_barTime = barT;
      g_impDn_QBrange = 0;

      for(int i = 0; i < InpImpulseQuietBars; i++)
      {
         int shift = i + 1 + InpImpulseShift;
         double barRange = iHigh(g_symbol, g_timeframe, shift) - iLow(g_symbol, g_timeframe, shift);
         if(barRange > g_impDn_QBrange)
            g_impDn_QBrange = barRange;
      }

      g_impDn_open = iOpen(g_symbol, g_timeframe, InpImpulseShift);
      newBar = true;
   }

   double minRange = PipsToPrice(InpImpulseQuietMinRange);

   if((sigType == "continuous" || g_impDn_barTime2 < barT)
      && ((InpImpulseShift == 0 && g_impDn_open > 0) || newBar)
      && g_impDn_QBrange > minRange)
   {
      double impSize = g_impDn_open - iClose(g_symbol, g_timeframe, InpImpulseShift);
      if(impSize >= g_impDn_QBrange * InpImpulseStrength)
      {
         if(sigType != "continuous")
            g_impDn_barTime2 = barT;
         pass = true;
      }
   }

   return pass;
}

// Combined signal: returns +1 for BUY, -1 for SELL, 0 for none
int DetectSignal()
{
   bool tbUp = DetectTimeBombUp();
   bool tbDn = DetectTimeBombDown();
   bool impUp = false, impDn = false;

   if(tbUp)  impUp = DetectImpulseUp();
   if(tbDn)  impDn = DetectImpulseDown();

   if(tbUp && impUp) return +1;
   if(tbDn && impDn) return -1;
   return 0;
}

//+------------------------------------------------------------------+
//|                    TRADE EXECUTION HELPERS                         |
//+------------------------------------------------------------------+
bool ExecuteTradeWithRetry(bool isBuy, double lots, double sl, double tp, long magicOffset, string comment, int maxRetries = 3)
{
   g_trade.SetExpertMagicNumber((ulong)(InpMagicBase + magicOffset));

   for(int attempt = 0; attempt < maxRetries; attempt++)
   {
      double price = isBuy ? SymbolInfoDouble(g_symbol, SYMBOL_ASK) : SymbolInfoDouble(g_symbol, SYMBOL_BID);
      double slPrice = 0, tpPrice = 0;

      if(sl > 0)
      {
         slPrice = isBuy ? NormalizePrice(price - PipsToPrice(sl)) : NormalizePrice(price + PipsToPrice(sl));
         // Respect stops level
         double minDist = g_stopsLevel * g_point;
         double dist = MathAbs(price - slPrice);
         if(dist < minDist && minDist > 0)
            slPrice = isBuy ? NormalizePrice(price - minDist) : NormalizePrice(price + minDist);
      }

      if(tp > 0)
      {
         tpPrice = isBuy ? NormalizePrice(price + PipsToPrice(tp)) : NormalizePrice(price - PipsToPrice(tp));
         double minDist = g_stopsLevel * g_point;
         double dist = MathAbs(price - tpPrice);
         if(dist < minDist && minDist > 0)
            tpPrice = isBuy ? NormalizePrice(price + minDist) : NormalizePrice(price - minDist);
      }

      string fullComment = InpTradeCommentPrefix + "_" + comment;
      bool result = false;

      if(isBuy)
         result = g_trade.Buy(lots, g_symbol, price, slPrice, tpPrice, fullComment);
      else
         result = g_trade.Sell(lots, g_symbol, price, slPrice, tpPrice, fullComment);

      if(result && g_trade.ResultRetcode() == TRADE_RETCODE_DONE)
      {
         Print("Trade OK: ", (isBuy?"BUY":"SELL"), " ", lots, " @ ", price,
               " SL=", slPrice, " TP=", tpPrice, " Magic=", InpMagicBase + magicOffset);
         g_lastTradeTime = TimeCurrent();
         return true;
      }
      else
      {
         Print("Trade attempt ", attempt+1, " failed: ", g_trade.ResultRetcodeDescription(),
               " Code: ", g_trade.ResultRetcode());
         if(g_trade.ResultRetcode() == TRADE_RETCODE_REQUOTE ||
            g_trade.ResultRetcode() == TRADE_RETCODE_PRICE_OFF)
         {
            Sleep(100);
            continue;
         }
         break; // non-retriable error
      }
   }
   return false;
}

bool PlacePendingOrder(bool isBuyDirection, bool isLimit, double price, double lots,
                       double slPips, double tpPips, long magicOffset, string comment)
{
   g_trade.SetExpertMagicNumber((ulong)(InpMagicBase + magicOffset));

   double slPrice = 0, tpPrice = 0;
   price = NormalizePrice(price);

   if(slPips > 0)
      slPrice = isBuyDirection ? NormalizePrice(price - PipsToPrice(slPips)) : NormalizePrice(price + PipsToPrice(slPips));
   if(tpPips > 0)
      tpPrice = isBuyDirection ? NormalizePrice(price + PipsToPrice(tpPips)) : NormalizePrice(price - PipsToPrice(tpPips));

   string fullComment = InpTradeCommentPrefix + "_" + comment;

   ENUM_ORDER_TYPE orderType;
   if(isBuyDirection)
      orderType = isLimit ? ORDER_TYPE_BUY_LIMIT : ORDER_TYPE_BUY_STOP;
   else
      orderType = isLimit ? ORDER_TYPE_SELL_LIMIT : ORDER_TYPE_SELL_STOP;

   bool result = g_trade.OrderOpen(g_symbol, orderType, lots, 0, price, slPrice, tpPrice,
                                    ORDER_TIME_GTC, 0, fullComment);

   if(result && g_trade.ResultRetcode() == TRADE_RETCODE_DONE)
   {
      Print("Pending OK: ", EnumToString(orderType), " ", lots, " @ ", price, " Magic=", InpMagicBase + magicOffset);
      return true;
   }
   else
   {
      Print("Pending failed: ", g_trade.ResultRetcodeDescription());
      return false;
   }
}

//+------------------------------------------------------------------+
//|                   ENTRY SYSTEM 1: SINGLE + REVERSAL                |
//+------------------------------------------------------------------+
void PlaceOriginalEntries(int signal)
{
   if(signal == 0) return;
   if(!InpSys1_EnableOriginal && InpEntrySystemMode == ENTRY_System1_SingleReversal) return;

   // Check position limits
   int totalPos = CountAllEAPositions();
   if(totalPos >= InpMaxPositionsTotal) return;

   int sys1Pos = CountPositions(MAGIC_ORIGINAL_SINGLE_BUY) + CountPositions(MAGIC_ORIGINAL_SINGLE_SELL);
   if(sys1Pos >= InpSys1_MaxPositions) return;

   // Check min seconds between entries
   if(TimeCurrent() - g_lastTradeTime < InpSys1_MinSecBetween) return;

   // Check opposite direction
   if(!InpAllowOppositeDirection)
   {
      if(signal > 0 && CountPositions(-1, -1) > 0) return; // has sells, don't buy
      if(signal < 0 && CountPositions(-1, +1) > 0) return; // has buys, don't sell
   }

   double lots = CalculateLotSize(InpOriginal_SL_Pips);
   bool isBuy = (signal > 0);
   long magicOff = isBuy ? MAGIC_ORIGINAL_SINGLE_BUY : MAGIC_ORIGINAL_SINGLE_SELL;
   string tag = isBuy ? "OrigBuy" : "OrigSell";

   if(ExecuteTradeWithRetry(isBuy, lots, InpOriginal_SL_Pips, InpOriginal_TP_Pips, magicOff, tag))
   {
      g_state = STATE_OriginalPositionsRunning;
   }
}

//+------------------------------------------------------------------+
//|                   REVERSAL ENTRIES                                  |
//+------------------------------------------------------------------+
void PlaceReversalEntries()
{
   if(!InpReversal_Enable) return;

   // Check if within reversal window
   if(g_lastTPHitTime == 0) return;
   if(TimeCurrent() - g_lastTPHitTime > InpReversal_MaxSecsAfterTP) return;

   // Check trigger mode
   bool triggerOK = false;
   double currentPrice = (g_lastTPHitDirection > 0) ?
      SymbolInfoDouble(g_symbol, SYMBOL_ASK) : SymbolInfoDouble(g_symbol, SYMBOL_BID);

   if(InpReversal_TriggerMode == REV_AfterTPHitOnly)
   {
      triggerOK = true;
   }
   else if(InpReversal_TriggerMode == REV_AfterBeyondTPByPips)
   {
      double beyondDist = PriceToPips(MathAbs(currentPrice - g_lastTPHitPrice));
      triggerOK = (beyondDist >= InpReversal_BeyondTP_Pips);
   }
   else // Both required
   {
      double beyondDist = PriceToPips(MathAbs(currentPrice - g_lastTPHitPrice));
      triggerOK = (beyondDist >= InpReversal_BeyondTP_Pips);
   }

   if(!triggerOK) return;

   // Update peak tracking
   if(g_lastTPHitDirection > 0) // was buy, track high for sell reversal
   {
      if(currentPrice > g_peakAfterTP) g_peakAfterTP = currentPrice;
   }
   else
   {
      if(g_peakAfterTP == 0 || currentPrice < g_peakAfterTP) g_peakAfterTP = currentPrice;
   }

   // Check pullback detection
   bool pullbackDetected = false;
   if(InpReversal_PullbackMode == PB_PipsFromPeak)
   {
      double pullback = PriceToPips(MathAbs(currentPrice - g_peakAfterTP));
      pullbackDetected = (pullback >= InpReversal_PullbackPips);
   }
   else if(InpReversal_PullbackMode == PB_OppositeSignal)
   {
      // Use opposite impulse/timebomb for reversal signal
      if(g_lastTPHitDirection > 0) // was BUY, look for SELL signal
         pullbackDetected = (DetectTimeBombDown() && DetectImpulseDown());
      else
         pullbackDetected = (DetectTimeBombUp() && DetectImpulseUp());
   }
   else // Candle pattern
   {
      // Simple engulfing pattern check
      double o1 = iOpen(g_symbol, g_timeframe, 1);
      double c1 = iClose(g_symbol, g_timeframe, 1);
      double o2 = iOpen(g_symbol, g_timeframe, 2);
      double c2 = iClose(g_symbol, g_timeframe, 2);

      if(g_lastTPHitDirection > 0) // look for bearish engulfing
         pullbackDetected = (c2 > o2 && c1 < o1 && c1 < o2 && o1 > c2);
      else // look for bullish engulfing
         pullbackDetected = (c2 < o2 && c1 > o1 && c1 > o2 && o1 < c2);
   }

   if(!pullbackDetected) return;

   // Check trend fading
   if(InpReversal_OnlyIfFading)
   {
      double mom0 = MathAbs(iClose(g_symbol, g_timeframe, 1) - iClose(g_symbol, g_timeframe, 1 + InpReversal_FadingBars));
      double mom1 = MathAbs(iClose(g_symbol, g_timeframe, 1 + InpReversal_FadingBars) -
                            iClose(g_symbol, g_timeframe, 1 + InpReversal_FadingBars * 2));
      if(mom1 > 0 && (mom0 / mom1 * 100.0) > (100.0 - InpReversal_FadingMomDrop))
         return; // momentum not fading enough
   }

   // Position limits
   if(CountAllEAPositions() >= InpMaxPositionsTotal) return;

   // Place reversal trade (opposite to original direction)
   bool isBuy = (g_lastTPHitDirection < 0); // reverse direction
   double lots = CalculateLotSize(InpReversal_SL_Pips);
   long magicOff = isBuy ? MAGIC_REVERSAL_SINGLE_BUY : MAGIC_REVERSAL_SINGLE_SELL;
   string tag = isBuy ? "RevBuy" : "RevSell";

   if(ExecuteTradeWithRetry(isBuy, lots, InpReversal_SL_Pips, InpReversal_TP_Pips, magicOff, tag))
   {
      g_state = STATE_ReversalPositionsRunning;
      g_lastTPHitTime = 0; // reset, one reversal per TP event
      g_peakAfterTP = 0;
   }
}

//+------------------------------------------------------------------+
//|                   ENTRY SYSTEM 2: GRID                             |
//+------------------------------------------------------------------+
void PlaceGridOrdersOriginal(int signal)
{
   if(signal == 0) return;
   if(!InpSys2_EnableOriginalGrid) return;

   // Activation delay check
   if(!g_gridActivated)
   {
      if(g_gridActivationTime == 0)
      {
         g_gridActivationTime = TimeCurrent();
         g_gridDirection = signal;
         Print("Grid activation delay started. Direction: ", signal > 0 ? "BUY" : "SELL");
         return;
      }
      if(TimeCurrent() - g_gridActivationTime < InpGrid_ActivationDelay) return;
      g_gridActivated = true;
   }

   if(CountAllEAPositions() >= InpMaxPositionsTotal) return;
   if(CountPendings() >= InpMaxPendingsTotal) return;

   bool isBuy = (g_gridDirection > 0);
   double basePrice = isBuy ? SymbolInfoDouble(g_symbol, SYMBOL_ASK) : SymbolInfoDouble(g_symbol, SYMBOL_BID);
   double baseLots = CalculateLotSize(InpOriginal_SL_Pips);
   double totalLots = 0;
   long baseMagic = isBuy ? MAGIC_GRID_ORIG_BUY : MAGIC_GRID_ORIG_SELL;

   int existingGrid = CountPositions(baseMagic) + CountPendings(baseMagic);

   for(int level = existingGrid; level < InpGrid_MaxOrders; level++)
   {
      double levelLots = baseLots;
      if(InpGrid_LotDist == GRID_Martingale && level > 0)
         levelLots = baseLots * MathPow(InpGrid_LotMultiplier, level);
      else if(InpGrid_LotDist == GRID_CustomMultiplier && level > 0)
         levelLots = baseLots * MathPow(InpGrid_LotMultiplier, level);

      levelLots = NormalizeLots(levelLots);
      totalLots += levelLots;
      if(totalLots > InpGrid_MaxTotalLots) break;

      if(level == 0 && InpGrid_Mode == GRID_MarketPlusPending)
      {
         // First order is market
         ExecuteTradeWithRetry(isBuy, levelLots, InpOriginal_SL_Pips, InpOriginal_TP_Pips,
                               baseMagic, "Grid_L" + IntegerToString(level));
      }
      else
      {
         // Pending orders
         double stepPrice = PipsToPrice(InpGrid_StepPips * level);
         double orderPrice;
         bool isLimit = (InpGrid_OrderType == GRID_Limits);

         if(isBuy)
            orderPrice = basePrice - stepPrice; // limits below for buy
         else
            orderPrice = basePrice + stepPrice; // limits above for sell

         if(orderPrice <= 0) continue;

         PlacePendingOrder(isBuy, isLimit, orderPrice, levelLots,
                           InpOriginal_SL_Pips, InpOriginal_TP_Pips,
                           baseMagic, "Grid_L" + IntegerToString(level));
      }
   }

   g_state = STATE_OriginalPositionsRunning;
}

void PlaceGridOrdersReversal()
{
   if(!InpSys2_EnableReversalGrid) return;
   if(!InpReversal_Enable) return;
   if(g_lastTPHitTime == 0) return;
   if(TimeCurrent() - g_lastTPHitTime > InpReversal_MaxSecsAfterTP) return;

   bool isBuy = (g_lastTPHitDirection < 0); // opposite
   double basePrice = isBuy ? SymbolInfoDouble(g_symbol, SYMBOL_ASK) : SymbolInfoDouble(g_symbol, SYMBOL_BID);
   double baseLots = CalculateLotSize(InpReversal_SL_Pips);
   double totalLots = 0;
   long baseMagic = isBuy ? MAGIC_GRID_REV_BUY : MAGIC_GRID_REV_SELL;

   int existingGrid = CountPositions(baseMagic) + CountPendings(baseMagic);
   if(existingGrid >= InpGrid_MaxOrders) return;

   for(int level = existingGrid; level < InpGrid_MaxOrders; level++)
   {
      double levelLots = baseLots;
      if(InpGrid_LotDist == GRID_Martingale && level > 0)
         levelLots = baseLots * MathPow(InpGrid_LotMultiplier, level);
      else if(InpGrid_LotDist == GRID_CustomMultiplier && level > 0)
         levelLots = baseLots * MathPow(InpGrid_LotMultiplier, level);

      levelLots = NormalizeLots(levelLots);
      totalLots += levelLots;
      if(totalLots > InpGrid_MaxTotalLots) break;

      if(level == 0 && InpGrid_Mode == GRID_MarketPlusPending)
      {
         ExecuteTradeWithRetry(isBuy, levelLots, InpReversal_SL_Pips, InpReversal_TP_Pips,
                               baseMagic, "RevGrid_L" + IntegerToString(level));
      }
      else
      {
         double stepPrice = PipsToPrice(InpGrid_StepPips * level);
         double orderPrice;
         bool isLimit = (InpGrid_OrderType == GRID_Limits);

         if(isBuy)
            orderPrice = basePrice - stepPrice;
         else
            orderPrice = basePrice + stepPrice;

         if(orderPrice <= 0) continue;

         PlacePendingOrder(isBuy, isLimit, orderPrice, levelLots,
                           InpReversal_SL_Pips, InpReversal_TP_Pips,
                           baseMagic, "RevGrid_L" + IntegerToString(level));
      }
   }

   g_state = STATE_ReversalPositionsRunning;
   g_lastTPHitTime = 0;
   g_peakAfterTP = 0;
}

//+------------------------------------------------------------------+
//|                    EXIT SYSTEM 1: Direct TP/SL                     |
//+------------------------------------------------------------------+
void ManageExitSystem1()
{
   // Nothing to do - TP/SL are set on the orders themselves
   // Just monitor state transitions
   if(g_state == STATE_OriginalPositionsRunning || g_state == STATE_ReversalPositionsRunning)
   {
      if(CountAllEAPositions() == 0 && CountPendings() == 0)
      {
         g_state = STATE_CoolingDown;
         g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
      }
   }
}

//+------------------------------------------------------------------+
//|                 EXIT SYSTEM 2: Partial Close                       |
//+------------------------------------------------------------------+
bool IsTicketPartialClosed(long ticket)
{
   for(int i = 0; i < g_partialClosedCount; i++)
      if(g_partialClosedTickets[i] == ticket) return true;
   return false;
}

void MarkTicketPartialClosed(long ticket)
{
   g_partialClosedCount++;
   ArrayResize(g_partialClosedTickets, g_partialClosedCount);
   g_partialClosedTickets[g_partialClosedCount - 1] = ticket;
}

bool ClosePositionPartial(long ticket, double closePct)
{
   if(!PositionSelectByTicket(ticket)) return false;

   double volume = PositionGetDouble(POSITION_VOLUME);
   double closeVol = NormalizeLots(volume * closePct / 100.0);

   if(closeVol < g_lotMin) return false;
   if(closeVol >= volume) closeVol = NormalizeLots(volume - g_lotMin);
   if(closeVol < g_lotMin) return false;

   ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
   double price = (posType == POSITION_TYPE_BUY) ?
      SymbolInfoDouble(g_symbol, SYMBOL_BID) : SymbolInfoDouble(g_symbol, SYMBOL_ASK);

   g_trade.SetExpertMagicNumber((ulong)PositionGetInteger(POSITION_MAGIC));

   bool result = false;
   if(posType == POSITION_TYPE_BUY)
      result = g_trade.Sell(closeVol, g_symbol, price, 0, 0, InpTradeCommentPrefix + "_Partial");
   else
      result = g_trade.Buy(closeVol, g_symbol, price, 0, 0, InpTradeCommentPrefix + "_Partial");

   if(result)
      Print("Partial close OK: ticket=", ticket, " closed=", closeVol, " of ", volume);
   else
      Print("Partial close FAILED: ", g_trade.ResultRetcodeDescription());

   return result;
}

void MoveSLToBreakeven(long ticket)
{
   if(!PositionSelectByTicket(ticket)) return;

   double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
   double currentSL  = PositionGetDouble(POSITION_SL);
   double currentTP  = PositionGetDouble(POSITION_TP);
   ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);

   double bePrice;
   double offset = InpExit2_BEOffsetPoints * g_point;

   if(posType == POSITION_TYPE_BUY)
   {
      bePrice = NormalizePrice(entryPrice + offset);
      if(currentSL >= bePrice) return; // already at or above BE
   }
   else
   {
      bePrice = NormalizePrice(entryPrice - offset);
      if(currentSL > 0 && currentSL <= bePrice) return;
   }

   // Respect freeze level
   double currentPrice = (posType == POSITION_TYPE_BUY) ?
      SymbolInfoDouble(g_symbol, SYMBOL_BID) : SymbolInfoDouble(g_symbol, SYMBOL_ASK);
   double dist = MathAbs(currentPrice - bePrice) / g_point;
   if(dist < g_freezeLevel) return;

   g_trade.SetExpertMagicNumber((ulong)PositionGetInteger(POSITION_MAGIC));
   if(g_trade.PositionModify(ticket, bePrice, currentTP))
      Print("BE move OK: ticket=", ticket, " new SL=", bePrice);
   else
      Print("BE move FAILED: ", g_trade.ResultRetcodeDescription());
}

void ManageExitSystem2_Partials()
{
   if(!InpExit2_EnablePartial) { ManageExitSystem1(); return; }

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      long ticket = g_posInfo.Ticket();
      if(IsTicketPartialClosed(ticket)) continue;

      double entryPrice = g_posInfo.PriceOpen();
      double currentPrice = g_posInfo.PriceCurrent();
      double tp = g_posInfo.TakeProfit();
      double sl = g_posInfo.StopLoss();
      ENUM_POSITION_TYPE posType = g_posInfo.PositionType();

      // PROFIT-SIDE PARTIAL
      if(tp > 0)
      {
         double tpDist = MathAbs(tp - entryPrice);
         double triggerDist = tpDist * InpExit2_ActivPctOfTP / 100.0;
         double profitDist = 0;

         if(posType == POSITION_TYPE_BUY)
            profitDist = currentPrice - entryPrice;
         else
            profitDist = entryPrice - currentPrice;

         if(profitDist >= triggerDist && profitDist > 0)
         {
            if(ClosePositionPartial(ticket, InpExit2_PartialClosePct))
            {
               MarkTicketPartialClosed(ticket);
               if(InpExit2_MoveSLtoBE)
                  MoveSLToBreakeven(ticket);
            }
            continue;
         }
      }

      // LOSS-SIDE PARTIAL (only in ProfitAndLoss mode)
      if(InpExit2_Mode == E2_ProfitAndLoss && sl > 0)
      {
         double slDist = MathAbs(entryPrice - sl);
         double triggerDist = slDist * InpExit2_ActivPctOfSL / 100.0;
         double lossDist = 0;

         if(posType == POSITION_TYPE_BUY)
            lossDist = entryPrice - currentPrice;
         else
            lossDist = currentPrice - entryPrice;

         if(lossDist >= triggerDist && lossDist > 0)
         {
            if(ClosePositionPartial(ticket, InpExit2_PartialClosePct))
               MarkTicketPartialClosed(ticket);
         }
      }
   }

   // State transition
   if(CountAllEAPositions() == 0 && CountPendings() == 0)
   {
      g_state = STATE_CoolingDown;
      g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
   }
}

//+------------------------------------------------------------------+
//|               EXIT SYSTEM 3: PORTFOLIO CLOSE                       |
//+------------------------------------------------------------------+
void CloseAllEAPositions()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      g_trade.SetExpertMagicNumber((ulong)posMagic);
      if(!g_trade.PositionClose(g_posInfo.Ticket()))
         Print("Close failed: ticket=", g_posInfo.Ticket(), " ", g_trade.ResultRetcodeDescription());
   }
}

void CancelAllEAPendings()
{
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!g_ordInfo.SelectByIndex(i)) continue;
      if(g_ordInfo.Symbol() != g_symbol) continue;

      long ordMagic = g_ordInfo.Magic();
      if(ordMagic < InpMagicBase || ordMagic > InpMagicBase + 9999) continue;

      if(!g_trade.OrderDelete(g_ordInfo.Ticket()))
         Print("Cancel pending failed: ticket=", g_ordInfo.Ticket(), " ", g_trade.ResultRetcodeDescription());
   }
}

void CloseAllEAOrdersAndPositions()
{
   CloseAllEAPositions();
   CancelAllEAPendings();
}

double CalculatePortfolioProfitPips()
{
   double totalPips = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      double entry = g_posInfo.PriceOpen();
      double current = g_posInfo.PriceCurrent();
      ENUM_POSITION_TYPE posType = g_posInfo.PositionType();

      double pips = 0;
      if(posType == POSITION_TYPE_BUY)
         pips = PriceToPips(current - entry);
      else
         pips = PriceToPips(entry - current);

      totalPips += pips;
   }
   return totalPips;
}

double CalculatePortfolioProfitMoney()
{
   double total = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      total += g_posInfo.Profit() + g_posInfo.Swap() + g_posInfo.Commission();
   }
   return total;
}

double CalculateWeightedAverageSL()
{
   double totalWeightedSL = 0;
   double totalLots = 0;

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      double vol = g_posInfo.Volume();
      double entry = g_posInfo.PriceOpen();
      ENUM_POSITION_TYPE posType = g_posInfo.PositionType();

      double slPrice;
      if(posType == POSITION_TYPE_BUY)
         slPrice = entry - PipsToPrice(InpExit3_SLValue);
      else
         slPrice = entry + PipsToPrice(InpExit3_SLValue);

      totalWeightedSL += slPrice * vol;
      totalLots += vol;
   }

   if(totalLots == 0) return 0;
   return totalWeightedSL / totalLots;
}

void PortfolioPartialCloseAll(double closePct)
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!g_posInfo.SelectByIndex(i)) continue;
      if(g_posInfo.Symbol() != g_symbol) continue;

      long posMagic = g_posInfo.Magic();
      if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;

      ClosePositionPartial(g_posInfo.Ticket(), closePct);
   }
}

void ManageExitSystem3_Portfolio()
{
   int posCount = CountAllEAPositions();
   if(posCount == 0)
   {
      if(CountPendings() == 0)
      {
         g_state = STATE_CoolingDown;
         g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
      }
      return;
   }

   double portfolioProfit = 0;
   if(InpExit3_ProfitMode == E3_Money)
      portfolioProfit = CalculatePortfolioProfitMoney();
   else
      portfolioProfit = CalculatePortfolioProfitPips();

   // FULL PORTFOLIO TP HIT
   if(portfolioProfit >= InpExit3_TargetValue)
   {
      Print("Portfolio TP reached! Profit=", portfolioProfit, " Target=", InpExit3_TargetValue);
      CloseAllEAOrdersAndPositions();
      g_state = STATE_CoolingDown;
      g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
      g_portfolioPartialProfitDone = false;
      g_portfolioPartialLossDone = false;
      return;
   }

   // PORTFOLIO PARTIAL PROFIT
   if(InpExit3_EnablePartialProfit && !g_portfolioPartialProfitDone)
   {
      double triggerLevel = InpExit3_TargetValue * InpExit3_PartialProfitTrigPct / 100.0;
      if(portfolioProfit >= triggerLevel)
      {
         Print("Portfolio partial profit trigger at ", portfolioProfit);
         PortfolioPartialCloseAll(InpExit3_PartialProfitClosePct);
         g_portfolioPartialProfitDone = true;

         if(InpExit3_MoveSLtoBEAfterPartial)
         {
            for(int i = PositionsTotal() - 1; i >= 0; i--)
            {
               if(!g_posInfo.SelectByIndex(i)) continue;
               if(g_posInfo.Symbol() != g_symbol) continue;
               long posMagic = g_posInfo.Magic();
               if(posMagic < InpMagicBase || posMagic > InpMagicBase + 9999) continue;
               MoveSLToBreakeven(g_posInfo.Ticket());
            }
         }
      }
   }

   // AVERAGE SL CHECK
   if(InpExit3_SLMode == E3_AverageSL)
   {
      double portfolioLoss = 0;
      if(InpExit3_ProfitMode == E3_Money)
         portfolioLoss = -CalculatePortfolioProfitMoney();
      else
         portfolioLoss = -CalculatePortfolioProfitPips();

      if(portfolioLoss >= InpExit3_SLValue)
      {
         Print("Portfolio SL hit! Loss=", portfolioLoss, " Limit=", InpExit3_SLValue);
         CloseAllEAOrdersAndPositions();
         g_state = STATE_CoolingDown;
         g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
         return;
      }

      // PORTFOLIO PARTIAL LOSS
      if(InpExit3_EnablePartialLoss && !g_portfolioPartialLossDone)
      {
         double lossTrigger = InpExit3_SLValue * InpExit3_PartialLossTrigPct / 100.0;
         if(portfolioLoss >= lossTrigger)
         {
            Print("Portfolio partial loss trigger at ", portfolioLoss);
            PortfolioPartialCloseAll(InpExit3_PartialLossClosePct);
            g_portfolioPartialLossDone = true;
         }
      }
   }
}

//+------------------------------------------------------------------+
//|                 TP-HIT DETECTION VIA OnTradeTransaction            |
//+------------------------------------------------------------------+
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
   // Detect when a position is closed
   if(trans.type == TRADE_TRANSACTION_DEAL_ADD)
   {
      if(trans.symbol != g_symbol) return;

      // Check if it's a closing deal (entry type OUT)
      ENUM_DEAL_ENTRY dealEntry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(trans.deal, DEAL_ENTRY);
      if(dealEntry != DEAL_ENTRY_OUT) return;

      long dealMagic = HistoryDealGetInteger(trans.deal, DEAL_MAGIC);
      if(dealMagic < InpMagicBase || dealMagic > InpMagicBase + 9999) return;

      double dealProfit = HistoryDealGetDouble(trans.deal, DEAL_PROFIT);
      ENUM_DEAL_REASON reason = (ENUM_DEAL_REASON)HistoryDealGetInteger(trans.deal, DEAL_REASON);

      // Check if this was an original position closed at TP
      long magicOffset = dealMagic - InpMagicBase;
      bool isOriginal = (magicOffset == MAGIC_ORIGINAL_SINGLE_BUY ||
                         magicOffset == MAGIC_ORIGINAL_SINGLE_SELL ||
                         magicOffset == MAGIC_GRID_ORIG_BUY ||
                         magicOffset == MAGIC_GRID_ORIG_SELL);

      if(isOriginal && dealProfit > 0 && reason == DEAL_REASON_TP)
      {
         g_lastTPHitTime = TimeCurrent();
         g_lastTPHitPrice = HistoryDealGetDouble(trans.deal, DEAL_PRICE);

         ENUM_DEAL_TYPE dealType = (ENUM_DEAL_TYPE)HistoryDealGetInteger(trans.deal, DEAL_TYPE);
         // If closing deal is SELL, original was BUY (direction +1)
         g_lastTPHitDirection = (dealType == DEAL_TYPE_SELL) ? +1 : -1;
         g_peakAfterTP = g_lastTPHitPrice;

         g_state = STATE_OriginalTPHit;
         Print("TP HIT detected! Direction=", g_lastTPHitDirection,
               " Price=", g_lastTPHitPrice, " Time=", g_lastTPHitTime);
      }

      // If all EA positions are now closed
      if(CountAllEAPositions() == 0)
      {
         if(g_state != STATE_OriginalTPHit)
         {
            g_state = STATE_CoolingDown;
            g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
         }

         // Reset grid state
         g_gridActivated = false;
         g_gridActivationTime = 0;
         g_gridDirection = 0;

         // Cancel pendings if configured
         if(InpGrid_CancelOnExit)
            CancelAllEAPendings();
      }
   }
}

//+------------------------------------------------------------------+
//|                      STATE MACHINE LOGIC                           |
//+------------------------------------------------------------------+
void RunStateMachine(int signal)
{
   switch(g_state)
   {
      case STATE_Idle:
      {
         if(signal != 0)
         {
            g_state = STATE_OriginalSignalTriggered;
            // Fall through to execute
         }
         else break;
      }
      // FALLTHROUGH
      case STATE_OriginalSignalTriggered:
      {
         if(InpEntrySystemMode == ENTRY_System1_SingleReversal)
            PlaceOriginalEntries(signal);
         else
            PlaceGridOrdersOriginal(signal);

         if(CountAllEAPositions() > 0)
            g_state = STATE_OriginalPositionsRunning;
         else
            g_state = STATE_Idle;
         break;
      }

      case STATE_OriginalPositionsRunning:
      {
         // Managed by exit systems and OnTradeTransaction
         // Grid: replace missing pendings
         if(InpEntrySystemMode == ENTRY_System2_GridReversal && InpGrid_ReplaceMissing)
         {
            long buyMagic = MAGIC_GRID_ORIG_BUY;
            long sellMagic = MAGIC_GRID_ORIG_SELL;
            int gridCount = CountPositions(buyMagic) + CountPositions(sellMagic)
                          + CountPendings(buyMagic) + CountPendings(sellMagic);
            if(gridCount < InpGrid_MaxOrders && g_gridActivated && g_gridDirection != 0)
            {
               PlaceGridOrdersOriginal(g_gridDirection);
            }
         }
         break;
      }

      case STATE_OriginalTPHit:
      {
         // Transition to reversal eligible
         if(InpReversal_Enable &&
            ((InpEntrySystemMode == ENTRY_System1_SingleReversal && InpSys1_EnableReversal) ||
             (InpEntrySystemMode == ENTRY_System2_GridReversal && InpSys2_EnableReversalGrid)))
         {
            g_state = STATE_ReversalEligibleWindow;
         }
         else
         {
            g_state = STATE_CoolingDown;
            g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
         }
         break;
      }

      case STATE_ReversalEligibleWindow:
      {
         // Check if window expired
         if(g_lastTPHitTime > 0 && TimeCurrent() - g_lastTPHitTime > InpReversal_MaxSecsAfterTP)
         {
            g_state = STATE_CoolingDown;
            g_cooldownUntil = TimeCurrent() + InpCooldownSecondsAfterTrade;
            g_lastTPHitTime = 0;
            break;
         }

         if(InpEntrySystemMode == ENTRY_System1_SingleReversal)
            PlaceReversalEntries();
         else
            PlaceGridOrdersReversal();

         if(CountAllEAPositions() > 0 && g_state != STATE_ReversalEligibleWindow)
            g_state = STATE_ReversalPositionsRunning;
         break;
      }

      case STATE_ReversalPositionsRunning:
      {
         // Managed by exit systems and OnTradeTransaction
         break;
      }

      case STATE_CoolingDown:
      {
         if(TimeCurrent() >= g_cooldownUntil)
         {
            g_state = STATE_Idle;
            g_portfolioPartialProfitDone = false;
            g_portfolioPartialLossDone = false;
            // Clean partial tracking
            g_partialClosedCount = 0;
            ArrayResize(g_partialClosedTickets, 0);
         }
         break;
      }
   }
}

//+------------------------------------------------------------------+
//|                         MAIN OnTick                                |
//+------------------------------------------------------------------+
void OnTick()
{
   // Pre-checks
   if(IsDailyLossLimitHit())
   {
      static datetime lastDailyMsg = 0;
      if(TimeCurrent() - lastDailyMsg > 60) { Print("Daily loss limit hit. No new trades."); lastDailyMsg = TimeCurrent(); }
      // Still manage exits
      ManageExits();
      return;
   }
   if(IsWeeklyLossLimitHit())
   {
      static datetime lastWeeklyMsg = 0;
      if(TimeCurrent() - lastWeeklyMsg > 60) { Print("Weekly loss limit hit. No new trades."); lastWeeklyMsg = TimeCurrent(); }
      ManageExits();
      return;
   }

   // Spread filter (optional, does NOT block if disabled)
   if(InpMaxSpreadFilterEnabled)
   {
      double spread = SymbolInfoInteger(g_symbol, SYMBOL_SPREAD);
      if(spread > InpMaxSpreadPoints)
      {
         ManageExits();
         return;
      }
   }

   // One trade per bar check
   if(InpOneTradePerBar)
   {
      datetime barTime = iTime(g_symbol, g_timeframe, 0);
      if(barTime == g_lastBarTime && g_state == STATE_Idle)
      {
         ManageExits();
         return;
      }
      if(barTime != g_lastBarTime)
         g_lastBarTime = barTime;
   }

   // Detect signal
   int signal = 0;
   if(g_state == STATE_Idle || g_state == STATE_OriginalSignalTriggered)
      signal = DetectSignal();

   // Run state machine
   RunStateMachine(signal);

   // Manage exits
   ManageExits();
}

//+------------------------------------------------------------------+
//|                    EXIT MANAGEMENT ROUTER                          |
//+------------------------------------------------------------------+
void ManageExits()
{
   switch(InpExitSystemMode)
   {
      case EXIT_DirectTPSL:
         ManageExitSystem1();
         break;
      case EXIT_DirectWithPartial:
         ManageExitSystem2_Partials();
         break;
      case EXIT_PortfolioClose:
         ManageExitSystem3_Portfolio();
         break;
   }
}

//+------------------------------------------------------------------+
