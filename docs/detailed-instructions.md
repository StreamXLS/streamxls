# StreamXLS Detailed Instructions

This document provides comprehensive guidance for using StreamXLS RTD formulas in Excel. It expands on [quickstart.md](quickstart.md) with detailed contract specification conventions, examples for complex securities (options, futures), and references to official IBKR documentation.

---

## Table of Contents

1. [RTD Formula Structure](#rtd-formula-structure)
2. [Contract Specification Methods](#contract-specification-methods)
3. [Specifying Options](#specifying-options)
4. [Specifying Futures](#specifying-futures)
5. [Specifying Forex (CASH)](#specifying-forex-cash)
6. [Using ConID for Precision](#using-conid-for-precision)
7. [Specifying Combo/Spread Orders](#specifying-combospread-orders)
8. [Position Topics with Contract Filters](#position-topics-with-contract-filters)
9. [Multi-Account Position Aggregation](#multi-account-position-aggregation)
10. [Position Metadata Fields](#position-metadata-fields)
11. [Market Data Field Reference](#market-data-field-reference)
12. [Staging Orders (StageOrder)](#staging-orders-stageorder)
13. [Cell States and Lifecycle](#cell-states-and-lifecycle)
14. [Troubleshooting Common Issues](#troubleshooting-common-issues)
15. [IBKR Campus and External References](#ibkr-campus-and-external-references)

---

## RTD Formula Structure

The Excel RTD function has the following structure:

```
=RTD(ProgId, Server, Topic1, Topic2, Topic3, ...)
```

For StreamXLS:

- **ProgId**: `"Tws.Rtd"` (required)
- **Server**: Leave empty or use `""` (StreamXLS ignores this argument)
- **Topic1, Topic2, ...**: Contract specification and field (parsed by StreamXLS)

### The Server Argument (Second Parameter)

**Important:** StreamXLS **ignores the second argument** (the "Server" parameter in Excel's RTD function). All connection parameters must be specified in the topic strings (third argument onward).

**Correct usage:**

```excel
=RTD("Tws.Rtd",,"AAPL","BID")                                    ' Default: 127.0.0.1:7496
=RTD("Tws.Rtd",,"host=192.168.1.100","port=7497","AAPL","BID")   ' Custom host and port
=RTD("Tws.Rtd",,"127.0.0.1:4001","AAPL","BID")                   ' Compact host:port format
```

**Incorrect usage (will use defaults):**

```excel
=RTD("Tws.Rtd","port=7496","status","IsConnected")   ' WRONG: port=7496 is in server arg, ignored!
=RTD("Tws.Rtd","127.0.0.1:7497","AAPL","BID")        ' WRONG: connection string in server arg, ignored!
```

If you put connection parameters in the second argument instead of the topic strings, StreamXLS will **not** see them and will use the defaults (`127.0.0.1:7496`). This can cause:

- Connection to the wrong TWS instance
- Unexpected values (e.g., `IsConnected` showing `0` when you expected `1`)
- Market data from the wrong account

### Connection Parameters

Connection parameters can be specified anywhere in the topic strings (order doesn't matter):

| Parameter | Format | Description | Default |
|-----------|--------|-------------|---------|
| Host | `host=<ip-or-hostname>` | TWS/Gateway IP address or hostname | `127.0.0.1` |
| Port | `port=<port-number>` | TWS/Gateway socket port | `7496` |
| Client ID | `clientid=<number>` | API client ID (0-2147483646) | Auto-generated |

**Alternative port aliases:**

| Alias | Port | Description |
|-------|------|-------------|
| `paper` | 7497 | TWS paper trading |
| `gw` | 4001 | IB Gateway live |
| `gwpaper` | 4002 | IB Gateway paper |

### Connection String Formats

**Key-value format:**

```excel
=RTD("Tws.Rtd",,"host=192.168.1.100","port=4001","clientid=5","AAPL","BID")
```

**Compact format (`host:port` or `host:port:clientid`):**

```excel
=RTD("Tws.Rtd",,"192.168.1.100:4001","AAPL","BID")
=RTD("Tws.Rtd",,"192.168.1.100:4001:5","AAPL","BID")
```

> **Note:** Use colons (`:`) as separators in the compact format, NOT underscores (`_`).  
> `127.0.0.1:7497:12345` works ✓  
> `127.0.0.1_7497_12345` does NOT work ✗  
> The underscore format (`host_port_clientid`) is used internally for connection keys but is not accepted as input.

**Port alias format:**

```excel
=RTD("Tws.Rtd",,"paper","AAPL","BID")           ' Connects to 127.0.0.1:7497
=RTD("Tws.Rtd",,"gw","AAPL","BID")              ' Connects to 127.0.0.1:4001
=RTD("Tws.Rtd",,"gwpaper","AAPL","BID")         ' Connects to 127.0.0.1:4002
```

### Multiple Connections

You can connect to multiple TWS/Gateway instances simultaneously. Each unique `host:port:clientid` combination creates a separate connection:

```excel
=RTD("Tws.Rtd",,"port=7496","AAPL","BID")       ' Live TWS
=RTD("Tws.Rtd",,"port=7497","AAPL","BID")       ' Paper TWS
=RTD("Tws.Rtd",,"port=4001","AAPL","BID")       ' Gateway Live
```

Each cell can use different connection parameters to compare quotes or monitor multiple accounts.

### Environment Variable Override

The client ID can also be set via environment variable:

- `TWS_RTD_CLIENT_ID`: If set, overrides the `clientid=` parameter in formulas

---

## Contract Specification Methods

StreamXLS supports multiple syntaxes for specifying contracts. The choice depends on your needs:

### Method 1: Simple Symbol (Stocks Only)

For plain equities, just use the symbol:

```
=RTD("Tws.Rtd",,"AAPL","BID")
=RTD("Tws.Rtd",,"IBM","LAST")
=RTD("Tws.Rtd",,"MSFT","MarketPrice")
```

This uses default assumptions: `SecType=STK`, `Exchange=SMART`, `Currency=USD`.

> **There is no "positional arguments" form.** A formula like
> `=RTD("Tws.Rtd",,"AAPL","STK","NASDAQ","USD","LAST")` does **not** set SecType/Exchange/Currency by
> position — bare contract tokens are not interpreted positionally. Contract qualifiers beyond the
> symbol must use the `key=value` form below (or the compact notation). A bare non-field token in a
> formula with no recognized field fails loud with an `Unknown field` error rather than silently
> resolving the `STK`/`SMART`/`USD` defaults.

### Method 2: Key-Value Pairs (Recommended for Complex Securities)

Use `key=value` syntax for explicit, self-documenting formulas:

```
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","BID")
=RTD("Tws.Rtd",,"sym=ES","sec=FUT","exch=CME","exp=202503","LAST")
```

**Common keys** (case-insensitive) — the complete set of contract keys, with every alias and the symbol-only defaults, is in [reference.md §3](reference.md#3-contract-specification-keys):

| Key | Aliases | Description |
|-----|---------|-------------|
| `sym` | `symbol` | Underlying symbol |
| `sec` | `sectype`, `securitytype` | Security type (STK, OPT, FUT, etc.) |
| `exch` | `exchange` | Exchange name |
| `cur` | `curr`, `currency` | Currency (USD, EUR, GBP, etc.) |
| `exp` | `expiry`, `expiration`, `lasttradedate` | Expiration date (YYYYMMDD format) |
| `strike` | `strikeprice` | Option strike price |
| `right` | `putcall`, `optiontype` | Option right (C or P) |
| `conid` | `contractid` | TWS Contract ID |

### Method 3: Compact Notation

Use `@` for exchange and `/` as delimiter:

```
=RTD("Tws.Rtd",,"AAPL@NASDAQ/STK/USD","BID")
=RTD("Tws.Rtd",,"ES@CME/FUT/202503/USD","LAST")
=RTD("Tws.Rtd",,"SPY@SMART/OPT/20251219/C/680/USD","BID")
```

**Format:** `SYMBOL@EXCH/PRIMEXCH/SECTYPE/EXP/RIGHT/STRIKE/CURRENCY` (omit trailing segments you don't need)

Other supported compact shorthand forms:

- **FX cash pairs:**

  ```
  =RTD("Tws.Rtd",,"EUR.USD/CASH","BID")
  ```

SymbolsCsv returns these same compact strings (stock symbols may be returned as just the symbol).

### Method 4: ConID (Most Precise)

Use TWS Contract ID for unambiguous identification:

```
=RTD("Tws.Rtd",,"conid=265598","BID")           // SPY
=RTD("Tws.Rtd",,"conid=643929299","BID")        // Specific option contract
```

**When to use ConID:**

- Options with similar strikes (e.g., mini options)
- When you have the ConID from TWS or another system
- For absolute precision in trading systems

**Finding the ConID:**

1. In TWS, right-click a contract → Financial Instrument Info → Description
2. The ConID is shown in the Contract Description dialog
3. Programmatically via `reqContractDetails` API call

---

## Specifying Combo/Spread Orders

Combo orders (also called spreads or BAG orders) allow you to define multi-leg option strategies, futures spreads, or other multi-contract combinations as a single instrument.

### Combo Leg Format

Each combo leg is specified with the format:

```
conid#ratio#action#exchange
```

| Field | Description | Example |
|-------|-------------|---------|
| `conid` | Contract ID of the leg | `265598` (SPY) |
| `ratio` | Number of contracts in this leg | `1`, `2`, etc. |
| `action` | BUY or SELL | `BUY`, `SELL` |
| `exchange` | Exchange for this leg | `SMART`, `ISE`, etc. |

Multiple legs are separated by semicolons (`;`).

### Combo Formula Pattern

```
=RTD("Tws.Rtd",,"sym=<UNDERLYING>","sec=BAG","cmb=<CONID1>#<RATIO>#<ACTION>#<EXCH>;<CONID2>#<RATIO>#<ACTION>#<EXCH>","<FIELD>")
```

**Note:** Security type must be `BAG` for combo orders.

### Examples: Option Spreads

**Bull Call Spread (buy lower strike, sell higher strike):**

```
=RTD("Tws.Rtd",,"sym=SPY","sec=BAG","cmb=643929299#1#BUY#SMART;643929301#1#SELL#SMART","BID")
```

**Iron Condor (4 legs):**

```
=RTD("Tws.Rtd",,"sym=SPY","sec=BAG","cmb=100001#1#SELL#SMART;100002#1#BUY#SMART;100003#1#SELL#SMART;100004#1#BUY#SMART","ASK")
```

(Replace 100001-100004 with actual ConIDs for the option legs)

### Examples: Futures Spread

**Calendar spread (sell front month, buy back month):**

```
=RTD("Tws.Rtd",,"sym=ES","sec=BAG","exch=CME","cmb=123456#1#SELL#CME;123457#1#BUY#CME","LAST")
```

### Finding ConIDs for Combo Legs

To construct a combo, you need the ConID for each leg:

1. **From TWS:** Right-click the contract → Financial Instrument Info → Description, and note the ConId.
2. **From a position you already hold:** read the `ConID` metadata field on the `position` topic,
   or the `ConIdCsv` field on the `positions` list topic.

### Important Notes

- The `cmb=` parameter is preserved as a single token and will not be split on semicolons
- Combo legs are executed as a single unit (all-or-none by default)
- Leg ratios allow unbalanced spreads (e.g., 2:1 ratio spreads)
- Some exchanges have specific requirements for combo legs

---

## Specifying Options

Options require additional parameters: expiration, strike, and right (call/put).

### Option Formula Pattern

```
=RTD("Tws.Rtd",,"sym=<UNDERLYING>","sec=OPT","exp=<YYYYMMDD>","strike=<STRIKE>","right=<C|P>","<FIELD>")
```

### Examples: SPY Options

**SPY $680 Call expiring December 19, 2025:**

```
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","BID")
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","ASK")
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","LAST")
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","MarketPrice")
```

**SPY $600 Put expiring January 17, 2025:**

```
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=600","right=P","exp=20250117","BID")
```

### Options: Additional Fields

Options have special fields for Greeks and implied volatility, exposed **per computation source**. Field names compose as `<Source><Measure>` where the source is `Bid`, `Ask`, `Last`, or `Model` — a bare measure name (e.g. `Delta` on its own) is not a valid field and errors loudly:

| Measure | Description |
|-------|-------------|
| `ImpliedVol` | Implied volatility |
| `Delta` | Option delta |
| `Gamma` | Option gamma |
| `Theta` | Option theta |
| `Vega` | Option vega |
| `OptPrice` | Model price |
| `PvDividend` | Present value of dividends |
| `UndPrice` | Underlying price used in calculation |

#### Example: Get SPY option Greeks

```
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","ModelDelta")
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","BidImpliedVol")
```

### Option Expiration Formats

The `exp` parameter accepts YYYYMMDD format:

- `20251219` = December 19, 2025
- `20250117` = January 17, 2025
- `20260116` = January 16, 2026

**Note:** Monthly options typically expire the third Friday; weekly options expire each Friday. Use the exact expiration date from TWS.

---

## Specifying Futures

Futures require the expiration month (and sometimes exchange).

### Futures Formula Pattern

```
=RTD("Tws.Rtd",,"sym=<SYMBOL>","sec=FUT","exch=<EXCHANGE>","exp=<YYYYMM>","<FIELD>")
```

### Examples: E-mini S&P 500 (ES)

**March 2025 ES future:**

```
=RTD("Tws.Rtd",,"sym=ES","sec=FUT","exch=CME","exp=202503","BID")
=RTD("Tws.Rtd",,"sym=ES","sec=FUT","exch=CME","exp=202503","LAST")
=RTD("Tws.Rtd",,"sym=ES","sec=FUT","exch=CME","exp=202503","MarketPrice")
```

### Examples: Micro E-mini (MES)

```
=RTD("Tws.Rtd",,"sym=MES","sec=FUT","exch=CME","exp=202503","BID")
```

### Continuous Futures

For continuous contract (front month), use the local symbol pattern:

```
=RTD("Tws.Rtd",,"loc=ESH5","sec=FUT","exch=CME","BID")
```

Where `ESH5` = ES March 2025 (H=March, 5=2025).

---

## Specifying Forex (CASH)

Forex pairs use `sec=CASH` and the IDEALPRO exchange:

### Forex Formula Pattern

```
=RTD("Tws.Rtd",,"sym=<BASE>","sec=CASH","exch=IDEALPRO","cur=<QUOTE>","<FIELD>")
```

### Examples

**EUR/USD:**

```
=RTD("Tws.Rtd",,"sym=EUR","sec=CASH","exch=IDEALPRO","cur=USD","BID")
=RTD("Tws.Rtd",,"sym=EUR","sec=CASH","exch=IDEALPRO","cur=USD","ASK")
```

**GBP/USD:**

```
=RTD("Tws.Rtd",,"sym=GBP","sec=CASH","exch=IDEALPRO","cur=USD","BID")
```

**USD/JPY:**

```
=RTD("Tws.Rtd",,"sym=USD","sec=CASH","exch=IDEALPRO","cur=JPY","BID")
```

---

## Using ConID for Precision

### When ConID is Recommended

1. **Options:** Many options have similar parameters; ConID is unambiguous
2. **Weekly vs Monthly options:** Same strike/expiry week but different contracts
3. **Trading systems:** Eliminate parsing ambiguity
4. **After obtaining from TWS:** If you have the ConID, use it

### Finding ConIDs

**In TWS:**

1. Right-click the contract row
2. Select "Financial Instrument Info" → "Description"
3. Note the "ConId" field

**From the RTD Server:**
Use position metadata if you hold the position (the contract is a single token — semicolon-joined
`key=value`, or compact notation):

```
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C;exp=20251219","ConID")
```

### ConID Examples

```
=RTD("Tws.Rtd",,"conid=265598","BID")           // SPY ETF
=RTD("Tws.Rtd",,"conid=8314","BID")             // IBM
=RTD("Tws.Rtd",,"conid=265529","BID")           // AAPL
=RTD("Tws.Rtd",,"conid=643929299","BID")        // SPY option
```

---

## Position Topics with Contract Filters

Position topics support contract filtering to specify which position you want.

### Basic Position Query

```
=RTD("Tws.Rtd",,"position","U1234567","AAPL","Position")
=RTD("Tws.Rtd",,"position","U1234567","AAPL","MarketValue")
```

### Position with Contract Details

For options or futures in your portfolio, the contract is a **single token** — join the
`key=value` qualifiers with semicolons (`;`), or use the compact `@`/`/` notation:

```
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C;exp=20251219","Position")
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C;exp=20251219","MarketValue")
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C;exp=20251219","UnrealizedPNL")
```

(A position cell exposes size, cost, value, and PnL — not a live price. For the option's
`MarketPrice`/`BID`/`ASK`, use the market-data topic: `=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","MarketPrice")`.)

### Position by ConID

```
=RTD("Tws.Rtd",,"position","U1234567","conid=643929299","Position")
=RTD("Tws.Rtd",,"position","U1234567","conid=643929299","MarketValue")
```

---

## Multi-Account Position Aggregation

You can aggregate positions across multiple accounts.

**What gets aggregated:**

- `Position`: Sum of shares across matched accounts
- `AverageCost`: Weighted average cost based on position sizes
- `MarketValue`: Sum of market values
- `DailyPNL`: Sum of daily P&L
- `RealizedPNL`: Sum of realized P&L
- `UnrealizedPNL`: Sum of unrealized P&L

(A live per-share price is **not** a position field — for a quote, query the market-data topic
for the contract, e.g. `=RTD("Tws.Rtd",,"AAPL","MarketPrice")`.)

### Single Account

```
=RTD("Tws.Rtd",,"position","U1234567","AAPL","Position")
```

### All Accounts

Use `*` or leave account blank:

```
=RTD("Tws.Rtd",,"position","*","AAPL","Position")
=RTD("Tws.Rtd",,"position",,"AAPL","Position")
```

### Specific Account List

Comma-separated account codes:

```
=RTD("Tws.Rtd",,"position","U1234567,U8901234","AAPL","Position")
```

---

## Position Metadata Fields

Position topics expose contract metadata from the matched position — `ConID`, `Symbol`, `SecType`, `Strike`, `Right`, `Expiry`, `Exchange`, `PrimaryExch`, `LocalSymbol`, `TradingClass`, `Currency`, and `Multiplier`. The complete position field set (value fields plus contract metadata, with every alias) is in [reference.md §8](reference.md#8-position-list-and-option-definition-fields).

### Example: Get Option Position Metadata

The contract is a **single token** (semicolon-joined `key=value`, or compact notation):

```
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C","ConID")
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C","LocalSymbol")
=RTD("Tws.Rtd",,"position","U1234567","sym=SPY;sec=OPT;strike=680;right=C","Expiry")
```

---

## Market Data Field Reference

Market-data fields are requested with a contract and a field name, e.g. `=RTD("Tws.Rtd",,"AAPL","BID")`. Field names are case-insensitive. The **complete** set — core price/size, odd-lot quotes, derived prices, data-tier indicators, option-computation greeks, generic ticks, and delayed variants, plus the per-family disconnect behavior — is enumerated in [reference.md §2](reference.md#2-market-data-fields).

The most-used fields — the complete set is in [reference.md §2](reference.md#2-market-data-fields):

| Field | Description |
|-------|-------------|
| `BID` / `ASK` | Best bid / ask price |
| `BIDSIZE` / `ASKSIZE` | Size at the best bid / ask |
| `LAST` | Last traded price |
| `LASTSIZE` | Last trade size |
| `VOLUME` | Session cumulative volume |
| `OPEN` / `HIGH` / `LOW` / `CLOSE` | Session open / high / low / prior close |
| `MarketPrice` | Mid (BID+ASK)/2 when both present, else LAST, else CLOSE |
| `LastOrClose` | LAST → CLOSE precedence |
| `MarketDataType` | Market-data tier TWS is actually serving this contract (1-4) |
| `IsDelayed` | 1 when the served tier is delayed (3/4), 0 when real-time/frozen (1/2) |

Option greeks are exposed **per computation source** as `<Source><Measure>` — Source is `Bid`, `Ask`, `Last`, or `Model`; Measure is `ImpliedVol`, `Delta`, `Gamma`, `Theta`, `Vega`, `OptPrice`, `PvDividend`, or `UndPrice` — e.g. `ModelDelta`, `BidImpliedVol`. See [Specifying Options](#specifying-options) for worked examples; the full greek and generic-tick lists are in [reference.md §2](reference.md#2-market-data-fields).

To enumerate an option chain on an underlying, use `OptionExpirationsCSV`, `OptionStrikesCSV`, and `StrikeStep` (see [reference.md §8](reference.md#8-position-list-and-option-definition-fields)).

### Market Data Requires a Modern TWS API (ServerVersion >= 206)

IBKR moved market data to a protobuf-only wire contract at negotiated **ServerVersion 206** (TWS API **10.38.01**, shipped 2025-07-08). A connection that negotiates a protocol **below 206** gets **zero market-data ticks** from a modern TWS server -- no error, no quotes -- while **orders, positions, account values, and PnL keep working** over their legacy paths.

StreamXLS detects this **per connection** (each socket negotiates its own protocol level) and **withholds market-data topics failing-loud** (an actionable error in the cell plus a logged error) so a silent blank isn't mistaken for "no subscription." Non-market-data topics are unaffected.

If your quotes are blank but positions/orders work, check this connection's diagnostics:

```
=RTD("Tws.Rtd",,"status","ServerVersion")       ' Negotiated protocol level (int), or "Not Connected"
=RTD("Tws.Rtd",,"status","MarketDataState")     ' "Ok" (>=206) / "TooOld" (1..205) / "Unknown" (0)
=RTD("Tws.Rtd",,"status","MarketDataMessage")   ' Actionable "update your TWS API" text when TooOld, else empty
```

(The bare token-free form also works, e.g. `=RTD("Tws.Rtd",,"ServerVersion")`.)

**Fix:** update the TWS API to the supported **>= 10.47.01** (the lowest release empirically verified-good for market data; StreamXLS does not support older versions). These status fields scope per-connection like other Status fields -- supply a `paper`/`gw`/`host=`/`port=` connection token, or omit to piggyback the single connection. See [Troubleshooting](#issue-market-data-farm-connection-is-ok-but-no-data) for details.

---

## Staging Orders (StageOrder)

StreamXLS supports staging orders via a dedicated RTD topic category: `StageOrder`. (`SendOrder` is an accepted synonym — both spellings parse identically. `StageOrder` is the canonical, docs-preferred name: every order is STAGED in TWS — never sent to market — and must be released by you in TWS.)

Important characteristics:

- Staging is a **side-effect of subscribing** to the topic (i.e., placing the formula in a cell).
- The topic returns a **status string**: `Sending` → `Staged`, then follows the order in TWS (`Submitted`, `Canceled`, `Filled`, ...) — or `SendOrder Error: ...`.
- If the formula is copied to multiple cells, each cell topic can stage an order. Use this deliberately.

### Basic Grammar

```excel
=RTD("Tws.Rtd",,"StageOrder","sym=AAPL","side=BUY","shares=100","type=LMT","limit=150.05","exch=SMART")
```

`StageOrder` uses `key=value` tokens; token order does not matter.

### Required Parameters

| Key | Aliases | Notes |
|-----|---------|-------|
| `sym` | `symbol` | Symbol (e.g., AAPL) |
| `side` | `action` | `BUY` or `SELL` |
| `shares` | `quantity`, `size`, `qty` | Integer quantity |
| `type` |  | `MKT`, `LMT`, `STP`, `STP LMT`, `TRAIL`, … (other IB order types as supported) |

Type-dependent requirements (each enforced with a loud error, never a silent default):

- `type=LMT` and `type=STP LMT` require `limit` > 0.
- `type=STP` and `type=STP LMT` require `stop` > 0.
- `type=TRAIL` requires exactly ONE of `stop` (the trailing amount) or `trailingpercent`.
- Conversely, `stop`/`trailingpercent` on a type that cannot use them (e.g. `MKT`) is rejected — TWS would
  silently ignore the value, which is exactly the silent mismatch StreamXLS refuses to stage.

### Optional Parameters (Common)

These are the most-used optional keys; the complete StageOrder key set is in [reference.md §4](reference.md#4-stageorder-write-keys).

| Key | Notes |
|-----|-------|
| `exch` | Exchange (defaults to `SMART` if omitted) |
| `account` | IB account code (sets `Order.Account`) |
| `fagroup` | FA group (sets `Order.FaGroup`) |
| `algo` | Algo strategy name (sets `Order.AlgoStrategy`) |
| `algoparams` | Encoded as `tag=value\|tag=value\|...` (sets `Order.AlgoParams`) |
| `tag` / `nonce` / `seq` / `submit` | Optional client tag used only for uniqueness/traceability. Composed into `Order.OrderRef` before the engine's identity token — the TWS-visible OrderRef reads `<your tag>\|SXLS:<token>` (your tag comes first, preserved verbatim; the `Order()` `ORDERREF` field reads back the bare tag). Use this to force a new StageOrder when staging the same parameters twice in a row. |
| `park` (synonyms `parked`, `saved`) | `TRUE`/`FALSE` (default FALSE). `park=true` stages the order as a local order-entry ticket (`Order.Transmit=false`, released with TWS's **Transmit** button): it is visible in the parking user's OWN TWS order list, but other TWS instances and the API (`reqAllOpenOrders`, the `Orders` topics) do not see it until it is transmitted. Default = a deactivated order visible in every instance's order list (released with **Submit**). See Return Values below for the full behavioral difference. |

### Order Attribute Keys (validated)

Beyond the required and common keys above, StageOrder accepts a full set of order-attribute keys — time-in-force (`tif`, `goodtilldate`, `goodaftertime`), `outsiderth`, `stop`/`trailingpercent`, `hidden`, `display`, `allornone`, `minqty`, and OCA (`ocagroup` + `ocatype`). **The complete, exhaustive key list — every alias, value grammar, and validation rule — is in [reference.md §4](reference.md#4-stageorder-write-keys).**

Every key is **validated at parse time** — an out-of-range or malformed value errors loudly in the cell
and nothing is staged. Unrecognized keys are rejected the same way (there is no silent drop and no
reflection pass-through: a key StreamXLS does not enumerate is an error, by design). Supplying two synonyms
of the same key (e.g. `stop=` and `aux=`) is also rejected rather than silently picking one.

> Whether a given attribute applies to a given order type/exchange (e.g. `hidden`, `display`, `minqty`) is
> enforced by TWS at staging/transmit time — TWS's validation errors surface in the cell. Order attributes not
> listed here (scale/hedge/delta-neutral/pegged families, regulatory fields, …) are deliberately not
> supported; if you need one, contact support — additions are trivial.
>
> Excel deduplicates RTD topics that have identical parameters. If you need to place back-to-back StageOrder requests with the same contract/price/size, include a unique `tag`/`nonce` value (e.g., `nonce=2`) so Excel issues a second subscription and StreamXLS can stage the additional order.

### Return Values

- `Sending`: topic registered and send scheduled
- `Staged`: the order was delivered to TWS in a deactivated/not-transmitted state and is awaiting your release there. (TWS sends no confirmation at staging time, so `Staged` means "delivered without error.") How the staged order presents in TWS is chosen **per formula** with the optional `park` key:
  - **Default (no `park`, or `park=false`):** the order appears in TWS order lists as a deactivated ("PreSubmitted") order with a **Submit** button, is visible to other TWS instances on the account, and survives a TWS restart. Click Submit in TWS to release it.
  - **`park=true`** (synonyms: `parked=true`, `saved=true`): a local order-entry ticket with a **Transmit** button. It is visible in the parking user's **own** TWS order list, but other TWS instances and the API (`reqAllOpenOrders`) — including the StreamXLS `Orders` topics — do not see it until it is transmitted. TWS assigns it a permanent id only when it is transmitted or discarded.
- After you act on the order in TWS, the cell follows TWS's own status reports: `PreSubmitted`/`Submitted` (working), `Filled`, `Inactive` (order held/rejected by TWS), or `Canceled` (shown for a discard of the staged order, a cancel of the working order, or removing the formula before the order was placed).
- `Error <code>: <message>` / `SendOrder Error: <message>`: TWS order error / validation/connection error. If the order is in fact still staged in TWS (e.g., a Transmit attempt failed for a fixable reason, like a missing account allocation on advisor setups), fixing and transmitting it in TWS updates the cell to the working status — TWS's report always wins.
- Note: deleting the formula after `Staged` does **not** cancel the staged order in TWS; it only stops the cell from tracking it.

> **Reopening a saved workbook does NOT re-stage `StageOrder` formulas.** Staging happens only when a formula is freshly entered (typed, edited, or written by a macro). When Excel reopens a saved workbook it re-subscribes each `StageOrder` cell, but StreamXLS recognizes the reopen (Excel marks re-subscriptions of saved formulas differently from fresh entries) and DISARMS the cell instead of staging: it shows `Disarmed: workbook reopen does not re-stage orders. Re-enter the formula to stage a new order; track existing orders with the orders topics.` To stage the same order again deliberately, re-enter the formula (select the cell, F2, Enter). Orders staged or transmitted in a prior session are unaffected by the reopen — track the live ones with the `orders`/`order` topics.
>
> ⚠️ **EDITING a staged order's parameters does NOT modify it — it stages a SECOND order.** Changing any token of a `StageOrder` formula that has already reached `Staged` (e.g. correcting `limit=150` to `limit=149`) makes Excel treat it as a new subscription: StreamXLS stages a new order at the corrected parameters and **leaves the original staged order untouched in TWS** (there is no modify/replace for a staged order). You will have two staged orders. **Cancel the old one in TWS** — do not rely on editing the cell to replace it.

---

## Cell States and Lifecycle

StreamXLS follows one rule everywhere: **it shows you what TWS delivers, where and when that
data is valid — and nothing else.** When data cannot be trusted, a cell fails *loud* (a visible
`#N/A` or a message), never quietly holding a stale number you might trade on. Wrong data is
worse than no data. This section explains exactly what each kind of cell shows as conditions
change: license, connection, order staging, and updates.

Throughout, "data cells" means market data, positions, account values, PnL, and order cells.
"Status and metadata cells" means the diagnostic topics — connection status, license status,
update notices, and the like.

### When your license or trial lapses mid-session

If your trial ends, a subscription lapses, or the license simply cannot be verified while Excel
is open, **data cells stop showing data and instead show a short license message** — a line of
*text*, for example:

- `StreamXLS trial expired — purchase at https://streamxls.com/buy; activate in StreamXLS Control Panel.`
- `StreamXLS license expired — renew at https://streamxls.com/buy; data resumes automatically after renewal (restart Excel to re-check immediately).`

This affects all data cells: market data, positions, account, PnL, and order cells, plus order
staging. **Status and metadata cells keep working** — connection status, `LICENSE_STATE`,
`LICENSE_MESSAGE`, and similar diagnostics still resolve, so you can always see *why* your data
stopped.

**The message replaces a value only when the cell next updates.** A cell shows the license
message the next time it receives a tick or re-resolves; a **quiet** cell — for example a quote
in a closed market that is not currently ticking — can keep displaying its last number until its
next update. Do not read "the old number is still there" as "the license is fine"; check
`LICENSE_STATE` for the authoritative answer.

**A brief license-server outage does not instantly blank your dashboard.** There is a difference
between a *definitive* result (the license genuinely expired or was revoked — data is withheld at
once) and an *indeterminate* one (the license **server is unreachable**, so entitlement cannot be
confirmed right now). On an indeterminate outage, a machine that was already verified keeps
serving data through a **bounded retention window (up to ~3 days / 72 hours)** measured from its
last confirmed check; only if the server is still unreachable past that window do the cells flip
to the license message. (A different failure — the local license component itself breaking, e.g.
quarantined by antivirus — carries a longer grace on a paid license.) A transient verification
hiccup therefore rides through without interrupting live data.

#### The formula hazard: text landing in numeric cells

The license message is **text**, and it lands in cells your formulas expect to be numbers. Excel
does not treat this as an error, so it will not propagate loudly the way `#N/A` does. Instead:

- **Comparisons flip silently.** In Excel, any text value is treated as *greater than any
  number*. A guard like `=IF(Price>100, "sell", "hold")` will see the license text in `Price`,
  evaluate the text as greater than 100, and quietly take the wrong branch.
- **Aggregates skip it silently.** `MAX`, `MIN`, `AVERAGE`, and `SUM` ignore text cells. An
  average across a column of quotes will simply compute over fewer points, with no error shown —
  the result looks plausible and is wrong.

If your workbook drives decisions off these cells, wrap price/quantity references so a
non-number is caught explicitly — e.g. `=IF(ISNUMBER(Price), …, …)` — rather than assuming a
lapse will surface as `#N/A`.

#### Recovery

- **Cells that were already live when the lapse happened repair themselves.** Once the license
  re-verifies (this happens automatically in the background), those cells resume showing data on
  the next refresh with no action from you. To force an immediate re-check, restart Excel.
- **Cells first entered *while* the license was withheld were never subscribed** — they will keep
  showing the license message until you re-enter the formula (F2, Enter) or reopen the
  workbook. (A plain recalculate — F9 — does **not** re-enter an RTD formula, so it will not clear
  a stuck cell.) If a cell is still stuck on the message after your license is valid again, re-enter it.

(During the very first seconds of a session, while the license is still being verified for the
first time, data cells briefly show `#N/A` rather than the message — a normal startup transient
that clears as soon as verification completes.)

### Disconnect and reconnect: what each cell family does

When the connection to TWS drops (or TWS reports a data-connectivity loss), cells fail loud
rather than freeze on old numbers. The exact behavior depends on the cell family. On reconnect,
StreamXLS re-requests everything automatically — **you do not need to touch your formulas.**

#### Market-data cells

The rules below describe a **socket disconnect** — the API↔TWS connection itself drops. A **TWS
data-connectivity-loss report (error 1100)** is stronger and is covered separately at the end of
this list.

- **Bid/Ask family → immediate `#N/A`.** `BID`, `ASK`, their sizes and exchanges, and the
  delayed and odd-lot variants all go to `#N/A` the moment the connection is down. A bid or ask
  that is not current is never shown as if it were live.
- **Last-trade cells depend on your market-data tier.** `LAST` (and `DELAYEDLAST`) go to `#N/A`
  on a socket disconnect **only** under the real-time and delayed tiers. Under the *frozen* tiers —
  including **DelayedFrozen, the default** — a frozen last price is an explicit opt-in to
  last-known values, so `LAST` **keeps its last-seen value** across a socket disconnect. So with
  default settings, `LAST` holds; if you configure a real-time or (non-frozen) delayed tier, it
  goes `#N/A`.
- **Everything else keeps its last value on a socket disconnect.** `CLOSE`, `OPEN`, `HIGH`,
  `LOW`, `VOLUME`, `MARKETPRICE`, `LASTORCLOSE`, and the rest retain their last-seen value when
  the socket drops.
- **Error 1100 (TWS lost its data connection to IB) blanks *every* market-data cell → `#N/A`.**
  When TWS reports a data-connectivity loss, the API↔TWS socket stays up (so `IsConnected` may
  still read `1`), but no quotes can be current — so StreamXLS flips **all** market-data topics to
  `#N/A`, **including the keep-last families above** (`CLOSE`/`OPEN`/`HIGH`/`LOW`/`VOLUME`/
  `MARKETPRICE`/`LASTORCLOSE` and a frozen `LAST`). This is deliberately stronger than a socket
  disconnect: nothing may keep serving a stale number while TWS is cut off from the market. Values
  repaint automatically when connectivity is restored (error 1101/1102).

#### Positions, account, and PnL cells

By default these go to **`#N/A` immediately** on disconnect (fail loud) — a balance or position
from before an outage is never shown as current. If you would rather keep the last-known values
on screen during an outage, set `TWS_RTD_PRESERVE_ON_DISCONNECT=true` (default is `false`). With
the knob on, cells hold their workbook-saved values until reconnect repaints them — except that a
position confirmed *gone* on reconnect still flips to `#N/A` regardless of the knob.

#### Order cells

Order and order-list cells ride the same `TWS_RTD_PRESERVE_ON_DISCONNECT` knob:

- **Default (`false`):** working (non-terminal) order rows go to `#N/A`; **terminal facts —
  Filled, Canceled, Inactive — are preserved** (they are completed realities TWS will not
  re-send).
- **Knob on (`true`):** cells keep their workbook-saved values until reconnect repaints them.

Staged-order cells (from `StageOrder`/`SendOrder`) behave the same way but **always fail loud
regardless of the knob**: on a disconnect, any cell still showing a non-terminal status
(`Sending`, `PreSubmitted`, `Submitted`, …) flips to `#N/A`, while terminal facts (Filled,
Canceled, Inactive) are preserved. This matters because IB could fill or cancel a transmitted
order during an uplink loss — the cell must not keep reading "Submitted" as if nothing changed.

#### Reconnect

On reconnect, StreamXLS re-subscribes market data, re-requests account/position/PnL values, and
re-queries open and completed orders — automatically. Each `#N/A` (or preserved) cell repaints
with fresh truth as soon as it arrives. No user action is required.

### Order staging lifecycle

StreamXLS **stages** orders; it never transmits them to the market on its own. A staged order
sits in TWS awaiting a human click there. The `StageOrder` formula (the preferred name;
`SendOrder` is an accepted synonym) drives one independent submission per cell.

#### What the cell shows, step by step

1. **`Sending`** — the moment the formula is entered, while the order is being placed.
2. **`Staged`** — once StreamXLS has handed the order to TWS without error.
3. **TWS's own status words, verbatim** — as the order progresses in TWS you'll see TWS's
   strings: `PreSubmitted`, `Submitted`, `Filled`, `Inactive`, and so on. (The one house spelling:
   TWS's `Cancelled`/`ApiCancelled` are shown as **`Canceled`**, the spelling StreamXLS has used
   since v1 so existing formulas keep matching.)

Never assume a status word not in TWS's vocabulary — these are TWS's own strings.

#### Two staging shapes: default vs. `park=true`

Choose per formula how the staged order appears in TWS:

- **Default (Submit shape):** the order appears in TWS's order lists as a **deactivated
  ("PreSubmitted")** order. It is visible to other TWS instances and survives a TWS restart. The
  TWS **Submit** button releases it to market in one click.
- **`park=true`:** a **local order-entry ticket** — visible in the parking user's **own** TWS
  order list (with a **Transmit** button), but not seen by other TWS instances or the API
  (`reqAllOpenOrders`, the `Orders` topics) until you transmit it. (`parked` and `saved` are
  accepted synonyms, for users arriving from CQG/thinkorswim vocabularies.)

Either way, the order cannot reach the market without your action in TWS.

#### Reopening a workbook does not re-stage

When you reopen a workbook, Excel re-subscribes your saved `StageOrder` formulas. StreamXLS does
**not** re-stage them — a reopen is not a deliberate order action. Each such cell shows:

> `Disarmed: workbook reopen does not re-stage orders. Re-enter the formula to stage a new order; track existing orders with the orders topics.`

To stage a fresh order, re-enter the formula. To track orders you already staged, use the order
topics.

#### Editing a staged formula stages a *second* order

Editing a `StageOrder` formula is a *fresh* order action (unlike a reopen). It stages a **new,
independent order** — the original staged order is **not** replaced or canceled; it remains in
TWS. If you edit a staged formula, expect two orders in TWS and cancel the unwanted one there.

#### Deleting a cell is not the same as canceling

Deleting or clearing a `StageOrder` cell tells StreamXLS to stop tracking it. StreamXLS attempts
a best-effort cancel **only if the order has not yet been staged** (i.e., it is still `Sending`).
**Once a cell reads `Staged` or later, deleting it does not cancel the order in TWS** — the order
is already live there. To cancel a staged order, act in TWS.

### What StreamXLS formulas cannot do

`StageOrder` is the only order-*writing* verb, and it can only *place* an order. **There is no
cancel, replace, or modify verb** — no StreamXLS formula can cancel or amend a working order.
Once an order is staged, every subsequent action (transmit, modify, cancel) happens **in TWS**.
This is deliberate: the workbook stages; the human in TWS decides.

`StageOrder` also stages **stock orders in USD only, one leg at a time**. Combination/spread
(multi-leg) orders cannot be staged. Contract keys such as `sec=`, `cur=`, `exp=`, `strike=`,
`right=`, and `conid=` are **rejected with a clear error** rather than silently ignored — so a
formula that tries to change the security type or build a combo fails loud instead of quietly
staging a plain stock order.

### Client ID (which API client StreamXLS connects as)

If you do not set `TWS_RTD_CLIENT_ID` (and pass no `clientid=` in a formula), StreamXLS picks a
**random client ID** for each connection, in the range **70,000 to ~2.14 billion** (kept above
70,000 so it is easy to tell apart from a port number).

- **It is not stable across Excel restarts.** A fresh random ID is chosen each session. If you
  need a *fixed* client ID — for example to reserve a "master" API client slot in TWS, or to
  reconcile against an order feed — set `TWS_RTD_CLIENT_ID` explicitly.
- **Two Excel instances almost never collide.** With a ~2.14-billion range, two auto-ID instances
  drawing the same number is astronomically unlikely. If you *pin* two instances to the **same**
  explicit ID, TWS rejects the second connection with error **326 (client ID already in use)** and
  StreamXLS retries. A pinned ID is never changed automatically (it reflects your intent); an
  auto ID may rotate to a new random value after a connection failure — including a single
  pre-handshake drop (TWS closing the socket before the handshake completes), a symptom of TWS
  rate-limiting a client ID. Rotation is rate-capped so it cannot thrash.

### Staying up to date

Updates are **offered, never forced.** A licensed copy may keep running an old build indefinitely.

- A small **per-user scheduled task** (no admin rights) checks for updates **daily at 03:00**, and
  the Control Panel also runs a quiet check on launch if the last one is stale — so a laptop
  asleep at 03:00 still catches up.
- The check writes a small local breadcrumb; the engine reads it **locally (no network call)** and
  surfaces it through the `UPDATE_AVAILABLE`, `UPDATE_CRITICAL`, `UPDATE_LATEST_VERSION`, and
  `UPDATE_MESSAGE` topics. These resolve as one-shot metadata topics *and* as live status fields
  (reached via the `status` token, e.g. `=RTD("Tws.Rtd",,"status","UPDATE_CRITICAL")`), so a
  critical flag that flips mid-session appears **without re-subscribing**.
- The engine composes the notice text itself from fixed phrasing — a plain `Update available
  (x.y.z).`, or a louder `IMPORTANT: a critical update (x.y.z) is available - install it before
  continuing.` for a critical update.
- To install, the Control Panel downloads the signed installer, verifies its signature, and — after
  **you confirm** — launches it and closes itself so the files can be replaced. Updates apply only
  when Excel is closed; an update never hot-swaps under you mid-session.

### Error display: MESSAGE vs. NA

`TWS_RTD_ERROR_DISPLAY` controls how **market-data error cells** appear:

- **`MESSAGE` (default):** a market-data error shows explanatory text, e.g. `RTD error: Requested
  market data is not subscribed`. This is the fail-loud default recommended for trading.
- **`NA`:** the same market-data error cells show a bare `#N/A` instead, for a cleaner display.

This setting changes **only market-data error cells** (TWS market-data error callbacks, and the
withhold when TWS's protocol is too old to stream market data). It does **not** change:

- **License-lapse cells** — always the license message text.
- **The `#N/A` disconnect placeholders** — Bid/Ask, positions/account/PnL/orders, and swept
  staged orders are always `#N/A`.
- **Formula/validation errors** — a bad `StageOrder` key, a missing contract, etc. always show
  `RTD error: …` text, so a malformed formula is never silently blanked.

---

## Troubleshooting Common Issues

### Issue: #N/A in Cell

**Possible causes:**

1. TWS not connected
2. No market data subscription for the security
3. Invalid contract specification
4. Market closed (for some tick types)
5. Expected during a TWS outage: by default (`TWS_RTD_PRESERVE_ON_DISCONNECT=false`) account, PnL, and position value cells show `#N/A` while disconnected rather than a stale number — this is the fail-loud default, not a fault.

**Solutions:**

1. Check `=RTD("Tws.Rtd",,"status","IsConnected")` → should be 1
2. Use `MarketPrice` instead of `LAST` for after-hours fallback
3. Verify contract parameters match TWS exactly
4. Check TWS for error messages
5. To keep the last-known account/PnL/position values on screen through an outage instead of `#N/A`, set `TWS_RTD_PRESERVE_ON_DISCONNECT=true` (Control Panel -> Settings -> "Preserve values on disconnect"). Values repaint when data returns; a cell that never had a value stays `#N/A`.

### Issue: "No security definition" Error

**Cause:** Contract specification doesn't match any IB contract.

**Solutions:**

1. Verify symbol spelling
2. Check expiration date format (YYYYMMDD for options, YYYYMM for futures)
3. Ensure strike price is exact (check for mini options)
4. Use ConID for precision

### Issue: "Market data farm connection is OK" but no data

**Possible causes:**

1. Market data subscription not active for the security
2. **TWS API too old for market data (ServerVersion < 206).** A connection that negotiates a protocol below 206 gets **zero market-data ticks** from a modern TWS while orders/positions still flow. This is now a primary cause of "no market data." Fix by installing the supported TWS API **>= 10.47.01**.
3. Some securities (e.g., pink sheets) may not have data

**Solutions:**

1. Check TWS Account Management for data subscriptions
2. Use delayed data: the default is `TWS_RTD_MARKET_DATA_TYPE=4` (DELAYED_FROZEN — delayed data with automatic fallback, plus frozen last-session values when the market is closed). Set `=3` for plain DELAYED without the frozen behavior. Note the type is what you *request*; TWS serves each contract at the best tier your subscriptions allow and reports it per contract — check `=RTD("Tws.Rtd",,"<contract>","IsDelayed")` (1 = delayed) or `"MarketDataType"` (1-4). Some venues offer no delayed data at all (error 354): those symbols get no quotes without a subscription regardless of the requested type.
3. **Check this connection's market-data diagnostics:**

   ```
   =RTD("Tws.Rtd",,"status","ServerVersion")       ' Negotiated protocol level, or "Not Connected"
   =RTD("Tws.Rtd",,"status","MarketDataState")     ' "Ok" / "TooOld" / "Unknown"
   =RTD("Tws.Rtd",,"status","MarketDataMessage")   ' Actionable message when TooOld, else empty
   ```

   If `MarketDataState` is `TooOld`, update the TWS API to **>= 10.47.01** — StreamXLS refuses to bind an older TWS API entirely, and market data is only served at 10.47.01 and above. StreamXLS withholds market-data topics fail-loud in this case, so an affected cell shows an error rather than going silently blank.

### Issue: My prices show "(delayed)" after the number / formulas referencing prices broke

**Cause:** `TWS_RTD_DELAYED_ANNOTATION=1` is enabled (StreamXLS Control Panel → Settings, or the env var), and TWS is serving those contracts delayed data (no market-data subscription). The annotation renders the delayed value as *text*, deliberately — so a 15-20-minute-old price cannot masquerade as live.

**Read this first — the silent failure mode:** Excel treats any text as **greater than any number**. On an annotated cell, `=IF(A1>threshold,...)` is always TRUE, `=IF(A1<stop,...)` never fires, and `MAX`/`MIN`/`AVERAGE`/`COUNTIF` silently skip the cell. Arithmetic (`+`, `SUM`) fails loud with `#VALUE!`. If you keep annotation on, route formulas through a strip: `=VALUE(SUBSTITUTE(A1," (delayed)",""))`.

**Solutions:**

1. Subscribe to real-time data for those symbols (TWS Account Management → Market Data Subscription Manager) — entitled symbols are never annotated.
2. Turn annotation off (Control Panel → Settings → "Delayed-data annotation" → Off, or `TWS_RTD_DELAYED_ANNOTATION=0`) and use the indicator fields instead: `=RTD("Tws.Rtd",,"<contract>","IsDelayed")` (1/0) or `"MarketDataType"` (1-4).
3. Keep annotation on and use the `VALUE(SUBSTITUTE(...))` strip in dependent formulas.

Note: an annotated (text) value can never place an order — a StageOrder pointed at one is rejected with a parse error.

### Issue: Position shows 0 but I have a position

**Cause:** Contract filter doesn't match.

**Solutions:**

1. Use broader filter (just symbol, not full contract spec)
2. Check security type matches (STK vs OPT)
3. Verify account code is correct
4. Use `positions` topic to see what symbols are active:

   ```
   =RTD("Tws.Rtd",,"positions",,"SymbolsCsv")
   ```

---

## IBKR Campus and External References

### Official IBKR Documentation

- **RTD Server Overview:** [IBKR Campus - RTD](https://www.interactivebrokers.com/campus/trading-lessons/downloading-ibkr-rtd/)
- **TWS API Reference:** [TWS API Documentation](https://interactivebrokers.github.io/tws-api/)
- **Contract Specification:** [TWS API - Contracts](https://interactivebrokers.github.io/tws-api/contracts.html)
- **Market Data Types:** [TWS API - Market Data Types](https://interactivebrokers.github.io/tws-api/market_data_type.html)

### Related Documentation

- [quickstart.md](quickstart.md) – Basic usage guide
- [Configuration-key reference](reference.md#11-configuration-keys) – Installation and logging
- [reference.md](reference.md) – Complete field and key reference
- [Market-data field reference](reference.md#2-market-data-fields) – Market data tick types and generic ticks

### Excel Sample Workbook

See `../examples/StreamXLS.xlsm` for a working example with all account fields listed.

### Diagnostics

If quotes or fields look wrong and the checks above don't explain it, contact support
(<support@streamxls.com>) — we can provide a standalone diagnostic script that probes TWS directly
for the market-data tick types a given contract returns.

---

## Quick Reference Card

### Stock

```
=RTD("Tws.Rtd",,"AAPL","BID")
```

### Option

```
=RTD("Tws.Rtd",,"sym=SPY","sec=OPT","strike=680","right=C","exp=20251219","BID")
```

### Future

```
=RTD("Tws.Rtd",,"sym=ES","sec=FUT","exch=CME","exp=202503","BID")
```

### Forex

```
=RTD("Tws.Rtd",,"sym=EUR","sec=CASH","exch=IDEALPRO","cur=USD","BID")
```

### By ConID

```
=RTD("Tws.Rtd",,"conid=265598","BID")
```

### Position

```
=RTD("Tws.Rtd",,"position","U1234567","AAPL","MarketValue")
```

### Account

```
=RTD("Tws.Rtd",,"account","U1234567","NetLiquidation")
```

### Status

```
=RTD("Tws.Rtd",,"status","IsConnected")
=RTD("Tws.Rtd",,"status","ConfigWarnings")  ' TWS_RTD_* configuration-validation warnings; empty when clean
=RTD("Tws.Rtd",,"status","UPDATE_CRITICAL") ' live update-notify field (re-resolves each heartbeat)
=RTD("Tws.Rtd",,"status","UPDATE_MESSAGE")  ' engine-composed update text; "" when up to date
```

> The four `UPDATE_*` fields exist BOTH as one-shot metadata topics (bare name, resolve at subscribe) AND as
> live status fields (the `status` token form above, re-resolve each heartbeat so a mid-session critical flip
> reaches an open cell). Same local breadcrumb, identical values; the bare name stays metadata for back-compat.
>
> **What `IsConnected` means:** it starts `0`, flips to `1` when a
> TWS API handshake is completed, and stays `1` as long as there is no indication that TWS has disconnected.
> **It does not guarantee a functioning TWS connection**: the TWS API socket has no protocol-level
> heartbeat/keepalive, so a TWS that dies silently (without closing the socket) can leave `IsConnected` at `1`
> until the next request or socket error surfaces the failure. Treat `IsConnected=0` as definitive
> ("not connected"); treat `IsConnected=1` as "connected as far as we can tell."

### Positions List

```
=RTD("Tws.Rtd",,"positions",,"SymbolsCsv")
```
