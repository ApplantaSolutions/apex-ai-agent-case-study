# APEX — architecture notes

Supplementary detail to the [README](../README.md). Still a case study, not
source. Alpaca **paper** trading only.

## The frozen evidence snapshot

Both AI roles reason over **one** object: a content-hashed capture of the
underlying price, the option chain and quotes, greeks/IV, and the market clock,
taken at decision time. It is hashed at capture (`evidence_sha256`). The
Proponent and the Skeptic each echo that hash back; if either doesn't match, the
decision fails closed before the Governor is ever consulted. This is what lets
APEX claim the two agents genuinely disagreed *about the same facts* rather than
about different reads of the market.

## Proponent / Skeptic isolation

- **Proponent** produces a structured proposal for exactly one single-leg
  contract: thesis, cited evidence points (each pointing at a field in the
  snapshot), acknowledged risks, a recommended action, a conviction level.
- **Skeptic** runs in a separate context — it does not see the Proponent's
  private reasoning, only the same snapshot and the concrete proposal. Each
  objection is structured: a `checkable_against` path into the evidence, an
  `if_true_then` consequence, and a severity. Objections that are vague or don't
  resolve against real data are caught by the assessor.
- **Challenge assessor** is rule-based, not another LLM. It verifies the Skeptic
  engaged: objection paths resolve, no blocklisted vagueness, concrete
  consequences, actual engagement with the Proponent's cited points. A
  schema-valid but non-genuine challenge is recorded as
  `skeptic_challenge_insufficient` and the Governor refuses — it is never
  silently treated as "no objection."
- Provider is abstracted (`anthropic` in prod, a local CLI in dev, a stub in
  tests). `APEX_USE_AI=false` runs a deterministic-only path. Bounded retries and
  a daily call ceiling (checked against Ledger entries) cap spend. No hidden
  chain-of-thought is persisted or displayed.

## The Governor as a pure function

```
decide(snapshot, proposal, policy, context, skeptic_signal) -> verdict + reason_codes
```

No model, no randomness, no I/O. Same inputs always produce the same output. That
output, plus a **policy fingerprint** (a hash of the active deterministic
limits), is persisted with every decision. Reason codes are concrete and
enumerable — e.g. `spread_too_wide`, `stale_quote`, `expiration_too_close`,
`max_concurrent_positions_reached`, `order_already_pending_for_symbol`,
`skeptic_veto`, `skeptic_challenge_insufficient`.

The Governor decides **whether** the proposed action is permitted under the
human-set limits. It does not decide whether a trade "looks good," and it does
not place the order.

## Execution as a separate step

`apex/execution.py` (non-AI) is a distinct component from the Governor. Given an
authorized decision it submits exactly one order with a deterministic
`client_order_id` (so a retry can't double-submit), and it submits nothing when
the Governor refuses or the interlock is closed. It contains no decision logic
and cannot originate or approve a trade. "Decide whether" and "carry out" stay
separately inspectable.

## The autonomous loop

A single sequential worker (no queues, no parallel agents). Each cycle, in order:

1. **Reconcile** against the broker — the broker is the source of truth;
   persisted on drift.
2. **Deterministic exits** — position-management logic runs *before any AI*.
3. **Drift check** — unexplained drift halts new entries (existing positions
   still managed).
4. Market-closed → a compact liveness record, nothing else.
5. At capacity → skip.
6. Deterministic pre-filter.
7. **Anti-resubmission** — a materially identical candidate that the Governor
   already rejected within the cooldown is suppressed with no AI call.
8. Proponent → Skeptic → assessor → Governor.
9. Execute (idempotent) if and only if authorized.

On restart, step 1 runs first and always: APEX recovers its true state from the
broker before it does anything else.

## The Ledger and replay

Every evaluated decision is a hash-chained row (each row chains to the prior
hash; a single sequential writer, so ordering is unambiguous). Altering any past
row breaks verification at exactly that row.

**Replay** rebuilds a past decision from its persisted frozen evidence,
proposal, policy, context and Skeptic signal, and re-runs the deterministic
Governor. It compares verdict, reason codes, inputs hash, and policy fingerprint
against what was recorded. The AI text is read back verbatim — never
regenerated. Replay does not fabricate P&L or backtest.

## Alpaca surface

- `alpaca-py` for account, options market data (free indicative feed —
  permitted), and the full order lifecycle. Paper endpoint only, verified.
- Official Alpaca **MCP server** integrated as an independent read-only channel:
  spawned over stdio, a 4-tool read-only allowlist, write-verb tools refused
  locally before the server is contacted. Used to cross-check account identity
  and status against the SDK. Every MCP call is a Ledger entry marked
  `authorization_path_touched: false`, `governor_invoked: false`. It verifies the
  connection — not the trading decisions.
- Deployed as a managed Postgres database + a read-only web dashboard; the
  autonomous worker runs against the same database.

## What this is not

Not a strategy. Not a backtester. Not multi-leg. Not a profit claim. The
strategy layer that generates candidates is deliberately simple — the point of
the project is the governance, challenge, and audit layer around a consequential
AI-assisted decision, not the alpha.
