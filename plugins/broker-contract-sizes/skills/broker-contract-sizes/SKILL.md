---
name: broker-contract-sizes
description: "Verify a broker's contract_size per instrument so your computed P&L matches the broker-reported number. Audit the contract-size table that survived a broker migration."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Broker Contract Sizes

## When to Use

- Closed-trade P&L in the app doesn't match the broker's reported
  value (any direction: 10× too high, 10× too low, 1000× too high)
- Porting a contract-size table from one broker to another
  (Genesis FX → the broker, demo → live, MT5 → the broker)
- Adding a new instrument class (crypto, indices, metals, FX,
  energy) to a broker integration
- Auditing an existing `contract-sizes.ts` file for values that
  survived a broker migration but were never re-verified

## What this skill covers

A CFD broker's `contract_size` per instrument is the multiplier
that converts `lots × price_diff` into broker-reported P&L.
Every instrument class (crypto, indices, metals, FX, energy) has
its own convention and **every broker differs**:

| Broker | XAUUSD 0.01 lot × $4.32 | BTCUSD 0.1 lot × $100 | US30 0.01 lot × 67pt |
|---|---|---|---|
| Genesis FX (CFD) | $4.32 (contract=100) | $10.00 (contract=1000) | $6.70 (contract=10) |
| the broker | $4.32 (contract=100) | $10.00 (contract=1) | $6.70 (contract=10) |
| MT5 default | varies by symbol | varies | varies |

Same P&L formula `lots × price_diff × contract_size`, but the
multiplier is broker-specific. Wrong multiplier = wrong P&L on
every closed trade.

This skill captures the **verification workflow** that
prevents the most common failure mode: shipping contract values
based on industry-standard conventions instead of actual
broker-reported data. The pattern is class-level — applies to
any broker integration, not just the broker.

## The verification workflow (always do this first)

**Before editing the contract_size table, gather ground truth:**

1. **Pick a known closed trade from the broker's UI.** Login to
   the broker's web/desktop app as the user, find a recent closed
   trade with all four known fields visible:
   - Lots (e.g. 0.03)
   - Price diff (e.g. 160.08)
   - Broker-reported P&L (e.g. $48.02)
   - Side (BUY/SELL)

2. **Solve for contract_size:**
   ```
   contract_size = broker_pnl / (lots × price_diff)
   ```
   Example: `$48.02 / (0.03 × 160.08) = 10.0` → contract=10

3. **Cross-check against another trade.** Don't trust one
   data point — find a second closed trade on a different
   instrument class (e.g. one XAUUSD + one NAS100) and solve
   again. Each instrument class has its own convention; what
   holds for XAUUSD doesn't hold for XAUUSD-as-metal vs
   XAUUSD-as-future.

4. **Look for the broker's own documentation.** Most brokers
   publish contract sizes in their instrument specs. If their
   docs contradict your math, the docs are the source of truth
   (and your math is the test for the docs).

5. **Only edit the table after at least two independent
   verifications.** One screenshot + one broker-doc reference
   is the minimum.

**Anti-pattern — DO NOT do this:**

> "the broker uses standard CFD conventions: crypto = 1 lot = 1
> coin, indices = 1 lot = $1/pt. So contract=1 for those."
>
> *Three guesses wrong. Verified against actual data:
> the broker BTCUSD contract=1, ETHUSD=1, but NAS100/US30/SPX500
> /GER40/UK100/JPN225 all = 10, not 1. Crypto matches
> convention; indices don't.*

Industry-standard conventions are good starting points but
**always cross-check against real broker data**. The broker
doesn't care what the convention says.

## The contract_size table — by instrument class

### Crypto
the broker convention:
- 1 lot = 1 BTC for BTCUSD/BTCEUR
- 1 lot = 1 ETH for ETHUSD/ETHEUR
- 1 lot = 1 SOL for SOLUSD
- 1 lot = 1000 XRP for XRPUSD (XRP is sub-dollar)
- 1 lot = 1000 DOGE/XLM/ADA/etc. (low-priced coins)

Genesis FX CFD broker convention is uniformly contract=1000 across
all crypto (0.01 lot = 10 units of underlying). **This convention
does NOT transfer to the broker** — the most common porting bug.

### Indices
the broker convention:
- 1 lot = $10 per index point (NAS100, US30, SPX500, GER40, UK100, JPN225)
- contract=10

This is the same as Genesis FX's convention. Most CFD brokers
default to contract=10 for indices.

### Metals
the broker convention:
- XAUUSD: 1 lot = 100 oz (contract=100)
- XAGUSD: 1 lot = 5000 oz (contract=5000)

Verify with: `XAUUSD 0.01 lot × $4.32 diff = $4.32` → contract=100
✓. `XAGUSD 0.01 lot × 50 oz × $0.05 diff = $0.025` → contract=5000 ✓.

### Forex
- Standard lots: 1 lot = 100,000 units of base currency.
  Contract=100000 for EURUSD, GBPUSD, AUDUSD, NZDUSD.
- USDJPY, EURJPY, GBPJPY, AUDJPY: 1 lot = 100,000 JPY × lots.
  Contract ≈ 667 (USDJPY ≈ 150, so 100,000/150 = 667) — this is an
  approximation with ~5-10% drift depending on actual rate at
  trade close time. Acceptable as a tradeoff vs. fetching the
  historical USDJPY rate per trade.
- Cross-currency (USDCAD, USDCHF, EURGBP, AUDCAD, EURCHF):
  contract=100000 but quote currency ≠ USD, so the P&L is in
  the quote currency (CAD, CHF, GBP), not USD. The display layer
  must convert.

### Energy
- USOIL: 1 lot = 1000 barrels (contract=1000)
- UKOIL: 1 lot = 1000 barrels (contract=1000)

the broker matches standard CFD conventions here.

## How to wire contract_size into the P&L math

The formula is the same across all instrument classes:

```ts
const contract_size = getContractSize(symbol);  // from contract-sizes.ts
const pnl = (avgSell - avgBuy) × matchedLots × contract_size;
```

Where:
- `avgBuy` / `avgSell` are weighted-average entry/exit prices
  for the position (matched buy and sell fills, FIFO)
- `matchedLots` is the lot size of the closed portion
- `contract_size` is looked up by symbol name

For BUY positions: `pnl > 0` when `avgSell > avgBuy`.
For SELL positions: `pnl > 0` when `avgSell < avgBuy`.

## The porting checklist (Genesis FX → the broker, etc.)

When porting a contract-size table from one broker to another:

1. **Identify which instrument classes differ.** Crypto is the
   most common porting bug (Genesis FX = 1000, the broker = 1
   for most coins). Indices are usually the same (10/10) but
   verify with actual data.

2. **Reset, don't merge.** Treat the new broker as a fresh
   start. Even if the values look "probably the same," verify
   each row with the workflow above.

3. **Don't trust comments.** Comments like "Verified against
   Genesis FX trade data" are the worst — they're literally
   documenting the WRONG broker. The comment is the
   information, not the value; verify both.

4. **Run a one-shot recompute against the broker.** After
   updating the table, hit
   `/api/<broker>/recompute-pnl?force=true` (or equivalent
   for the broker). It re-derives P&L for every closed trade
   from the new table, fixes any drift in the lifetime P&L
   aggregate stored on the user row.

## Debugging when the table looks wrong

Four tools, in order.

### 1. Solve for contract_size from one closed trade

```python
broker_pnl = 48.02
lots = 0.03
price_diff = 51110.79 - 50950.71  # = 160.08
contract_size = broker_pnl / (lots * price_diff)
# → 10.0
```

If the answer is a clean integer (1, 10, 100, 1000), you've
identified the right value. If it's a weird decimal like 9.99
or 10.0003, check for rounding in the broker's display.

### 2. Check a second trade in a different instrument class

Different class confirms the lookup is correctly keyed by symbol.
If XAUUSD also matches with contract=100, the table is correct
for both metals and indices.

### 3. Check the table file's own verified-data comments

Many contract-size tables include a "VERIFIED CONTRACT SIZES"
comment block listing test trades that were used to derive each
value. These are **gold** — they were correct at write time.
If your math doesn't match the comment, the comment is
authoritative; the current value is wrong.

### 4. Edge case: when contract=1 fails the math

If `contract=1` gives `lots × diff` but the broker reports a
larger value, the contract isn't 1 — it's something else.
Don't paper over with `*1000` without verifying. Solve the
math first.

## Anti-patterns

### ❌ Don't port a comment that documents the wrong broker

```
// Verified against a the broker account:
//   NAS100 0.01 lot × 67.61 diff → $6.76 (broker-reported)
NAS100: 10,
```

This comment is **trade data from the WRONG broker**. If
$6.76 came from a Genesis FX report, the contract=10 is a
Genesis FX rule, not a the broker rule. The comment sounds like
evidence but is actually the bug. Always verify the comment
itself, not just the value.

### ❌ Don't trust industry-standard conventions

- "Crypto is 1 lot = 1 coin everywhere" — wrong for Genesis FX
- "Indices are 1 lot = $1/pt everywhere" — wrong for both Genesis
  FX and the broker (both are $10/pt)
- "Forex is always contract=100000" — wrong for JPY pairs

Use conventions as a starting point. Verify against actual
broker-reported data before committing the value.

### ❌ Don't paper over with multipliers when the math is wrong

If `lots × diff × contract` gives the wrong answer, **solve
for the correct contract** rather than guessing a multiplier
that makes the math work. The right answer is usually a clean
integer (1, 10, 100, 1000) — a 10× or 1000× bug in the contract
value is the most common pattern.

### ❌ Don't leave old stored P&L values stale after a fix

Once the contract_size is corrected, the lifetime P&L stored on
the user row is still wrong (it was computed with the old value).
Two paths to fix:

- **User-triggered:** hit `/api/<broker>/recompute-pnl` to
  re-derive all stored values
- **Auto-trigger:** the signal write-back endpoint (the handler
  that fires when a TP/SL is hit) recomputes the lifetime value
  as a side-effect. Natural refresh window:
  5 minutes after any new position-close event.

Don't ship the contract-size fix without telling the user about
the stale lifetime value — they'll see correct per-trade P&L
but wrong lifetime aggregate, which is confusing.

## Worked example

**Symptom:** the dashboard showed US30 closed trades with P&L 10×
too small vs the broker's authoritative number.

**Math (from the broker's own trade report):**
- the broker: US30 0.03 lot × 160.08 diff = $48.02
- App: same trade = $4.80
- contract_1 = $48.02 / (0.03 × 160.08) = 10.0
- contract_2 = $18.32 / (0.03 × 61.08) = 10.0
- cross-check: NAS100 0.01 × 67.61 × ? = $6.76 → contract=10
  (verified value from the old comment block)

**Root cause:** an earlier change had switched indices from
contract=10 to contract=1, citing "industry-standard convention"
and a note in the risk-calculator view component that said
contractSize=1. That note was wrong (the risk calculator doesn't
use that field), and the convention was wrong for the broker.

**Fix:** revert indices to contract=10, add a verification comment
block with the two new test trades.

**Lesson:** the convention was wrong; the verified data was
right. The existing comment block was the canonical source — it
just got dismissed as "Genesis FX data" because the note said so.
Wrong. It was the broker data the whole time.

## Other broker-specific patterns

### the broker contract conventions in detail

the broker uses different conventions per instrument class:

- **Crypto majors** (BTC, ETH, SOL, etc.): contract=1 (1 lot = 1
  coin)
- **Crypto sub-dollar** (XRP, DOGE, XLM, SHIB): contract=1000
  (1 lot = 1000 coins)
- **Indices** (NAS100, US30, SPX500, GER40, UK100, JPN225):
  contract=10 (1 lot = $10/pt)
- **Metals** (XAU, XAG): contract=N where N = oz per lot
  (gold=100, silver=5000)
- **Forex standard**: contract=100000
- **Forex JPY**: contract ≈ 667 (USDJPY ≈ 150; 100000/150)
- **Energy** (USOIL, UKOIL): contract=1000

### MT5 contract conventions (for cross-broker compatibility)

MT5 defaults are similar to the broker for indices (1 lot =
$1/pt in the index's base currency). Crypto is mostly 1 lot =
1 coin. Forex is 100000. JPY pairs may require quote-currency
conversion to USD before P&L display.

If you're building a multi-broker abstraction (registry pattern
in `src/lib/brokers/`), the contract_size is a property of the
**broker**, not the **instrument**. Each adapter overrides it
per symbol.

## File location conventions

- the broker: `src/lib/the broker/contract-sizes.ts`
- Cross-broker: `src/lib/brokers/<broker-name>.ts` (each adapter
  has its own table)
- Legacy broker after a migration: `src/lib/<legacy-broker>/contract-sizes.ts`
  (delete after migration, do not keep as a "just in case" file —
  it'll bit-rot and confuse future sessions)

The file should have a top-level comment block documenting the
verified contract sizes for at least 2-3 instruments, with the
math. Future sessions will check those comments first before
touching the values.

## Pitfalls

- **Don't override a verified value based on convention alone.**
  Verified data wins every time. If the math doesn't match
  convention, the convention is wrong for this broker.

- **Don't commit a contract_size change without a test trade.**
  Add a comment block citing at least one test trade from the
  broker's UI that confirms the value.

- **Don't keep the old contract_size in a comment as "previous
  value" without explaining why it was wrong.** Future readers
  will assume it's a valid alternative. Either delete the
  historical value or explain the bug.

- **Don't use `*1000` or `*10` multipliers as band-aids.**
  Solve the math first; if the answer is a clean integer, you
  have the right contract value. Multipliers that "happen to
  work" are coincidences that break when the next instrument
  comes through.

- **Don't trust the risk-calculator view or any other doc that
  quotes contract_size values.** That file is a separate
  surface with its own math (pipValue-based). The
  contract-sizes.ts table is the canonical source. Cross-check
  both before either wins an argument.
