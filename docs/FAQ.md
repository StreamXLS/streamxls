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

By default a staged order arrives in TWS **deactivated** — visible in the TWS order list (to every TWS session on the account, and it survives a TWS restart) with a **Submit** button, and released to the market only when you click it there; `park=true` instead stages a local order-entry ticket — visible in the parking user's own TWS order list with a **Transmit** button, but not seen by other TWS instances or the API until you transmit it. Either way, nothing reaches the market without a human action in TWS. Staging requires TWS's **Read-Only API** setting to be off (*File → Global Configuration → API → Settings*). Details: [Staging orders](manual.md#staging-orders-stageorder).

## How are TWS client IDs allocated across multiple Excel instances?

Each Excel process that loads StreamXLS opens its own TWS API client connection. By default, client IDs are chosen automatically to reduce collisions across instances. To pin a specific client ID, set the `TWS_RTD_CLIENT_ID` environment variable before launching Excel.

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

In TWS or IB Gateway, *File → Global Configuration → API → Settings*: enable **ActiveX and Socket Clients**, confirm the **Socket Port**, and uncheck **Read-Only API**. StreamXLS never places or modifies orders on its own (staged tickets still need your click in TWS to transmit), but TWS classifies its completed-order history query as a write action: with Read-Only checked, TWS refuses that query (error 321), pops an "API client is attempting to send a request that needs API write access" dialog at you, and StreamXLS then cannot list orders that were already complete when it connected or confirm the final status of an order that leaves the open-orders list. StreamXLS logs one warning naming the setting and stops asking until it reconnects; open orders, executions, positions, and market data are unaffected. Read-Only must also be off for `StageOrder` to draft order tickets in TWS. API permissions, trusted-IP configuration, and master-client-ID setup follow the standard guidance in the [TWS API documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/twsapi-doc/). These prerequisites apply to StreamXLS in the same way they apply to any TWS API client.

## Why do `BIDSIZE` and `ASKSIZE` read 0 on liquid US stocks?

Because Interactive Brokers is sending sizes in **round lots**, and an inside quote of fewer than 100 shares truncates to zero. In TWS/IB Gateway *Global Configuration → API → Settings*, checkbox "Send market data in lots for US stocks for dual-mode API clients" — IBKR marks it *(Not recommended)* — divides `BIDSIZE`, `ASKSIZE` and `LASTSIZE` by 100 and rounds down, so a 40-share bid reports `0`. A separate box, "Send volumes in lots from all market data sources for US stocks", does the same to `VOLUME`: a session that traded 74.3 million shares reports `742,668`. The two are independent, so sizes in shares alongside volume in lots is possible — verify both. StreamXLS reports what TWS sends.

The price beside a `0` size is still live: the zero means "under one round lot", not "no quote" (a genuinely absent quote returns `#N/A`). Clear a checkbox to receive share counts instead — those fields then return values 100× larger. Details: [Size and volume units](reference.md#size-and-volume-units).

## Every StreamXLS cell shows `#N/A` — on a VPS, or when Excel runs as administrator

By default StreamXLS installs for a single user, and Windows does not let a program running with full administrator rights use an installation registered that way. An Excel that runs that way cannot start StreamXLS at all: **every** `=RTD("Tws.Rtd", …)` formula returns `#N/A`, including the ones that need no TWS connection (`VERSION`, `LICENSE_STATE`), and no log is written, because the server never starts.

There are two ways to be in that situation:

- **You are signed in as the built-in `Administrator` account.** This is the default on most trading-VPS images, and it is the common case. Every program on that account runs with full administrator rights whether you asked for it or not, so it is not something you can switch off for Excel alone. The same applies on a machine where User Account Control has been turned off, where every administrator account behaves this way.
- **You launched Excel with "Run as administrator".** Start Excel normally instead; StreamXLS needs no administrator rights for anything.

**The fix, in order of preference:**

1. **Uninstall StreamXLS (Settings → Apps), then run the installer again and choose "Install for all users."** The uninstall comes first: a re-run over an existing single-user installation keeps that installation's mode and never shows the choice. (Alternatively, run the installer with the `/ALLUSERS` switch; it then offers to remove the single-user copy for you.) Installing for all users registers StreamXLS for the whole machine, which any account can use — including the built-in `Administrator`, and including an Excel you deliberately run as an administrator. It needs administrator rights to install, and afterwards applying an update will ask for administrator approval too.
2. **Or sign in with a named administrator account** — any account you created yourself — and install StreamXLS there. It then works normally, with no other change. This needs no administrator rights at all, but it only helps that account.

From version 1.1.1 the installer tells you when the account you are installing from cannot use a single-user installation, and checks at the end of the installation that Excel will actually be able to start StreamXLS. The Control Panel runs the same check when it opens.

This is standard Windows behaviour for installations registered to one user, not specific to any one Windows version. If only *some* cells read `#N/A` while others stream normally, this is not the cause — if the dead ones are quote cells, see [Quotes stopped](#quotes-stopped--no-market-data-during-competing-live-session); otherwise see [Field reference](reference.md) for the per-field conditions.

## Quotes stopped — `No market data during competing live session`

Interactive Brokers licenses market data per user and streams it to one session at a time. If a second session is signed in to the same IBKR user — TWS or IB Gateway on another machine, IBKR Mobile, or Client Portal — that session holds the line, and the one StreamXLS is connected to receives a refusal instead of prices. A paper-trading session holds no market-data subscriptions of its own; it borrows the live user's, so a paper session is the one that goes quiet when a live session appears anywhere. Switching to delayed data did not help in our tests.

This is not the same fault as [every cell showing `#N/A`](#every-streamxls-cell-shows-na--on-a-vps-or-when-excel-runs-as-administrator). There, Excel never starts the StreamXLS engine and *every* formula is dead, including the ones that need no TWS connection (`VERSION`, `LICENSE_STATE`), with no log written. Here the connection is healthy — `IsConnected` reads `1`, and orders, positions and account values keep updating — and market data specifically stops, on the refused contracts only.

From version 1.1.1, every cell on a refused contract reads `RTD error: No market data during competing live session` for as long as the refusal is in effect rather than a bare `#N/A`, and StreamXLS keeps re-requesting the contract — three times at five-minute intervals, then once an hour — instead of giving up for the session. Close the other session and data comes back on its own, usually within a few minutes of it going away, with nothing to do in Excel.

Full diagnosis, including how to confirm it from the sheet without a log and what to do if data does not come back: [Quotes stopped, but the connection is healthy](manual.md#quotes-stopped-but-the-connection-is-healthy).

## What versions of Windows / Excel / TWS are supported?

Windows 10 / 11; desktop Microsoft Excel for Windows — Microsoft 365 or Office 2016+, 32- or 64-bit. RTD is a Windows capability, so Excel for Mac, Excel on the web, and mobile Excel are not supported. You must also install the [**TWS API**](https://interactivebrokers.github.io/) — version **10.47.01** is the current minimum.

## How does pricing work?

There is a 30-day, full-featured trial that starts on first run — no sign-up required. Subscriptions start at **$59/month**; current pricing is on the [pricing page](https://streamxls.com/buy). A license key arrives by email after purchase and is activated in the StreamXLS Control Panel. The trial is the evaluation window: paid subscriptions are non-refundable, but you can cancel future renewals at any time (effective at the end of the paid period).

## What happens when the trial or a subscription ends?

Live data stops at the end of the trial or paid period. Nothing is deleted — the software does not remove itself, your workbooks, or any logged data — and full function resumes when you activate or renew a license. A temporary licensing-service problem is not a lapse: after activation, StreamXLS stores a license token on your machine and continues to operate for up to 30 days without reaching the license server, alerting you well before the token expires (unless you have switched the reminders off — see below). Details: [streamxls.com/faq](https://streamxls.com/faq) and the [EULA](https://streamxls.com/eula).

StreamXLS warns you — as Windows notifications raised by its Control Panel — before your trial ends, and before a verification problem can stop your data. The notifications appear as a free trial nears its end (roughly seven days out, then three, then one, then on the last day) and, once a paid license's stored token is running short of time to re-verify, about a week, three days and one day before it runs out; that is the case a reconnection (or allowlisting the license server through a firewall or VPN) resolves. At most one such notification appears per day, and a license that has already lapsed is announced once rather than repeatedly. A healthy paid subscription is not counted down to: its stored date is the end of the current billing period and moves forward each time you renew. The reminders can be switched off in the Control Panel ("Remind me before my trial or license expires"), which takes effect immediately — including withdrawing a reminder still sitting in the notification center — and Windows' own per-app notification settings apply to them like any other notification.

## StreamXLS says "Accept the StreamXLS licence agreement in the StreamXLS Control Panel"

Every Windows user account accepts the licence agreement once. If StreamXLS was installed for all users — on a shared machine, or on a pre-built image — the person who installed it accepted for themselves only, so your account has not accepted yet. `LICENSE_STATE` reads `AssentRequired` while that is the case.

Open **Start menu → StreamXLS → StreamXLS Control Panel**, read the agreement, and click **I accept**.

If you decline, the Control Panel closes; open it again whenever you want to accept. The full account of what each formula family shows while the agreement is outstanding is in the manual: ["Accept the StreamXLS licence agreement…" — a Windows account that has not accepted yet](manual.md#accept-the-streamxls-licence-agreement--a-windows-account-that-has-not-accepted-yet).

## StreamXLS says "the Microsoft Visual C++ runtime is missing"

StreamXLS's licensing component needs the Microsoft Visual C++ 2015–2022 runtime. Nearly every Windows PC already has it — many programs install it — but a freshly built Windows Server image or VPS template may not. Until it is present StreamXLS cannot verify any licence: `LICENSE_STATE` reads `Unknown`, and the cells that need a licence carry that message instead of data.

Install the [x64 redistributable](https://aka.ms/vs/17/release/vc_redist.x64.exe) (and the [x86 one](https://aka.ms/vs/17/release/vc_redist.x86.exe) as well if you use 32-bit Excel), then restart Excel. StreamXLS itself does not need to be reinstalled. If you are building an image, install the runtime on the template first — see [Pre-installing StreamXLS in a Windows image](install.md#pre-installing-streamxls-in-a-windows-image).

## Does it send my trading data anywhere?

No. The software never transmits your market data, positions, orders, account values, or spreadsheet contents — that data is processed locally, between your own TWS and your own Excel — and it contains no usage analytics or behavioral telemetry. The only data it transmits is the limited licensing data used for activation and periodic re-validation and, if update checking is enabled, update checks. Full policy: [streamxls.com/privacy](https://streamxls.com/privacy).

## How do updates work?

New releases publish on this repository's [Releases](https://github.com/StreamXLS/streamxls/releases) page and are offered to installed copies through the product's update channel. Updates are offered, not forced: you are notified, nothing installs without your action, and nothing ever installs mid-session. (Excel must be restarted to load a new RTD server.) Each update is cryptographically verified before it runs.

## I have a bug to report / a feature to request.

Open an [Issue](https://github.com/StreamXLS/streamxls/issues) using the appropriate template. For binary-product bugs, reproduction steps with the `=RTD(...)` formula, the contract / account context, and TWS / Excel / Windows versions help triage. For account or license issues, email [support@streamxls.com](mailto:support@streamxls.com) instead — those usually involve details that shouldn't be public.

## I want to integrate StreamXLS into another product (white-label / OEM).

Email [sales@streamxls.com](mailto:sales@streamxls.com).
