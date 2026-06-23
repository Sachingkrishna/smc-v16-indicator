# SMC + SMT indicators + strategy (TradingView Pine v6)

Three Pine Script v6 files in this repo (git, branch `main`). Read the changelog
block at the top of each file for full per-version history.

## The three artifacts
1. **smcV16.pine** — "Smart Money Concepts [v20.16]" (~5,300 lines, LuxAlgo-derived).
   OBs/breakers, FVG/iFVG, BOS/CHoCH/MSS, swing+internal structure, liquidity
   sweeps, EQH/EQL, OTE, kill zones, sessions, NDOG/NWOG, rVOL/institutional OB, CVD.
   Exposes ONE hidden plot **"SMT Sweep/BOS/CHoCH Export"** (`smcV16.pine:5202`),
   packed `sweep + 3*BOS + 9*CHoCH` (per field 0/1/2 = none/bull/bear).
2. **smtDivergence.pine** — "SMT + VP + AVWAP + Ice [v3.21]" (~3,300 lines). Multi-symbol
   SMT divergence + Volume Profile (current/prev/composite, naked POCs) + Anchored
   VWAP ± σ bands + Iceberg/absorption + confluence (★N) + dashboard.
3. **smtDivergenceStrategy.pine** — "SMT Divergence Strategy [v1.11]" (~1,200 lines).
   Backtest harness mirroring the SMT engine + entries/exits/risk + filters.

## Cross-indicator wiring
SMT indicator & strategy read smcV16's export via `input.source` ("SMC Sweep Export
source"); decode `%3` (sweep), `floor(/3)%3` (BOS), `floor(/9)%3` (CHoCH); gate
signals on a same-direction sweep. Same chart/symbol/TF; the user wires the dropdown.
NOTE: BOS/CHoCH are ALREADY in that one export — a structure-confluence gate needs
NO new plot (no 64-plot-cap issue). Strategy currently decodes only sweep.

## Opt-in filters (ALL default-OFF → default behaviour = legacy)
- **SMT indicator**: sweep gate (Filter/Tag), min-swing, cooldown+same-level dedup,
  structure-reset, correlation mute (basis Returns/Price), recency, σ confluence
  Touch/Zone.
- **Strategy** mirrors these (no Tag mode — entries are binary) PLUS: Kill Zone
  filter (v1.7), Session-window filter (v1.8).
- **Shared (indicator+strategy)**: comparison-symbol auto-preset (strategy v1.9 /
  indicator v3.21), SL/TP R:R zone boxes (strategy v1.10), one-trade=one-zone fix
  (strategy v1.11).

## Comparison auto-preset (`cmpPreset`, default 'Auto')
Detects the chart instrument and fills Symbol 1/2/3 + polarity:
- XAUUSD or GC → Gold: `XPTUSD(+)`, `USDX(−)`, `XAGUSD(+)`
- GBPUSD or 6B → Pound: `EURUSD(+)`, `USDX(−)`, `USDCHF(−)`
- else → manual Symbol 1/2/3 inputs.
All comparison symbols hardcoded `PEPPERSTONE:` (no broker auto-detect). Detection:
`syminfo.ticker` contains XAUUSD/GBPUSD, or `syminfo.root == GC/6B`. Override done by
reassigning `sym1/2/3` via `:=` (stays `simple` → request.security safe).
**Known limitation**: `input.symbol` widgets CANNOT be rewritten by code — when a
preset is active the Symbol 1/2/3 boxes still show the stale manual values; the
ACTIVE basket is shown in the correlation table + on-chart signal labels, not the
input boxes. This is expected, not a bug.

## REQUIRED workflow (every change)
**Plan → Fix → Commit (each change) → Regression-check.** Bump the in-file version +
changelog on every edit. New inputs go at the END of the input list (TradingView
maps saved settings by input ORDER; mid-list inserts scramble saved settings on
in-place updates). Keep new features default-OFF.

## Git workflow (IMPORTANT — user preference)
Edit and commit **directly in the main repo folder**. `cd "C:\Users\rgksa\Documents\New project\smc-v16-indicator"`
before every git command; commit straight to `main`. **Do NOT use the git worktree.**

## Compile-checking (Pine only compiles in TradingView)
CLI: `node C:\Users\rgksa\Documents\tradingview-mcp\src\cli\index.js` (Node at
`C:\Program Files\nodejs\node.exe`; refresh PATH from Machine+User first).
- `pine analyze --file X` → OFFLINE static check.
- `pine check --file X` → SERVER-SIDE compile (needs CDP); returns
  `{compiled, error_count, warning_count}`. Throws a harmless
  `Assertion failed ... async.c` (exit 9) AFTER printing the JSON — IGNORE it.
Expected: smtDivergence & strategy = 0/0; smcV16 = 0 errors / 8 pre-existing warnings.

## TradingView CDP + MCP (live) — capabilities VERIFIED this session
Launch via `launch-tv-debug.ps1` / the "TradingView (CDP)" shortcut. Verify with
`node <cli> status`. CDP DROPS when the user interacts with TradingView — keep idle.
- **Reads that WORK**: `status`; `screenshot --region full|chart|strategy_tester`
  (use screenshots to read the strategy report).
- **`data strategy` does NOT work** here — returns "No strategy found on chart";
  do not rely on it for the report.
- **`indicator set --inputs` is UNSAFE — DO NOT USE.** It has no entity targeting,
  so it can hit the big SMT study (~131 inputs) and corrupt it (loses ALL inputs,
  goes red). To change settings: edit source defaults + re-add, or the USER changes
  them in the Settings UI. Read results via screenshot.

## Hard gotchas
- **NEVER `indicator set`/setInputs on a study** (esp. the ~131-input smcV16) or any
  color input — corrupts it. And `indicator set` can't be targeted, so avoid it.
- `input.symbol`/input widgets can't be set by code (auto-preset shows stale boxes —
  by design; see above).
- Warning-free discipline: never call a history-reading user function inside
  `not X or func()` (→ "call on every bar" warning). Assign to a var first.
- smcV16 sits at the **64-plot cap** (12 plots + 52 alertconditions; alertcondition
  counts, bgcolor doesn't). Adding a plot needs freeing a slot.
- Compile-clean ≠ correct: repaint / max_bars_back / trading-logic bugs need
  reasoning + visual/backtest verification.
- **Backtest data window is short**: 5m history (made worse by cmp `request.security`
  calls) ≈ a few months. Can't run multi-year/multi-regime tests; treat any
  "optimal" settings as fragile / curve-fitting risk. Prioritise robustness
  (plateaus, slippage stress, in-sample vs out-of-sample) over peak numbers.

## My role expectation
Senior quant developer/trader. Audit for real logic/trading flaws, not just compile.
Honest about uncertainty; verify against current code before asserting (file:line
claims may be stale). Distinguish compile/warning issues (catchable by `pine check`)
from logic/runtime issues (need reasoning + on-chart/backtest verification).

## Git state
Branch `main`. Versions: smcV16 v20.16, smtDivergence **v3.21**, strategy **v1.11**.
Review a change: `git diff HEAD~1` (run from the main repo folder). Use `git log` for
the current head rather than hardcoding a hash here (avoids staleness).
