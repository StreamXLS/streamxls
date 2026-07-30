# Frequently Asked Questions

Basic product questions — install, trial, formulas, troubleshooting — are answered at [streamxls.com/docs](https://streamxls.com/docs) and [streamxls.com/faq](https://streamxls.com/faq). This page covers the questions a GitHub visitor tends to ask.

## What is RTD?

RTD (Real-Time Data) is Microsoft Excel's native streaming-data protocol. An RTD server is a COM component that Excel queries via the `=RTD(progID, server, topic1, topic2, ...)` worksheet function. Excel owns the subscription lifecycle (`ConnectData` / `RefreshData` / `DisconnectData`); the server's job is to maintain the underlying subscriptions and provide the latest data when requested by Excel.

RTD is Microsoft's recommended replacement for DDE (Dynamic Data Exchange) and is capable of serving realtime data at the scale required for modern trading and asset management operations.

## What is StreamXLS?

A production-grade `RtdServer` implementation that connects to Interactive Brokers' TWS (or IB Gateway) and streams Market Data, Account values, Positions, and Orders using Excel's native `=RTD()` formula.

## How does this relate to the `TwsRtdServer` sample in `C:\TWS API\samples\Excel`?

Interactive Brokers ships a sample "TwsRtdServer" with the TWS API. From IBKR's Excel RTD documentation:

> [The sample applications] are not intended to be used as production level trading tools.

StreamXLS is a separate, independent implementation of the same `IRtdServer` COM contract, designed for live-trading deployment. It is not affiliated with Interactive Brokers. Differences include:

- **Topic schema beyond Market Data.** Account values (130+ fields, per-currency), Positions (with streaming P&L), Order monitoring ([roughly 80 fields](reference.md#5-order-read-fields)), and Order staging via a `StageOrder` topic family are all exposed as `=RTD()` formulas.
- **Excel UI-priority handling.** Excel stops calculating while you edit a formula, interact with a dialog box or setting, or while Excel is otherwise busy. This tends to break realtime data feeds that expect the application to always be available for updates.  The RTD protocol is different: StreamXLS listens to updates and maintains the latest data internally, then provides that only when Excel asks and is ready to receive it.
- **Multi-instance behaviour.** Separate `EXCEL.EXE` processes can connect to the same TWS (or IB Gateway) instance.  And a single Excel workbook can connect to multiple TWS/Gateway instances.
- **Subscription deduplication within a process.** Many `=RTD()` cells referencing the same logical topic share a single upstream subscription.
- **Automatic reconnection** and resubscription.
- **Test coverage** engineered for reliability — every release is verified with a suite of more than 3,000 automated tests.

## Can StreamXLS place orders?

**No, but it can *stage* them.** A `StageOrder` topic family stages an order as the side-effect of subscribing to the topic. Example:

```excel
=RTD("Tws.Rtd",,"StageOrder","sym=AAPL","side=BUY","shares=100","type=LMT","limit=150.05","exch=SMART")
```

By default a staged order arrives in TWS **deactivated** — visible in the TWS order list with a **Submit** button, and released to the market only when you click it there; `park=true` instead stages an invisible `Transmit=false` ticket that TWS shows only in its order-entry row, released with **Transmit**. Either way, nothing reaches the market without a human action in TWS (see [Cell states and lifecycle](detailed-instructions.md#cell-states-and-lifecycle)).

## How are TWS client IDs allocated across multiple Excel instances?

Each Excel process that loads StreamXLS opens its own TWS API client connection. By default, a per-process client ID is chosen automatically to reduce collisions across instances. To pin a specific client ID, set the `TWS_RTD_CLIENT_ID` environment variable before launching Excel.

## Is the source code available?

No. StreamXLS is closed-source commercial software. The repository you are looking at contains documentation, examples, FAQ, and binary releases — not source.

Source-license inquiries (e.g., for in-house trading-firm use, white-label OEM, or integration partnership) are welcome by email: [sales@streamxls.com](mailto:sales@streamxls.com).

## Where do I download it, and is it signed?

The signed installer is on the [Releases](https://github.com/StreamXLS/streamxls/releases) page — `StreamXLS-Setup-<version>.exe`, a per-user installer (no admin rights) signed by **StreamXLS LLC**. Each installer's SHA-256 is published with the release (the asset's digest) and at [streamxls.com/download](https://streamxls.com/download). Because launch volume is low, Windows SmartScreen may still warn on first run; [streamxls.com/download](https://streamxls.com/download) walks through verifying the publisher and checksum. Full install and first-run guidance lives at [streamxls.com/docs](https://streamxls.com/docs).

## What do I need on the IBKR side?

A live or paper IBKR account with API access enabled (TWS or IB Gateway). The server connects to TWS over the documented socket-API client port:

| Endpoint | Live | Paper |
|---|---|---|
| TWS | `7496` | `7497` |
| IB Gateway | `4001` | `4002` |

API permissions, trusted-IP configuration, and master-client-ID setup follow the standard guidance in the [TWS API documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/twsapi-doc/). These prerequisites apply to StreamXLS in the same way they apply to any TWS API client.

## What versions of Windows / Excel / TWS are supported?

Windows 10 / 11, Microsoft Excel for Windows (desktop, 32- or 64-bit), .NET Framework 4.8 (present by default on modern Windows), Interactive Brokers' current TWS / IB Gateway.  You must also install the [**TWS API**](https://interactivebrokers.github.io/) — version **10.47.01** is the current minimum.

## How does pricing work?

There is a 30-day, full-featured trial that is keyless — it starts automatically the first time StreamXLS is used in Excel, no sign-up required. Subscriptions start at **$59/month**; current pricing is on the [pricing page](https://streamxls.com/buy). A license key arrives by email after purchase and is activated in the StreamXLS Control Panel. The trial is the evaluation window: paid subscriptions are non-refundable, but you can cancel future renewals at any time (effective at the end of the paid period).

## What happens when the trial or a subscription ends?

Live data stops at the end of the trial or paid period. Nothing is deleted — the software does not remove itself, your workbooks, or any logged data — and full function resumes when you activate or renew a license. A temporary licensing-service problem is not a lapse: after activation, StreamXLS stores a license token on your machine and continues to operate for up to 30 days without reaching the license server, alerting you well before the token expires. Details: [streamxls.com/faq](https://streamxls.com/faq) and the [EULA](https://streamxls.com/eula).

## Does it send my trading data anywhere?

No. The software never transmits your market data, positions, orders, account values, or spreadsheet contents — that data is processed locally, between your own TWS and your own Excel — and it contains no usage analytics or behavioral telemetry. The only data it transmits is the limited licensing data used for activation and periodic re-validation and, if update checking is enabled, update checks. Full policy: [streamxls.com/privacy](https://streamxls.com/privacy).

## How do updates work?

New releases publish on this repository's [Releases](https://github.com/StreamXLS/streamxls/releases) page and are offered to installed copies through the product's update channel. Updates are offered, not forced: you are notified, nothing installs without your action, and nothing ever installs mid-session. (Excel must be restarted to load a new RTD server.) Each update is cryptographically verified before it runs.

## I have a bug to report / a feature to request.

Open an [Issue](https://github.com/StreamXLS/streamxls/issues) using the appropriate template. For binary-product bugs, reproduction steps with the `=RTD(...)` formula, the contract / account context, and TWS / Excel / Windows versions help triage. For account or license issues, email [support@streamxls.com](mailto:support@streamxls.com) instead — those usually involve details that shouldn't be public.

## I want to integrate StreamXLS into another product (white-label / OEM).

Email [sales@streamxls.com](mailto:sales@streamxls.com).
