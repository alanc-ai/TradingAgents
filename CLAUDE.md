# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Install (editable, dev):
```bash
pip install -e .
```

Run the interactive CLI:
```bash
tradingagents          # console script (cli.main:app)
python -m cli.main     # equivalent
python main.py         # minimal scripted entry: builds TradingAgentsGraph and calls .propagate()
```

Tests (pytest, configured in `pyproject.toml`, `testpaths=["tests"]`):
```bash
pytest                                          # full suite
pytest tests/test_structured_agents.py          # one file
pytest tests/test_signal_processing.py::test_x  # one test
pytest -m unit                                  # by marker (unit/integration/smoke)
```
`tests/conftest.py` auto-injects placeholder values for every provider API key and exposes a `mock_llm_client` fixture that patches `tradingagents.llm_clients.factory.create_llm_client`. Tests should not require live network/keys unless explicitly marked `integration`.

Docker:
```bash
docker compose run --rm tradingagents
docker compose --profile ollama run --rm tradingagents-ollama
```

## Architecture

This is a LangGraph-based multi-agent pipeline. The orchestrator is `tradingagents.graph.trading_graph.TradingAgentsGraph`; `propagate(ticker, date)` returns `(final_state, processed_signal)`.

**LLM provider abstraction** (`tradingagents/llm_clients/`)
`create_llm_client(provider, model, base_url, **kwargs)` is the single entry point. OpenAI, xAI, DeepSeek, Qwen, GLM, Ollama, and OpenRouter all go through `OpenAIClient` (OpenAI-compatible chat completions); Anthropic, Google, and Azure each have dedicated clients. Provider-specific reasoning controls (`google_thinking_level`, `openai_reasoning_effort`, `anthropic_effort`) flow from config → `_get_provider_kwargs()` → client kwargs. **Do not** put provider-specific defaults in `backend_url` in `default_config.py` — leaving it `None` lets each client use its own base URL (a previous bug forwarded OpenAI's `/v1` to Gemini).

**Agent graph** (`tradingagents/agents/` + `tradingagents/graph/setup.py`)
Pipeline stages, wired in `GraphSetup.setup_graph()`:
1. Analyst team (`analysts/`): market, social, news, fundamentals — each is a tool-using agent backed by a `ToolNode` (see `_create_tool_nodes` in `trading_graph.py`).
2. Researcher debate (`researchers/` bull vs bear) — `max_debate_rounds` rounds.
3. Research Manager (`managers/research_manager.py`) — produces an investment plan.
4. Trader (`trader/trader.py`) — produces a trade proposal.
5. Risk debate (`risk_mgmt/` aggressive/conservative/neutral) — `max_risk_discuss_rounds` rounds.
6. Portfolio Manager (`managers/portfolio_manager.py`) — final approve/reject decision.

Research Manager, Trader, and Portfolio Manager use **structured output** via `agents/utils/structured.py`. The pattern: `bind_structured()` wraps the LLM with `with_structured_output(Schema)` (schemas in `agents/schemas.py`); `invoke_structured_or_freetext()` runs the structured call and falls back to plain `llm.invoke` on failure (weak models / malformed JSON / transient errors). Keep all three agents on this shared helper so fallback logging stays consistent.

**Data layer** (`tradingagents/dataflows/`)
Two vendors are pluggable per category via config: `data_vendors` (category default) and `tool_vendors` (per-tool override). Vendors are `yfinance` and `alpha_vantage`. Tool functions in `agents/utils/agent_utils.py` (`get_stock_data`, `get_indicators`, `get_fundamentals`, `get_news`, etc.) dispatch to the chosen vendor.

**State and persistence**
- `agents/utils/agent_states.py` defines `AgentState`, `InvestDebateState`, `RiskDebateState` — the LangGraph state schema.
- **Decision log** (always on): `agents/utils/memory.TradingMemoryLog` appends each run's decision to `~/.tradingagents/memory/trading_memory.md`. On the next same-ticker run, `_resolve_pending_entries` fetches realised return + alpha vs SPY via yfinance, generates a one-paragraph reflection, and injects past context into the PM prompt. Override path with `TRADINGAGENTS_MEMORY_LOG_PATH`.
- **Checkpoint resume** (opt-in via `config["checkpoint_enabled"]` or `--checkpoint`): `graph/checkpointer.py` uses LangGraph `SqliteSaver` per ticker at `~/.tradingagents/cache/checkpoints/<TICKER>.db`. Thread ID is `(ticker, date)`, so re-running the same ticker+date resumes; a different date starts fresh. Checkpoint is cleared on successful completion. Override base with `TRADINGAGENTS_CACHE_DIR`.

**Ticker safety**: any code that builds a filesystem path from a user-supplied ticker must go through `dataflows.utils.safe_ticker_component` (validated path-component check; see `tests/test_safe_ticker_component.py` and commit 2c97bad).

## Configuration

`tradingagents/default_config.py` is the source of truth for runtime knobs. All paths default under `~/.tradingagents/` and are overridable by env vars (`TRADINGAGENTS_RESULTS_DIR`, `TRADINGAGENTS_CACHE_DIR`, `TRADINGAGENTS_MEMORY_LOG_PATH`). Callers typically `DEFAULT_CONFIG.copy()` then override `llm_provider`, `deep_think_llm`, `quick_think_llm`, `max_debate_rounds`, etc. before constructing `TradingAgentsGraph`.

The CLI (`cli/main.py`) sets `backend_url` per provider when the user picks one; do not bake provider-specific URLs into `DEFAULT_CONFIG`.
