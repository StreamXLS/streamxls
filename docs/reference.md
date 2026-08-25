# StreamXLS RTD reference

Complete reference for the RTD vocabulary the StreamXLS engine serves: topics, market-data fields, contract keys, StageOrder write keys, order and position fields, status and metadata fields, account keys, configuration keys, and cell-error semantics. Every RTD formula takes the form `=RTD("tws.rtd", , "{arguments…}")`.

Generated from the engine. The topics, fields, and keys below are taken from the shipping code and re-checked against it by the test suite on every run, so the names here are the names your installed engine answers to. Every name the code yields is either described here, listed as unsupported, or held back by name in the suite — a few developer-only settings and one unpublished field — so nothing drifts out of the reference silently. The descriptions beside the names are written and reviewed by hand.

**Numbers quoted in prose are documentation, not contract.** Ranges, defaults, and limits written into the text — configuration value ranges especially — are maintained by hand and can lag a release. Where a number matters, trust the engine over this page: it validates the value you supply, falls back or clamps when the value is out of range, and reports what it did in the `CONFIGWARNINGS` status field. The prose around the names is written and reviewed by hand; the suite checks the names, not the sentences.

## 1. Topic and category arguments

A category argument as the first non-connection argument selects the request type. Market-data topics carry no category argument — a contract plus a field name routes to market data.

| Argument | Request type |
| --- | --- |
| `ACCOUNT` | Account value for one account (e.g. NetLiquidation). |
| `POSITION` | A single position (one contract, one or more accounts). |
| `POSITIONS` | A positions list (SymbolsCsv / ConIdCsv). |
| `ORDER` | A single order addressed by permId. |
| `ORDERS` | An orders list (ListCsv / OpenListCsv). |
| `STATUS` | A connection status field addressed by name — the closed status-field set (including the four UPDATE_* update-check fields it shares with metadata). Metadata fields (VERSION, LICENSE_*, TWSAPI_*, …) are addressed by *bare* name, not through this argument. |
| `SENDORDER` | Stage an order in TWS (synonym of StageOrder). |
| `STAGEORDER` | Stage an order in TWS (the canonical name; SENDORDER is an accepted synonym). |

### Resolution order

The request type is resolved by scanning the arguments in this order, first match wins: (1) a bare metadata field name; (2) an explicit category argument from the table above; (3) a bare option-definition field name; (4) a bare name-addressable status field. If none match, the request is treated as market data. `MARKETDATATYPE` is deliberately excluded from the bare-status fall-through so a per-contract `MARKETDATATYPE` stays market data; reach the connection-level status field via the explicit `status` argument.

First match wins, but the arguments it did not match are not ignored. A `status` field answers for the connection as a whole and a metadata field for the whole add-in — neither takes an account, a contract, or a second field — so once a formula resolves to `status` or to metadata, any other argument that is not a [connection argument](#connection-arguments) fails the cell loud, naming the argument it cannot honour, instead of answering a different question from the one written. This holds wherever the argument sits, before or after the field. Blank arguments are connection slots, not extra arguments, so a trailing empty cell is always safe. Name one field per cell: `=RTD("tws.rtd", , "ISCONNECTED", "U1234567")`, `=RTD("tws.rtd", , "AAPL", "BID", "ISCONNECTED")` and `=RTD("tws.rtd", , "AAPL", "BID", "status", "ISCONNECTED")` are errors, not an `ISCONNECTED` reading.

## 2. Market-data fields

Request with a contract and a field name, e.g. `=RTD("tws.rtd", , "AAPL", "BID")`. Field names are case-insensitive. The full supported set is enumerated below by group. *Every* formula names the field it wants: a field name outside the set below fails loud rather than serving some other number under your header.

Two arguments are documented as blank-able — they carry a *meaning* when left empty: the `{Accounts}` filter on `position`/`positions`/`orders` (blank = all accounts, same as `*`), and the connection argument (blank = the default connection — the demo workbook's `A2` "Custom connection" cell). Every *other* argument left blank is simply dropped, provided the formula names its field as above. Write a blank-able connection reference *last*: on `position` and `positions` the first argument slot belongs to the `{Accounts}` filter, so a blank connection cell placed ahead of the contract takes that slot, and a filled accounts cell then lands in the next slot — the contract on `position`, the field on `positions` — and fails loud.

The *first* bare argument is always read as the ticker (or contract spec), never as a field — even when it spells a field name (several field names, e.g. `OPEN` and `LOW`, are also real ticker symbols). Name your field in a later argument; a formula whose only bare argument is a field name draws the field-is-required error. With the contract supplied entirely by keys (`sym=`/`conid=`/`loc=`), a lone bare argument is read as the field.

The field can also be named explicitly, as its own `qt=BID` argument or as a segment of a combined spec (`sym=SPY;qt=BID`). The vocabulary is the same as the bare form; a `qt=` value outside the set below fails loud rather than falling back to `LAST`, and naming two *different* fields (two disagreeing `qt=` arguments, or `qt=` against a differing bare field name) fails loud rather than guessing which you meant.

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
| `LASTTIME` | Timestamp of the last trade: Unix epoch *seconds* as text. Convert to Excel UTC with `=value/86400 + DATE(1970,1,1)`. |
| `HALTED` | Trading-halt indicator (0 = not halted; >0 = halted). |

### Size and volume units

**Sizes and volume for US stocks may arrive in round lots (i.e., units of 100 shares) rather than in shares.** The unit is set in TWS, via two separate checkboxes under *Global Configuration → API → Settings*:

- **"Send market data in lots for US stocks for dual-mode API clients"** — governs `BIDSIZE`, `ASKSIZE`, `LASTSIZE` and their `DELAYED*` twins. IBKR labels this one *(Not recommended)*.
- **"Send volumes in lots from all market data sources for US stocks"** — governs `VOLUME` and `DELAYEDVOLUME`.

Because they are set separately, sizes in shares beside a volume still counted in lots is a perfectly ordinary configuration — check both before trusting either. While a box is checked, its fields arrive divided by 100 and rounded down. Two symptoms to recognise:

- **A `0` size standing beside a live price.** An inside bid of 40 shares truncates to `0` while `BID` keeps quoting: the `0` means "less than one round lot", not "no size" — where a quote is genuinely absent the formula reports `#N/A`. Odd-lot insides are common on higher-priced single stocks and rare on large ETFs, so a block of `0 x 0` sizes under real prices on the former is faithful data, not a dropped subscription.
- **A session that traded 74.3 million shares reporting `742,668`.**

Clearing a checkbox switches its fields to share counts, and those cells then read 100× larger. The settings name US stocks; other markets follow their own exchange conventions. The `ODDLOT*` fields below are a *separate* quote feed rather than a shares-unit view of the fields above — they report an odd-lot best bid and ask where your subscription supplies one.

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
| `MARKETPRICE` | Derived mid: `(BID+ASK)/2` when both present, else `LAST`, else `CLOSE`; blank until a source is available. Never rolls back, on the same rule as `LASTORCLOSE`. |
| `LASTORCLOSE` | LAST when available, otherwise CLOSE. Never rolls back: once a trade has been shown, the prior session's CLOSE does not replace it, so the cell holds the last price it knew rather than stepping backwards — unless the cell has gone blank (as on a connection loss), which the restored CLOSE then fills, or TWS delivers a settle from a later session. |

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
| `BIDOPTPRICE` | Option price used in the bid computation. |
| `BIDPVDIVIDEND` | Present value of dividends from the bid computation. |
| `BIDGAMMA` | Option gamma from the bid computation. |
| `BIDVEGA` | Option vega from the bid computation. |
| `BIDTHETA` | Option theta from the bid computation. |
| `BIDUNDPRICE` | Underlying price used in the bid computation. |
| `ASKTICKATTRIB` | Tick attributes for the ask option-computation snapshot. |
| `ASKIMPLIEDVOL` | Implied volatility from the ask option computation. |
| `ASKDELTA` | Option delta from the ask computation. |
| `ASKOPTPRICE` | Option price used in the ask computation. |
| `ASKPVDIVIDEND` | Present value of dividends from the ask computation. |
| `ASKGAMMA` | Option gamma from the ask computation. |
| `ASKVEGA` | Option vega from the ask computation. |
| `ASKTHETA` | Option theta from the ask computation. |
| `ASKUNDPRICE` | Underlying price used in the ask computation. |
| `LASTTICKATTRIB` | Tick attributes for the last option-computation snapshot. |
| `LASTIMPLIEDVOL` | Implied volatility from the last option computation. |
| `LASTDELTA` | Option delta from the last computation. |
| `LASTOPTPRICE` | Option price used in the last computation. |
| `LASTPVDIVIDEND` | Present value of dividends from the last computation. |
| `LASTGAMMA` | Option gamma from the last computation. |
| `LASTVEGA` | Option vega from the last computation. |
| `LASTTHETA` | Option theta from the last computation. |
| `LASTUNDPRICE` | Underlying price used in the last computation. |
| `MODELTICKATTRIB` | Tick attributes for the model option-computation snapshot. |
| `MODELIMPLIEDVOL` | Implied volatility from the model option computation. |
| `MODELDELTA` | Option delta from the model computation. |
| `MODELOPTPRICE` | Option price used in the model computation. |
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
| `IBDIVIDENDS` | IBKR dividend estimates. |
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

The four ETF NAV fields (`ETFNAVLAST`, `ETFFROZENNAVLAST`, `ETFNAVHIGH`, `ETFNAVLOW`) depend on your account's market-data entitlements. StreamXLS requests them by default on stock and ETF subscriptions, but an account without the entitlement receives no NAV data and no error from TWS — so those cells read `#N/A` and stay there. That is the fail-loud rule working as intended: the cell reports that it has no value rather than showing a blank or a stale one. To check whether the data is yours to receive, look at the fund in TWS itself — if TWS shows no NAV column data for it, StreamXLS cannot show any either. (The converse does not follow: TWS's own display can draw on data paths the API does not expose, so NAV visible in TWS does not guarantee the API delivers it.)

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
| `DELAYEDLASTTIMESTAMP` | Delayed last-trade timestamp, in the same Unix epoch seconds as `LASTTIME`. |
| `DELAYEDHALTED` | Delayed halt indicator. |

### Disconnect behavior (the #N/A class)

On a TWS disconnect, market-data cells follow one of three rules, keyed off the *configured* market-data tier (`TWS_RTD_MARKET_DATA_TYPE`), not the per-request served tier.

- Quote family → #N/A in every tier. A standing bid/ask dies with the session that carried it, so every quote cell fails loud on disconnect regardless of tier: `ASK`, `ASKEXCH`, `ASKSIZE`, `BID`, `BIDEXCH`, `BIDSIZE`, `DELAYEDASK`, `DELAYEDASKSIZE`, `DELAYEDBID`, `DELAYEDBIDSIZE`, `ODDLOTASK`, `ODDLOTASKEXCH`, `ODDLOTASKSIZE`, `ODDLOTBID`, `ODDLOTBIDEXCH`, `ODDLOTBIDSIZE`.
- `LAST` / `DELAYEDLAST` → tier-dependent. Under the real-time (1) and delayed (3) tiers a stale last reads `#N/A` (you did not opt into last-known values); under the frozen (2) and delayed-frozen (4, the default) tiers it keeps its last-seen value (frozen tiers are the explicit opt-in to last-known values).
- Everything else → keep-last. CLOSE, MARKETPRICE, LASTORCLOSE, VOLUME, the option greeks, and the last-trade attributes (LASTSIZE / LASTEXCH / LASTTIME) keep their last value regardless of tier — they are facts about a completed trade, not standing-quote state.

A TWS data-connectivity loss (error 1100) is stronger: every market-data cell goes `#N/A`, including the keep-last families, until connectivity returns.

## 3. Contract specification keys

Contracts can be given as a simple ticker (`AAPL`), a slash form (`AAPL@SMART/STK/USD`), a pipe form (`SPY|OPT|20251219|C|450`), an FX pair (`EUR.USD/CASH`), or explicit `key=value` arguments. The canonical keys and their accepted aliases are below (case-insensitive). Worked examples of the common forms are in [manual.md](manual.md).

**One key per argument, or every key in one argument.** The keys below can be written as separate RTD arguments (`"sym=SPY", "sec=OPT", …`) or joined into a *single* argument with semicolons (`"sym=SPY;sec=OPT;strike=680;right=C;exp=20261218"`); market-data, option-definition and `StageOrder` formulas parse the two identically, and a `StageOrder` argument may mix contract and order keys freely. The `position` topic accepts *only* the joined form — it reads one argument as the contract and the next as the field, so a second `key=value` argument fails loud as an unsupported position field. Some `StageOrder` keys must be written as their *own* argument, for two different reasons: `tag`, `account`, `fagroup`, `ocagroup` (and `tag`'s synonyms) are free text, so each keeps everything after its `=` and a `;` there is *data*, not a separator; `algoparams` is judged *whole* by its own grammar, which rejects a `;` outright ([section 4](#4-stageorder-write-keys)). Written joined — *behind* another key, or with a recognized key inside its own value (`"tag=ORD17;exch=CBOT"`) — the argument fails loud rather than guessing where the value ends. A *connection* argument (`paper`, `gw`, `gwpaper`, `host=`, `port=`, `clientid=`, `host:port`) must stand alone for a different reason again: it is read from the *whole* argument, so joined with a `;` it would be invisible to the connection and the order would stage on the default TWS — it fails loud instead.

| Canonical key | Aliases | Description |
| --- | --- | --- |
| `symbol` | `sym` | Underlying symbol. |
| `sectype` | `sec`, `securitytype` | Security type (`STK`, `OPT`, `FUT`, `FOP`, `CASH`, `IND`, …). |
| `exchange` | `exch` | Routing exchange (`SMART` for the universal router). |
| `primaryexch` | `prim`, `primary`, `primaryexchange`, `primexch` | Primary listing exchange (disambiguates `SMART`-routed symbols). |
| `currency` | `cur`, `curr` | Trading currency. |
| `expiry` | `exp`, `expiration`, `lasttradedate` | Expiration date (YYYYMM or YYYYMMDD) for options/futures. |
| `strike` | `strikeprice` | Option strike price. |
| `right` | `optiontype`, `putcall` | Option right: `C` (call) or `P` (put). |
| `multiplier` | `mult` | Contract multiplier. |
| `localsymbol` | `loc`, `local` | Exchange-specific local symbol. |
| `tradingclass` | `class`, `tc` | Trading class. It identifies a contract; on the option-definition fields of [section 8](#8-position-list-and-option-definition-fields) it does not filter the chain. |
| `conid` | `contractid` | TWS contract ID (authoritative when present). |

Defaults when a symbol-only request omits them, for market-data and option-definition contract parsing: security type `STK`, exchange `SMART`, currency `USD`. A position-topic contract defaults the same security type and currency but deliberately leaves the exchange *unconstrained* (blank), so a symbol-only spec matches the position on whatever exchange TWS reports rather than being pinned to `SMART`. When a `conid` is supplied the security type and currency are left to TWS; the exchange still defaults to `SMART` unless the spec supplies `exch` or a `localsymbol`.

**Case, and one contract per formula.** Values are upper-cased for you on every notation — `sym=spy`, `SPY|opt|…` and `spy@smart/stk` ask for the same contracts as their upper-case spellings, and two case spellings of one contract therefore share one subscription. The two exceptions are `localsymbol` and `tradingclass`, which are case-significant on TWS and are passed through exactly as typed. A `key=value` argument beats a compact-form segment for the same field, so a column can override one part without rewriting the spec string — *except* the symbol: a ticker shorthand that contradicts `sym=`, or that stands beside a `conid=`, names a second instrument rather than overriding an attribute, and fails loud. The pipe form reads exactly five segments (`SYMBOL|SECTYPE|EXPIRY|RIGHT|STRIKE`); a sixth fails loud rather than being dropped — give an exchange or currency as its own `exch=`/`cur=` argument. In the slash form the security type is positional — the first segment after the exchange, or after the primary exchange if you give one — and must be one of TWS's security types (`STK`, `OPT`, `FUT`, `FOP`, `CASH`, `IND`, `BOND`, `CFD`, `CRYPTO`, `WAR`, `IOPT`, `FUND`, `CMDTY`, `BAG`, `BILL`, `CONTFUT`, `FWD`, `FIXED`, `SLB`, `NEWS`, `BSK`, `ICU`, `ICS`); anything else in that slot fails loud with the spelling to use, so `AAPL@SMART/NASDAQ` (a primary exchange with no type after it) is refused rather than sent to TWS as `SecType=NASDAQ`. `sec=` is not checked against the list.

**Every key you write must carry a value.** A key with nothing after the `=` — the shape `"cur="&B2` produces when `B2` is blank — fails loud rather than falling back to the default above or discarding a value the ticker shorthand supplied. The same applies to an empty segment of the compact form (`"BHP@ASX/STK/"&B2`, `"BHP@"&B2`, `"EUR."&B2&"/CASH"`) and to the four extension arguments below. *Omitting* a key is always legal; writing it and leaving it empty is not, because an omitted key and an empty one describe different instruments and StreamXLS will not guess which was meant. Build optional arguments with `IF` so they collapse to nothing when the cell is blank (`IF(B2="","","cur="&B2)`) — that idiom leaves a *blank* argument behind, which is dropped so long as the formula names its field ([section 2](#2-market-data-fields)). Two exceptions: the *pipe* form's segments are positional, so an empty segment is how that notation spells "omitted" (`SPY||||` is the plain stock) — its symbol position is required, and so is its *security-type* position whenever a *later* segment is filled in, because an empty security type is read as `STK` and `ES||202612` would otherwise ask for a *stock* carrying a futures expiry; and `StageOrder`'s *order* keys ([section 4](#4-stageorder-write-keys)) keep reading a blank as "not supplied", while its *contract* keys, `exch=` included, follow the rule — except `sym=`/`symbol=`, which are read on that order-key path, so a blank one reads as "not supplied" there: with no `conid=` the order then fails loud for a missing symbol, and with one the `conid` is authoritative anyway. The failure arrives as error text in the cell rather than an Excel error value, so `IFERROR`/`IFNA` will not hide it.

### Combo and extension arguments

Four further arguments extend a *market-data* request beyond the contract keys above. Each is written `key=value` like the rest and carries its own punctuation inside the value, and each must carry a value (above). `cmb=` is a *list*, so every leg position written must carry a leg: a stray or doubled `;` from a blank cell fails loud rather than quoting a spread with a leg missing, and each leg's four fields (`conId#ratio#action#exchange`) must each carry a value — a blank venue cell fails loud rather than shipping a leg with no exchange, since the leg's venue is part of which price the combo reflects. `StageOrder` accepts none of these arguments — an unrecognized key there fails loud ([section 4](#4-stageorder-write-keys)).

| Argument | Description |
| --- | --- |
| `cmb=` | Legs of a combo (multi-leg) contract, for a `sec=BAG` request: one `conId#ratio#action#exchange` leg per leg, semicolon-separated between legs. Leg actions are `BUY`/`B`, `SELL`/`S`, or `SSHORT`/`SS`, case-insensitive; an unrecognized action fails loud instead of assuming a side. The argument is passed through whole — it is never split on its own semicolons. `combo=` is an accepted spelling. Combos are quotable, not stageable ([section 4](#4-stageorder-write-keys)). |
| `und=` | Delta-neutral hedge contract for a combo request: `conId#delta#price`. All three parts are required, and a malformed one fails loud. |
| `opt=` | TWS request options, passed through verbatim with the market-data request as `tag#value` pairs, semicolon-separated between pairs. No StreamXLS behavior is attached to them — omit this argument unless you have been given a specific option to send. |
| `genticks=` | Explicit TWS generic-tick list for the request, comma-separated tick IDs. When it is omitted, StreamXLS sends a default list chosen by security type — the list that carries the generic-tick fields of [section 2](#2-market-data-fields). When it is supplied, it *replaces* that default, so any generic-tick field whose tick is left out stops updating. |

## 4. StageOrder write keys

`=RTD("tws.rtd", , "StageOrder", "sym=AAPL", "side=BUY", "shares=100", "type=LMT", "limit=150")` stages an order in TWS. Every key is `key=value`. An unrecognized `key=value` argument fails loud (a dropped key could stage an order you did not describe); an argument that follows the StageOrder argument but contains no `=` — anything other than a connection argument — fails loud too rather than being silently dropped, so write every argument as `key=value`. Keys may be written one per argument or semicolon-joined into a single argument ([section 3](#3-contract-specification-keys)), mixing contract and order keys freely; the rows marked below, and any connection argument, are the exceptions and must each be written as their own argument. The recognized keys, grouped by the logical field they set:

| Keys | Description |
| --- | --- |
| `symbol`, `sym` | Underlying symbol (required). |
| `action`, `side` | BUY or SELL (required). |
| `quantity`, `shares`, `qty`, `size` | Order quantity, integer > 0 (required). |
| `type` | Order type, required; passed to TWS uppercased (e.g. LMT, MKT, STP, STP LMT, TRAIL, TRAIL LIMIT). Internal whitespace is collapsed, so a doubled space between STP and LMT (a concatenation artifact) is read and sent as `STP LMT` instead of slipping past the type-dependent rules. Not an engine whitelist: LMT/STP LMT require limit and the stop/trailing keys constrain the stop family, but an unlisted type is TWS's to accept or reject. One exception: a short list of near-miss spellings of the modelled types — `LIMIT`, `MARKET`, `STOP`, `STPLMT`, `TRAILING`, `TRAILLIMIT` and their spaced variants — fails loud and names the modelled spelling to write instead. Whether or not TWS would accept the near-miss, StreamXLS matches the type-dependent rules on the exact spellings above, so a near-miss would silently switch those price rules *off*. |
| `limit` | Limit price. Required for LMT and STP LMT, and on every type it must be a finite number greater than zero. Rejected loud on the types that cannot carry one (MKT, STP, TRAIL, MOC, MIT, MTL), where TWS would ignore it and stage an order at no price you named. |
| `stop`, `aux`, `stopprice` | Stop / auxiliary trigger price (STP, STP LMT, TRAIL, TRAIL LIMIT). |
| `trailingpercent` | Trailing percent (TRAIL, TRAIL LIMIT). |
| `trailstop`, `trailstopprice` | Initial trailing *trigger* price, decimal > 0 (TRAIL, TRAIL LIMIT). *Required* for TRAIL LIMIT — TWS rejects a TRAIL LIMIT that has none, answering "Please enter a stop price". Optional for TRAIL, which TWS accepts without one. |
| `limitoffset`, `lmtoffset` | Distance from the trailing trigger to the limit price, any finite decimal (TRAIL LIMIT only; zero and negative offsets are accepted and passed through — their meaning is defined by TWS). Mutually exclusive with `limit`: TRAIL LIMIT requires *exactly one* of the two, and supplying both is rejected by TWS itself ("You must specify one value: limit price or limit price offset value"). Use `limit=` when you want the limit at an exact price. |
| `exchange`, `exch` | Routing exchange (defaults to SMART). |
| `account` | IBKR account to stage into. [Cannot be joined with `;`](#3-contract-specification-keys). |
| `fagroup` | Financial-advisor allocation group. [Cannot be joined with `;`](#3-contract-specification-keys). |
| `algostrategy`, `algo` | Algo strategy name. |
| `algoparams` | Algo parameters as pipe-delimited tag=value pairs. Requires `algo=`: the parameters attach to the algo strategy, so without one the whole list would be dropped at staging. The pipe is the only separator — a `,` or `;` inside the value, a segment with no `=`, an empty value, or a repeated tag each fail loud rather than shipping a truncated or corrupted parameter. [Cannot be joined with `;`](#3-contract-specification-keys). |
| `tif` | Time in force (see grammars). |
| `outsiderth` | Allow fills outside regular trading hours (boolean). |
| `goodtilldate`, `gtd` | Good-till date/time (required with tif=GTD). |
| `goodaftertime`, `gat` | Good-after date/time. |
| `hidden` | Hidden order flag (boolean). |
| `display`, `displaysize` | Iceberg display size (integer > 0). |
| `allornone`, `aon` | All-or-none flag (boolean). |
| `minqty` | Minimum fill quantity (integer > 0). |
| `ocagroup` | One-cancels-all group name (with ocatype). [Cannot be joined with `;`](#3-contract-specification-keys). |
| `ocatype` | One-cancels-all type (with ocagroup; see grammars). |
| `park`, `parked`, `saved` | park=true stages a local ticket visible only in the parking user's own TWS (Transmit button) instead of the default deactivated order (Submit button), which is visible to all TWS sessions and survives a TWS restart. |
| `tag`, `nonce`, `seq`, `submit`, `clienttag` | User order tag (composed into OrderRef; see composition). [Cannot be joined with `;`](#3-contract-specification-keys). |

### Synonym groups

Within each group the keys are synonyms — supply at most one; supplying two of the same group fails loud rather than silently dropping a value. Supplying the *same* key twice also fails loud, so `side=BUY`,`side=SELL` is rejected rather than silently staging the later value.

- `symbol` = `sym`
- `action` = `side`
- `quantity` = `shares` = `qty` = `size`
- `exchange` = `exch`
- `algostrategy` = `algo`
- `tag` = `nonce` = `seq` = `submit` = `clienttag`
- `park` = `parked` = `saved`
- `stop` = `aux` = `stopprice`
- `trailstop` = `trailstopprice`
- `limitoffset` = `lmtoffset`
- `goodtilldate` = `gtd`
- `goodaftertime` = `gat`
- `display` = `displaysize`
- `allornone` = `aon`

### Value grammars

- Booleans (`park`, `outsiderth`, `hidden`, `allornone`): `TRUE`/`1`/`YES` or `FALSE`/`0`/`NO`; anything else fails loud.
- Time-in-force (`tif`): one of `DAY`, `GTC`, `IOC`, `FOK`, `OPG`, `GTD`. `GTD` requires `goodtilldate`, and `goodtilldate` requires `tif=GTD`.
- OCA type (`ocatype`): `1`, `2`, `3` (1 = cancel remaining with block, 2 = reduce remaining with block, 3 = reduce remaining without block); `ocagroup` and `ocatype` are required together.
- Date/time (`goodtilldate`, `goodaftertime`): `YYYYMMDD [HH:MM:SS [TZ]]`, validated as a real calendar date/time.
- Prices (`limit`, `stop`): `limit`, when supplied, must be a finite decimal `> 0` *whatever* the order type, and `LMT` / `STP LMT` *require* it; it is refused on the types that cannot carry one (`MKT`, `STP`, `TRAIL`, `MOC`, `MIT`, `MTL`), where TWS would ignore the price and stage an order at none you named. `stop` (aliases `aux` / `stopprice`), when supplied, must be a finite decimal `> 0`. `STP` and `STP LMT` *require* `stop`; `stop` is accepted only for `STP`, `STP LMT`, `TRAIL`, `TRAIL LIMIT`.
- Trailing (`trailingpercent`): when supplied, a finite decimal in `(0, 100]`; accepted only for `TRAIL` / `TRAIL LIMIT`. `TRAIL` and `TRAIL LIMIT` each require *exactly one* of `stop` (the trailing amount) or `trailingpercent`.
- Trailing trigger and limit offset (`trailstop`, `limitoffset`): `trailstop` (alias `trailstopprice`) is a finite decimal `> 0` and is accepted only for `TRAIL` / `TRAIL LIMIT`; `limitoffset` (alias `lmtoffset`) is any finite decimal (zero and negative offsets are accepted by TWS and pass through verbatim) and is accepted only for `TRAIL LIMIT`. `TRAIL LIMIT` *requires* `trailstop` and *exactly one* of `limit` or `limitoffset` — TWS rejects the order outright without a trigger price, and rejects a limit price and a limit offset supplied together.

### Contract keys

StageOrder accepts the full single-leg contract vocabulary: the contract keys of [section 3](#3-contract-specification-keys) (`sec`, `cur`, `exp`, `strike`, `right`, `mult`, `loc`, `tc`, `prim`) plus `conid`, resolved through the same contract-key aliases documented there. When omitted, security type defaults to `STK` and currency to `USD`. Every order/contract key must appear *after* the `StageOrder` argument (a key placed before it fails loud), and a contract key supplied with an empty value fails loud rather than silently defaulting.

- `conid`: integer > 0. Identifies the contract by itself — supplying `sym` or any other contract descriptor alongside it fails loud; only `exchange` may accompany `conid`.
- `sec`: one of `STK`, `OPT`, `FUT`, `FOP`, `CASH`, `IND`, `CFD`, `BOND`, `FUND`, `CMDTY`, `WAR`. `BAG` (combo) is rejected. `OPT`/`FOP` require `exp`, `strike`, and `right` (or a `loc` that resolves them), and so does `WAR` (a warrant is named the same way an option is — IBKR's own warrant contract carries an expiry, a strike and a right); `FUT` requires `exp` (or `loc`); `exp` is accepted for `OPT`/`FOP`/`FUT`/`WAR`; `strike` and `right` for `OPT`/`FOP`/`WAR` only. `BOND` must be named by a security identifier: write the CUSIP or ISIN as `sym` (this is how IBKR identifies a bond), or supply `loc`, or `conid` alone. An issuer's ticker in `sym` matches every bond that issuer has outstanding and is rejected, so an order can never be staged against a bond TWS picked rather than the one you named. One caveat on `WAR`: the expiry/strike/right quad narrows a warrant but does not always single one out — several issuers can list warrants with the same terms on the same venue — so use `loc` or `conid` when you need certainty about *which* warrant.
- `cur`: 3-letter currency code.
- `exp`: `YYYYMM` or `YYYYMMDD`, validated as a real calendar date.
- `strike`: decimal > 0 (invariant parse — a comma decimal fails loud).
- `right`: `C`/`CALL` or `P`/`PUT` (normalized to `C`/`P`).
- `mult`: decimal > 0 (fractional ratios are ordinary on warrants — IBKR's own warrant example carries a multiplier of 0.01).
- `loc` / `tc`: free text, preserved verbatim (case- and space-significant on TWS).
- `prim`: primary listing exchange (upper-cased).

### Capability boundary

StageOrder stages single-leg orders across the full contract vocabulary above, defaulting to `STK`/`USD` only when `sec`/`cur` are omitted. Combo (multi-leg / `sec=BAG`) staging is *not* supported. There is no cancel or replace verb from a formula: a staged order sits in TWS with auto-transmit suppressed and reaches the market only when a human transmits it there; all subsequent modification, cancellation, and transmission happen in TWS. The engine re-checks this at the socket boundary before every StageOrder reaches TWS — the order must be either deactivated or transmit-suppressed, and the send is refused otherwise; the test suite pins that check. By default the order is staged deactivated (visible in TWS order lists, survives a TWS restart); `park=true` stages it instead as a local order-entry ticket, visible only in the parking user's own TWS.

### Order reference composition

The `tag` key (aliases `nonce`, `seq`, `submit`, `clienttag`) is preserved verbatim and tail-composed with an engine identity token into the order's OrderRef as `{tag}|SXLS:{token}` (bare `SXLS:{token}` when no tag). The engine token at the tail is how a whole-day order replay is matched back to the exact cell that staged it; the composed reference is capped at 64 characters and an over-length order fails loud before staging.

## 5. Order read fields

`=RTD("tws.rtd", , "order", "{permId}", "{field}")` reads one field of one order (addressed by permId; the field is named, never inferred). The vocabulary is a closed set — an unknown field fails loud. Canonical fields:

| Field | Aliases | Description |
| --- | --- | --- |
| `STATUS` | — | Current order status (see [section 7](#7-order-status-vocabulary)). |
| `FILLED` | — | Filled quantity. |
| `REMAINING` | — | Remaining (unfilled) quantity. |
| `AVGFILLPRICE` | — | Average fill price. |
| `WHYHELD` | — | Reason the order is held, if any. |
| `WARNINGTEXT` | — | TWS warning text for the order. |
| `PERMID` | — | Permanent order ID (stable across sessions). |
| `ORDERID` | — | Client order ID. |
| `PARENTID` | — | Parent order ID (bracket/child orders). |
| `CLIENTID` | — | API client ID the order originated from. |
| `ACCOUNT` | — | IBKR account number. #N/A when the order was submitted to an advisor group rather than one account (e.g. the "All" account): TWS sends no account field for an all-accounts order, because it is not attributed to a single account — read `FAGROUP` for the group it was staged to. |
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
| `ORDREF` | `ORDERREF` | Order tag as staged — the engine's identity token is stripped before display. |
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
| `CONID` | — | Contract ID. |
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

[Section 5](#5-order-read-fields) lists every order field you can read. This section lists the rest — each remaining TWS API order, contract, and order-state property, grouped by the reason it is not offered as a cell. Nothing here was dropped by oversight: the test suite fails if a property in the TWS API is neither published as a field in [section 5](#5-order-read-fields) nor listed below with a reason. Asking for one of these names in a formula fails loud rather than leaving the cell blank.

### Unsupported — Order properties

- `Transmit`, `Deactivate`, `PostOnly`, `IgnoreOpenAuction` — Unreported booleans — TWS does not populate these flags on the open-order feed, so a cell could only show a fabricated default rather than the order's real setting, and a boolean has no way to report "unknown". `Transmit` and `Deactivate` would be the worst of it: an order still parked in TWS would read as though it were already working. Check the order in TWS itself.
- `TriggerMethod`, `OverridePercentageConstraints`, `Rule80A`, `Origin`, `ShortSaleSlot`, `DesignatedLocation`, `ExemptCode`, `OptOutSmartRouting`, `ProfessionalCustomer`, `ExtOperator` — Regulatory / routing / origin — compliance metadata, not a value a user reads back per order.
- `AuctionStrategy`, `StartingPrice`, `StockRefPrice`, `Delta`, `StockRangeLower`, `StockRangeUpper` — BOX / auction / stock-range price monitoring — niche order-entry modes.
- `Volatility`, `VolatilityType`, `ContinuousUpdate`, `ReferencePriceType`, `RandomizeSize`, `RandomizePrice` — Volatility / pegged-to-volatility family.
- `DeltaNeutralOrderType`, `DeltaNeutralAuxPrice`, `DeltaNeutralConId`, `DeltaNeutralSettlingFirm`, `DeltaNeutralClearingAccount`, `DeltaNeutralClearingIntent`, `DeltaNeutralOpenClose`, `DeltaNeutralShortSale`, `DeltaNeutralShortSaleSlot`, `DeltaNeutralDesignatedLocation` — Delta-neutral family — institutional order parameters with no clear per-cell meaning.
- `BasisPoints`, `BasisPointsType`, `RefFuturesConId` — EFP (exchange-for-physical) family + futures reference.
- `ScaleInitLevelSize`, `ScaleSubsLevelSize`, `ScalePriceIncrement`, `ScalePriceAdjustValue`, `ScalePriceAdjustInterval`, `ScaleProfitOffset`, `ScaleAutoReset`, `ScaleInitPosition`, `ScaleInitFillQty`, `ScaleRandomPercent`, `ScaleTable` — Scale-order family.
- `HedgeType`, `HedgeParam`, `HedgeMaxSize`, `DontUseAutoPriceForHedge` — Hedge family.
- `SettlingFirm`, `ClearingAccount`, `ClearingIntent`, `Shareholder`, `CustomerAccount` — Clearing / settling / account identifiers.
- `Mifid2DecisionMaker`, `Mifid2DecisionAlgo`, `Mifid2ExecutionTrader`, `Mifid2ExecutionAlgo` — MiFID 2 reporting fields.
- `AdjustedOrderType`, `TriggerPrice`, `AdjustedStopPrice`, `AdjustedStopLimitPrice`, `AdjustedTrailingAmount`, `AdjustableTrailingUnit` — Adjusted-stop family.
- `ReferenceContractId`, `IsPeggedChangeAmountDecrease`, `PeggedChangeAmount`, `ReferenceChangeAmount`, `ReferenceExchange` — Pegged-to-benchmark family.
- `MinTradeQty`, `MinCompeteSize`, `CompeteAgainstBestOffset`, `MidOffsetAtWhole`, `MidOffsetAtHalf` — IBKRATS / pegged-best family.
- `Conditions`, `ConditionsIgnoreRth`, `ConditionsCancelOrder` — Conditional-order family.
- `SmartComboRoutingParams`, `OrderComboLegs`, `OrderMiscOptions`, `Tier` — Collection / nested types — not cell-representable (AlgoParams *is* published as a read field, formatted explicitly).
- `FaPercentage` — FA allocation percentage; FAGROUP and FAMETHOD are available, this is not.
- `AlgoId` — internal algo identifier; ALGOSTRATEGY and ALGOPARAMS carry the detail you can read back.
- `AutoCancelDate` — auto-cancel date; niche order-entry option.
- `AutoCancelParent` — bracket auto-cancel flag; niche.
- `FilledQuantity` — a second filled-quantity value from a different feed; the FILLED read field is the authoritative one, so this duplicate is not offered.
- `ImbalanceOnly` — imbalance-only auction flag; niche.
- `RouteMarketableToBbo` — routing flag (bool?); niche.
- `ParentPermId` — parent's permId; PARENTID (parent orderId) is the published linkage.
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

- `Status` — `ORDERSTATUS` was retired in favor of `STATUS`. Its underlying value is refreshed by only one of the paths that can move an order to a terminal state, so it could keep reading "Submitted" after an order had already filled or been cancelled. `STATUS` is refreshed by every path, including the ones that infer a terminal state. A formula asking for `ORDERSTATUS` fails loud rather than returning a stale value, and the TWS-delivered terminal status string remains available verbatim via `COMPLETEDSTATUS`.
- `InitMarginBeforeOutsideRTH`, `MaintMarginBeforeOutsideRTH`, `EquityWithLoanBeforeOutsideRTH`, `InitMarginChangeOutsideRTH`, `MaintMarginChangeOutsideRTH`, `EquityWithLoanChangeOutsideRTH`, `InitMarginAfterOutsideRTH`, `MaintMarginAfterOutsideRTH`, `EquityWithLoanAfterOutsideRTH` — outside-regular-hours counterparts of the margin figures above; niche.
- `SuggestedSize` — whatIf sizing hint; niche.
- `OrderAllocations` — collection/nested type — not cell-representable.

## 7. Order status vocabulary

Two kinds of cell report this vocabulary, and they differ on exactly one point — how they handle an API-side cancel. An `order` topic's `STATUS` cell passes TWS's status string through almost unchanged, including the two-L spellings `Cancelled` and `ApiCancelled`. Two values on that cell are engine-supplied rather than TWS-delivered: an order that leaves the open set with nothing left to fill reads `Filled`, and one TWS stops reporting with quantity remaining reads `Cancelled` — both in TWS's own spelling. A `StageOrder` cell reports the same names in TWS's own spelling, with one difference: it collapses a terminal `ApiCancelled` into `Cancelled`, so a staged cell shows a single cancel word. Every other value below reads the same on both.

| Status | Class | Meaning |
| --- | --- | --- |
| `ApiPending` | transient | Received by the API client but not yet sent to TWS. |
| `PendingSubmit` | active | Accepted by the API; transmission to TWS pending. |
| `PendingCancel` | active | Cancel requested; not yet confirmed by the destination. |
| `PreSubmitted` | active | Simulated order accepted; election criteria not yet met. |
| `Submitted` | active | Accepted and working at the destination. |
| `Filled` | terminal | Completely filled. |
| `Cancelled` | terminal | Cancellation confirmed; a StageOrder cell shows the same word. |
| `ApiCancelled` | terminal | Cancelled via the API before TWS acknowledged the order; a StageOrder cell collapses it to `Cancelled`. |
| `Inactive` | transient | Received but not currently active (e.g. rejected, or held at the destination). Not treated as final by StreamXLS; excluded from `OPENLISTCSV`. |
| `Unknown` | sentinel | StreamXLS's parse sentinel for an unrecognized status string — not a TWS status. |

A StageOrder cell publishes its own states before any TWS status arrives: `Sending` (the staging request is in flight), `Staged` (the request reached TWS without error; the order waits there for you to transmit it), and `Cancelled` (the terminal-cancel spelling described above). Reopening a workbook disarms a StageOrder cell rather than re-staging it: the cell shows a `Disarmed:` message and stages nothing. If the TWS order on the cell's order id stops carrying the identity tag StreamXLS stamped at staging (for example, the Order Ref field was edited in TWS), the cell shows an `Order identity unverifiable:` message and stops tracking that order id; check the order in TWS or track it with the orders topics.

## 8. Position, list, and option-definition fields

### Position fields

`=RTD("tws.rtd", , "position", "{accounts}", "{contract}", "{field}")` (the field is named, never inferred; `{accounts}` may be blank for all accounts). Across multiple matched accounts, values aggregate (sizes and P&L sum; average cost is weighted by *absolute* size; contract-metadata fields return the matched contract when the spec resolves to a single contract ID, and fail loud with the same `Ambiguous position` error naming the matched contract IDs when it resolves to more than one). A *partial* contract spec matters: in the aggregate (blank / `*` / CSV-accounts) form it aggregates across *every* contract it matches, whereas a single-account request does *not* aggregate — it must resolve to exactly one position. A partial spec that matches several contracts in that account is ambiguous and fails loud (an `Ambiguous position` error naming the matched contract IDs) rather than returning an arbitrary one; the error clears on its own once the ambiguity resolves. Pin a single contract with enough contract detail, or `conid=`, for an unambiguous single-contract read.

| Field | Aliases | Description |
| --- | --- | --- |
| `POSITION` | `QTY`, `QUANTITY`, `SHARES`, `SIZE` | Position quantity (shares/contracts); sums across accounts. |
| `AVERAGECOST` | `AVGCOST`, `COST` | Average cost per share/contract (size-weighted across accounts). |
| `MARKETVALUE` | `VALUE` | Market value (sums across accounts). |
| `DAILYPNL` | `DPNL` | Daily P&L (sums across accounts). |
| `REALIZEDPNL` | `RPNL` | Realized P&L (sums across accounts). |
| `UNREALIZEDPNL` | `UPNL` | Unrealized P&L (sums across accounts). |
| `CONID` | `CONTRACTID` | Contract ID. |
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

`=RTD("tws.rtd", , "positions", "{accounts}", "{field}")` returns a list across positions.

| Field | Description |
| --- | --- |
| `SYMBOLSCSV` | Semicolon-delimited list of position contract specs. |
| `CONIDCSV` | Semicolon-delimited list of position contract IDs; a position that lacks a conid contributes the literal sentinel token `Missing ConID` (appended to the CSV, or standing alone as the whole value when no position has a conid). |

### Orders-list fields

`=RTD("tws.rtd", , "orders", "{accounts}", "{field}")` returns a list of order permIds.

An *account-filtered* list contains only the orders TWS attributes to one of the named accounts. An order submitted to an advisor *group* rather than a single account (the "All" account, for instance) carries no account attribution on the wire, so it appears in the unfiltered list (blank or `*`) and in no account-filtered one — the same reason `ACCOUNT` reads #N/A for it and `FAGROUP` names the group instead ([section 5](#5-order-read-fields)).

| Field | Description |
| --- | --- |
| `LISTCSV` | All orders, including filled/cancelled. |
| `OPENLISTCSV` | Active orders only (terminal statuses and Inactive excluded). |

### Option-definition fields

Requested like market data — an underlying contract plus a field name, e.g. `=RTD("tws.rtd", , "AAPL", "OPTIONSTRIKESCSV")`. Given an underlying contract, these enumerate its option chain.

**These fields aggregate every trading class TWS lists for the underlying** — weeklies alongside monthlies (`SPX` with `SPXW`) — and they cannot be narrowed to one class. `tc=` does not narrow them: the TWS chain request takes no trading class at all. Neither can the sheet: the returned dates and strikes carry no class marker, so nothing in the CSV says which class listed a given expiry or strike, and `STRIKESTEP` is the minimum increment across the whole aggregate — it can be finer than any single class uses. Treat all three as an answer about the underlying, and name the exact contract you want on the market-data formula that quotes it.

Exchange behaves differently from class. For a stock or index underlying the answer spans every exchange TWS reports. For a `FUT` or `FOP` underlying the chain request carries the spec's `exch=` when one is set — and `SMART` is not a futures routing exchange, so a futures underlying normally needs an explicit `exch=`. A spec that names the contract with `localsymbol` and no `exch=` sends no exchange at all, so that chain spans every exchange TWS reports.

`tc=` is not inert on these fields; it just cannot shorten the chain. When the underlying is named by symbol, the class is still sent while TWS resolves that symbol, so it can decide whether the cell resolves at all: a class that matches nothing fails the cell loud instead of returning a chain, and deleting a `tc=` that was disambiguating a dual-listed symbol gives `Ambiguous underlying`. Keep the `tc=` you need to *name* the underlying. When the underlying is named by `conid=`, the class is dropped entirely: it can neither narrow the chain nor fail the cell, and two cells differing only by `tc=` share one request.

| Field | Description |
| --- | --- |
| `OPTIONEXPIRATIONSCSV` | CSV of expiration dates that have listed options. |
| `OPTIONSTRIKESCSV` | CSV of strikes that have listed options. |
| `STRIKESTEP` | Smallest reported strike increment. |

## 9. Status and metadata fields

### Status fields

`=RTD("tws.rtd", , "status", "{field}")` reports connection state. These re-resolve on every heartbeat, so an already-subscribed cell tracks live changes. That covers the *value*, not the routing: the connection each status topic reads is chosen once, at first subscription (see [Connection arguments](#connection-arguments)).

Most of these fields answer for one connection. Seven do not — `ACTIVETOPICCOUNT`, `MARKETDATATYPE`, `CONFIGWARNINGS` and the four `UPDATE_*` fields report add-in-wide facts, so they read the same from every connection and a connection argument changes only which connection the cell binds to, never the answer.

| Field | Description |
| --- | --- |
| `ISCONNECTED` | 1 when the engine is connected to TWS (as far as it can tell), 0 when definitively not. |
| `ACTIVETOPICCOUNT` | How many distinct subscriptions the add-in currently holds. Two formulas that resolve to the same subscription — say `AAPL` and `AAPL@SMART` for the same field — count once between them, and that subscription leaves the count only when the last formula referring to it is gone. Add-in-wide, not per connection, and counted from subscription rather than from data arriving — so it does not fall when TWS disconnects. Status and metadata cells count themselves, and StageOrder is the one exception to sharing: each StageOrder formula is its own submission and counts separately. |
| `ACCOUNTSCSV` | Comma-separated managed account IDs from the connection handshake. |
| `LASTUPDATEUTC` | UTC timestamp of the last successful data update. |
| `SERVERHEARTBEATUTC` | UTC timestamp of the server's most recent status pass. Excel's heartbeat usually drives it, but engine-side connection events stamp it too, and when the Excel heartbeat is disabled or raised above Excel's floor the server's own cadence takes over — so it measures StreamXLS running, not Excel calling, and never TWS liveness. |
| `SERVERVERSION` | This connection's negotiated TWS ServerVersion, or 'Not Connected'. |
| `MARKETDATATYPE` | Configured default market-data tier (1–4); distinct from the per-contract market-data field of the same name. |
| `MARKETDATASTATE` | Market-data version state: Ok / TooOld / Unknown. |
| `MARKETDATAMESSAGE` | Actionable message when the negotiated version is too old; empty otherwise. |
| `MARKETDATAFARMS` | Which market-data farms TWS has told this connection are broken, and since when (UTC) — one entry per farm, reading like `usfarm down since 2026-08-16T13:45:09Z`. TWS connects to every market-data farm your entitlements touch, and it announces each one independently — so a farm named here is very often not one serving the contracts in your sheet, and StreamXLS deliberately changes no price cell because of it: this is an explanation to reach for when a quote looks frozen, not a verdict on any cell. Empty means only that no uncleared farm-down report is outstanding — never that every farm is healthy. TWS clears an entry itself, by re-announcing that farm as OK or as dormant. Everything here is also cleared wholesale on a reconnect, and on TWS regaining its own connection to IB after an outage: in both cases a farm report from before the break describes a connection that no longer exists, and StreamXLS will not carry a claim it can no longer stand behind. A farm that is still down is re-announced, so it comes back with a fresh time. Historical-data farms are not reported: StreamXLS never requests historical data, so they cannot affect a cell. |
| `ORDERDATASTATE` | Order-subsystem data state (see data-state values). |
| `LASTORDERLISTCHANGEUTC` | UTC time the order-list membership last changed — across every order the connection's API client sees, regardless of account (an orders-list account filter narrows the rendered CSV, not this timestamp). |
| `LASTORDERUPDATEUTC` | UTC time TWS last delivered order information that actually *changed* something — a poll that re-delivers identical orders does not advance it. Covers any order on the connection regardless of account (not scoped to the accounts in a given orders-list subscription). To tell "nothing happened" apart from "the order feed stopped", pair it with LASTORDERPOLLUTC. |
| `LASTORDERPOLLUTC` | UTC time the last complete open-orders response arrived from TWS, whether or not anything changed. Order-feed liveness: it keeps advancing every poll (15s by default) while the feed is healthy and freezes if TWS stops answering, which the change stamps above cannot show. |
| `POSITIONDATASTATE` | Position-subsystem data state (see data-state values). |
| `LASTPOSITIONLISTCHANGEUTC` | UTC time the position-list membership last changed — a contract entering or leaving the active set, not a size change or a price tick. Connection-wide (never scoped to one cell's account filter), and it advances only while position topics keep position data flowing. |
| `LASTPOSITIONUPDATEUTC` | UTC time of the last accepted position callback. Freshness, not change: unlike LASTORDERUPDATEUTC it advances even when the delivered values are identical. |
| `CONFIGWARNINGS` | Configuration-validation warnings, joined; empty when the configuration is clean. |
| `CONNECTIONKEY` | Which TWS *this* cell is actually reading, as `host:port:clientId`. It exists because a status formula carrying no connection argument can bind to a connection you never named — it piggybacks the sole connection when there is exactly one, and otherwise takes the default (see [Connection arguments](#connection-arguments)). Binding is per formula, not per sheet, so give this field the same connection argument as the cells you are checking. The client ID tracks automatic rotations; the host and port cannot change under a subscribed cell. If the cell had piggybacked the sole connection and a second connection has since been created, this field reports that instead — the binding is no longer meaningful, and every other status field on that formula reports the same. This is a report, not a connection argument: an automatically assigned client ID is not part of the connection's identity, so pasting the value back into a formula names a *different* connection. `#N/A` means the binding could not be determined. |
| `ROTATIONCOUNT` | Lifetime automatic client-ID rotations on this connection. |
| `UPDATE_AVAILABLE` | '1' when a newer release is available, '0' when an update check has run and found none, and `#N/A` when this computer has no completed update check on record (or the last one is more than two weeks old) — StreamXLS will not claim you are current on evidence it does not have. Re-checked live; `UPDATE_MESSAGE` says which case you are in. |
| `UPDATE_CRITICAL` | '1' when the available update is critical, '0' when it is not, `#N/A` on the same no-evidence terms as `UPDATE_AVAILABLE`. |
| `UPDATE_LATEST_VERSION` | Latest available version from the update check; empty when no newer version is known. |
| `UPDATE_MESSAGE` | Engine-composed update guidance message: empty when up to date, the update notice when one is available, and an explanation when the update status is unknown. |

### Data-state values

`OrderDataState` and `PositionDataState` take one of: `Disconnected`, `Idle`, `Requested`, `Receiving`, `Ready`.

### Metadata fields

Addressed by bare name (`=RTD("tws.rtd", , "VERSION")`). A metadata cell resolves once, when it is first subscribed. The `LICENSE_*` fields are the one exception: they re-resolve around the initial verification window (the non-blocking first evaluation), so a cell subscribed while entitlement was still verifying repaints once the definitive state lands. `LICENSE_*` and `TWSAPI_*` also repaint mid-session, without re-entry, when what they report changes — a trial running out, a subscription lapsing, an activation, or an installed TWS API that turns out to be incompatible when a connection is attempted. That re-verification is deliberately infrequent (about every 6 hours while the license grants data, about every 10 minutes once it does not), so a license changed elsewhere normally reaches the cell at the next check rather than promptly. The installed TWS API itself is inspected once per Excel session, so installing or upgrading it takes effect on the next Excel restart. The build and `UPDATE_*` metadata fields resolve once at subscription and do not re-resolve. For a live-tracking view of update availability, use the `UPDATE_*` status fields above, which re-resolve every heartbeat.

| Field | Description |
| --- | --- |
| `VERSION` | Deployed product SemVer. |
| `BUILD_TIME` | UTC build timestamp of the deployed DLL. |
| `SERVER_PATH` | Filesystem location of the deployed DLL. |
| `CONFIGURATION` | Build configuration (Debug/Release). |
| `ASSEMBLY_NAME` | Executing assembly simple name. |
| `LICENSE_STATE` | License entitlement state. |
| `LICENSE_MESSAGE` | License status message. |
| `LICENSE_DAYS_REMAINING` | Trial days remaining; empty when not on trial, `#N/A` while a trial is running but its expiry date could not be read. |
| `TWSAPI_STATE` | TWS-API binding state. |
| `TWSAPI_MESSAGE` | TWS-API binding message (actionable guidance). |
| `TWSAPI_VERSION` | Detected TWS-API version. |
| `UPDATE_AVAILABLE` | Update-available flag ('0'/'1'), or `#N/A` when no completed update check is on record for this computer. Resolved once at subscription. |
| `UPDATE_CRITICAL` | Update-critical flag ('0'/'1'), `#N/A` on the same terms. |
| `UPDATE_LATEST_VERSION` | Latest available version from the update check; empty when no newer version is known. |
| `UPDATE_MESSAGE` | Update guidance message from the update check, or an explanation when the update status is unknown. |

## 10. Account value keys

`=RTD("tws.rtd", , "account", "{account}", "{key}")` returns any account value TWS delivers for the account; keys are case-insensitive and ignore spaces and underscores (other punctuation is significant), and are *not* a fixed list. Common keys:

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

Engine-computed additions: `OPENPOSITIONCOUNT` (count of positions with a non-zero size or market value), `DAILYPNL`, `REALIZEDPNL`, and `UNREALIZEDPNL` (from the P&L feed).

**A key this account does not report.** Because the key set is *not* a fixed list, StreamXLS cannot check the spelling when you write the formula — it asks TWS instead, and answers once TWS has answered. When TWS has finished sending an account's values, and has reported other keys for that account, a key that received nothing is named in the cell: `RTD error: Account key 'NETLIQUIDATOIN' was not reported. TWS finished sending this account's values without reporting it — check the spelling; TWS also does not report every key for every account type. See streamxls.com/docs-reference.` Until then the cell reads `#N/A`: a key whose values have simply not arrived yet is never flagged. Two different causes produce this message and, from outside TWS, they are indistinguishable — a misspelled key, and a real key (including one listed on this page) that TWS does not report for *this* account type. Either way the cell will not fill while that remains true, and the message clears itself when it stops being true: a later delivery replaces it with the value — or with `#N/A`, where the only data that arrived is under the key's `…ByCurrency` counterpart rather than the spelling your formula used. Never flagged: `cur=BASE` cells (see below), and `OPENPOSITIONCOUNT`, `DAILYPNL`, `REALIZEDPNL` and `UNREALIZEDPNL` asked for without a currency, which come from other feeds rather than from the account download. Like other formula-fix messages this text appears even under `TWS_RTD_ERROR_DISPLAY=NA`.

**Per-currency values.** Append a currency code as a trailing argument to request the per-currency breakdown (e.g. account, key, then `EUR`). Per-currency values require the TWS setting *Global Config > API > Settings > 'Use "$LEDGER-" prefix for per-currency keys.'* to be enabled; while it is disabled, affected cells fail loud with: `#LEDGER-DISABLED — enable TWS Global Config > API > Settings > 'Use "$LEDGER-" prefix for per-currency keys.'`. While the setting is disabled the sentinel *also* poisons the affected *bare* (non-currency-suffixed) dual-class keys — the aggregates TWS delivers in both an account-level and a per-currency category (CashBalance, AccruedCash, RealizedPnL, UnrealizedPnL, StockMarketValue, and the other dual-class values). Without the prefix the engine cannot tell an account-level aggregate from a currency row, so those bare aggregates are ambiguous and fail loud too. Bare non-dual-class keys (e.g. NetLiquidation) are unaffected.

**Keys TWS reports only per currency.** With the prefix setting enabled, TWS reports a set of keys *only* inside the per-currency ledger and publishes no account-wide value for them — `AccountOrGroup` (which `AccountName` maps to), `Currency`, `RealCurrency`, `ExchangeRate`, `CashBalance`, `TotalCashBalance`, `NetLiquidationByCurrency`, `StockMarketValue`, `OptionMarketValue`, `FxCashBalance`, and the other per-currency balances. Asked for *with* a currency, such a key returns that currency's row (or `#N/A` if the account holds nothing in it). Asked for *without* one, what you get depends on the account:

- **One currency for that key** — the value. With a single currency there is nothing to aggregate, so that currency's figure is the account's. A currency sitting at exactly *zero* does not count as a second one (it adds nothing at any exchange rate), so an account that has zeroed out its other currencies keeps showing the value.
- **Several currencies, one identical value in each** — that value. Keys like `AccountOrGroup` are a property *of* the account rather than a balance *in* a currency, so TWS repeats them verbatim in every currency row and the answer is unambiguous. Only non-numeric values qualify: several currencies agreeing on a *number* is not evidence (three rows of 5 does not make the account's figure 5), so a numeric key takes the message below.
- **`Currency` and `RealCurrency`, several currencies that disagree** — the account's currency codes as *text*, comma-separated and ascending with no spaces, e.g. `EUR,USD` (`TEXTSPLIT`-able). These two keys report a currency *identity* rather than an amount held in one, so where a single-currency account answers with that one ledger's code, a multi-currency account answers with the set — asking you to pick one would send you round a loop. Two caveats. A currency whose balance has gone to *zero* still has a ledger, so it is still listed here even though the money keys above have stopped counting it; and StreamXLS publishes the list only when TWS's own ledger confirms it (every currency's row names that currency, and the key is present in every one of the account's currency ledgers). When it does not, the cell falls back to the message below rather than show a list that might be short or wrong.
- **Several currencies with different values** (any other key) — an actionable message naming the currencies that *are* available and asking you to add one, e.g. `RTD error: CASHBALANCE is reported per currency on this account; specify one to get its value — available: EUR, USD (e.g. "cur=EUR")`. StreamXLS never adds balances across currencies — a cross-currency total needs an exchange rate TWS did not send, so inventing one would put a fabricated number in your cell. Like other formula-fix messages, this text appears even under `TWS_RTD_ERROR_DISPLAY=NA`. The message re-renders as the currency set changes, and becomes a value if the account ends up with one currency for that key.
- **Not resolvable yet** — `#N/A`. These cells stay `#N/A` until TWS signals that it has finished sending the account, because mid-download a multi-currency account is indistinguishable from a single-currency one and showing one currency's figure as the account's would be worse than showing nothing. The same applies again after a reconnection or a TWS connectivity blip until the account has been re-sent.

All of the above describes a connection with the `$LEDGER-` setting *enabled*, which is the only regime in which the engine can tell a per-currency row from an account-level one. With the setting *disabled* these keys arrive indistinguishable from account level, so they take the disabled-state rules in the previous paragraph instead (the sentinel for the dual-class ones; `#N/A` for a bare key whose rows collide under two or more currency tags). Enable it and reconnect.

The demo workbook enumerates the full TWS-delivered account key set — open it from the StreamXLS Control Panel (*Open the demo workbook*); a copy ships in this repository at `../examples/StreamXLS.xlsm`.

## 11. Configuration keys

Configuration is read from environment variables, then a config file (`%LOCALAPPDATA%\StreamXLS\config.json`), then code defaults — environment wins. A malformed (non-numeric) value warns and falls back to the default. An out-of-range *integer* is *clamped* to the nearest in-range bound (with a warning), not dropped to the default — except where the minimum encodes a disabled mode (e.g. a below-minimum positions stale-timeout) and for the discrete selectors (market-data tier, port), which warn and fall back to the default rather than clamp toward an unintended mode. All such warnings appear in the `CONFIGWARNINGS` status field.

Most of these are also editable — with their valid range and default in view — from the StreamXLS Control Panel Settings dialog, which writes the same config file. For the setting-name mapping and the rules that decide whether the Control Panel value or an environment variable wins, see [Advanced: environment variables](manual.md#advanced-environment-variables).

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `TWS_RTD_MARKET_DATA_TYPE` | 1–4 | 4 | Market-data tier to request (1 real-time, 2 frozen, 3 delayed, 4 delayed-frozen); delayed-frozen falls back gracefully without subscriptions. |
| `TWS_RTD_ERROR_DISPLAY` | MESSAGE / NA | MESSAGE | How TWS errors surface in cells (see error-display effect). |
| `TWS_RTD_PRESERVE_ON_DISCONNECT` | bool | false | false = fail loud (account/order/position/PnL values go #N/A on disconnect); true keeps last-known values. Market data is out of scope. |
| `TWS_RTD_DELAYED_ANNOTATION` | bool | false | When true, delayed-tier numeric values render as the text `150.25 (delayed)` instead of a bare number. Excel then ranks that text above any number, so `=IF(A1>100,…)` is silently TRUE and `SUM`/`MAX`/`AVERAGE`/`COUNTIF` skip the cell, while `+` arithmetic fails loud with `#VALUE!`; strip the suffix to recover the value with `=VALUE(SUBSTITUTE(A1," (delayed)",""))`. Annotation is presentation only. For a plain flag, read the `ISDELAYED` field instead. |
| `TWS_RTD_DIRECT_ROUTE_VENUES` | venue list | (empty) | US stock exchanges you hold a real-time exchange-direct market-data subscription for, separated by commas, semicolons or spaces (`NASDAQ,CBOE`; case and spacing do not matter). There is no wildcard — each venue is a separate assertion — and an entry that is not a US stock venue opts nothing in and is reported in `CONFIGWARNINGS`. A market-data formula that routes a US stock or warrant direct to a US stock venue *not* named here — `AAPL@NASDAQ`, `XYZ@NYSE/WAR/USD`, or a `conid=`/`loc=` whose `exch=` names a venue that carries no options — is refused with a message instead of being sent, because on an exchange your account is not entitled to, that one request makes TWS stop delivering that symbol's last price, size, volume and daily open/high/low/close until the session ends (in Excel and in TWS's own watchlists; pressing Ctrl+Alt+F in TWS clears it). On the eight venues IBKR also routes US options to — `AMEX`, `BATS`, `CBOE`, `EDGX`, `ISE`, `MEMX`, `PEARL`, `PHLX` — a `conid=`/`loc=` naming no security type is likelier to be an option than a stock and is left alone; write `sectype=STK` beside it to have a stock request refused there. The venues covered are the ones IBKR publishes as carrying US stocks: the lit exchanges, plus its OTC books (`PINK`, `OTCLNKECN`, `ARCAEDGE`) and overnight destinations (`OVERNIGHT`, `IBEOS`). SMART routing, every security type but stocks and warrants, non-USD contracts, and venues that are not US stock venues are all unaffected — write `AAPL` or `AAPL@SMART/NASDAQ/STK/USD` where you were naming an exchange for the listing rather than for the route. The list is read when a formula first calculates, so adding an exchange does not repaint already-refused cells: re-enter those formulas, or restart Excel. |
| `TWS_RTD_HOST` | host | 127.0.0.1 | Default TWS host when a formula names none. |
| `TWS_RTD_PORT` | 1–65535 | 7496 | Default TWS port when a formula names none (7496 = live TWS). |
| `TWS_RTD_CLIENT_ID` | int | auto | Fixed API client ID; unset auto-generates a unique ID per process. |
| `TWS_RTD_LOG_FILE` | path | (none) | Log-file path (supports env-var expansion); unset disables file logging. |
| `TWS_RTD_LOG_LEVEL` | enum | Info | Log verbosity threshold: None (or Off), Error, Warn (or Warning), Info, Debug, Trace, Verbose; an unrecognized value falls back to Info with a warning. |
| `TWS_RTD_LOG_RETENTION_DAYS` | -1–365 | 5 | Days to retain rotated logs; -1 disables cleanup. |
| `TWS_RTD_THROTTLE_MS` | 0–10000 | 500 | Minimum interval between position-refresh requests (ms). |
| `TWS_RTD_ORDER_REFRESH_SECONDS` | 5–300 | 15 | Order-polling interval (seconds). |
| `TWS_RTD_HEARTBEAT_INTERVAL_MS` | ≥15000 or -1 | (Excel default) | Overrides Excel's RTD heartbeat interval; -1 disables. Setting -1, or any value above 15000, also starts a server-owned background cadence, so a heartbeat you moved out of the way is not the only thing that can re-arm a dropped update. |
| `TWS_RTD_POSITION_REQUEST_TIMEOUT_MS` | 0–120000 | 8000 | Positions watchdog: re-issue reqPositions when a snapshot stalls this long (0 disables, which also disables the reconnect-escalation below — it counts these re-requests). |
| `TWS_RTD_POSITION_REQUEST_MAX_RETRIES` | -1–20 | -1 | Max watchdog re-requests per stuck episode; -1 = unlimited. |
| `TWS_RTD_POSITION_STALE_TIMEOUT_MS` | 0–600000 | 30000 | Flip individual/aggregate position values to #N/A after this long stuck (0 disables; negatives rejected). Independent of the request timeout. |
| `TWS_RTD_POSITION_RECONNECT_AFTER_RETRIES` | 0–100 | 4 | Force a full reconnect after this many futile re-requests (0 disables); inert when the request timeout is 0, since nothing counts the re-requests. The wait before each re-request doubles — 8 s, 16 s, 32 s, then 60 s — so at the defaults the reconnect follows about two minutes of silence from the positions stream, not 32 seconds. If you also set the stale timeout above to 0, nothing flips the cells to #N/A, and this wait becomes the only bound on how long a position cell can go on showing a pre-stall number: about two minutes if the connection has gone quiet, ten minutes if it is still delivering other data (see the quiet window below). |
| `TWS_RTD_POSITION_ESCALATION_QUIET_MS` | 0–600000 | 15000 | Before forcing that reconnect, StreamXLS asks whether its own inbound data is running behind — because if it is, the missing position rows are queued rather than lost. Where the market-data liveness probe has timed a round trip within the last two probe windows, *that measurement* answers the question: a round trip at or above `TWS_RTD_MD_LIVENESS_RTT_WARN_MS` plus one probe window means the data really is behind, so the connection is left alone and the re-requests continue; a prompt round trip means the data is current and the rows are simply not being served, so the reconnect goes ahead. **This window is the fallback where no such measurement exists** — a workbook with no market-data formulas never probes, and the probe can be switched off: there, StreamXLS falls back to asking whether TWS delivered *anything at all* — any quote, any account or order message — within this window, and leaves the connection alone if it did. Either way the reason is written to the log once, and the wait is not open-ended: after ten minutes without position data the reconnect happens regardless, so a connection that has quietly lost only its position stream is still repaired. Set 0 to reconnect on the re-request count alone, without any of these checks. Must be longer than the request timeout above. |
| `TWS_RTD_MD_LIVENESS_PROBE_MS` | 0–600000 | 20000 | Market-data liveness probe: how long a `reqCurrentTime` round trip may go unanswered on a socket reporting connected before that window counts as a miss. It also spaces the probes (one per window), and the probe runs only while market-data subscriptions exist. 0 disables it; a negative value is rejected (warns, falls back to the default) rather than clamped into the disabled mode. |
| `TWS_RTD_MD_LIVENESS_MISSES` | 0–20 | 2 | How many *consecutive* unanswered probes force a full reconnect — the recovery path out of a wedged-but-alive TWS socket, where `ISCONNECTED` reads 1 while quotes silently freeze at pre-wedge prices. Any reply — and any other inbound callback — resets the count, so neither a slow TWS nor a busy one triggers it; only a socket that delivers nothing at all does. 0 disables the probe, 1 is raised to the design minimum of 2, and a negative value is rejected (warns, falls back to the default). |
| `TWS_RTD_MD_LIVENESS_RTT_WARN_MS` | 0–600000 | 5000 | How long the liveness probe's round trip may take before the log says so. The probe already times how long TWS takes to answer, and that number is StreamXLS's own end-to-end delivery lag: when Excel is busy enough that the engine falls behind on the messages TWS has already sent, every value in the sheet is that far behind the market. A round trip at or above this threshold writes one warning, and one more when it recovers. It is also the bar the positions check above compares against, with one probe window added as margin (the round-trip measurement can read one window high after a probe TWS never answered, and a decision that *suppresses* a reconnect carries that known error rather than trusting it). Set 0 for no warning — which also removes the round-trip evidence from that check, leaving it on the quiet window alone (the timing is still recorded at debug level). Inert when the probe is off. |
| `TWS_RTD_UPDATE_NOTIFY_MIN_MS` | 0–60000 | 0 | Minimum interval between UpdateNotify attempts (0 = no throttle). |
| `TWS_RTD_UPDATE_NOTIFY_PENDING_STALE_MS` | 500–60000 | 1000 | Window after which pending topics are considered stale. Smaller values are accepted but raised to 500 at runtime. |
| `STREAMXLS_TWSAPI_PATH` | path | (auto) | Override the TWS-API client location (directory or DLL); resolver reads env then config file. |
| `STREAMXLS_CONFIG_FILE` | path | %LOCALAPPDATA%\StreamXLS\config.json | Location of the optional config file; read from the environment only. |

### Retired configuration keys

These keys are recognized only to warn that they no longer have any effect:

- `TWS_RTD_ACCOUNT_VALUES_NA_ON_DISCONNECT` and `TWS_RTD_PNL_VALUES_NA_ON_DISCONNECT` — retired; use `TWS_RTD_PRESERVE_ON_DISCONNECT` instead.

### Connection arguments

A topic can name its TWS connection inline with these arguments (otherwise the default host/port are used). Defaults: host `127.0.0.1`, port `7496`.

`status` topics are the exception. A status topic carrying no connection argument piggybacks the sole connection when StreamXLS has exactly one; with two or more, or with none created yet, it takes the default instead (creating it if absent). The count is of connections *created* — every formula except the bare-name metadata fields creates one, a connection whose TWS never answers still counts, and none is removed before the session ends.

The piggyback holds only while one connection is all there is. If a second connection is ever created, a status cell that piggybacked stops answering and says so — `#AMBIGUOUS-CONNECTION`, naming the `host:port` it had been reading — rather than keep reporting one connection out of several without saying which. It does not move to another connection, and re-entering the formula does not restore it: with two or more connections an argument-less status formula takes the default. Name the connection in every formula, status topics included. A cell that took the default keeps answering; so do `ACTIVETOPICCOUNT`, `MARKETDATATYPE`, `CONFIGWARNINGS` and the four `UPDATE_*` fields, whose answers never came from a connection. The `CONNECTIONKEY` status field reports the connection a status topic actually bound to, so the choice can be read rather than inferred.

| Argument | Meaning |
| --- | --- |
| `paper` | Paper-trading TWS (port 7497). |
| `gw` | IB Gateway, live (port 4001). |
| `gwpaper` | IB Gateway, paper (port 4002). |
| `host={host}` | Explicit host. |
| `port={port}` | Explicit port. |
| `clientid={id}` | Explicit API client ID. |
| `{host}:{port}` | Host and port in colon form. |

### Error-display effect

`TWS_RTD_ERROR_DISPLAY` selects how errors render: `MESSAGE` (default) shows the error text; `NA` shows Excel `#N/A`. It affects *only* market-data error displays (every site that renders a market-data error honors the switch); license text and `#N/A` placeholders are unaffected, and so is the exchange-direct refusal described in [section 12](#12-cell-error-vocabulary), which reports a formula to fix rather than a market-data condition and therefore keeps its text under `NA`.

## 12. Cell error vocabulary

Errors surface in a cell as text prefixed `RTD error:` (with a trailing space). For *market-data* subscription errors only, `TWS_RTD_ERROR_DISPLAY=NA` renders them as Excel `#N/A` instead (see [section 11](#11-configuration-keys)); the rest of the messages below are always text. A cell also shows Excel `#N/A` (VBA's `xlErrNA`, error number 2042) when data is unavailable or has been failed loud. Delayed values, when annotation is on, render as '150.25 (delayed)'. Notable messages:

| Constant | Message |
| --- | --- |
| `ERROR_PREFIX` | RTD error: |
| `ERROR_MAX_TICKERS` | Max tickers reached: TWS's market-data subscription limit is full. Reduce the number of market-data cells for this to stream. |
| `ERROR_CONTRACT_REQUIRED` | Contract description is required for 'position' topics. Use 'positions' (plural) for position lists. |
| `ERROR_ACCOUNT_CODE_REQUIRED` | Account number is required for account topics. |
| `ERROR_ACCOUNT_VALUE_KEY_REQUIRED` | Account value key is required. |
| `ERROR_MARKET_DATA_FIELD_REQUIRED` | Market data field is required (e.g., LAST, BID, ASK). Every formula names its field. See streamxls.com/docs-reference for supported fields. |
| `ERROR_ORDER_FIELD_REQUIRED` | Order field is required (e.g., STATUS, FILLED, AVGFILLPRICE). Every formula names its field. See streamxls.com/docs-reference for supported order fields. |
| `ERROR_POSITION_FIELD_REQUIRED` | Position field is required (e.g., POSITION, MARKETVALUE, UNREALIZEDPNL). Every formula names its field. See streamxls.com/docs-reference for supported position fields. |
| `ERROR_STATUS_FIELD_REQUIRED` | Status field is required. |
| `ERROR_METADATA_FIELD_REQUIRED` | Metadata field is required. |
| `ERROR_POSITIONS_FIELD_REQUIRED` | Positions field is required (e.g., SymbolsCsv, ConIdCsv). |
| `ERROR_POSITIONS_FIELD_INVALID` | Positions field is invalid. Expected SymbolsCsv or ConIdCsv. |
| `ERROR_INVALID_ACCOUNT_CODE` | Invalid account number |
| `ERROR_FORMULA_CORRUPTED` | Formula corrupted; please re-enter. |

Four further fail-loud cell values are composed at the moment they are needed rather than fixed strings, so they are described here instead of tabled above. The account per-currency misconfiguration sentinel (see [section 10](#10-account-value-keys)). The exchange-direct refusal, which opens `real-time exchange-direct data for` and then names the venue, what the request would have done to your TWS session, how to clear a session already affected, and the remedies — SMART routing, or naming the venue in `TWS_RTD_DIRECT_ROUTE_VENUES` (see [section 11](#11-configuration-keys)). It offers the venue as a primary exchange (`AAPL@SMART/NASDAQ/STK/USD`) only where that venue lists stocks; on a routing destination it says so instead, and on a venue that also carries options it adds how to say the contract is an index rather than a stock. It is shown as text even under `TWS_RTD_ERROR_DISPLAY=NA`: it reports a formula to fix, not a market-data condition, and `#N/A` would read as "TWS has no value for this". The extra-argument message, which opens `Unexpected extra argument` and then names the argument, the field the formula did resolve to, and the remedy: a `status` field answers for the connection as a whole and a metadata field for the whole add-in, so neither takes an account, a contract, or a second field — name one field per cell (see [Resolution order](#resolution-order)). And `#AMBIGUOUS-CONNECTION`, which a `status` cell carrying no connection argument publishes once it can no longer say which connection it means — it had piggybacked the sole connection, and a second connection has since been created. The text names the `host:port` it had been reading and the remedy: add a connection argument. See [Connection arguments](#connection-arguments).
