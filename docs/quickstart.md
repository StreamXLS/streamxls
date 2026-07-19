# StreamXLS for Excel

This RTD server exposes IBKR account, positions, orders, and market data to Excel. In addition to long-standing market data functionality, it now includes:

- Account values
- Positions with streaming PnL data
- Open Orders, plus the ability to send orders from Excel
- Production-grade multi-threading and connection management
- Safer build/registration workflow and [logging](reference.md#11-configuration-keys) for troubleshooting

## Usage in Excel

All topics and fields are case-insensitive.

## RTD Status

Examples:

```
=RTD("Tws.Rtd", , "status", "IsConnected")
=RTD("Tws.Rtd", , "status", "LastUpdateUtc")
=RTD("Tws.Rtd", , "status", "ActiveTopicCount")
=RTD("Tws.Rtd", , "status", "ServerHeartbeatUtc")
=RTD("Tws.Rtd", , "status", "LastOrderListChangeUtc")
=RTD("Tws.Rtd", , "status", "LastOrderUpdateUtc")
=RTD("Tws.Rtd", , "status", "OrderDataState")
=RTD("Tws.Rtd", , "status", "LastPositionListChangeUtc")
=RTD("Tws.Rtd", , "status", "LastPositionUpdateUtc")
=RTD("Tws.Rtd", , "status", "PositionDataState")
=RTD("Tws.Rtd", , "status", "MarketDataType")
=RTD("Tws.Rtd", , "status", "AccountsCSV")
=RTD("Tws.Rtd", , "status", "ServerVersion")
=RTD("Tws.Rtd", , "status", "MarketDataState")
=RTD("Tws.Rtd", , "status", "MarketDataMessage")
```

- *LastUpdateUtc* and *ServerHeartbeatUtc* are Excel DateTime values in UTC.
- *IsConnected* refers to the connection between the RTD server and TWS, not between Excel and the RTD server.
- *ServerHeartbeatUtc*: Excel periodically calls the `Heartbeat()` method on the RTD server.  However, the frequency of Excel's heartbeat can be unpredictable.  In testing we routinely see Excel instances run for several minutes without a heartbeat.
- *MarketDataType (default)*: 1=REALTIME, 2=FROZEN, 3=DELAYED, 4=DELAYED_FROZEN
- *AccountsCSV*: A comma-separated list of all managed account identifiers available through the current TWS connection (e.g., "DU123456,DU789012,DU345678"). This is populated from the TWS `managedAccounts` callback during the connection handshake.  In Excel 365, use `TEXTSPLIT` to spill to `AccountsCSV` to columns or rows.  Suggested patterns:
  - Horizontal (across columns): `=TEXTSPLIT(RTD("Tws.Rtd", , "status", , "AccountsCSV"), ",")`
  - Vertical (down rows): `=TEXTSPLIT(RTD("Tws.Rtd", , "status", , "AccountsCSV"), , ",")`
- *ServerVersion*, *MarketDataState*, and *MarketDataMessage* are **per-connection** diagnostics: each TWS connection negotiates its own protocol level, so supply a connection token (e.g. `paper`, `gw`, or `host=`/`port=`) to target a specific connection, or omit it to piggyback the single connection.
  - *ServerVersion*: this connection's negotiated TWS `ServerVersion` as a number, or `"Not Connected"` if no version has been negotiated yet.
  - *MarketDataState*: `Ok` when the negotiated protocol can stream market data (ServerVersion >= 206), `TooOld` when it cannot (1..205), or `Unknown` before a version is negotiated. See [Market Data](#market-data).
  - *MarketDataMessage*: when *MarketDataState* is `TooOld`, the actionable "update your TWS API" message explaining why quotes are withheld; otherwise empty.

### Position Data State Topics

Three additional status topics track the state of the position data system:

- **LastPositionListChangeUtc**: Updates whenever the membership of the symbols list actually changes (symbol added/removed from open positions). Use this with `SymbolsCsv` to detect when the list of open symbols has changed.
- **LastPositionUpdateUtc**: Updates on every position data callback from TWS, even when the symbol list membership hasn't changed. Useful for monitoring data freshness.
- **PositionDataState**: Displays the current state of the position data system. Possible values:
  - **Disconnected**: Not connected to TWS
  - **Idle**: Connected but no position topics subscribed
  - **Requested**: Position subscription requested, waiting for first callback
  - **Receiving**: Position callbacks arriving (snapshot in progress)
  - **Ready**: Initial snapshot complete, all position data available and presumed current

These topics help distinguish between data updates (new prices, quantities) and membership changes (symbols added/removed).

## Account Data

Formula: `=RTD("Tws.Rtd",, "account", "<AccountNumber>", "<field>" [, "<currency>"])`. Account keys are a pass-through of whatever TWS delivers (not a fixed list); common keys and the per-currency `$LEDGER-` rules are in [reference.md §10](reference.md#10-account-value-keys), and the full TWS-delivered set is enumerated on the Account worksheet in [StreamXLS.xlsm](../examples/StreamXLS.xlsm).

Examples:

```
=RTD("Tws.Rtd", , "account", "U1234567", "NetLiquidation")
=RTD("Tws.Rtd", , "account", "U1234567", "AvailableFunds", "cur=USD")
=RTD("Tws.Rtd", , "account", "U1234567", "OpenPositionCount")
```

- Numeric values are returned as numbers when possible (Excel auto-format applies).
- If a currency filter is omitted, generic fields (e.g., *NetLiquidation*) will not be overwritten by ByCurrency values.
- *OpenPositionCount* is computed per-account by tracking non-zero positions (or non-zero market values) and updates live with portfolio data.

## Market Data

For the complete list of available fields see [reference.md §2](reference.md#2-market-data-fields).  For instructions on specifying securities contracts see [DETAILED_INSTRUCTIONS](detailed-instructions.md).

**Special fields and notes:**

- **Last** (strict) `=RTD("Tws.Rtd",, "<contract>", "Last")` Shows #N/A when unset under real-time/frozen; shows numeric Last once trades arrive.
- **LastOrClose** `=RTD("Tws.Rtd",, "<contract>", "LastOrClose")` Last → Close precedence.
- **MarketPrice** `=RTD("Tws.Rtd",, "<contract>", "MarketPrice")` Mid → Last → Close precedence; returns blank until any source is available.
- **MarketDataType** `=RTD("Tws.Rtd",, "<contract>", "MarketDataType")` indicates the actual market data type TWS is serving for the specific contract, which can vary from the default.
- **IsDelayed** `=RTD("Tws.Rtd",, "<contract>", "IsDelayed")` returns `1` when TWS is serving this contract delayed data (MarketDataType 3 or 4), `0` when it is serving real-time or frozen real-time data (1 or 2), and #N/A before TWS has reported which. The simplest honest-data check — put it next to a price and drive conditional formatting off it.

**How the data types actually work** — `TWS_RTD_MARKET_DATA_TYPE` sets the *requested* type; TWS then serves **each contract at the best tier your market-data subscriptions allow** and reports what it served per contract (the `MarketDataType`/`IsDelayed` fields above). Under the default (4), a subscribed symbol streams real-time while an unsubscribed one silently falls back to 15-20-minute-delayed data — *in the same session, on the same connection* (live-verified 2026-07-09: one Type-4 session served IDEALPRO FX real-time, an unsubscribed CME future delayed). So "Type 4 is set" does **not** mean your data is delayed; check `IsDelayed` per symbol. To instead get an error whenever real-time data is unavailable, set `TWS_RTD_MARKET_DATA_TYPE=1`.

| Type | ID | Behavior |
|------|----|----|
| REALTIME | 1 | Requires API subscription; errors if missing |
| FROZEN | 2 | Last known price when market closed |
| DELAYED | 3 | **Auto-fallback**: real-time if available, delayed (15-20min) otherwise |
| DELAYED_FROZEN | 4 | Delayed + frozen data |

**⚠ Delayed prices look live in a plain cell.** A delayed quote is just a number — nothing in the cell marks it as 15-20 minutes old. Two ways to make delayed-ness visible:

1. **Indicator fields** (recommended): display `IsDelayed` or `MarketDataType` next to your prices, or drive conditional formatting off them.
2. **In-cell annotation** (opt-in): set `TWS_RTD_DELAYED_ANNOTATION=1` (or flip it in the StreamXLS Control Panel → Settings) and every delayed-tier numeric value renders as text, e.g. `150.25 (delayed)`. **Trade-off — read before enabling:** the value becomes *text*, and Excel treats any text as **greater than any number**, so comparison formulas on an annotated cell are silently wrong (`=IF(A1>100,...)` is always TRUE; `MAX`/`AVERAGE`/`COUNTIF` skip the cell), while arithmetic fails loud with `#VALUE!`. To recover the number in a formula: `=VALUE(SUBSTITUTE(A1," (delayed)",""))`. Annotation is presentation-only — it never alters stored values, and text can never feed a StageOrder (the order is rejected with a parse error, not placed).

**TWS API version floor for market data:** Modern TWS serves market data over a protobuf-only wire contract introduced at negotiated protocol `ServerVersion` 206 (TWS API **>= 10.38.01**). A connection whose TWS API is older negotiates a sub-206 protocol and receives **zero** market-data ticks from TWS — no error, no quotes — while orders, positions, account values, and PnL keep working normally. Because silent stale/blank quotes are dangerous, StreamXLS **withholds market-data topics fail-loud** on such a connection (with an actionable error in the [log](reference.md#11-configuration-keys)) rather than letting them sit blank. Check the per-connection status fields to diagnose this: `MarketDataState` reports `Ok`/`TooOld`/`Unknown`, and `MarketDataMessage` carries the "update your TWS API" detail. If quotes are missing but orders/positions work, update the TWS API.

## Position Data

Formula: `=RTD("Tws.Rtd",, "position", "<AccountNumber>", "<contract>", "<field>")` where `field` is one of (the complete field set with aliases is in [reference.md §8](reference.md#8-position-list-and-option-definition-fields)):

- AverageCost (per share)
- Position (shares)
- MarketValue
- DailyPNL
- RealizedPNL
- UnrealizedPNL
- Contract metadata fields: `ConID, Symbol, SecType, Strike, Exchange, LocalSymbol, TradingClass, Right, Expiry, Currency, Multiplier`

Examples:

```
=RTD("Tws.Rtd", , "position", , "AAPL", "Position")
=RTD("Tws.Rtd", , "position", "*", "AAPL@SMART", "MarketValue")
=RTD("Tws.Rtd", , "position", "U1234567,U8901234", "AAPL@SMART/STK", "MarketValue")
```

## Positions list (active positions)

Formula: `=RTD("Tws.Rtd",, "positions", "<Accounts>", "[SymbolsCsv|ConIdCsv|PositionsChangedUtc]")`

Examples:

```
=RTD("Tws.Rtd", , "positions", , "SymbolsCsv")
=RTD("Tws.Rtd", , "positions", "*", "SymbolsCsv")
=RTD("Tws.Rtd", , "positions", "U1234567,U7654321", "SymbolsCsv")
=RTD("Tws.Rtd", , "positions", , "PositionsChangedUtc")
=RTD("Tws.Rtd", , "positions", , "ConIdCsv")
```

- `SymbolsCsv` returns a semicolon-delimited list of unique position identifiers across all accounts by default.
  - For stocks: just the symbol (e.g., `AAPL`)
  - For options: `SYMBOL@EXCH/OPT/Expiry/Right/Strike/Currency` (e.g., `AAPL@SMART/OPT/20251219/C/180/USD`)
  - For futures: `SYMBOL@EXCH/FUT/Expiry/Currency` (e.g., `ES@CME/FUT/202503/USD`)
  - For other contracts: compact slash notation matching the [contract specification methods](./detailed-instructions.md#contract-specification-methods).
- `ConIdCsv` returns a semicolon-delimited list of contract IDs (ConId) for the same active positions.
- `<Accounts>` argument accepts a comma-separated account filter to restrict to specific accounts; use "*" or leave blank for all accounts.
- `PositionsChangedUtc` updates whenever the membership of the position set changes; pair it with `SymbolsCsv` or `ConIdCsv` to detect changes.  (Legacy synonym: `SymbolsChangedUtc`.)
- *Active positions* definition: every position where `(position size != 0) or (MarketValue != 0)` for any covered account.
- In Excel 365, use `TEXTSPLIT` to spill to `SymbolsCsv` to columns or rows.  Suggested patterns:
  - Horizontal (across columns): `=TEXTSPLIT(RTD("Tws.Rtd", , "positions", , "SymbolsCsv"), ";")`
  - Vertical (down rows): `=TEXTSPLIT(RTD("Tws.Rtd", , "positions", , "SymbolsCsv"), , ";")`
  - Guard for first-load blank: `=IFERROR(TEXTSPLIT(RTD("Tws.Rtd", , "positions", , "SymbolsCsv"), ";"), "")`

Notes on initial load:

- The RTD server waits to publish the first `SymbolsCsv`, `ConIdCsv`, and `PositionsChangedUtc` until the initial positions snapshot completes (IB API `positionEnd`).
- After the snapshot, subsequent membership changes publish immediately with updated `PositionsChangedUtc`.

## Orders list

Returns a comma-separated list of PermIDs, filtered by the given accounts.

### All Orders (including filled/cancelled)

Formula: `=RTD("tws.rtd",,"orders","<Accounts>","ListCsv")`

Returns all orders the RTD server has seen, including those that have been filled, cancelled, or otherwise completed.

### Open Orders Only

Formula: `=RTD("tws.rtd",,"orders","<Accounts>","OpenListCsv")`

Returns only active/open orders, excluding terminal statuses (Filled, Cancelled, Inactive, ApiCancelled).

### Account Filtering

As with `positions`, the `<Accounts>` argument accepts a comma-separated account filter to restrict to specific accounts; use "*" or leave blank for all accounts.

### Status Topics

```
=RTD("Tws.Rtd", , "status", "LastOrderListChangeUtc")
=RTD("Tws.Rtd", , "status", "LastOrderUpdateUtc")
=RTD("Tws.Rtd", , "status", "OrderDataState")
```

- `LastOrderListChangeUtc` updates whenever the order-list membership the RTD server tracks changes — a new order appears, or an order transitions to a terminal state. This is **not scoped to the accounts your `ListCsv`/`OpenListCsv` formulas subscribe to**: an order that appears for an account you are not displaying can still bump this timestamp, because the server tracks every order TWS reports on the connection.
- `LastOrderUpdateUtc` updates whenever TWS sends any *new* orderStatus.
- `OrderDataState` is one of: Disconnected, Idle, Requested, Receiving, Ready

## Order Data

```
=RTD("tws.rtd",,"order","<permId>","<field>")
```

`<permId>` is the TWS permanent order id — take it from the orders-list topics (`ListCsv`/`OpenListCsv`), which return permIds, not the small per-client Order IDs TWS shows in its own order window.

Common fields (the complete closed set — canonical names and synonyms — is in [reference.md §5](reference.md#5-order-read-fields)):

- `STATUS` — current order status (see Order Status Values below)
- `SYMBOL`, `CONID`, `SECTYPE`, `EXCHANGE`, `CURRENCY`
- `SIDE` (`ACTION`), `ORDERTYPE` (`TYPE`), `QUANTITY`
- `LMTPRICE` (`LIMITPRICE`), `STOPPRICE` (`AUXPRICE`), `AVGFILLPRICE`
- `FILLED`, `REMAINING`
- `TIF` (`TimeInForce`), `GOODTILLDATE`, `GOODAFTERTIME`
- `ACCOUNT`, `PERMID`, `PARENTID`, `FirstSeenUtc`, `LastUpdateUtc`

The supported order fields are a **COMPLETE, closed set** (canonical names with synonyms). There is no reflection fallback: an **unknown or misspelled field errors loudly** at formula-registration time (e.g. `Unknown order field 'FOO'. See streamxls.com/docs-reference for supported order fields.`) rather than silently leaving the cell at `#N/A`. For the full field list see [reference.md §5](reference.md#5-order-read-fields); if you need a field that isn't listed, it is a trivial addition — ask.

### Order Status Values

An `order` topic `STATUS` cell reports TWS's own status strings **verbatim** — `PendingSubmit`, `PendingCancel`, `PreSubmitted`, `Submitted`, `Filled`, `Inactive`, and the cancel strings `Cancelled`/`ApiCancelled` exactly as TWS spells them (two L's). Only a **staged-order** cell (`StageOrder`) respells those cancel strings to the house `Canceled` (one L, the spelling used since v1 so existing formulas keep matching); before TWS speaks, a staged order also shows the house states `Sending` then `Staged`. The complete status vocabulary with active/terminal classification is in [reference.md §7](reference.md#7-order-status-vocabulary); the staging lifecycle is in [Cell States and Lifecycle](detailed-instructions.md#cell-states-and-lifecycle).

## FAQ

### What happens if TWS closes while I have active RTD requests?

- The server's connection drops. Account and position values stop updating. By default, account values, PnL, and position cells display `#N/A` to prevent reliance on stale data (fail loud).
- **Prefer to keep the last values on screen during an outage?** Set `TWS_RTD_PRESERVE_ON_DISCONNECT=true` (StreamXLS Control Panel -> Settings -> "Preserve values on disconnect"). Existing cells then keep their last-known value through the outage and repaint when data returns; a cell that had no value stays `#N/A`, and a **position** a completed reconnect snapshot no longer reports flips to `#N/A` regardless of this setting (account balances re-download in full and repaint). **`order` and orders-list cells ride this knob too:** with the default (`false`), working orders go `#N/A` while their terminal facts (`Filled`/`Canceled`/`Inactive`) are preserved; with the knob on, they keep their workbook-saved values until reconnect repaints them. **`StageOrder` cells are the exception — a non-terminal staged status is swept to `#N/A` unconditionally, ignoring the knob** (IB may fill or cancel a transmitted order during the outage, so a stale "Submitted" must never persist). Market-data cells follow their own per-field rules. The exact per-family behavior is in [Cell States and Lifecycle](detailed-instructions.md#cell-states-and-lifecycle). *(One narrow edge: a per-currency balance for a currency fully closed out during the outage may keep its last value until the next update; the default fail-loud mode avoids this.)*
- You can monitor connection state via status topics:
  - `=RTD("Tws.Rtd",, "status","IsConnected")` → 1 when connected, 0 otherwise
  - `=RTD("Tws.Rtd",, "status","LastUpdateUtc")` → ISO 8601 timestamp of the last successful update published by the server
  - `=RTD("Tws.Rtd",, "status","ActiveTopicCount")` → Number of topics subscribed.
- **Exactly what each cell family shows on disconnect, reconnect, or a license/trial lapse** is documented in [Cell States and Lifecycle](detailed-instructions.md#cell-states-and-lifecycle) in detailed-instructions.md.

### I want to open a saved workbook and see the last values, without connecting to TWS. How?

- Put Excel into **Manual Calculation mode** (Formulas -> Calculation Options -> Manual) *before* opening the workbook. Excel then does not re-evaluate the RTD formulas on open, so the cell values saved with the workbook stay on screen — the same move Bloomberg users know. This works with no TWS connection and zero engine risk.
- Switch back to Automatic (or press F9) when you want live data again. (A future opt-in "cold-open preserve" engine setting is queued; until then, Manual Calculation is the workaround.)

### My orders and positions update, but I get no market-data quotes. Why?

- Most likely your TWS API is too old for market data. Modern TWS serves quotes only over a protocol negotiated at `ServerVersion` 206 (TWS API **>= 10.38.01**). An older API negotiates a sub-206 protocol and TWS sends it **zero** market-data ticks while orders/positions/account/PnL keep working. StreamXLS withholds market-data topics fail-loud in this case (see [Market Data](#market-data)).
- Diagnose with the per-connection status fields:
  - `=RTD("Tws.Rtd",, "status","MarketDataState")` → `Ok` / `TooOld` / `Unknown`
  - `=RTD("Tws.Rtd",, "status","MarketDataMessage")` → the actionable detail when `TooOld`
  - `=RTD("Tws.Rtd",, "status","ServerVersion")` → this connection's negotiated `ServerVersion`
- Fix: update the TWS API to a current release and reconnect.

### What happens if I restart TWS?

- While TWS is restarting the RTD server will recognize the lost connection and immediately set `IsConnected` to 0.

### How do I reconnect to TWS?

If you open Excel before TWS, or have it open while TWS restarts, the RTD server will automatically reconnect and re-subscribe to all topics you have requested.  If Excel is very busy that automatic reconnection could take a minute or longer to occur.  You can force a reconnection by closing and reopening Excel, or by making the following VBA call: `Application.RTD.RefreshData`

> **`StageOrder` formulas stage only when freshly entered.** Reopening a saved workbook does NOT re-stage them — StreamXLS recognizes the reopen and shows `Disarmed: workbook reopen does not re-stage orders...` instead; re-enter the formula (F2, Enter) to stage again deliberately. TWS reconnects do not re-stage either. ⚠️ **Editing a staged order's parameters stages a *second* order** rather than modifying the first — the original stays in TWS; cancel it there. Track live orders with the `orders`/`order` topics. See [Staging Orders](detailed-instructions.md#staging-orders-stageorder) for details.

### Multiple Excel instances: can conflicts arise?

- Multiple Excel processes can each host their own RTD server instance. Each instance maintains its own TWS connection and subscriptions. If you prefer shared connections, keep to one Excel process.

### What if the RTD server crashes?

- Excel will show `#N/A` or last-known values. Excel will attempt to reinstantiate an RTD server instance every heartbeat so long as it has RTD formulas.

### How can I determine the last time of a successful RTD update?

- Use the status topic `LastUpdateUtc`:
  - `=RTD("Tws.Rtd",, "status","LastUpdateUtc")`

### How can I detect stale subscriptions?

- Combine `IsConnected` and `LastUpdateUtc` to flag stale data. If `IsConnected`=1 but `LastUpdateUtc` hasn’t advanced for several seconds during market hours, investigate.

### Application.RTD.ThrottleInterval

You can use a macro (VBA) to modify Excel's `Application.RTD.ThrottleInterval`.  This determines how often Excel calls `RefreshData()` to pull queued topic updates.
