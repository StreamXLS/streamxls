# Frequently Asked Questions

## What is RTD?

RTD (Real-Time Data) is Microsoft Excel's native streaming-data protocol. An RTD server is a COM component that Excel queries via the `=RTD(progID, server, topic1, topic2, ...)` worksheet function. Excel owns the subscription lifecycle (`ConnectData` / `RefreshData` / `DisconnectData`); the server's job is to maintain the underlying subscriptions and surface value updates that Excel reads on its next refresh pass.

RTD is the documented streaming-data path for new Excel work. From [IBKR's Excel RTD documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/excel-rtd/):

> RTD is Microsoft's recommended replacement for DDE (Dynamic Data Exchange).

## What is StreamXLS?

A production-grade `IRtdServer` implementation that connects to Interactive Brokers' TWS (or IB Gateway) and exposes Market Data, Account values, Positions, Order monitoring, and Order submission as `=RTD()` topics. Written in C#, registered as a COM component with ProgID `Tws.Rtd`. An optional registry alias maps the IBKR sample's legacy ProgID `Tws.TwsRtdServerCtrl` onto this server's CLSID for workbooks built against the sample.

## How does this relate to the `TwsRtdServer` sample in `C:\TWS API\samples\Excel`?

Interactive Brokers ships a sample Excel RTD application of the same name with the TWS API distribution. From IBKR's Excel RTD documentation:

> [The sample applications] are not intended to be used as production level trading tools.

This project is a separate, independent implementation of the same `IRtdServer` COM contract — a drop-in replacement designed for live-trading deployment. It is not affiliated with Interactive Brokers; the shared `TwsRtdServer` name reflects the shared COM contract and topic-stream model. Differences include:

- **Topic schema beyond Market Data.** Account values (136 fields, per-currency), Positions (with streaming P&L), Order monitoring (all clients, 30+ fields), and Order submission via a `SendOrder` topic family are all exposed as `=RTD()` formulas.
- **Excel UI-priority handling.** Excel pauses its RTD pull while a modal dialog is open, while a cell is in edit mode, or while Excel is otherwise busy. IBKR's docs describe this as inherent to Excel's design as a trading application. StreamXLS maintains its internal cache independently of Excel's polling cadence so the data sitting in the cache when Excel returns to a ready state is the latest, not whatever happened to arrive during Excel's polling window.
- **Multi-instance behaviour.** Two `EXCEL.EXE` processes running on the same host each open their own TWS API client connection with separate auto-allocated client IDs.
- **Subscription deduplication within a process.** Many `=RTD()` cells referencing the same logical topic share a single upstream subscription.
- **Automatic reconnection** with subscription re-establishment + non-volatile-field preservation across the disconnect window.
- **Test coverage** sized for live-trading deployment.
- **Migration path.** Existing workbooks built against the IBKR sample can be migrated either by replacing the `Tws.TwsRtdServerCtrl` ProgID with `Tws.Rtd` or by installing the optional legacy-ProgID alias and leaving the formulas alone.

## Can StreamXLS place orders?

**Yes.** A dedicated `SendOrder` topic family submits an order as the side-effect of subscribing to the topic. Example:

```excel
=RTD("Tws.Rtd",,"SendOrder","sym=AAPL","side=BUY","shares=100","type=LMT","limit=150.05","exch=SMART")
```

The cell returns a status string — `Sending` while the submission is queued, `Sent` once `PlaceOrder` has been invoked successfully, or `SendOrder Error: <message>` on validation / connection / API errors. Required parameters are `sym` / `side` / `shares` / `type` (plus `limit` if `type=LMT`); common optional parameters include `exch` (defaults to `SMART`), `account`, and a `tag` / `nonce` token for uniqueness when the same parameters need to be submitted twice in a row (Excel deduplicates identical RTD topics, so a per-submission tag forces a fresh subscription).

The `SendOrder` topic family is documented in full in `DETAILED_INSTRUCTIONS.md` which ships alongside the binary; a worked-example workbook (`StreamXLS.xlsm`, "SendOrder" sheet) ships with the installer.

## How are TWS client IDs allocated across multiple Excel instances?

Each Excel process that loads StreamXLS opens its own TWS API client connection. By default, a per-process client ID is chosen automatically to reduce collisions across instances. To pin a specific client ID, set the `TWS_RTD_CLIENT_ID` environment variable before launching Excel.

## Is the source code available?

No. StreamXLS is closed-source commercial software. The repository you are looking at contains documentation, examples, FAQ, and binary releases — not source.

Source-license inquiries (e.g., for in-house trading-firm use, white-label OEM, or integration partnership) are welcome via [GitHub Discussions](https://github.com/StreamXLS/streamxls/discussions) under the **Commercial inquiries** category.

## When will signed binaries be available?

Signed releases are gated on completion of code-signing certificate onboarding. When the cert is in place, the first signed release publishes to the [Releases](https://github.com/StreamXLS/streamxls/releases) page on this repository, accompanied by verified install + first-run instructions and example workbooks.

## What do I need on the IBKR side?

A live or paper IBKR account with API access enabled (TWS or IB Gateway). The server connects to TWS over the documented socket-API client port:

| Endpoint | Live | Paper |
|---|---|---|
| TWS | `7496` | `7497` |
| IB Gateway | `4001` | `4002` |

API permissions, trusted-IP configuration, and master-client-ID setup follow the standard guidance in the [TWS API documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/twsapi-doc/). These prerequisites apply to StreamXLS in the same way they apply to any TWS API client.

## What versions of Windows / Excel / TWS are supported?

Targeted: Windows 10 / 11, Excel for Microsoft 365 (Desktop), recent TWS / IB Gateway builds. Specific supported version matrices publish with the first Releases drop.

## How does pricing work?

Pricing and end-user distribution model are in development; details publish when the first Releases drop. Source-license terms (for trading-firm in-house use) are negotiated case-by-case via [GitHub Discussions](https://github.com/StreamXLS/streamxls/discussions).

## I have a bug to report / a feature to request.

Open an [Issue](https://github.com/StreamXLS/streamxls/issues) using the appropriate template. For binary-product bugs, reproduction steps with the `=RTD(...)` formula, the contract / account context, and TWS / Excel / Windows versions help triage.

## I want to integrate StreamXLS into another product (white-label / OEM).

[GitHub Discussions](https://github.com/StreamXLS/streamxls/discussions) → **Commercial inquiries** category.
