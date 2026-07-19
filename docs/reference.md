# StreamXLS RTD reference

Complete reference for the RTD vocabulary the StreamXLS engine serves: topics, contract keys, StageOrder write keys, order and position fields, status and metadata fields, account keys, configuration keys, and cell-error semantics. Every RTD formula takes the form `=RTD("tws.rtd", , "<tokens…>")`.

**Generated file — do not hand-edit.** This file is produced from engine ground truth by `ReferenceDocTests` in the test suite. Field and key NAMES and their COVERAGE are code-asserted (every name the code yields is either described here or explicitly skipped-with-reason, enforced by partition tests); the DESCRIPTIONS are authored and reviewed, not code-asserted. To change the file, edit the generator's description maps and re-run the suite with the environment variable `STREAMXLS_UPDATE_REFERENCE_DOCS=1`, which rewrites this file.

**Range/limit caveat:** numeric ranges, defaults, and limits quoted in prose (e.g. configuration value ranges) are authored literals — they are NOT asserted against the code's own range constants and can drift; treat them as documentation, and the engine's validation messages as authoritative. Sections marked *(authored)* carry prose whose content the ratchet does not verify.

## 1. Topic and category tokens

A category token as the first non-connection argument selects the request type. Market-data topics carry no category token — a contract plus a field name routes to market data.

| Token | Request type |
| --- | --- |
| `ACCOUNT` | Account value for one account (e.g. NetLiquidation). |
| `POSITION` | A single position (one contract, one or more accounts). |
| `POSITIONS` | A positions list (SymbolsCsv / ConIdCsv / PositionsChangedUtc). |
| `ORDER` | A single order addressed by permId. |
| `ORDERS` | An orders list (ListCsv / OpenListCsv). |
| `STATUS` | A connection status field addressed by name — the closed status-field set (including the four UPDATE_* breadcrumb fields it shares with metadata). Metadata fields (VERSION, LICENSE_*, TWSAPI_*, …) are addressed by BARE name, not through this token. |
| `SENDORDER` | Stage an order in TWS (synonym of StageOrder). |
| `STAGEORDER` | Stage an order in TWS (the canonical name; SENDORDER is an accepted synonym). |

### Resolution order

*(authored — not code-asserted.)* The request type is resolved by scanning the argument tokens in this order, first match wins: (1) a bare metadata field name; (2) an explicit category token from the table above; (3) a bare option-definition field name; (4) a bare name-addressable status field. If none match, the request is treated as market data. `MARKETDATATYPE` is deliberately excluded from the bare-status fall-through so a per-contract `MARKETDATATYPE` stays market data; reach the connection-level status field via the explicit `status` token.

## 2. Market-data fields

Request with a contract and a field name, e.g. `=RTD("tws.rtd", , "AAPL", "BID")`. Field names are case-insensitive. The full supported set is enumerated below by group.

### Market data — core price and size

| Field | Description |
| --- | --- |
| `BID` | Best (highest) bid price. |
| `BIDSIZE` | Size available at the best bid. |
| `ASK` | Best (lowest) ask price. |
| `ASKSIZE` | Size available at the best ask. |
| `LAST` | Last traded price. |
| `LASTSIZE` | Size of the last trade. |
| `OPEN` | Today's opening price. |
| `HIGH` | Today's session high. |
| `LOW` | Today's session low. |
| `CLOSE` | Previous session's closing price. |
| `VOLUME` | Cumulative traded volume for the session. |
| `BIDEXCH` | Exchange(s) posting the best bid. |
| `ASKEXCH` | Exchange(s) posting the best ask. |
| `LASTEXCH` | Exchange of the last trade. |
| `LASTTIME` | Timestamp of the last trade. |
| `HALTED` | Trading-halt indicator (0 = not halted; >0 = halted). |

### Market data — odd-lot quotes

| Field | Description |
| --- | --- |
| `ODDLOTBID` | Odd-lot best bid price. |
| `ODDLOTASK` | Odd-lot best ask price. |
| `ODDLOTBIDSIZE` | Size at the odd-lot best bid. |
| `ODDLOTASKSIZE` | Size at the odd-lot best ask. |
| `ODDLOTBIDEXCH` | Exchange posting the odd-lot best bid. |
| `ODDLOTASKEXCH` | Exchange posting the odd-lot best ask. |

### Market data — derived prices

| Field | Description |
| --- | --- |
| `MARKETPRICE` | Derived mid: (BID+ASK)/2 when both present, else LAST, else CLOSE; blank until a source is available. |
| `LASTORCLOSE` | LAST when available, otherwise CLOSE. |

### Market data — data-tier indicators

| Field | Description |
| --- | --- |
| `MARKETDATATYPE` | Market-data tier TWS is actually serving this contract (1 real-time, 2 frozen, 3 delayed, 4 delayed-frozen); #N/A until reported. |
| `ISDELAYED` | 1 when the served tier is delayed (3 or 4), 0 when real-time/frozen (1 or 2); #N/A until reported. |

### Market data — option computation (greeks)

| Field | Description |
| --- | --- |
| `BIDTICKATTRIB` | Tick attributes for the bid option-computation snapshot. |
| `BIDIMPLIEDVOL` | Implied volatility from the bid option computation. |
| `BIDDELTA` | Option delta from the bid computation. |
| `BIDOPTPRICE` | Model option price from the bid computation. |
| `BIDPVDIVIDEND` | Present value of dividends from the bid computation. |
| `BIDGAMMA` | Option gamma from the bid computation. |
| `BIDVEGA` | Option vega from the bid computation. |
| `BIDTHETA` | Option theta from the bid computation. |
| `BIDUNDPRICE` | Underlying price used in the bid computation. |
| `ASKTICKATTRIB` | Tick attributes for the ask option-computation snapshot. |
| `ASKIMPLIEDVOL` | Implied volatility from the ask option computation. |
| `ASKDELTA` | Option delta from the ask computation. |
| `ASKOPTPRICE` | Model option price from the ask computation. |
| `ASKPVDIVIDEND` | Present value of dividends from the ask computation. |
| `ASKGAMMA` | Option gamma from the ask computation. |
| `ASKVEGA` | Option vega from the ask computation. |
| `ASKTHETA` | Option theta from the ask computation. |
| `ASKUNDPRICE` | Underlying price used in the ask computation. |
| `LASTTICKATTRIB` | Tick attributes for the last option-computation snapshot. |
| `LASTIMPLIEDVOL` | Implied volatility from the last option computation. |
| `LASTDELTA` | Option delta from the last computation. |
| `LASTOPTPRICE` | Model option price from the last computation. |
| `LASTPVDIVIDEND` | Present value of dividends from the last computation. |
| `LASTGAMMA` | Option gamma from the last computation. |
| `LASTVEGA` | Option vega from the last computation. |
| `LASTTHETA` | Option theta from the last computation. |
| `LASTUNDPRICE` | Underlying price used in the last computation. |
| `MODELTICKATTRIB` | Tick attributes for the model option-computation snapshot. |
| `MODELIMPLIEDVOL` | Implied volatility from the model option computation. |
| `MODELDELTA` | Option delta from the model computation. |
| `MODELOPTPRICE` | Model option price from the model computation. |
| `MODELPVDIVIDEND` | Present value of dividends from the model computation. |
| `MODELGAMMA` | Option gamma from the model computation. |
| `MODELVEGA` | Option vega from the model computation. |
| `MODELTHETA` | Option theta from the model computation. |
| `MODELUNDPRICE` | Underlying price used in the model computation. |

### Market data — generic ticks

| Field | Description |
| --- | --- |
| `AUCTIONVOLUME` | Auction volume. |
| `AUCTIONIMBALANCE` | Auction imbalance. |
| `AUCTIONPRICE` | Auction (indicative) price. |
| `REGULATORYIMBALANCE` | Regulatory imbalance. |
| `PLPRICE` | Current P&L mark price. |
| `CREDITMANMARKPRICE` | CreditMan mark price. |
| `CREDITMANSLOWMARKPRICE` | CreditMan slow mark price. |
| `CALLOPTIONVOLUME` | Call option volume. |
| `PUTOPTIONVOLUME` | Put option volume. |
| `CALLOPTIONOPENINTEREST` | Call option open interest. |
| `PUTOPTIONOPENINTEREST` | Put option open interest. |
| `OPTIONHISTORICALVOL` | Option historical volatility. |
| `RTHISTORICALVOL` | Real-time historical volatility (futures only). |
| `OPTIONIMPLIEDVOL` | Option implied volatility. |
| `INDEXFUTUREPREMIUM` | Index future premium (indices only). |
| `SHORTABLE` | Shortability indicator (higher = easier to borrow). |
| `SHORTABLESHARES` | Number of shortable shares available. |
| `ESTIMATEDIPOMIDPOINT` | Estimated IPO midpoint price. |
| `FINALIPOLAST` | Final IPO opening/last price. |
| `TRADECOUNT` | Trade count for the session. |
| `TRADERATE` | Trades per minute. |
| `VOLUMERATE` | Volume per minute. |
| `LASTRTHTRADE` | Last regular-trading-hours trade price. |
| `IBDIVIDENDS` | IB dividend estimates. |
| `BONDMULTIPLIER` | Bond multiplier. |
| `AVGVOLUME` | Average daily volume. |
| `WEEK13HI` | 13-week high. |
| `WEEK13LO` | 13-week low. |
| `WEEK26HI` | 26-week high. |
| `WEEK26LO` | 26-week low. |
| `WEEK52HI` | 52-week high. |
| `WEEK52LO` | 52-week low. |
| `SHORTTERMVOLUME3MIN` | 3-minute short-term volume. |
| `SHORTTERMVOLUME5MIN` | 5-minute short-term volume. |
| `SHORTTERMVOLUME10MIN` | 10-minute short-term volume. |
| `FUTURESOPENINTEREST` | Futures open interest. |
| `AVGOPTVOLUME` | Average option volume. |
| `ETFNAVLAST` | ETF net asset value, last. |
| `ETFFROZENNAVLAST` | ETF NAV, frozen last. |
| `ETFNAVHIGH` | ETF NAV session high. |
| `ETFNAVLOW` | ETF NAV session low. |

### Market data — delayed fields

| Field | Description |
| --- | --- |
| `DELAYEDBID` | Delayed best bid price. |
| `DELAYEDASK` | Delayed best ask price. |
| `DELAYEDLAST` | Delayed last trade price. |
| `DELAYEDBIDSIZE` | Delayed best-bid size. |
| `DELAYEDASKSIZE` | Delayed best-ask size. |
| `DELAYEDLASTSIZE` | Delayed last-trade size. |
| `DELAYEDHIGH` | Delayed session high. |
| `DELAYEDLOW` | Delayed session low. |
| `DELAYEDVOLUME` | Delayed cumulative volume. |
| `DELAYEDCLOSE` | Delayed previous close. |
| `DELAYEDOPEN` | Delayed session open. |
| `DELAYEDLASTTIMESTAMP` | Delayed last-trade timestamp. |
| `DELAYEDHALTED` | Delayed halt indicator. |

### Disconnect behavior (the #N/A class)

On a TWS disconnect, market-data cells follow one of three rules, keyed off the CONFIGURED market-data tier (`TWS_RTD_MARKET_DATA_TYPE`), not the per-request served tier.

- **Quote family → #N/A in every tier.** A standing bid/ask dies with the session that carried it, so every quote cell fails loud on disconnect regardless of tier: `ASK`, `ASKEXCH`, `ASKSIZE`, `BID`, `BIDEXCH`, `BIDSIZE`, `DELAYEDASK`, `DELAYEDASKSIZE`, `DELAYEDBID`, `DELAYEDBIDSIZE`, `ODDLOTASK`, `ODDLOTASKEXCH`, `ODDLOTASKSIZE`, `ODDLOTBID`, `ODDLOTBIDEXCH`, `ODDLOTBIDSIZE`.
- **`LAST` / `DELAYEDLAST` → tier-dependent.** Under the Realtime (1) and Delayed (3) tiers a stale last reads `#N/A` (you did not opt into last-known values); under the Frozen (2) and DelayedFrozen (4) tiers it keeps its last-seen value (frozen tiers are the explicit opt-in to last-known values).
- **Everything else → keep-last.** CLOSE, MARKETPRICE, LASTORCLOSE, VOLUME, the option greeks, and the last-trade attributes (LASTSIZE / LASTEXCH / LASTTIME) keep their last value regardless of tier — they are facts about a completed trade, not standing-quote state.

## 3. Contract specification keys

Contracts can be given as a simple ticker (`AAPL`), a slash form (`AAPL@SMART/STK/USD`), a pipe form (`SPY|OPT|20251219|C|450`), an FX pair (`EUR.USD/CASH`), or explicit `key=value` tokens. The canonical keys and their accepted aliases are below (case-insensitive). See the guide for worked examples of each form.

| Canonical key | Aliases | Description |
| --- | --- | --- |
| `symbol` | `sym` | Underlying symbol. |
| `sectype` | `sec`, `securitytype` | Security type (STK, OPT, FUT, FOP, CASH, IND, …). |
| `exchange` | `exch` | Routing exchange (SMART for the universal router). |
| `primaryexch` | `prim`, `primary`, `primaryexchange`, `primexch` | Primary listing exchange (disambiguates SMART-routed symbols). |
| `currency` | `cur`, `curr` | Trading currency. |
| `expiry` | `exp`, `expiration`, `lasttradedate` | Expiration date (YYYYMM or YYYYMMDD) for options/futures. |
| `strike` | `strikeprice` | Option strike price. |
| `right` | `optiontype`, `putcall` | Option right: C (call) or P (put). |
| `multiplier` | `mult` | Contract multiplier. |
| `localsymbol` | `loc`, `local` | Exchange-specific local symbol. |
| `tradingclass` | `class`, `tc` | Trading class. |
| `conid` | `contractid` | TWS contract id (authoritative when present). |

Defaults when a symbol-only request omits them, for market-data and option-definition contract parsing: security type `STK`, exchange `SMART`, currency `USD`. A position-topic contract defaults the same security type and currency but deliberately leaves the exchange UNCONSTRAINED (blank), so a symbol-only spec matches the position on whatever exchange TWS reports rather than being pinned to `SMART`. When a `conid` is supplied the security type / exchange / currency are left to TWS.

## 4. StageOrder write keys

`=RTD("tws.rtd", , "StageOrder", "symbol=AAPL", "action=BUY", "quantity=100", "type=LMT", "limit=150")` stages an order in TWS. Every key is `key=value`. An unrecognized `key=value` token fails loud (a dropped key could stage an order you did not describe); a token that follows the StageOrder token but contains no `=` — anything other than a connection token — is currently IGNORED rather than rejected, so always write `key=value`. The recognized keys, grouped by the logical field they set:

| Keys | Description |
| --- | --- |
| `symbol`, `sym` | Underlying symbol (required). |
| `action`, `side` | BUY or SELL (required). |
| `quantity`, `shares`, `qty`, `size` | Order quantity, integer > 0 (required). |
| `type` | Order type, required; passed to TWS uppercased (e.g. LMT, MKT, STP, STP LMT, TRAIL, TRAIL LIMIT). Not an engine whitelist: LMT/STP LMT require limit and the stop/trailing keys constrain the stop family, but an unlisted type is TWS's to accept or reject. |
| `limit` | Limit price (required for LMT and STP LMT). |
| `stop`, `aux`, `stopprice` | Stop / auxiliary trigger price (STP, STP LMT, TRAIL, TRAIL LIMIT). |
| `trailingpercent` | Trailing percent (TRAIL, TRAIL LIMIT). |
| `exchange`, `exch` | Routing exchange (defaults to SMART). |
| `account` | IB account to stage into. |
| `fagroup` | Financial-advisor allocation group. |
| `algostrategy`, `algo` | Algo strategy name. |
| `algoparams` | Algo parameters as pipe-delimited tag=value pairs. |
| `tif` | Time in force (see grammars). |
| `outsiderth` | Allow fills outside regular trading hours (boolean). |
| `goodtilldate`, `gtd` | Good-till date/time (required with tif=GTD). |
| `goodaftertime`, `gat` | Good-after date/time. |
| `hidden` | Hidden order flag (boolean). |
| `display`, `displaysize` | Iceberg display size (integer > 0). |
| `allornone`, `aon` | All-or-none flag (boolean). |
| `minqty` | Minimum fill quantity (integer > 0). |
| `ocagroup` | One-cancels-all group name (with ocatype). |
| `ocatype` | One-cancels-all type (with ocagroup; see grammars). |
| `park`, `parked`, `saved` | park=true stages an invisible ticket (Transmit=false) instead of the default deactivated order. |
| `tag`, `nonce`, `seq`, `submit`, `clienttag` | User order tag (composed into OrderRef; see composition). |

### Synonym groups

Within each group the keys are synonyms — supply at most one; supplying two of the same group fails loud rather than silently dropping a value.

- `symbol` = `sym`
- `action` = `side`
- `quantity` = `shares` = `qty` = `size`
- `exchange` = `exch`
- `algostrategy` = `algo`
- `tag` = `nonce` = `seq` = `submit` = `clienttag`
- `park` = `parked` = `saved`
- `stop` = `aux` = `stopprice`
- `goodtilldate` = `gtd`
- `goodaftertime` = `gat`
- `display` = `displaysize`
- `allornone` = `aon`

### Value grammars

- **Booleans** (`park`, `outsiderth`, `hidden`, `allornone`): `TRUE`/`1`/`YES` or `FALSE`/`0`/`NO`; anything else fails loud.
- **Time-in-force** (`tif`): one of `DAY`, `GTC`, `IOC`, `FOK`, `OPG`, `GTD`. `GTD` requires `goodtilldate`, and `goodtilldate` requires `tif=GTD`.
- **OCA type** (`ocatype`): `1`, `2`, `3` (1 = cancel remaining with block, 2 = reduce remaining with block, 3 = reduce remaining without block); `ocagroup` and `ocatype` are required together.
- **Date/time** (`goodtilldate`, `goodaftertime`): `YYYYMMDD [HH:MM:SS [TZ]]`, validated as a real calendar date/time.
- **Prices** (`limit`, `stop`): `limit` is required and must be `> 0` for `LMT` and `STP LMT`; `stop` (aliases `aux` / `stopprice`), when supplied, must be a finite decimal `> 0`. `STP` and `STP LMT` REQUIRE `stop`; `stop` is accepted only for `STP`, `STP LMT`, `TRAIL`, `TRAIL LIMIT`.
- **Trailing** (`trailingpercent`): when supplied, a finite decimal in `(0, 100]`; accepted only for `TRAIL` / `TRAIL LIMIT`. `TRAIL` requires EXACTLY ONE of `stop` (the trailing amount) or `trailingpercent`.

### Capability boundary

*(authored — code-verified at review.)* StageOrder handles US-stock, USD, single-leg orders only (`STK`/`USD` are hardcoded); `sec`/`cur`/`exp`/`strike`/`right`/`conid` and combo staging are not accepted. There is no cancel or replace verb from a formula: a staged order is placed in TWS with auto-transmit suppressed and reaches the market only when a human transmits it there; all subsequent modification, cancellation, and transmission happen in TWS. By default the order is staged deactivated (visible in TWS order lists, survives a TWS restart); `park=true` stages it as an invisible order-entry ticket instead.

### Order reference composition

*(authored.)* The `tag` key (aliases `nonce`, `seq`, `submit`, `clienttag`) is preserved verbatim and tail-composed with an engine identity token into the order's OrderRef as `<tag>|SXLS:<token>` (bare `SXLS:<token>` when no tag). The engine token at the tail is how a whole-day order replay is matched back to the exact cell that placed it; the composed reference is capped at 64 characters and an over-length order fails loud before placement.

## 5. Order read fields

`=RTD("tws.rtd", , "order", "<permId>", "<field>")` reads one field of one order (addressed by permId; the field defaults to STATUS). The vocabulary is a closed set — an unknown field fails loud. Canonical fields:

| Field | Aliases | Description |
| --- | --- | --- |
| `STATUS` | — | Current order status (see section 7). |
| `FILLED` | — | Filled quantity. |
| `REMAINING` | — | Remaining (unfilled) quantity. |
| `AVGFILLPRICE` | — | Average fill price. |
| `WHYHELD` | — | Reason the order is held, if any. |
| `WARNINGTEXT` | — | TWS warning text for the order. |
| `PERMID` | — | Permanent order id (stable across sessions). |
| `ORDERID` | — | Client order id. |
| `PARENTID` | — | Parent order id (bracket/child orders). |
| `CLIENTID` | — | API client id that placed the order. |
| `ACCOUNT` | — | IB account code. |
| `ACTION` | `SIDE` | BUY or SELL. |
| `QUANTITY` | `TOTALQUANTITY` | Total order quantity. |
| `ORDERTYPE` | `TYPE` | Order type (LMT, MKT, STP, …). |
| `LMTPRICE` | `LIMITPRICE` | Limit price. |
| `AUXPRICE` | `STOPPRICE` | Auxiliary / stop price. |
| `TIF` | `TIMEINFORCE` | Time in force. |
| `TRAILSTOPPRICE` | — | Trailing stop price. |
| `TRAILINGPERCENT` | — | Trailing percent. |
| `OCAGROUP` | — | One-cancels-all group name. |
| `OCATYPE` | — | One-cancels-all type. |
| `DISPLAYSIZE` | — | Iceberg display size. |
| `OUTSIDERTH` | — | Allow fills outside regular trading hours. |
| `HIDDEN` | — | Hidden order flag. |
| `GOODAFTERTIME` | — | Good-after time. |
| `GOODTILLDATE` | — | Good-till date. |
| `ALLOWPREOPEN` | — | Allow pre-open activation. |
| `SUBMITTER` | — | Order submitter. |
| `MODELCODE` | — | Model code. |
| `FAGROUP` | — | Financial-advisor group. |
| `FAMETHOD` | — | Financial-advisor allocation method. |
| `OPENCLOSE` | — | Open/close indicator. |
| `ALGOSTRATEGY` | — | Algo strategy name. |
| `ALGOPARAMS` | — | Algo parameters (semicolon-delimited tag=value). |
| `ORDREF` | `ORDERREF` | User order tag (engine token un-composed away). |
| `MINQTY` | — | Minimum fill quantity. |
| `PERCENTOFFSET` | — | Percent offset (relative/pegged orders). |
| `DISCRETIONARYAMT` | — | Discretionary amount. |
| `CASHQTY` | — | Cash quantity (monetary order size). |
| `LMTPRICEOFFSET` | — | Limit price offset. |
| `ACTIVESTARTTIME` | — | Order active-start time. |
| `ACTIVESTOPTIME` | — | Order active-stop time. |
| `BLOCKORDER` | — | Block order flag. |
| `SWEEPTOFILL` | — | Sweep-to-fill flag. |
| `ALLORNONE` | — | All-or-none flag. |
| `NOTHELD` | — | Not-held flag. |
| `SOLICITED` | — | Solicited flag. |
| `WHATIF` | — | What-if (margin preview) flag. |
| `INCLUDEOVERNIGHT` | — | Include-overnight flag. |
| `CONID` | — | Contract id. |
| `SYMBOL` | — | Underlying symbol. |
| `SECTYPE` | — | Security type. |
| `EXPIRY` | `LASTTRADEDATE` | Expiration date. |
| `STRIKE` | — | Option strike. |
| `RIGHT` | — | Option right (C/P). |
| `MULTIPLIER` | — | Contract multiplier. |
| `EXCHANGE` | — | Exchange. |
| `PRIMARYEXCHANGE` | — | Primary exchange. |
| `CURRENCY` | — | Currency. |
| `LOCALSYMBOL` | — | Exchange-specific local symbol. |
| `TRADINGCLASS` | — | Trading class. |
| `COMMISSIONANDFEES` | — | Commission and fees. |
| `MINCOMMISSIONANDFEES` | — | Minimum commission and fees. |
| `MAXCOMMISSIONANDFEES` | — | Maximum commission and fees. |
| `COMMISSIONANDFEESCURRENCY` | — | Currency of the commission/fees figures. |
| `MARGINCURRENCY` | — | Currency of the margin figures. |
| `INITMARGINBEFORE` | — | Initial margin before the order. |
| `MAINTMARGINBEFORE` | — | Maintenance margin before the order. |
| `EQUITYWITHLOANBEFORE` | — | Equity-with-loan before the order. |
| `INITMARGINCHANGE` | — | Initial-margin change from the order. |
| `MAINTMARGINCHANGE` | — | Maintenance-margin change from the order. |
| `EQUITYWITHLOANCHANGE` | — | Equity-with-loan change from the order. |
| `INITMARGINAFTER` | — | Initial margin after the order. |
| `MAINTMARGINAFTER` | — | Maintenance margin after the order. |
| `EQUITYWITHLOANAFTER` | — | Equity-with-loan after the order. |
| `REJECTREASON` | — | Rejection reason. |
| `COMPLETEDTIME` | — | Completion time. |
| `COMPLETEDSTATUS` | — | TWS-delivered terminal status string. |
| `FIRSTSEENUTC` | `FIRSTSEEN` | UTC time the engine first saw this order. |
| `LASTUPDATEUTC` | `LASTUPDATE` | UTC time of the most recent change to this order. |

`ORDERSTATUS` was retired (use `STATUS`, the live status field refreshed by every path); the TWS-delivered terminal status string remains available verbatim via `COMPLETEDSTATUS`.

## 6. Deliberately-unsupported order properties

These IBApi order/contract/order-state properties are intentionally NOT surfaced as read fields. Grouped by the shared reason; the reasons come from the engine's census skip lists.

### Unsupported — Order properties

- `Transmit`, `Deactivate`, `PostOnly`, `IgnoreOpenAuction` — Undecoded booleans (engineering review finding): the openOrder decoder never populates these on the legacy binary path (Transmit: on NEITHER path), so a resolver would publish the fabricated ctor default — booleans have no unset sentinel to guard with. Transmit/Deactivate would claim a parked order is transmitted (staging-shape readback must come from our own SendOrder record if ever wanted, not the decoder-reconstructed Order).
- `TriggerMethod`, `OverridePercentageConstraints`, `Rule80A`, `Origin`, `ShortSaleSlot`, `DesignatedLocation`, `ExemptCode`, `OptOutSmartRouting`, `ProfessionalCustomer`, `ExtOperator` — Regulatory / routing / origin — compliance metadata, not a value a user reads back per order.
- `AuctionStrategy`, `StartingPrice`, `StockRefPrice`, `Delta`, `StockRangeLower`, `StockRangeUpper` — BOX / auction / stock-range price monitoring — niche order-entry modes.
- `Volatility`, `VolatilityType`, `ContinuousUpdate`, `ReferencePriceType`, `RandomizeSize`, `RandomizePrice` — Volatility / pegged-to-volatility family.
- `DeltaNeutralOrderType`, `DeltaNeutralAuxPrice`, `DeltaNeutralConId`, `DeltaNeutralSettlingFirm`, `DeltaNeutralClearingAccount`, `DeltaNeutralClearingIntent`, `DeltaNeutralOpenClose`, `DeltaNeutralShortSale`, `DeltaNeutralShortSaleSlot`, `DeltaNeutralDesignatedLocation` — Delta-neutral family (institutional; unclear cell semantics — design ruling: skip and note).
- `BasisPoints`, `BasisPointsType`, `RefFuturesConId` — EFP (exchange-for-physical) family + futures reference.
- `ScaleInitLevelSize`, `ScaleSubsLevelSize`, `ScalePriceIncrement`, `ScalePriceAdjustValue`, `ScalePriceAdjustInterval`, `ScaleProfitOffset`, `ScaleAutoReset`, `ScaleInitPosition`, `ScaleInitFillQty`, `ScaleRandomPercent`, `ScaleTable` — Scale-order family.
- `HedgeType`, `HedgeParam`, `HedgeMaxSize`, `DontUseAutoPriceForHedge` — Hedge family.
- `SettlingFirm`, `ClearingAccount`, `ClearingIntent`, `Shareholder`, `CustomerAccount` — Clearing / settling / account identifiers.
- `Mifid2DecisionMaker`, `Mifid2DecisionAlgo`, `Mifid2ExecutionTrader`, `Mifid2ExecutionAlgo` — MiFID 2 reporting fields.
- `AdjustedOrderType`, `TriggerPrice`, `AdjustedStopPrice`, `AdjustedStopLimitPrice`, `AdjustedTrailingAmount`, `AdjustableTrailingUnit` — Adjusted-stop family.
- `ReferenceContractId`, `IsPeggedChangeAmountDecrease`, `PeggedChangeAmount`, `ReferenceChangeAmount`, `ReferenceExchange` — Pegged-to-benchmark family.
- `MinTradeQty`, `MinCompeteSize`, `CompeteAgainstBestOffset`, `MidOffsetAtWhole`, `MidOffsetAtHalf` — IBKRATS / pegged-best family.
- `Conditions`, `ConditionsIgnoreRth`, `ConditionsCancelOrder` — Conditional-order family.
- `SmartComboRoutingParams`, `OrderComboLegs`, `OrderMiscOptions`, `Tier` — Collection / nested types — not cell-representable (AlgoParams IS surfaced, formatted explicitly).
- `FaPercentage` — FA allocation percentage; FaGroup/FaMethod are surfaced, this is not.
- `AlgoId` — algo internal id; AlgoStrategy/AlgoParams carry the useful surface.
- `AutoCancelDate` — auto-cancel date; niche order-entry option.
- `AutoCancelParent` — bracket auto-cancel flag; niche.
- `FilledQuantity` — openOrder-provided filled; FILLED (from orderStatus) is authoritative — avoid a second, divergent field.
- `ImbalanceOnly` — imbalance-only auction flag; niche.
- `RouteMarketableToBbo` — routing flag (bool?); niche.
- `ParentPermId` — parent's permId; PARENTID (parent orderId) is the surfaced linkage.
- `AdvancedErrorOverride` — advanced-reject JSON passthrough; internal.
- `ManualOrderTime`, `ManualOrderIndicator` — manual-entry metadata.
- `BondAccruedInterest` — bond-only field.
- `Duration` — GTD duration seconds; niche.
- `PostToAts` — IBKRATS parking seconds; niche.
- `IsOmsContainer` — OMS ticket flag; niche.
- `DiscretionaryUpToLimitPrice` — D-peg conversion flag; niche.
- `UsePriceMgmtAlgo` — price-management algo flag (bool?); CTCI-only.
- `SeekPriceImprovement` — price-improvement flag (bool?); niche.
- `WhatIfType` — whatIf internal subtype.
- `SlOrderId`, `SlOrderType`, `PtOrderId`, `PtOrderType` — attached stop-loss / profit-taker child descriptors.

### Unsupported — Contract properties

- `LastTradeDate` — redundant with LastTradeDateOrContractMonth (EXPIRY); rarely populated on order contracts.
- `IncludeExpired` — a contract-query flag, not an order attribute.
- `SecIdType`, `SecId` — ISIN/CUSIP identifier metadata; not useful in a cell.
- `Description` — free-text contract description.
- `IssuerId` — issuer identifier.
- `ComboLegsDescription` — combo free-text.
- `ComboLegs` — collection/nested type — not cell-representable.
- `DeltaNeutralContract` — nested type — not cell-representable.

### Unsupported — Order-state properties

- `Status` — ORDERSTATUS was retired (design review, 2026-07-18): it was refreshed by only one of the engine's update paths, so it could keep reading a stale "Submitted" after an order had actually completed — while STATUS is refreshed by every path and never goes stale that way. A typed ORDERSTATUS now fails loud at parse. The TWS-delivered terminal status string remains available verbatim via COMPLETEDSTATUS.
- `InitMarginBeforeOutsideRTH`, `MaintMarginBeforeOutsideRTH`, `EquityWithLoanBeforeOutsideRTH`, `InitMarginChangeOutsideRTH`, `MaintMarginChangeOutsideRTH`, `EquityWithLoanChangeOutsideRTH`, `InitMarginAfterOutsideRTH`, `MaintMarginAfterOutsideRTH`, `EquityWithLoanAfterOutsideRTH` — The *OutsideRTH margin doubles: a parallel outside-RTH margin surface; the standard (string) margin fields above are surfaced, these niche doubles are not.
- `SuggestedSize` — whatIf sizing hint; niche.
- `OrderAllocations` — collection/nested type — not cell-representable.

## 7. Order status vocabulary

Two surfaces report this vocabulary, and they differ on exactly one point — the cancel spelling. An `order` topic's `STATUS` cell reports TWS's own status string VERBATIM, including the two-L spellings `Cancelled` and `ApiCancelled`. A `StageOrder` cell reports the same names while the order works, but maps a terminal `Cancelled` / `ApiCancelled` from TWS to the house spelling `Canceled` (one L) so formulas written against v1 keep matching. Every other value below is reported unchanged on both surfaces.

| Status | Class | Meaning |
| --- | --- | --- |
| `ApiPending` | transient | Received by the API client but not yet sent to TWS. |
| `PendingSubmit` | active | Accepted by the API; transmission to TWS pending. |
| `PendingCancel` | active | Cancel requested; not yet confirmed by the destination. |
| `PreSubmitted` | active | Simulated order accepted; election criteria not yet met. |
| `Submitted` | active | Accepted and working at the destination. |
| `Filled` | terminal | Completely filled. |
| `Cancelled` | terminal | Cancellation confirmed. Shown verbatim on an order STATUS cell; a StageOrder cell shows the house spelling Canceled. |
| `ApiCancelled` | terminal | Cancelled via the API before TWS acknowledged the order. Verbatim on an order STATUS cell; a StageOrder cell maps it to Canceled. |
| `Inactive` | terminal | Received but no longer active (e.g. rejected). |
| `Unknown` | sentinel | House parse sentinel for an unrecognized status string — NOT a TWS status. |

House staging states a StageOrder cell publishes before TWS speaks: `Sending` (a placement is in flight), `Staged` (placeOrder delivered; the order awaits human transmission in TWS), and `Canceled` (the house spelling a StageOrder cell uses for a terminal cancel, to which it maps TWS's `Cancelled`/`ApiCancelled`; an `order` topic's STATUS cell does NOT remap — it shows `Cancelled`/`ApiCancelled` verbatim). A workbook reopen disarms a StageOrder cell rather than re-staging: it shows a `Disarmed:` message and stages nothing.

## 8. Position, list, and option-definition fields

### Position fields

`=RTD("tws.rtd", , "position", "<accounts>", "<contract>", "<field>")` (field defaults to POSITION). Across multiple matched accounts, values aggregate (sizes and P&L sum; average cost is size-weighted; contract-metadata fields return the first matched contract). A PARTIAL contract spec matters: in the aggregate (blank / `*` / CSV-accounts) form it aggregates across EVERY contract it matches, whereas a single-account request does NOT aggregate — it resolves to one matching position, so a partial spec that matches several contracts in that account is ambiguous. Pin a single contract with enough contract detail, or `conid=`, for an unambiguous single-contract read.

| Field | Aliases | Description |
| --- | --- | --- |
| `POSITION` | `QTY`, `QUANTITY`, `SHARES`, `SIZE` | Position quantity (shares/contracts); sums across accounts. |
| `AVERAGECOST` | `AVGCOST`, `COST` | Average cost per share/contract (size-weighted across accounts). |
| `MARKETVALUE` | `VALUE` | Market value (sums across accounts). |
| `DAILYPNL` | `DPNL` | Daily P&L (sums across accounts). |
| `REALIZEDPNL` | `RPNL` | Realized P&L (sums across accounts). |
| `UNREALIZEDPNL` | `UPNL` | Unrealized P&L (sums across accounts). |
| `CONID` | `CONTRACTID` | Contract id. |
| `SYMBOL` | `SYM` | Underlying symbol. |
| `SECTYPE` | `SEC`, `SECURITYTYPE` | Security type. |
| `STRIKE` | — | Option strike. |
| `RIGHT` | `PUTCALL` | Option right (C/P). |
| `EXPIRY` | `EXP`, `EXPIRATION`, `LASTTRADEDATEORCONTRACTMONTH` | Expiration date. |
| `EXCHANGE` | `EXCH` | Exchange. |
| `PRIMARYEXCH` | `PRIM`, `PRIMARY`, `PRIMARYEXCHANGE`, `PRIMEXCH` | Primary exchange. |
| `LOCALSYMBOL` | `LOC`, `LOCAL` | Exchange-specific local symbol. |
| `TRADINGCLASS` | `CLASS`, `TC` | Trading class. |
| `CURRENCY` | `CUR` | Currency. |
| `MULTIPLIER` | `MULT` | Contract multiplier. |

### Positions-list fields

`=RTD("tws.rtd", , "positions", "<accounts>", "<field>")` returns a list across positions.

| Field | Description |
| --- | --- |
| `SYMBOLSCSV` | Semicolon-delimited list of position contract specs. |
| `CONIDCSV` | Semicolon-delimited list of position contract ids; a position that lacks a conid contributes the literal sentinel token `Missing ConID` (appended to the CSV, or standing alone as the whole value when no position has a conid). |
| `POSITIONSCHANGEDUTC` | UTC timestamp updated when list membership changes (legacy alias `SYMBOLSCHANGEDUTC`); shows the transient value `Requested` while a list publish is pending. |

### Orders-list fields

`=RTD("tws.rtd", , "orders", "<accounts>", "<field>")` returns a list of order permIds.

| Field | Description |
| --- | --- |
| `LISTCSV` | All orders, including filled/cancelled. |
| `OPENLISTCSV` | Active orders only (terminal statuses excluded). |

### Option-definition fields

Given an underlying contract, these enumerate its option chain.

| Field | Description |
| --- | --- |
| `OPTIONEXPIRATIONSCSV` | CSV of expiration dates that have listed options. |
| `OPTIONSTRIKESCSV` | CSV of strikes that have listed options. |
| `STRIKESTEP` | Smallest reported strike increment. |

## 9. Status and metadata fields

### Status fields

`=RTD("tws.rtd", , "status", "<field>")` reports connection state. These re-resolve on every heartbeat, so an already-subscribed cell tracks live changes.

| Field | Description |
| --- | --- |
| `ISCONNECTED` | 1 when the engine is connected to TWS (as far as it can tell), 0 when definitively not. |
| `ACTIVETOPICCOUNT` | Number of currently subscribed topics. |
| `ACCOUNTSCSV` | Comma-separated managed account ids from the connection handshake. |
| `LASTUPDATEUTC` | UTC timestamp of the last successful data update. |
| `SERVERHEARTBEATUTC` | UTC timestamp of the last Excel heartbeat. |
| `SERVERVERSION` | This connection's negotiated TWS ServerVersion, or 'Not Connected'. |
| `MARKETDATATYPE` | Configured default market-data tier (1–4); distinct from the per-contract market-data field of the same name. |
| `MARKETDATASTATE` | Market-data version state: Ok / TooOld / Unknown. |
| `MARKETDATAMESSAGE` | Actionable message when the negotiated version is too old; empty otherwise. |
| `ORDERDATASTATE` | Order-subsystem data state (see data-state values). |
| `LASTORDERLISTCHANGEUTC` | UTC time the order-list membership last changed — across every order the connection's API client sees, regardless of account (an orders-list account filter narrows the rendered CSV, not this stamp). |
| `LASTORDERUPDATEUTC` | UTC time of the last order update from TWS, for any order on the connection regardless of account (not scoped to the accounts in a given orders-list subscription). |
| `POSITIONDATASTATE` | Position-subsystem data state (see data-state values). |
| `LASTPOSITIONLISTCHANGEUTC` | UTC time the position-list membership last changed. |
| `LASTPOSITIONUPDATEUTC` | UTC time of the last position update. |
| `CONFIGWARNINGS` | Configuration-validation warnings, joined; empty when the configuration is clean. |
| `ROTATIONCOUNT` | Lifetime automatic client-id rotations on this connection. |
| `UPDATE_AVAILABLE` | '1' when a newer release is available, else '0' (live breadcrumb). |
| `UPDATE_CRITICAL` | '1' when the available update is critical, else '0'. |
| `UPDATE_LATEST_VERSION` | Latest available version from the update breadcrumb. |
| `UPDATE_MESSAGE` | Engine-composed update guidance message. |

### Data-state values

`OrderDataState` and `PositionDataState` take one of: `Disconnected`, `Idle`, `Requested`, `Receiving`, `Ready`.

### Metadata fields

Addressed by bare name (`=RTD("tws.rtd", , "VERSION")`). A metadata cell resolves once, when it is first subscribed. The `LICENSE_*` fields are the one exception: they re-resolve around the initial verification window (the non-blocking first evaluation), so a cell subscribed while entitlement was still verifying repaints once the definitive state lands. A license change LATER in the session does NOT repaint an already-subscribed metadata cell — it shows on re-entry or workbook reopen. `TWSAPI_*` (and the build / `UPDATE_*` fields) resolve once at subscription and do not re-resolve. For a live-tracking view of the update breadcrumb, use the `UPDATE_*` status fields above, which re-resolve every heartbeat.

| Field | Description |
| --- | --- |
| `VERSION` | Deployed product SemVer. |
| `BUILD_TIME` | UTC build timestamp of the deployed DLL. |
| `SERVER_PATH` | Filesystem location of the deployed DLL. |
| `CONFIGURATION` | Build configuration (Debug/Release). |
| `ASSEMBLY_NAME` | Executing assembly simple name. |
| `LICENSE_STATE` | License entitlement state. |
| `LICENSE_MESSAGE` | License status message. |
| `LICENSE_DAYS_REMAINING` | Trial days remaining (empty when not on trial). |
| `TWSAPI_STATE` | TWS-API binding state. |
| `TWSAPI_MESSAGE` | TWS-API binding message (actionable guidance). |
| `TWSAPI_VERSION` | Detected TWS-API version. |
| `UPDATE_AVAILABLE` | Update-available breadcrumb flag ('0'/'1'), resolved once at subscription. |
| `UPDATE_CRITICAL` | Update-critical breadcrumb flag ('0'/'1'). |
| `UPDATE_LATEST_VERSION` | Latest available version from the update breadcrumb. |
| `UPDATE_MESSAGE` | Update guidance message from the breadcrumb. |

## 10. Account value keys

*(authored — the key set is a pass-through.)* `=RTD("tws.rtd", , "account", "<account>", "<key>")` returns any account value TWS delivers for the account; keys are case- and separator-insensitive and are NOT a fixed list. Common keys:

| Key | Description |
| --- | --- |
| `NETLIQUIDATION` | Total account net liquidation value. |
| `TOTALCASHVALUE` | Total cash value. |
| `BUYINGPOWER` | Buying power. |
| `AVAILABLEFUNDS` | Funds available for trading. |
| `EXCESSLIQUIDITY` | Excess liquidity. |
| `GROSSPOSITIONVALUE` | Gross position value. |
| `EQUITYWITHLOANVALUE` | Equity with loan value. |
| `INITMARGINREQ` | Initial margin requirement. |
| `MAINTMARGINREQ` | Maintenance margin requirement. |

Engine-computed additions: `OPENPOSITIONCOUNT` (count of non-zero positions), `DAILYPNL`, `REALIZEDPNL`, and `UNREALIZEDPNL` (from the P&L feed).

**Per-currency values.** Append a currency code as a trailing token to request the per-currency breakdown (e.g. account, key, then `EUR`). Per-currency values require the TWS setting *Global Config > API > Settings > 'Prepend $LEDGER- prefix to per-currency account values'* to be enabled; while it is disabled, affected cells fail loud with: `#LEDGER-DISABLED — enable TWS Global Config > API > Settings > 'Prepend $LEDGER- prefix to per-currency account values' (affects all API clients on this TWS), then reconnect`. While the setting is disabled the sentinel ALSO poisons the affected BARE (non-currency-suffixed) dual-class keys — the aggregates TWS delivers in both an account-level and a per-currency category (CashBalance, AccruedCash, RealizedPnL, UnrealizedPnL, StockMarketValue, and the other dual-class values). Without the prefix the engine cannot tell an account-level aggregate from a currency row, so those bare aggregates are ambiguous and fail loud too. Bare non-dual-class keys (e.g. NetLiquidation) are unaffected.

The demo workbook enumerates the full TWS-delivered account key set: `../examples/StreamXLS.xlsm`.

## 11. Configuration keys

Configuration is read from environment variables, then a config file (`%LOCALAPPDATA%\StreamXLS\config.json`), then code defaults — environment wins. A malformed (non-numeric) value warns and falls back to the default. An out-of-range INTEGER is CLAMPED to the nearest in-range bound (with a warning), not dropped to the default — except where the minimum encodes a disabled mode (e.g. a below-minimum positions stale-timeout) and for the discrete selectors (market-data tier, port), which warn and fall back to the default rather than clamp toward an unintended mode. All such warnings surface via the `ConfigWarnings` status field.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `TWS_RTD_MARKET_DATA_TYPE` | 1–4 | 4 | Market-data tier to request (1 real-time, 2 frozen, 3 delayed, 4 delayed-frozen); delayed-frozen falls back gracefully without subscriptions. |
| `TWS_RTD_ERROR_DISPLAY` | MESSAGE / NA | MESSAGE | How TWS errors surface in cells (see error-display effect). |
| `TWS_RTD_PRESERVE_ON_DISCONNECT` | bool | false | false = fail loud (account/order/position/PnL values go #N/A on disconnect); true keeps last-known values. Market data is out of scope. |
| `TWS_RTD_DELAYED_ANNOTATION` | bool | false | When true, delayed numeric values render as '150.25 (delayed)' text instead of a bare number. |
| `TWS_RTD_HOST` | host | 127.0.0.1 | Default TWS host when a formula names none. |
| `TWS_RTD_PORT` | 1–65535 | 7496 | Default TWS port when a formula names none (7496 = live TWS). |
| `TWS_RTD_CLIENT_ID` | int | auto | Fixed API client id; unset auto-generates a unique id per process. |
| `TWS_RTD_LOG_FILE` | path | (none) | Log-file path (supports env-var expansion); unset disables file logging. |
| `TWS_RTD_LOG_LEVEL` | enum | Info | Log verbosity threshold (Error/Warn/Info/Debug/Trace). |
| `TWS_RTD_LOG_RETENTION_DAYS` | -1–365 | 5 | Days to retain rotated logs; -1 disables cleanup. |
| `TWS_RTD_THROTTLE_MS` | 0–10000 | 500 | Minimum interval between position-refresh requests (ms). |
| `TWS_RTD_ORDER_REFRESH_SECONDS` | 5–300 | 15 | Order-polling interval (seconds). |
| `TWS_RTD_HEARTBEAT_INTERVAL_MS` | ≥15000 or -1 | (Excel default) | Overrides Excel's RTD heartbeat interval; -1 disables. |
| `TWS_RTD_POSITION_REQUEST_TIMEOUT_MS` | 0–120000 | 8000 | Positions watchdog: re-issue reqPositions when a snapshot stalls this long (0 disables). |
| `TWS_RTD_POSITION_REQUEST_MAX_RETRIES` | -1–20 | -1 | Max watchdog re-requests per stuck episode; -1 = unlimited. |
| `TWS_RTD_POSITION_STALE_TIMEOUT_MS` | 0–600000 | 30000 | Flip individual/aggregate position values to #N/A after this long stuck (0 disables; negatives rejected). |
| `TWS_RTD_POSITION_RECONNECT_AFTER_RETRIES` | 0–100 | 4 | Force a full reconnect after this many futile re-requests (0 disables). |
| `TWS_RTD_UPDATE_NOTIFY_MIN_MS` | 0–60000 | 0 | Minimum interval between UpdateNotify attempts (0 = no throttle). |
| `TWS_RTD_UPDATE_NOTIFY_PENDING_STALE_MS` | 100–60000 | 1000 | Window after which pending topics are considered stale. |
| `TWS_RTD_NOTIFY_EXPECT_REFRESH_MS` | 0–60000 | 0 | Warn if Excel does not call RefreshData within this window after a notify (0 disables). |
| `STREAMXLS_TWSAPI_PATH` | path | (auto) | Override the TWS-API client location (directory or DLL); resolver reads env then config file. |
| `STREAMXLS_CONFIG_FILE` | path | %LOCALAPPDATA%\StreamXLS\config.json | Location of the optional config file; read from the environment only. |

### Retired configuration keys

These keys are recognized only to warn that they no longer have any effect:

- `TWS_RTD_ACCOUNT_VALUES_NA_ON_DISCONNECT` and `TWS_RTD_PNL_VALUES_NA_ON_DISCONNECT` — retired; use `TWS_RTD_PRESERVE_ON_DISCONNECT` instead.

### Connection tokens

A topic can name its TWS connection inline with these tokens (otherwise the default host/port are used). Defaults: host `127.0.0.1`, port `7496`.

| Token | Meaning |
| --- | --- |
| `paper` | Paper-trading TWS (port 7497). |
| `gw` | IB Gateway, live (port 4001). |
| `gwpaper` | IB Gateway, paper (port 4002). |
| `host=<host>` | Explicit host. |
| `port=<port>` | Explicit port. |
| `clientid=<id>` | Explicit API client id. |
| `<host>:<port>` | Host and port in colon form. |

### Error-display effect

`TWS_RTD_ERROR_DISPLAY` selects how errors render: `MESSAGE` (default) shows the error text; `NA` shows Excel `#N/A`. It affects ONLY market-data subscription errors (the three sites that route through the error-display switch); license text and `#N/A` placeholders are unaffected.

## 12. Cell error vocabulary

Errors surface in a cell as text prefixed `RTD error:` (with a trailing space). For MARKET-DATA subscription errors only, `TWS_RTD_ERROR_DISPLAY=NA` renders them as Excel `#N/A` instead (see section 11); the rest of the messages below are always text. A cell also shows Excel `#N/A` (COM error 2042) when data is unavailable or has been failed loud. Delayed values, when annotation is on, render as '150.25 (delayed)'. Notable messages:

| Constant | Message |
| --- | --- |
| `ERROR_PREFIX` | RTD error: |
| `ERROR_MAX_TICKERS` | Max tickers reached: TWS's market-data subscription limit is full. Reduce the number of market-data cells for this to stream. |
| `ERROR_CONTRACT_REQUIRED` | Contract description is required for 'position' topics. Use 'positions' (plural) for position lists. |
| `ERROR_ACCOUNT_CODE_REQUIRED` | Account code is required for account topics. |
| `ERROR_ACCOUNT_VALUE_KEY_REQUIRED` | Account value key is required. |
| `ERROR_STATUS_FIELD_REQUIRED` | Status field is required. |
| `ERROR_METADATA_FIELD_REQUIRED` | Metadata field is required. |
| `ERROR_POSITIONS_FIELD_REQUIRED` | Positions field is required (e.g., SymbolsCsv, ConIdCsv, PositionsChangedUtc). |
| `ERROR_POSITIONS_FIELD_INVALID` | Positions field is invalid. Expected SymbolsCsv, ConIdCsv, or PositionsChangedUtc. |
| `ERROR_INVALID_ACCOUNT_CODE` | Invalid account code |
| `ERROR_FORMULA_CORRUPTED` | Formula corrupted; please re-enter. |

The account per-currency misconfiguration sentinel (see section 10) is another fail-loud cell value.
