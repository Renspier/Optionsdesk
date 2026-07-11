# OptionsDesk

Single-file (`index.html`) portfolio tracker + trade-idea generator for a DEGIRO
account trading DAX index options (ODAX), mainly bull put spreads. No backend,
no build step — plain JS, runs by opening the file in a browser.

## Data sources

- `Portfolio.csv` / `Transactions.csv`: DEGIRO exports checked into this repo
  as reference snapshots. DEGIRO has no API, so refreshing these means
  re-exporting from DEGIRO and re-importing via the app's DEGIRO tab (or
  pasting a screenshot into chat).
- DAX spot + VDAX-NEW (implied vol): fetched client-side from Stooq
  (`stooq.com/q/d/l/...`, CORS-friendly, no key). Delayed, not real-time.
- ODAX strikes/expiries + pricing: fetched client-side from the Deutsche
  Börse Eurex GraphQL API (`api.developer.deutsche-boerse.com/eurex-prod-graphql/`).
  The schema is PascalCase and response-wrapped — e.g. the `Contracts` query
  returns `{ data: [Contracts] }` (the item type is also called `Contracts`),
  filtered with `filter:{Product:{eq:"ODAX"}}`. Real field names on a
  contract: `ContractID`, `ExpirationDate`, `Strike`, `CallPut`,
  `PreviousDaySettlementPrice`, `OptionsDelta` — see `loadIdeasChain()` in
  `index.html`. This schema was reverse-engineered via the API's own GraphQL
  error messages plus manual introspection (`__type` queries) since the API
  isn't reachable from this project's remote dev sandbox — only from the
  user's own browser. Re-verify field names via introspection if a query
  starts erroring again (the schema/field casing has already shifted once).
- Option premiums: the Eurex `Contracts` query above includes
  `PreviousDaySettlementPrice` (real official settlement, not live intraday)
  — used as the premium whenever present. When a contract has no settlement
  yet (e.g. a brand-new strike), falls back to a Black-Scholes estimate
  (flat VDAX-NEW as IV, no skew, r=2.5% constant, q=0) — see `bsPut()` in
  `index.html`. Each Trade Idea card is badged EUREX SETTLEMENT / PARTLY
  ESTIMATED / ESTIMATED depending on which source priced its two legs.
  Even settlement-priced ideas are prior-day data, not live quotes — always
  verify the actual premium in DEGIRO before entering a trade.
- IBKR Web API live quotes were discussed as a future upgrade (would need the
  user's IBKR API access enabled + running the Client Portal Gateway) but are
  **not implemented** — out of scope for a remote sandbox with no access to
  the user's local network.

## "Give me trade options" workflow

Whenever asked for trade ideas (in the app's IDEAS tab, or conversationally
in chat), the method is the same:

1. Inputs needed: current DAX spot, current VDAX-NEW level (or override IV%),
   and a max-risk-per-trade budget in EUR.
2. Generate candidates for **three strategies** — Bull Put Spread (credit),
   Bear Put Spread (debit), Bull Call Spread (debit) — each with 3 profiles
   (Conservative/Balanced/Aggressive) that vary by **DTE and width** (not a
   fixed OTM%). By default the near strike uses a vol-scaled offset from spot:
   `nearStrike ≈ spot * (1 + dirSign * kSigma * IV * sqrt(T))` (dirSign is -1
   for puts/below spot, +1 for calls/above spot) — this keeps strikes
   realistic across different vol/DTE combos instead of producing
   near-worthless or absurdly risky spreads. See `STRATEGY_SPECS` and
   `IDEA_PROFILES` in `index.html` for the current parameters.
   If an **expected move %** is given (optional input, e.g. "5"), the SHORT
   strike (whichever leg is sold — the crossing point where the spread's max
   profit is achieved, not an option Greek) is instead placed exactly that %
   away from spot: `shortStrike = spot * (1 + dirSign * movePct/100)`, with
   the other (bought) leg `width` further out for a credit spread or `width`
   closer to spot for a debit spread — applied uniformly across all three
   profiles for that generation (they still differ by DTE/width, so absolute
   strikes still differ per profile even at the same expected move %). Each
   leg's actual option delta (Eurex live `OptionsDelta` when present, else
   Black-Scholes `bsDeltaAbs()`) is shown alongside it for reference only —
   it does not drive selection.
   Bear Put Spread reuses the *exact same two strikes* as Bull Put Spread —
   it's the mirror position (buy/sell legs swapped) on the same put pair, not
   a separately-selected spread. Bull Call Spread needs the call side of the
   chain and strikes above spot; call premiums are estimated via put-call
   parity (`bsCall()`) when no live settlement price is available.
3. Snap to real strikes/expiries from the Eurex chain (not arbitrary levels).
4. Size contracts so max risk stays within the stated budget (never round up
   past it — see the `maxRiskEURPerContract > state.ideasMaxRisk` guard).
   For debit spreads the debit paid **is** the max risk (no further subtraction
   from width); for the credit spread max risk is `width - credit`.
5. Report for each idea: strikes, width, contracts, **net credit received**
   (credit spread) or **net debit paid** (debit spreads), max risk, max
   profit, breakeven, approx POP (relative to breakeven, not the short
   strike), ROI on risk, and portfolio impact (open credit / portfolio risk
   / best-case open P&L, before → after if the trade were added — debit
   spreads reduce the "open credit" figure rather than adding to it).
6. Always caveat: premiums are prior-day Eurex settlement prices where
   available, else Black-Scholes estimates — neither is a live DEGIRO
   quote. Confirm actual fill price before trading. Deep-OTM debit spreads
   can show very high ROI% (small debit, big theoretical payoff) alongside a
   correspondingly low approx POP — that's expected, not a bug.

When doing this conversationally (screenshot input instead of the app), ask
for current DAX level + relevant ODAX premiums if not provided, and present
the same fields for each idea across all three strategies (unless the user
asks for one strategy specifically).
