# StreamXLS

**Stream Interactive Brokers data into Excel with native `=RTD()` formulas.**
Market data, account values, positions with live P&L, order monitoring, and order staging — a production-grade RTD engine built for live trading, where the IBKR sample is a teaching artifact.

```excel
=RTD("Tws.Rtd", , "status", "IsConnected")
=RTD("Tws.Rtd", , "AAPL", "MarketPrice")
=RTD("Tws.Rtd", , "account", "U1234567", "NetLiquidation")
=RTD("Tws.Rtd", , "positions", "*", "SymbolsCsv")
```

Watch the [72-second demo](https://vimeo.com/1191256765).

*StreamXLS is closed-source commercial software; this repository hosts its signed installer releases, demo workbook, and issue tracker — [more below](#what-this-repository-is).*

## Documentation

**The curated documentation site is [streamxls.com/docs](https://streamxls.com/docs).** The complete reference documentation also lives in this repository under [`docs/`](docs/) — start with the [index](docs/README.md):

- [quickstart.md](docs/quickstart.md) — start here: your first `=RTD()` formulas across status, account, market-data, positions, and orders, plus the market-data version floor.
- [detailed-instructions.md](docs/detailed-instructions.md) — contract specification in depth (options, futures, forex, combos, ConID), order staging, and cell states & lifecycle.
- [reference.md](docs/reference.md) — the exhaustive reference: every topic, market-data and order field, status value, account key, configuration key, and cell-error value.

The curated website surface links out to these and remains the recommended entry point:

- [Get Started](https://streamxls.com/docs) — requirements, install and the 30-day trial, TWS / IB Gateway API setup, your first formulas, troubleshooting.
- [Reference](https://streamxls.com/docs-reference) — the complete `=RTD()` grammar: topic families, contract specification, every field.
- [Configuration](https://streamxls.com/docs-config) — environment variables, logging, market-data type, the TWS-API version floor.

## Downloads

- **[Releases](https://github.com/StreamXLS/streamxls/releases)** — signed, per-user installer (no admin rights required). Each installer's SHA-256 is published with the release (the asset's digest) and at [streamxls.com/download](https://streamxls.com/download), which also documents how to verify the publisher. SmartScreen may still warn on first run at launch volume.
- **[Demo workbook](examples/StreamXLS.xlsm)** — every feature illustrated in one workbook, so you can start without reading further. The recommended copy is the one the installer places (Start menu → *StreamXLS demo workbook*); a copy downloaded from here or from [streamxls.com/StreamXLS.xlsm](https://streamxls.com/StreamXLS.xlsm) carries Windows' Mark-of-the-Web, so Excel blocks its (signed) macros until you right-click the file → Properties → **Unblock**.

## What this repository is

StreamXLS is **closed-source commercial software** by StreamXLS LLC; this repository does not contain source. It exists to host verifiable artifacts — signed installer releases and the demo workbook — and as a feedback surface. The 30-day, full-featured trial is keyless: it starts automatically the first time StreamXLS is used in Excel. Subscriptions start at $590/year — current pricing at [streamxls.com/buy](https://streamxls.com/buy).

- Bug reports and feature requests: [GitHub Issues](https://github.com/StreamXLS/streamxls/issues). Account or license issues: email support instead — those usually involve details that shouldn't be public.
- Commercial inquiries (source license, integration partnership, OEM redistribution) and pre-sales questions: [sales@streamxls.com](mailto:sales@streamxls.com).
- Customer support: [support@streamxls.com](mailto:support@streamxls.com) — we aim to respond within one US business day.

StreamXLS is actively developed and commercially supported; the current release is always available on the [Releases](https://github.com/StreamXLS/streamxls/releases) page. Releases publish here and reach installed copies through the product's update channel (updates are offered, never forced — nothing installs without your action); release notes call out changes, including any change to the topic-string schema.

Repository contents are © 2026 StreamXLS LLC, all rights reserved (see [LICENSE](LICENSE)); the software product ships under its own End-User License Agreement — see [streamxls.com/eula](https://streamxls.com/eula).

---

*StreamXLS is not affiliated with, endorsed by, or sponsored by Interactive Brokers. Trader Workstation, the TWS API, and Excel are products of their respective owners; their use is governed by their respective licenses. StreamXLS does not provide investment advice or recommendations.*
