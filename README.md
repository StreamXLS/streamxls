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

## Documentation

**All documentation lives at [streamxls.com/docs](https://streamxls.com/docs).**

- [Get Started](https://streamxls.com/docs) — requirements, install and the 30-day trial, TWS / IB Gateway API setup, your first formulas, troubleshooting.
- [Reference](https://streamxls.com/docs-reference) — the complete `=RTD()` grammar: topic families, contract specification, every field.
- [Configuration](https://streamxls.com/docs-config) — environment variables, logging, market-data type, the TWS-API version floor.

## Downloads

- **[Releases](https://github.com/StreamXLS/streamxls/releases)** — signed, per-user installer (no admin rights required), with a published SHA-256. The installer is signed by StreamXLS LLC; SmartScreen may still warn on first run at launch volume — [streamxls.com/download](https://streamxls.com/download) documents how to verify the publisher and checksum.
- **[Demo workbook](examples/StreamXLS.xlsm)** — every feature illustrated in one workbook, so you can start without reading further. The recommended copy is the one the installer places (Start menu → *StreamXLS demo workbook*); a copy downloaded from here or from [streamxls.com/StreamXLS.xlsm](https://streamxls.com/StreamXLS.xlsm) carries Windows' Mark-of-the-Web, so Excel blocks its (signed) macros until you right-click the file → Properties → **Unblock**.

## What this repository is

StreamXLS is **closed-source commercial software** by StreamXLS LLC; this repository does not contain source. It exists to host verifiable artifacts — signed installer releases and the demo workbook — and as a feedback surface. The 30-day, full-featured trial is keyless: it starts automatically the first time StreamXLS is used in Excel.

- Bug reports and feature requests: [GitHub Issues](https://github.com/StreamXLS/streamxls/issues).
- Commercial inquiries (source license, integration partnership, OEM redistribution): a [GitHub Discussion](https://github.com/StreamXLS/streamxls/discussions) in the **Commercial inquiries** category, or [sales@streamxls.com](mailto:sales@streamxls.com).
- Customer support: [support@streamxls.com](mailto:support@streamxls.com).

Repository contents are © 2026 StreamXLS LLC, all rights reserved (see [LICENSE](LICENSE)); the software product ships under its own End-User License Agreement — see [streamxls.com/eula](https://streamxls.com/eula).

---

*StreamXLS is not affiliated with, endorsed by, or sponsored by Interactive Brokers. Trader Workstation, the TWS API, and Excel are products of their respective owners; their use is governed by their respective licenses. StreamXLS does not provide investment advice or recommendations.*
