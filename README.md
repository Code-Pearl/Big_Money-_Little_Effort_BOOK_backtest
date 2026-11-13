# LTIS - Long-Term Investment System

Strategy spec, pseudocode and backtest notes. Source: Mark Shipman, *Big Money, Little Effort* (Kogan Page, 2008). Implementation: `ltis_backtest.py`.
![image]([/ltis_output/GSPC_signal_close_chart.png) 

## 1. What the book contains

One mechanical strategy, the **Long-Term Investment System** ("the System", Ch.13-16), plus three ways to apply it and some built-in evidence to test against. Everything else (Ch.1-5, 10-12, appendices) is advice about advisers, psychology and the Turtles story. No further rules are hidden there.

| # | Strategy / use | Book location | In the script |
| --- | --- | --- | --- |
| A | Stock-market **index timing**: all-in index or all-in cash | Ch.14 p.75, Ch.16 p.91-97 | Menu 1, 4, 5 |
| B | Same rules on an **individual stock or sector** weekly chart | Ch.16 p.97 | Menu 6 |
| C | Index signal as a **bull/bear filter** for stocks ("a rising tide lifts all boats") | Ch.16 p.97 | Menu 7 (flagged interpretation) |

Built-in evidence the book supplies for testing: the published S&P 500 trade list 1951-2007 (Ch.18, 21 positions), the buy-and-hold vs System worked example 1995-2007 (Ch.9), and the 1987 crash behaviour (Ch.19).

## 2. The rules (Ch.14, p.75-76)

- Use **weekly** bars. Moving averages are simple averages of weekly **closing** prices.
- **BUY** signal is active whenever the 30-week MA is higher than the 50-week MA.
- **SELL** signal is active whenever the 50-week MA is higher than the 30-week MA.
- The System is always in exactly one of two states. BUY = invested in the stock market index (tracker or ETF). SELL = 100% in an instant-access interest-bearing savings account.
- Already invested and BUY signal again: do nothing. Already in cash and SELL signal again: do nothing.
- Beginning to operate while in BUY mode: invest immediately (p.95).
- Analyse **after** the weekly close (book does it Saturday/Sunday), act when the market reopens at the beginning of the next week (p.85).
- No stops, no extra filters, no position sizing. Ch.11 says beyond 2-4 rules performance degrades.
- Markets the book applies it to (Ch.15, 17): S&P 500, Dow Jones, NASDAQ-100, FTSE 100, DAX 30, CAC 40, Nikkei 225, Hang Seng.

## 3. Pseudocode

```
load daily OHLC for ticker                       # cached on disk
weekly = for each Mon..Sun week: last session's close
drop the unfinished week                         # analyse only after the weekly close
ma30 = SMA(weekly.close, 30)
ma50 = SMA(weekly.close, 50)

for each weekly bar t with date >= START and ma50 available:
    if ma30 > ma50:  state = BUY
    elif ma50 > ma30: state = SELL
    else:             state = previous state
    want_invested = (state == BUY)
    if want_invested != invested:                # act on a state CHANGE only
        if EXEC_MODE == next_open:    fill at OPEN of first session after t
        if EXEC_MODE == signal_close: fill at CLOSE of bar t
        invested = want_invested

daily equity:
    in cash   -> equity *= (1 + CASH_RATE) ^ (calendar days / 365)
    invested  -> equity = shares * close   (all-in, fractional shares, full compounding)
    costs     -> commission + slippage on every fill, ANNUAL_FEE drag while invested

open position at END -> mark to market at last close (the book does this, p.144)

report: book-style trade statistics (Ch.7 list) + equity curve + drawdown
        vs buy-and-hold started on the same bar with the same capital
```

## 4. Decisions where the book is silent

The book never specifies these. Each choice is configurable at the top of the script (or in Menu 8) and the script prints the active values in every report.

| Gap | Choice made | Why |
| --- | --- | --- |
| 30-wk MA exactly equals 50-wk MA | Keep previous state | Book allows only two states; ties are vanishingly rare |
| Fill price | Default **next\_open**. Alternative **signal\_close** | p.85 says act at next week's open. But every date in the published record is a weekly close, so the validation menu forces signal\_close |
| Cash interest | 4% p.a. default, calendar-day accrual | The only rate the book uses (p.53). Set 0 to isolate the timing effect |
| Costs | Commission, slippage, fee all 0 by default | The book's record is gross. Tracker fee is quoted at about 0.5%/yr (p.90): set `ANNUAL_FEE = 0.005` |
| Dividends | Price-only | Same basis as the published trade list. DAX on Yahoo and in the book is a total-return index (p.115), so its numbers include dividends |
| History before START | Loaded for MA warm-up | Need 50 weekly bars before the first signal. Short-history indices start once the MAs exist |
| Open position at the end | Marked to market, counted in the statistics | Exactly how the book treats the final position |
| Unfinished current week | Dropped | p.85: analysis before the weekly close is inaccurate |
| Saturday sessions pre-1953 | Week bins end Sunday, so Saturday stays in its own week | Book dates include Saturdays (6 Jan 1951) |
| Stock + market gate (Menu 7) | Stock held only if its own signal AND the index signal are both BUY | Book describes the idea but gives no rule. Use Menu 6 for book-exact stock rules |

## 5. Book evidence to test against

### 5.1 Published S&P 500 record, Ch.18 p.124-147

Every entry and exit date in the book falls on the last session of a week, and profits equal close/open minus 1 (20.77 to 24.75 = 19.16%). That is why the validation menu uses `signal_close`.

| # | Open | Price | Close | Price | Result |
| --- | --- | --- | --- | --- | --- |
| 1 | 6 Jan 1951 | 20.77 | 14 Aug 1953 | 24.75 | +19.16% |
| 2 | 5 Mar 1954 | 26.25 | 8 Feb 1957 | 44.53 | +69.64% |
| 3 | 6 Sep 1957 | 45.44 | 13 Dec 1957 | 40.92 | -9.95% |
| 4 | 25 Jul 1958 | 46.33 | 4 Mar 1960 | 56.12 | +21.13% |
| 5 | 30 Dec 1960 | 57.52 | 29 Jun 1962 | 52.45 | -8.81% |
| 6 | 15 Mar 1963 | 65.51 | 22 Jul 1966 | 86.99 | +32.79% |
| 7 | 28 Apr 1967 | 92.62 | 10 May 1968 | 98.35 | +6.19% |
| 8 | 21 Jun 1968 | 100.13 | 11 Jul 1969 | 99.03 | -1.10% |
| 9 | 29 Jan 1971 | 95.28 | 10 Dec 1971 | 96.51 | +1.29% |
| 10 | 24 Mar 1972 | 107.59 | 13 Jul 1973 | 102.14 | -5.07% |
| 11 | 23 May 1975 | 90.53 | 7 Apr 1977 | 98.23 | +8.51% |
| 12 | 25 Aug 1978 | 103.89 | 12 Apr 1979 | 102.87 | -0.98% |
| 13 | 27 Jul 1979 | 101.59 | 21 Aug 1981 | 131.66 | +29.60% |
| 14 | 12 Nov 1982 | 142.12 | 23 Mar 1984 | 159.18 | +12.00% |
| 15 | 30 Nov 1984 | 166.90 | 15 Jan 1988 | 243.46 | +45.87% |
| 16 | 2 Sep 1988 | 259.69 | 29 Jun 1990 | 355.43 | +36.87% |
| 17 | 7 Sep 1990 | 322.56 | 19 Oct 1990 | 300.03 | -6.98% |
| 18 | 10 May 1991 | 380.78 | 15 Jul 1994 | 449.56 | +18.06% |
| 19 | 6 Jan 1995 | 459.21 | 5 Jan 2001 | 1280.21 | +178.78% |
| 20 | 13 Jun 2003 | 987.76 | 5 Nov 2004 | 1130.20 | +14.42% |
| 21 | 23 Dec 2004 | 1194.20 | 6 Jun 2007 (open) | 1517.38 | +27.06% |

Book summary (p.147): 21 positions, 15 profitable (71.43%), largest gain 178.78%, average profit 34.76%, largest loss -9.95%, average loss -5.48%, ratio 6.3 to 1. The self-test re-derives all of these from the table above and they reconcile, including counting the open position as a winner.

### 5.2 Buy-and-hold vs System, Ch.9 p.51-56

S&P 500, January 1995 to June 2007, 10,000 invested, 4% on cash. Book result: System about 45,000 (+350%) vs buy-and-hold 33,260 (+230%). Buy-and-hold is simply 10,000 x 1,530 / 460. The System exits 5 Jan 2001 near 1,280, re-enters mid-2003 near 990, exits Nov 2004 at 1,130, re-enters Dec 2004 near 1,195. Menu 3 reruns this and prints both sets of numbers.

### 5.3 1987 crash, Ch.19 p.149

The System entered 1987 in BUY mode and stayed in BUY through the October crash (index down about 36% peak to trough), finishing the year with a small gain. To check: Menu 8, set start 1987-01-01 and end 1987-12-31, run Menu 1 on S&P 500. The trade list should show one position held all year.

### 5.4 Individual stock examples, Ch.16 p.97

| Stock | Window | Book claim | Status in script |
| --- | --- | --- | --- |
| British Airways | Jul 1991 - Aug 1994 | +140% | Not on Yahoo (delisted before 2011). Cannot replicate |
| Sears Roebuck | Mar 1995 - Dec 1996 | +75% | Not on Yahoo (pre-2005 merger). Cannot replicate |
| BHP Billiton | Jun 2003 - Dec 2006 | +200% | Preset `BHP.L` (pence). Window opens a year early so the real cross is found |

## 6. Menu map

| Menu | Does what |
| --- | --- |
| 1 | One market, using your Settings. Trade list, book-style summary, strategy vs buy-and-hold table, 4-panel chart |
| 2 | Book validation: S&P 500 1951-01-01 to 2007-06-06, signal\_close, no costs. Pairs each published trade with the nearest of yours and flags OK / DIFF / MISSING / EXTRA |
| 3 | Ch.9 worked example vs buy-and-hold with the book's final figures printed beside yours |
| 4 | All 8 indices: comparison table and small-multiples equity chart |
| 5 | Weekly dashboard: current BUY/SELL state, MAs, gap %, date and age of last cross for every index |
| 6 | Individual stock/sector: presets or custom Yahoo ticker |
| 7 | Stock + market-regime gate (interpretation, flagged in the report) |
| 8 | Session settings: dates, capital, cash rate, fee, commission, slippage, exec mode, MA weeks |
| 9 | Cache: list, refresh all, delete |

Outputs per run go to `./ltis_output/`: `*_trades.csv`, `*_equity.csv`, `*_chart.png`. Price history is cached in `./ltis_cache/` as one CSV per ticker; refresh fetches only the missing tail once the cache is older than `CACHE_REFRESH_DAYS`.

## 7. Config knobs (top of the script)

`START_DATE`, `END_DATE`, `INITIAL_CAPITAL`, `CASH_RATE_ANNUAL`, `ANNUAL_FEE`, `COMMISSION_PCT`, `SLIPPAGE_PCT`, `EXEC_MODE`, `FAST_WEEKS` (30), `SLOW_WEEKS` (50), `USE_ADJ_CLOSE_FOR_STOCKS`, `CACHE_DIR`, `OUTPUT_DIR`, `CACHE_REFRESH_DAYS`, `SHOW_PLOTS`, `VALID_DATE_TOL_DAYS`, `VALID_PRICE_TOL_PCT`. Leave 30/50 alone for any run that is meant to represent the book.

## 8. Known limits and what is not yet verified

- **Not run against live Yahoo data.** The build environment had no access to Yahoo Finance. The engine was tested offline (`--selftest`): no look-ahead (a truncated run reproduces the earlier trades exactly), signal states match an independent 30/50 computation, equity reconciles to the product of trade returns, fills always land after the signal in next\_open mode, costs lower and cash interest raises results. Data download, cache and incremental refresh were tested with a mocked Yahoo response only. Run Menu 2 first on your machine.
- **Expect some DIFF rows in Menu 2.** The book used CSI cash data; Yahoo `^GSPC` can differ in older closes, and Yahoo has no Saturday sessions. The tolerances (5 days, 1.5%) are adjustable. A DIFF is a prompt to look, not proof of a bug. Systematic extra or missing trades would be a real signal that something in data or logic needs checking.
- **Opens before about 1962** on Yahoo index data are often equal to the close. In that case next\_open degrades to next-session close; the report says how many rows are affected.
- **Price index only.** No dividends, so results understate what a total-return tracker would give. DAX is the exception.
- **A trend-following system needs trends.** The book says so itself (p.44, p.147): sideways markets produce whipsaws and small losses. The trade table includes MAE and MFE per trade so you can see them.
- **Not investment advice.** This reproduces a published method for study and testing.
