# Examples

Excel workbook examples demonstrating `=RTD(...)` usage patterns against StreamXLS.

## Status: placeholder

This directory will be populated alongside the first signed Releases drop. Planned contents:

- **`portfolio-view.xlsx`** — multi-tab workbook with Status, Accounts, Positions, Orders, and a Market Data chain. Mirrors the `PortfolioView` tab shown in the 72-second demo video.
- **`scratch.xlsx`** — minimal one-tab workbook for first-run smoke testing. Single `=RTD(...)` formula resolving to a top-of-book quote.
- **`option-chain.xlsx`** — 40-strike × 2-side chain with greeks, comparable to the chain in Beat 1 of the demo video.

Each example will include a comment row at the top explaining the `=RTD(...)` topic-string format used.

## Why these aren't included in this initial commit

The example workbooks ship with the installer; publishing them ahead of the binary product would reference a ProgID that no installed COM component yet provides. Once the binary lands, the workbooks land alongside.

If you need a working example before then, the 72-second demo video shows real workbooks in use:

- Narrated: [youtu.be/Jq7d6iHN2R0](https://youtu.be/Jq7d6iHN2R0)
- Unnarrated: [youtu.be/YczPzTFefBs](https://youtu.be/YczPzTFefBs)
