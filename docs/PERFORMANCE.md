# Performance cost benchmarks

The contract keeps gas/resource expectations transparent by measuring major public entrypoints in `contracts/src/tests/cost_benchmarks.rs`.

## Generate the table

Run:

```text
cargo test --package xelma-contract cost_benchmarks -- --nocapture
```

Each benchmark prints both a machine-readable line:

```text
[cost-benchmark] name=create_round cpu_instructions=... memory_bytes=...
```

and a markdown table row that can be copied into this document.

## Latest local benchmark table

The exact numbers depend on the Soroban SDK version and host runtime. Regenerate the table before release and replace the rows below with the `--nocapture` output.

| Function / path | CPU instructions | Memory bytes |
|---|---:|---:|
| `create_round` | _regenerate_ | _regenerate_ |
| `place_bet` | _regenerate_ | _regenerate_ |
| `resolve_round` | _regenerate_ | _regenerate_ |
| `claim_winnings` | _regenerate_ | _regenerate_ |
| `get_updown_positions_page` | _regenerate_ | _regenerate_ |
| `get_precision_predictions_page` | _regenerate_ | _regenerate_ |
| `get_precision_predictions_cursor` | _regenerate_ | _regenerate_ |
| `get_updown_positions_cursor` | _regenerate_ | _regenerate_ |
| `get_leaderboard_by_wins` | _regenerate_ | _regenerate_ |
| `get_leaderboard_by_streak` | _regenerate_ | _regenerate_ |

## Regression policy

Every benchmark asserts the measured CPU instructions and memory bytes stay within the standard Soroban per-transaction resource budget. Treat any benchmark failure as a hard regression. If a passing run still shows a spike of more than 20% versus the last published table, call it out in the pull request and either optimize the path or document the reason for the higher cost.

## CI artifact guidance

The `rust-test` job in `.github/workflows/ci.yml` runs the generation command
above with `--nocapture`, tees the output to `cost-benchmarks.log`, and
uploads it as a `cost-benchmarks` build artifact (via `actions/upload-artifact`,
7-day retention) — including on a failed/regressed run, so the exact numbers
that tripped a `*_CPU_MAX`/`*_MEM_MAX` assertion are always reviewable from the
workflow run's Artifacts section, not just the truncated job log. When you
touch a benchmark-sensitive path, download that artifact from your PR's CI
run and paste the relevant rows into this file's table in the same change.

## Pagination query limits (Issue #430)

To prevent adversarial over-limit requests from bypassing CPU/memory budgets,
paginated query functions enforce strict pagination limits:

| Function | Max page size | Error on exceed |
|---|---:|---|
| `get_precision_predictions_cursor` | 100 | `PageSizeExceeded` (94) |
| `get_updown_positions_cursor` | 100 | `PageSizeExceeded` (94) |
| `get_leaderboard_by_wins` | 100 | `PageSizeExceeded` (94) |
| `get_leaderboard_by_streak` | 100 | `PageSizeExceeded` (94) |
| `get_user_archive_history` | 100 | `PageSizeExceeded` (94) |

**Key behaviors**:
- Requests with `limit == 0` or `limit > 100` are rejected with error code 94.
- The limit is **not** clamped; over-limit requests fail fast.
- Valid limits are `1..=100` (inclusive).
- Cursor-based functions (leaderboard, predictions, positions) return `(Vec<T>, Option<Address>)`.
- When results are exhausted, `next_cursor` is `None` and the page is empty.

**Gas guard rationale**:
The 100-item limit ensures that even under worst-case data density (each item fetches
from persistent storage), query CPU and memory consumption remains bounded within
Soroban's per-transaction budget. Rejecting over-limit requests prevents callers from
accidentally or maliciously requesting unbounded batches that could fail during
settlement or cause timeouts.

## Updating a cost-benchmark ceiling

The `*_CPU_MAX`/`*_MEM_MAX` constants in `contracts/src/tests/cost_benchmarks.rs`
are the enforcement mechanism — a benchmark failing them is what "flags a cost
regression." See [`contracts/BENCHMARKS.md`](../contracts/BENCHMARKS.md) for
the full baseline-recording and ceiling-tightening procedure (currently every
path is gated at the full Soroban per-transaction budget; tightening these to
real measured baselines ± tolerance is the documented next step there). Any
PR that intentionally raises a ceiling must also update the table in this file
and in `contracts/BENCHMARKS.md` with the new baseline and the commit/date it
was captured on, exactly like `docs/wasm-size-budget.md`'s baseline-bump
procedure for the separate WASM size gate.
