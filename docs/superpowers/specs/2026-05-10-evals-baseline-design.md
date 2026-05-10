# Evals Baseline — Design Spec

**Date:** 2026-05-10
**Status:** Approved, ready for implementation plan
**Purpose:** Plan 0 of the optimization roadmap. Provides the A/B verification gate that every subsequent architecture change (ReAct analysts, parallelization, dynamic debate, context isolation) must pass before merge.

## Goal

A reproducible offline evaluation harness that runs `TradingAgentsGraph.propagate()` over a fixed dataset of (ticker, decision_date) pairs and produces comparable metrics across runs. Two result files from different commits/configs can be diffed to answer "did this change help?".

## Non-Goals

- Not a research-grade backtester. We accept calendar-window data leakage (≤ 7 weeks) as a tradeoff for tractable implementation.
- Not a CI gate. Runs are on-demand, manual, expensive.
- Not a hyperparameter sweep harness.
- No statistical significance testing — sample size is too small.
- No HTML reports, no DB. JSONL + plain-text tables.

## Methodology

### Dataset

`evals/datasets/baseline.yaml` — pinned, version-controlled list:

- **8 tickers:** AAPL, MSFT, GOOGL, AMZN, META, NVDA, TSLA, JPM
  - First seven concentrate large-cap tech / AI beta exposure
  - JPM provides a low-beta financial-sector control
- **6 decision dates:** equally spaced trading days from window `[T-50, T-30]` (T = dataset creation date)
  - Window guarantees the 20-day forward outcome is observable
  - Calendar-window data leakage stays under ~7 weeks (acceptable per Q2)
- **48 total samples** per A/B run

Dates are written as explicit ISO strings, not computed at runtime — re-running on a different day must produce the same dataset.

### Decision-to-Signal Mapping

`SignalProcessor` already normalizes PM output to one of `{Strong Buy, Buy, Hold, Sell, Strong Sell}` (with `Overweight` → Strong Buy and `Underweight` → Strong Sell). Map to integer signal:

| Processed decision | Signal |
|---|---|
| Strong Buy / Overweight | +2 |
| Buy | +1 |
| Hold | 0 |
| Sell | -1 |
| Strong Sell / Underweight | -2 |

Unrecognized output → `signal = null`, `error = "unparseable: <raw>"`. Sample is excluded from metrics but not from error count.

### Outcome Computation

For each `(ticker, decision_date)` and horizon `N ∈ {5, 20}` trading days:

```
realized_return = close[ticker, decision_date + Nd] / close[ticker, decision_date] - 1
spy_return      = close[SPY,    decision_date + Nd] / close[SPY,    decision_date] - 1
alpha           = realized_return - spy_return
```

Beta is held at 1 (no regression estimate). Closes via yfinance, same vendor as the runtime pipeline.

### Metrics (per horizon)

- **score** (primary) — `mean(signal × alpha)` across samples with non-null signal. Higher is better. Sign-aware, magnitude-aware.
- **IC** — Spearman rank correlation between `signal` and `alpha`.
- **hit_rate** — among non-Hold samples, fraction where `sign(signal) == sign(alpha)`.

Errors and Holds are reported as separate counts, not folded into score.

## Architecture

### Layout

```
evals/
├── __init__.py
├── datasets/
│   └── baseline.yaml          # version-controlled (pinned tickers + dates)
├── results/
│   └── .gitkeep               # gitignore the rest
├── outcomes.py                # alpha/return computation (pure data)
├── scoring.py                 # signal mapping + metrics (pure functions)
├── runner.py                  # orchestrator: dataset → propagate → JSONL
├── compare.py                 # diff two result files
└── cli.py                     # console entrypoint (`python -m evals.cli`)
```

### Module Boundaries

**`outcomes.py`** — given `(ticker, date, horizon_days)`, return `{realized, spy, alpha}`. Pure data layer; no LLM calls. Independently unit-testable with vendor-mocked closes.

**`scoring.py`** — given a list of `(signal, alpha)` tuples, return `{score, ic, hit_rate}`. Pure functions. Unit tests cover happy path, all-Hold, empty, single-sample edge cases.

**`runner.py`** — orchestrator. Loads YAML, instantiates `TradingAgentsGraph` per sample (treating it as a black box via `propagate()`), looks up outcomes, writes JSONL line. Per-sample try/except so one failure doesn't kill the run.

**`compare.py`** — reads two JSONL files, validates dataset identity, prints metrics-diff table + per-sample decision-flip list. No LLM, no vendor calls.

**`cli.py`** — argparse, two subcommands (`run`, `compare`). Thin wrapper.

### Reuse from Existing Code

- `tradingagents/agents/utils/memory.py` already computes `raw_return / alpha_return / holding_days` inside `_resolve_pending_entries`. Extract that block into `evals/outcomes.py:compute_alpha(ticker, date, horizon)`. Update `memory.py` to call the new helper. Single source of truth, no double implementation.
- `tradingagents/graph/signal_processing.py` already normalizes PM output. `runner.py` uses `SignalProcessor` directly; `scoring.py` only sees the normalized 5-class string.

### Configuration

`runner.py` accepts `--config-overrides KEY=VALUE [KEY=VALUE ...]` and applies them to `DEFAULT_CONFIG.copy()` before constructing the graph. No eval-specific config file. Examples:

```bash
--config-overrides llm_provider=anthropic deep_think_llm=claude-opus-4-7
--config-overrides max_debate_rounds=2
```

`backend_url` defaults to `None` (per CLAUDE.md). Type-cast for known keys (max_debate_rounds → int) with a small lookup table; otherwise pass strings through.

## Output Formats

### Per-sample JSONL line (`<sha>-<tag>-<timestamp>.jsonl`)

```json
{
  "ticker": "META",
  "decision_date": "2026-04-02",
  "experiment_id": "d9b9b69-quickdeep-v1",
  "config": {"llm_provider": "anthropic", "deep_think_llm": "claude-opus-4-7"},
  "decision_raw": "BUY",
  "decision_processed": "Buy",
  "signal": 1,
  "outcomes": {
    "5d":  {"realized": 0.024, "spy": 0.011, "alpha": 0.013},
    "20d": {"realized": 0.061, "spy": 0.028, "alpha": 0.033}
  },
  "elapsed_sec": 612,
  "error": null
}
```

Errored samples: `signal: null`, `outcomes: null` if outcome lookup also failed, `error: "<message>"`.

### Summary file (`<sha>-<tag>-<timestamp>.summary.json`)

Written after the run completes (or on Ctrl-C, with whatever samples are done):

```json
{
  "experiment_id": "d9b9b69-quickdeep-v1",
  "git_sha": "d9b9b69",
  "config": {"llm_provider": "anthropic", "deep_think_llm": "claude-opus-4-7"},
  "n_total": 48, "n_completed": 47, "n_errors": 1,
  "metrics": {
    "5d":  {"score": 0.0082, "ic": 0.14, "hit_rate": 0.58, "n_nonhold": 31},
    "20d": {"score": 0.0231, "ic": 0.21, "hit_rate": 0.62, "n_nonhold": 31}
  }
}
```

### Compare table

```
                    baseline    variant     Δ
5d   score          0.008       0.012      +0.004
5d   IC             0.14        0.18       +0.04
5d   hit_rate       0.58        0.62       +0.04
20d  score          0.023       0.031      +0.008
20d  IC             0.21        0.27       +0.06
20d  hit_rate       0.62        0.65       +0.03
errors              1           0          -1
mean_elapsed_sec    612         189        -69%

Decision flips (3 of 47):
  META  2026-04-02  Hold → Buy   (20d alpha: +3.3%)
  TSLA  2026-04-09  Buy  → Sell  (20d alpha: -1.2%)
  JPM   2026-03-26  Hold → Hold  (no flip — config diff only)
```

Compare aborts with an error if dataset (ticker × date set) differs between the two files.

## CLI UX

### `run`

```bash
python -m evals.cli run \
  --dataset evals/datasets/baseline.yaml \
  --tag quickdeep-v1 \
  --config-overrides llm_provider=anthropic deep_think_llm=claude-opus-4-7 \
  [--parallel N]      # default 1; process-level concurrency
  [--resume FILE]     # skip samples already in FILE
  [-y]                # skip dry-run confirmation
  [--allow-dirty]     # allow non-clean working tree
```

Behavior:

1. **Dry-run plan** — prints dataset (ticker × date), config, estimated cost & time, asks for confirmation. `-y` skips.
2. **Git check** — refuses to run on dirty working tree unless `--allow-dirty`. Records `git rev-parse HEAD` in result file.
3. **Per-sample progress line** — `[12/48] META 2026-04-02 ... ✓ Buy  elapsed 542s`
4. **Append-on-completion** — each sample writes its JSONL line as soon as it finishes. No in-memory accumulation; safe to Ctrl-C.
5. **Resume** — `--resume FILE` reads existing lines, builds skip set keyed by `(ticker, date)`, runs only the rest.
6. **Parallelism** — `--parallel N` runs N samples concurrently via `multiprocessing.Pool`. Default 1. Anthropic rate limits make N>2 risky; warn on N>2.

### `compare`

```bash
python -m evals.cli compare \
  evals/results/d9b9b69-baseline-20260510.jsonl \
  evals/results/3608e0b-quickdeep-20260510.jsonl
```

Reads JSONL + adjacent `.summary.json`, prints diff table + decision flips, exits 0.

## Testing

`tests/evals/`:

- `test_outcomes.py` — `compute_alpha` against fixture-mocked yfinance closes (happy path, missing date, weekend rolling, vendor error).
- `test_scoring.py` — score/IC/hit_rate on fixed input arrays; covers all-Hold, empty, all-correct, all-wrong.
- `test_runner.py` — runner with `propagate` mocked to return canned decisions; verifies JSONL line shape, error handling, resume logic.
- `test_compare.py` — compare with two synthetic JSONL files; mismatched datasets must error.

No integration test that actually calls LLMs. The harness is the integration test for the pipeline; we don't test the harness with the pipeline.

## Open Questions Deferred to Plan 1+

These are intentionally out of scope for the eval harness itself:

- How to make the runtime pipeline strictly as-of-date (currently uses real-time vendor data). Plan 1+ may address; eval just records what happens.
- Whether to add per-sector control tickers beyond JPM. Out of scope until baseline metrics suggest sector-skew is masking signal.
- Whether to track token cost per sample. Useful but not required for A/B; add later if needed.

## Success Criteria

The harness is "done" when:

1. `python -m evals.cli run --dataset evals/datasets/baseline.yaml --tag baseline -y` produces a result file on `main` HEAD.
2. The same command with a deliberately broken commit (e.g., a forced `Hold` everywhere) produces a result file with measurably worse score.
3. `compare` between the two prints a table with finite, sane numbers and a decision-flip list.

This makes the harness self-validating before any optimization work begins.
