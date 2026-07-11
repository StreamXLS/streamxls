# Design Notes

Selected design decisions and the rationale behind them. Useful context for engineers evaluating the product or considering a source license.

## Why RTD and not DDE / ActiveX / a Python add-in?

- **DDE** is Microsoft-deprecated for new development and has well-known data-loss characteristics under load. IBKR's TWS DDE pipe predates the RTD-as-replacement story and remains usable but is not the recommended path for new builds.
- **ActiveX-on-worksheet** integrations require a hosted control object in the workbook, with the threading and persistence consequences that follow. RTD is data-only, formula-addressable, and survives `Save → Close → Reopen` without UI-state baggage.
- **Python add-ins** (xlwings, PyXLL) require a Python runtime alongside Excel and pull spreadsheets out of the COM-only deployment model that most trading-desk IT departments are tooled for.

The RTD protocol is the one Excel-native streaming surface that all of: (a) speaks COM, (b) addresses by formula, (c) survives Excel's UI-priority quirks if the server is built for them, (d) is documented by Microsoft as the streaming successor to DDE.

## Why a separate product rather than a patch to IBKR's sample?

The IBKR Excel RTD sample at `C:\TWS API\samples\Excel\TwsRtdServer.sln` is a reference implementation of the RTD contract. It demonstrates the mechanism. It is — by IBKR's own description — not engineered for production deployment, and the gap is structural, not a missing-feature list:

- The sample's topic schema is Market Data only. Adding Accounts / Positions / Order monitoring / Order staging families is not a patch — it is a new dimension on the subscription map, with its own caching, filter, and refresh semantics.
- Modal-dialog tolerance is not a single code change; it is the consequence of decoupling the upstream cache from Excel's polling cadence. That decoupling shape decision propagates through the whole component.
- Automatic reconnection with subscription re-establishment and non-volatile-field preservation across the disconnect window is a project-wide invariant, not a localised feature.
- Test coverage sized for live-trading deployment is a project posture, not a patch. (The suite currently exceeds 2,400 automated tests.)

A ground-up build was the cheaper path to a hardened RTD server than a patch series against the sample. For workbooks built against the sample, migration is mechanical: the COM contract is identical and the formula syntax is compatible, so re-pointing a workbook's formulas from the sample's ProgID (`Tws.TwsRtdServerCtrl`) to `Tws.Rtd` is the whole change.

## Per-Excel-process connection model

The decision to give each Excel process its own TWS API connection — rather than multiplex a single connection across processes — has two motivations:

1. **TWS API client-ID semantics.** A single client connection is logically a single agent to TWS. Multiplexing N agents over one connection requires the server to maintain an N×K subscription map and demultiplex callbacks to the right Excel instance, with all the pacing-limit consequences amplified by N.
2. **Failure-isolation.** A misbehaving formula or a hung Excel instance does not stall API delivery to the other instance.

The cost is N API client connections. For typical desktop use (1–3 Excel instances) this is well within IBKR's documented limits.

## Subscription deduplication within a process

Within a single Excel process, the opposite optimisation applies: every `=RTD()` cell that references the same logical topic shares one TWS subscription. The deduplication key is the canonicalised topic-string tuple, not the cell address.

Without this, a workbook with a single `=RTD(...)` formula copy-dragged across 200 rows of a `SPY` chain would issue 200 subscription requests. With it: one.

`ActiveTopicCount` on the Status tab surfaces the deduplicated count directly so that the user can see (and trust) what is going out over the wire.

## Closed-source posture

The repository you are reading is documentation, examples, and binary releases. The source itself stays private. The rationale: source distribution would commit the project to a community-support model (issue triage on patches, downstream-fork compatibility, etc.) that is not the product surface intended for the current customer set. Commercial source-license terms for trading-firm in-house use are available case-by-case: [sales@streamxls.com](mailto:sales@streamxls.com).
