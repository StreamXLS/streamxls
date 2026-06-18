# StreamXLS

**A production-grade Excel RTD server for Interactive Brokers TWS.**

A drop-in replacement for the Excel RTD server that ships in the IBKR TWS API distribution, with error handling, reconnection, multi-cell deduplication, and a topic schema that covers accounts, positions, orders, and order submission alongside market data.

---

## What this is

Interactive Brokers ships a sample Excel RTD application — also named `TwsRtdServer` — under `C:\TWS API\samples\Excel\` as part of the TWS API distribution. The IBKR sample is the right starting point for understanding how the Excel RTD contract plugs into TWS. [IBKR's Excel RTD documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/excel-rtd/) explicitly characterises the sample as a learning aid, noting that it is

> not intended to be used as production level trading tools

and that the sample applications do not have robust error handling.

**This is a separate, hardened C# implementation of the same `IRtdServer` COM contract** — built for live-trading deployment. The COM contract is identical, the formula syntax is identical, and an [optional registry alias](docs/install.md#optional-legacy-progid-alias) maps the IBKR sample's legacy ProgID `Tws.TwsRtdServerCtrl` onto this server's CLSID so existing workbooks built against the IBKR sample keep working without modification.

What that hardening covers, concretely:

- **Topic schema beyond Market Data.** Account values (per-currency), Positions (with streaming P&L), Orders (with order monitoring across all clients), and **Order submission** are all exposed as `=RTD()` formulas alongside Market Data. The schema the IBKR sample doesn't cover.
- **Order submission from Excel.** A dedicated `SendOrder` topic family lets a formula submit an order as the side-effect of subscribing — the cell returns a status string (`Sending` → `Sent`, or `SendOrder Error: <message>`).
- **Automatic reconnection with subscription re-establishment.** When TWS restarts mid-session, the server reconnects automatically and re-subscribes to every previously-active topic. Non-volatile fields preserve their last-known value through the disconnect window rather than going `#N/A`.
- **Excel UI events don't block subscription updates.** Microsoft Excel pauses its RTD pull while a modal dialog is open, while a cell is in edit mode, or while Excel is otherwise busy — IBKR's docs describe this as inherent to Excel's design as a trading application. StreamXLS maintains its internal cache independently of Excel's polling cadence, so when Excel returns to a ready state it reads the latest cached value rather than an update that arrived while it was busy and is now lost.
- **Multi-instance: per-process TWS connection.** Each `EXCEL.EXE` process that loads StreamXLS opens its own TWS API client connection with an auto-allocated client ID (configurable via `TWS_RTD_CLIENT_ID`). No cross-instance contention; no shared subscription map.
- **Subscription deduplication within a process.** Duplicate `=RTD()` cells referencing the same logical topic share a single upstream subscription. Copy-drag a formula across 200 rows of a chain → one subscription, not 200. Visible to the user as `ActiveTopicCount` on the Status tab.

## How a formula looks

StreamXLS registers itself with the ProgID `Tws.Rtd` (shorter than the IBKR sample's `Tws.TwsRtdServerCtrl`; case-insensitive in formulas). Some examples:

```excel
=RTD("Tws.Rtd", , "status", "IsConnected")
=RTD("Tws.Rtd", , "AAPL@SMART", "Last")
=RTD("Tws.Rtd", , "account", "U1234567", "NetLiquidation")
=RTD("Tws.Rtd", , "position", "U1234567", "AAPL@SMART", "MarketValue")
=RTD("Tws.Rtd", , "positions", "*", "SymbolsCsv")
=RTD("Tws.Rtd", , "orders", "*", "OpenListCsv")
=RTD("Tws.Rtd", , "SendOrder", "sym=AAPL", "side=BUY", "shares=100", "type=LMT", "limit=150.05", "exch=SMART")
```

The full topic-string grammar — Market Data, Accounts, Positions, Orders (monitoring + submission), Status — publishes alongside the first Releases drop. Existing workbooks built against the IBKR sample can be migrated either by replacing `Tws.TwsRtdServerCtrl` with `Tws.Rtd` or by installing the optional legacy-ProgID alias and leaving the formulas alone.

## 72-second demo

A short walkthrough — option chain (80 instruments, 7 fields each, 560 `=RTD()` formulas), a Format Cells dialog held open over the streaming chain, a roundtripped buy order with the new position reflected in Excel as it fills, and two Excel instances each running an independent connection.

Watch on [YouTube](https://youtu.be/Jq7d6iHN2R0) or [Vimeo](https://vimeo.com/1191256765).

## Where this fits relative to IBKR's documented surface

| Path | Surface | Coverage | Production-grade? |
|---|---|---|---|
| **TWS DDE** | Legacy Excel pipe | Market data, limited orders | Legacy / discouraged for new development |
| **IBKR Excel RTD sample** (`C:\TWS API\samples\Excel\TwsRtdServer.{xls,dll}`) | Reference C# RTD server | Market data | Per IBKR docs: learning aid, not production |
| **TWS API client libraries** (C# / Java / Python / `ib_async`) | General-purpose programmatic clients | Full TWS API surface | Yes, but not Excel-native |
| **StreamXLS (this project)** | Hardened RTD server (drop-in replacement for the sample) | Market Data + Accounts + Positions + Order monitoring + Order submission | Yes (Excel-specific surface) |

Relevant IBKR documentation:

- [Excel RTD — IBKR Campus](https://www.interactivebrokers.com/campus/ibkr-api-page/excel-rtd/) — the official Excel RTD page (source of the production-readiness disclaimer and the Excel UI-priority note)
- [TWS API documentation root — IBKR Campus](https://www.interactivebrokers.com/campus/ibkr-api-page/twsapi-doc/) — full TWS API reference

## Target platform

- **Server runtime:** .NET Framework 4.8, COM-registered via `regasm /codebase`.
- **Excel:** Microsoft 365 / 2019+, 32-bit or 64-bit (registration bitness must match Excel's bitness).
- **Windows:** Windows 10 / 11.
- **IBKR side:** TWS or IB Gateway with the socket API enabled. Standard `Configure → API → Settings → Enable ActiveX and Socket Clients` plus trusted-IP setup; no IBKR-side software extensions required.

## Status

- **Source:** closed. Commercial source-license inquiries are welcome — see [`docs/licensing.md`](docs/licensing.md).
- **Binaries:** the server runs in production and is verified-by-use on every trading day. Signed public Releases are pending code-signing certificate provisioning; the first signed `.msi`, with verified install and first-run instructions, publishes alongside that drop.
- **CI:** the GitHub Actions workflow in this repository runs docs lint and link checks on each push. The binary build runs against the private source repository and is gated on the same signing-cert milestone.

The directories `examples/`, `screenshots/`, and `releases/` currently hold placeholder READMEs and will populate alongside the first signed Releases drop.

## License

The contents of this repository are © 2026 StreamXLS LLC, all rights reserved (see [LICENSE](LICENSE)). **The StreamXLS software product is closed-source commercial software**, distributed under a separate End-User License Agreement that ships with the installer. See [`docs/licensing.md`](docs/licensing.md).

## Repository contents

- [`docs/`](docs/) — [architecture overview](docs/architecture.md), [design notes](docs/design-notes.md), [FAQ](docs/FAQ.md), [licensing posture](docs/licensing.md), [install + first run](docs/install.md)
- [`examples/`](examples/) — Excel workbook examples (placeholder)
- [`screenshots/`](screenshots/) — UI screenshots (placeholder)
- [`releases/`](releases/) — release-notes landing (signed binaries publish to the [Releases](https://github.com/StreamXLS/streamxls/releases) tab; placeholder until cert)
- [`.github/`](.github/) — issue templates, PR template, CI workflows

## Issues & feedback

Bug reports and feature requests are welcome via [GitHub Issues](https://github.com/StreamXLS/streamxls/issues). For commercial inquiries (source license, integration partnership, OEM redistribution), please open a [GitHub Discussion](https://github.com/StreamXLS/streamxls/discussions) in the **Commercial inquiries** category.
