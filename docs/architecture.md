# Architecture Overview

This document gives a high-level picture of how StreamXLS is structured. It is intended for engineers evaluating the product (or considering a source license) — not as a complete API reference.

## Component layout

```
+------------------+      COM (RTD)      +------------------+      TWS Socket API     +------------------+
|                  | <-----------------> |                  | <---------------------> |                  |
|   Microsoft      |  IRtdServer.        |  StreamXLS       |  EClientSocket /        |   TWS / IB       |
|   Excel          |  ConnectData /      |  (this product)  |  EWrapper callbacks     |   Gateway        |
|   (EXCEL.EXE)    |  RefreshData /      |                  |                         |                  |
|                  |  DisconnectData     |                  |                         |                  |
+------------------+                     +------------------+                         +------------------+
```

- **Excel** owns the call into the RTD server. It calls `ConnectData()` when a worksheet introduces a new `=RTD(...)` topic, polls `RefreshData()` on the throttle interval, and calls `DisconnectData()` when a cell formula referencing the topic is removed.
- **StreamXLS** is a COM component registered with the ProgID `Tws.Rtd` (with an optional legacy-ProgID alias `Tws.TwsRtdServerCtrl` for workbooks built against the IBKR sample). It receives RTD calls on a COM-managed apartment thread, maps topics to TWS subscriptions, and pushes value updates back into the cached snapshot that Excel reads on the next `RefreshData()` pass.
- **TWS / IB Gateway** is the upstream — the IBKR client process that holds the broker session. StreamXLS is a TWS API client of the same kind as `ib_async`, the official `EClientSocket` samples, or a custom C++ client; it just speaks Excel's RTD protocol on the other side.

## Per-Excel-process connection model

Each `EXCEL.EXE` process that loads StreamXLS establishes its own TWS API client connection. This is the standard Excel + COM behaviour — every instance of Excel loads its own copy of the in-process COM server — and StreamXLS leans into it rather than fighting it.

Consequence: running two Excel windows side by side (a "Live" workbook and a "Scratch" workbook in a separate instance, for example) produces two independent TWS API client connections. They do not contend for a shared subscription map.

### clientId allocation

TWS accepts one connection per client ID. Two Excel instances therefore need two distinct client IDs. By default, StreamXLS picks a per-process client ID automatically to reduce collisions across instances and with concurrent `ib_async` or sample-code sessions. To pin a specific client ID (for example, to satisfy a TWS configuration that requires `clientId=0` for certain features), set the `TWS_RTD_CLIENT_ID` environment variable before launching Excel.

Note that `reqAutoOpenOrders(true)` — the TWS API call that pushes order-status updates from *other* clients (FlexTrader, mobile app, additional API sessions) into your client — only works with `clientId=0`. Because StreamXLS's default client ID is randomised, it does not receive auto-pushed order updates from other clients; instead, it polls `reqAllOpenOrders()` at a configurable interval (default 15 seconds, via `TWS_RTD_ORDER_REFRESH_SECONDS`) whenever order topics are active. Set `TWS_RTD_CLIENT_ID=0` if you want auto-push instead, accepting that it precludes running other API clients alongside.

## Subscription deduplication within a process

Inside a single Excel process, however, the picture is reversed. Many cells in the same workbook (or across multiple workbooks open in the same instance) can reference the same logical topic — for example, a top-of-book quote for `SPY` shown in 30 different cells.

StreamXLS deduplicates: there is one TWS subscription per unique topic per process, regardless of how many `=RTD()` cells reference it. The deduplication is visible to the user as `ActiveTopicCount` on the Status tab — that count tracks distinct upstream subscriptions, not Excel cell references.

This is what lets a workbook with hundreds of `=RTD()` formulas run on a single API client without saturating pacing limits.

## UI-priority tolerance

Microsoft Excel gives its UI thread priority over RTD updates. IBKR's own Excel RTD documentation describes this clearly:

> [B]y design, Microsoft Excel gives precedence to the UI. Updates are ignored when a modal dialog is displayed, a cell is being edited, or Excel is busy. ([IBKR Campus, Excel RTD page](https://www.interactivebrokers.com/campus/ibkr-api-page/excel-rtd/))

A naive RTD implementation drops the data delivered during those periods. StreamXLS maintains its internal cache independently of Excel's polling, so a modal dialog open over a streaming chain does not cause data loss — when Excel returns to a ready state, it reads the latest cached value rather than a stale or null one. The [demo video](https://vimeo.com/1191256765) shows this behaviour explicitly (Beat 2, modal Format Cells dialog over a streaming options chain).

## Topic schema

StreamXLS exposes six topic families. The exact topic-string grammar — every family, contract-specification syntax, and field — is documented at [streamxls.com/docs-reference](https://streamxls.com/docs-reference). The families are:

| Family | Topic-string shape | Examples |
|---|---|---|
| **Status** | `status, <field>` | `IsConnected`, `ActiveTopicCount`, `LastUpdateUtc`, `ServerHeartbeatUtc`, `PositionDataState`, `OrderDataState`, `AccountsCSV` |
| **Market Data** | `<contract>, <field>` | top-of-book quotes (bid / ask / last / size / volatility / Greeks), option chains, contract details; derived fields `MarketPrice`, `LastOrClose` |
| **Accounts** | `account, <AccountNumber>, <field>[, <currency>]` | account values across 136 fields including NLV, buying power, currency balances, margin, OpenPositionCount |
| **Positions** | `position, <Accounts>, <contract>, <field>` and `positions, <Accounts>, <ListField>` | per-account, per-contract position size, average cost, market value, realised / unrealised P&L; positions-list topics return `SymbolsCsv` / `ConIdCsv` / `PositionsChangedUtc` |
| **Order monitoring** | `orders, <Accounts>, <ListField>` and `order, <orderID>, <field>` | list topics return `ListCsv` (all orders, including filled/cancelled) or `OpenListCsv` (open only); per-order topics return 30+ fields including `Status`, `Filled`, `Remaining`, `LmtPrice`, `AvgFillPrice`, `Side`, `OrderType`, `TIF` |
| **Order staging** | `StageOrder, <key>=<value>, ...` | stages an order as the side-effect of subscribing; topic returns a status string (`Sending` → `Staged`, or `SendOrder Error: <message>`) |

Every family is queried through the same `=RTD()` worksheet function. There is no separate add-in, no VBA glue, no ActiveX object on the worksheet — only native RTD formulas.

## Thread model

The server uses a multi-threaded apartment (MTA) for the COM side and a dedicated EWrapper-callback thread for the TWS side, with a thread-safe internal cache between them. Pacing and reconnection logic live in the upstream-facing layer; Excel only ever sees the cache.

## What this does not do

- **Replace TWS.** StreamXLS connects to a running TWS or IB Gateway; it does not implement the broker session itself.
- **Run without TWS API socket permissions.** Standard IBKR account API enablement applies (Configure → API → Settings → Enable ActiveX and Socket Clients, plus trusted-IP setup).
