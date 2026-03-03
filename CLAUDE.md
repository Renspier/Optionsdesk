# CLAUDE.md — AI Assistant Guide for Optionsdesk

## Project Overview

**Optionsdesk** is a self-contained, single-file web application for options traders. It enables tracking multi-leg option positions (spreads), importing trade history from the DeGiro broker, analysing P&L and win rates, monitoring implied volatility (VDAX), and deploying updates directly to GitHub Pages.

The entire application lives in one file: `index.html` (~2,100 lines, ~110 KB). There are no external npm dependencies, no build tools, and no backend server.

---

## Repository Layout

```
Optionsdesk/
├── index.html    # Complete single-file application (HTML + CSS + JS)
└── README.md     # Minimal project description
```

All code — markup, styles, and logic — is in `index.html`.

---

## Technology Stack

| Layer        | Technology                                |
|--------------|-------------------------------------------|
| Language     | Vanilla JavaScript (ES5-compatible)       |
| Markup       | HTML5 (inline in index.html)              |
| Styling      | CSS3 (embedded `<style>` block)           |
| Persistence  | Browser `localStorage` (credentials only) |
| Data import  | CSV file upload + FileReader API          |
| Market data  | Deutsche Börse Eurex GraphQL API          |
| Volatility   | Yahoo Finance (VDAX-NEW / V1X.DE)         |
| Deployment   | GitHub Pages via GitHub REST API          |

No TypeScript, no React/Vue/Angular, no bundlers, no test framework.

---

## Architecture

### State Management

There is one global mutable `state` object. All application data lives here:

```javascript
state = {
  view: 'dashboard',       // Active view
  positions: [],           // Trading positions (open/closed)
  transactions: [],        // Trade history
  selectedId: null,        // Currently selected position
  eurexResults: [],        // Eurex market search results
  eurexContracts: [],      // Contracts for selected product
  importStep: 'idle',      // CSV import wizard step
  importTxns: [],          // Parsed CSV transactions
  importPositions: [],     // Detected positions from CSV
  txnHistTxns: [],         // DeGiro transaction history
  vdax: {},                // VDAX/IV data
  vdaxLoading: false,
  ghToken: '',             // GitHub PAT (from localStorage)
  ghRepo: '',              // GitHub repo (from localStorage)
  ghBranch: 'main',        // GitHub branch (from localStorage)
  newPos: null,            // Position being created
  newLegs: []              // Legs for new position
}
```

**State mutation pattern:**
1. User triggers an event listener (registered by `bindEvents()`)
2. Handler mutates `state` directly (e.g., `state.view = 'detail'`)
3. Handler calls `render()` to regenerate the entire UI
4. `render()` calls view-specific render functions
5. `bindEvents()` re-attaches all event listeners

### Render Cycle

```
User action → event handler → mutate state → render() → innerHTML update → bindEvents()
```

`render()` picks the active view from `state.view` and delegates to a view function:

| `state.view`   | Render function         | Purpose                              |
|----------------|-------------------------|--------------------------------------|
| `dashboard`    | `renderDashboard()`     | Portfolio overview and open P&L      |
| `positions`    | `renderPositions()`     | All positions with filtering         |
| `detail`       | `renderDetail()`        | Single position, legs, trade history |
| `analytics`    | `renderAnalytics()`     | 8 analysis views from transaction history |
| `market`       | `renderMarket()`        | Eurex contract search                |
| `import`       | `renderImport()`        | CSV import wizard                    |
| `add`          | `renderAdd()`           | Manual position creation             |
| `github`       | `renderGitHub()`        | GitHub Pages deployment              |

---

## Key Constants and Configuration

Defined near the top of the `<script>` block:

```javascript
var EUREX_ENDPOINT = 'https://api.developer.deutsche-boerse.com/eurex-prod-graphql/';
var EUREX_KEY      = '68cdafd2-c5c1-49be-8558-37244ab4f513';  // Hardcoded API key
var MULTIPLIER     = 5;   // EUR 5 per index point (ODAX)
var MONTH_MAP      = { jan:'01', feb:'02', ... };  // DeGiro month abbreviation map
var SPREAD_TYPES   = ['Bull Call Spread', 'Bear Put Spread', 'Iron Condor', ...];
var STATUSES       = ['Open', 'Closed', 'Expired'];
```

There are no environment variables or `.env` files. Configuration is either hardcoded or stored in `localStorage` (GitHub credentials only).

---

## Data Models

### Position

```javascript
{
  id: '<timestamp>',
  name: 'My Spread',
  type: 'Bull Call Spread',  // SPREAD_TYPES value
  status: 'Open',            // STATUSES value
  legs: [Leg, ...],
  transactions: [Transaction, ...]
}
```

### Leg

```javascript
{
  id: '<timestamp>',
  ticker: 'ODAX',
  optType: 'C',              // 'C' (call) or 'P' (put)
  strike: 19000,
  expiry: '2026-02-20',
  direction: 'buy',          // 'buy' or 'sell'
  contracts: 1,
  entryPremium: 150,
  currentPremium: 200
}
```

### Transaction

```javascript
{
  date: '2025-01-15',
  ticker: 'ODAX',
  optType: 'C',
  strike: 19000,
  expiry: '2026-02-20',
  direction: 'buy',
  contracts: 1,
  price: 150,
  currency: 'EUR'
}
```

---

## P&L Calculation

The core P&L formula (see comments in source):

```
P&L = sign × (currentPremium - entryPremium) × netContracts × MULTIPLIER
```

- **sign**: `+1` for buy, `−1` for sell
- **netContracts**: net position across legs (buys minus sells)
- **MULTIPLIER**: `5` (EUR 5 per index point for ODAX)

---

## CSV Import

Two DeGiro CSV formats are supported:

| Parser                    | Source                          | Notes                              |
|---------------------------|---------------------------------|------------------------------------|
| `parseDeGiroCSV()`        | DeGiro portfolio export         | Detects Dutch and English headers  |
| `parseDeGiroTransactions()` | DeGiro transaction history    | Parses option name strings         |

**Option name parsing** (e.g., `"ODAX p19000.00 20feb26"`):
- `ticker`: `ODAX`
- `optType`: `P` (put)
- `strike`: `19000.00`
- `expiry`: parsed via `MONTH_MAP`

---

## External API Integrations

### 1. Eurex GraphQL (Deutsche Börse)

- **Endpoint**: `EUREX_ENDPOINT` constant
- **Auth**: `X-DBP-APIKEY` header with `EUREX_KEY`
- **Queries**:
  - Products search by product ID (`ODAX`, `OESX`, etc.)
  - Contracts query for available expiries and strikes

### 2. Yahoo Finance (VDAX)

- Used to fetch V1X.DE (VDAX-NEW) historical data
- URL is dynamically constructed based on date range
- Calculates **IV Rank** (percentile over 252-day window)
- Calculates **IV Crush** (IV change from trade entry to current)

### 3. GitHub REST API

- **Purpose**: Deploy `index.html` to a GitHub Pages repository
- **Auth**: GitHub Personal Access Token (PAT) stored in `localStorage`
- **Endpoint**: `GET/PUT /repos/{owner}/{repo}/contents/index.html`
- Reads current SHA, then updates file content via base64-encoded PUT

---

## Self-Deployment Pattern

The app can deploy itself to GitHub Pages through the `renderGitHub()` view. Users supply:
- GitHub PAT (`ghToken`) — stored to `localStorage`
- Repository (`ghRepo`) — e.g., `username/my-optionsdesk`
- Branch (`ghBranch`) — default `main`

The app calls GitHub API to overwrite `index.html` in the target repository.

There is also a `downloadSelf()` utility that opens the page content in a new window (primarily for iOS Safari "Save to Files" workflows).

---

## Analytics Views

The analytics view (`state.view === 'analytics'`) provides eight sub-analyses driven by `state.txnHistTxns` (DeGiro transaction history):

1. **P&L by closed position** — realized gains/losses per trade
2. **Rolling analysis** — positions that were rolled/adjusted
3. **Win rate** — winners vs. losers breakdown
4. **Short strike entries** — entry strikes for short legs
5. **Premium decay capture** — theta capture as percentage of max premium
6. **IV Rank history** — VDAX percentile at trade entry
7. **IV crush impact** — IV change from entry to close
8. **Additional metrics** — additional computed stats

---

## Code Style Conventions

Since there is no linter or formatter, follow the existing style:

- **Indentation**: 2 spaces
- **Semicolons**: always present
- **Variable declarations**: `var` (not `const`/`let`) for ES5 compatibility
- **Function declarations**: `var fn = function() {}` style
- **Arrow functions**: avoid — not used in the codebase
- **String concatenation**: use `+` for HTML strings, not template literals
- **Quotes**: single quotes for JavaScript strings, double quotes inside HTML attribute values
- **Constants**: `UPPER_SNAKE_CASE`
- **State fields / local vars**: `camelCase`
- **Null checks**: `x != null ? x : fallback` pattern
- **Error handling**: `try-catch` blocks around localStorage access and API calls
- **HTML generation**: build strings with `var html = ''; html += '...'; return html;`

---

## Development Workflow

There is no build step. To develop:

1. Open `index.html` directly in a modern browser (Chrome or Firefox recommended).
2. Edit `index.html` in any text editor.
3. Refresh the browser to see changes.
4. For API testing (Eurex), ensure network connectivity.
5. Deploy via the built-in GitHub view or manually copy the file.

### When making changes

- **All changes go into `index.html`** — there are no other source files.
- Keep the file self-contained. Do not introduce external script or style imports.
- Maintain ES5 compatibility unless specifically upgrading the project.
- When adding a new view, add a `renderXxx()` function and a branch in the main `render()` dispatcher.
- When adding new state fields, initialise them in the `state` object near the top of the script.
- Re-attach event listeners in `bindEvents()` after every render.

---

## Common Pitfalls

- **Do not use `const`/`let`** — the codebase uses `var` throughout for compatibility.
- **Do not use arrow functions** — use `function` expressions.
- **Do not use template literals** — use string concatenation.
- **Do not introduce npm or build tools** — the zero-dependency design is intentional.
- **The hardcoded `EUREX_KEY` is a public demo key** — do not treat it as a secret.
- **GitHub PAT is sensitive** — it is stored only in `localStorage` and never logged.
- **No test suite exists** — verify changes manually in the browser.
- **`localStorage` is used only for credentials** — all position/transaction data is in-memory and lost on refresh unless the user saves and reloads the file.

---

## Test Suite

A browser-based test suite lives in `tests.html`. It has zero external dependencies and requires no build step — open it directly in any modern browser.

### Running the tests

```
# Option 1: open directly in the browser
open tests.html          # macOS
xdg-open tests.html      # Linux

# Option 2: serve locally (avoids any file:// quirks)
python3 -m http.server 8080
# then navigate to http://localhost:8080/tests.html
```

### What is tested

| Suite                          | Functions covered                             |
|--------------------------------|-----------------------------------------------|
| `fmt`                          | Number formatting, null/undefined handling    |
| `fmtE`                         | Euro sign formatting                          |
| `fmtPct`                       | Percentage sign formatting                    |
| `pcClass`                      | CSS class selection by sign                   |
| `esc`                          | HTML special-character escaping               |
| `parseNum`                     | European number parsing (comma decimal)       |
| `parseCsvLine`                 | CSV tokeniser, quoted fields, separators      |
| `parseOptionName`              | DeGiro option name parser, Dutch month aliases|
| `calcPnL`                      | P&L formula, BUY/SELL sign, multiplier, pct  |
| `parseDeGiroCSV`               | Full portfolio CSV parse, Dutch & English     |
| `parseDeGiroTransactions`      | Transaction history CSV parse                 |
| `groupIntoPositions`           | Spread type detection from transaction list   |
| `enrichLegsWithTransactions`   | VWAP entry price, net contracts, matching     |

### Test runner implementation

`tests.html` contains a minimal self-contained test runner (no Jest/Mocha):
- `describe(name, fn)` — groups related tests
- `test(name, fn)` — runs a single assertion block
- `expect(actual).toBe/toEqual/toBeNull/toBeCloseTo/toThrow(expected)` — assertion helpers

Results are rendered as a pass/fail summary with a progress bar. The browser tab title also reflects pass/fail state for quick glance.

### Adding new tests

1. Copy the pure function(s) you want to test from `index.html` into the "Source under test" block in `tests.html`.
2. Add a `describe` block at the end of the test section.
3. Open `tests.html` in the browser to verify.

> **Keep functions in sync.** When you modify a function in `index.html`, update the copy in `tests.html` as well, otherwise tests may pass against stale code.

---

## Git Workflow

- Default development branch: `master`
- Feature branches follow the pattern: `claude/<description>-<id>`
- Commit messages should be clear and descriptive.
- There are no pre-commit hooks, CI pipelines, or automated checks.

---

## File Size Guidance

The current `index.html` is ~110 KB. When adding features:
- Keep HTML string templates concise.
- Avoid large inline data sets.
- Minimise repeated style declarations by reusing existing CSS classes.
