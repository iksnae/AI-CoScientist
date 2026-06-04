# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AI-CoScientist is a single-package Python library implementing the "Towards an AI Co-Scientist" methodology: a multi-agent pipeline that generates, peer-reviews, ranks (via Elo tournaments), and iteratively evolves scientific research hypotheses. It is published to PyPI as `ai-coscientist`.

## Architecture

Nearly all logic lives in **`ai_coscientist/main.py`** (~2000 lines). `ai_coscientist/__init__.py` only re-exports the public API. There is no separate module per agent — the entire framework is one file.

- **`AIScientistFramework`** is the single orchestrator class. Its `run_research_workflow(research_goal)` is the only public entry point and returns a `WorkflowResult` dict (or an `{"error": ...}` dict on failure — the workflow catches all exceptions rather than raising).
- The framework is built on the external **`swarms`** library: it instantiates 8 `swarms.Agent` instances (generation, reflection, ranking, evolution, meta-review, proximity, tournament, supervisor), each with `max_loops=1` and a hard-coded system prompt from a `_get_*_agent_prompt()` method. Conversation history is tracked via `swarms.Conversation`. Note: the `supervisor_agent` is initialized but not yet wired into the workflow loop.
- Agents communicate by returning **JSON in their text output**, which is extracted by `_safely_parse_json()` (handles markdown code fences and malformed output with fallbacks). When editing agent prompts, the expected JSON shape and the parsing/consumption code must stay in sync — they are coupled by convention, not by schema enforcement.
- Typed structures (`Hypothesis` dataclass, plus many `TypedDict`s like `ReviewScores`, `HypothesisReview`, `WorkflowResult`) define the data contracts. `Hypothesis` carries Elo rating, review history, and similarity-cluster membership; `Hypothesis.update_elo()` implements the pairwise Elo update.

### Workflow phases (in `run_research_workflow`)

Initial: generation → reflection → ranking → tournament. Then for `max_iterations`: meta-review → evolution (top-k) → reflection → ranking → tournament → proximity analysis. Each phase is a `_run_*_phase()` method wrapped by `_time_execution()` for per-agent timing in `execution_metrics`.

## Configuration

Construct `AIScientistFramework(model_name=..., max_iterations=..., tournament_size=..., hypotheses_per_generation=..., evolution_top_k=..., base_path=..., verbose=...)`. `model_name` is passed straight to `swarms.Agent` (LiteLLM-style strings, e.g. `gpt-4o-mini`, `gemini/gemini-2.0-flash`, `claude-...`). API keys come from a `.env` file (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`).

Agent state is persisted under `base_path` (default `./ai_coscientist_states/`) via `save_state()` / `load_state()`.

## Commands

```bash
pip install -e .          # install from source (or: poetry install)
python example.py         # run an end-to-end research workflow example
black .                   # format (line-length 70)
ruff check .              # lint (line-length 70)
mypy ai_coscientist       # type-check
```

There are no tests in this repo despite the CI workflow scaffolding under `.github/workflows/` (those are generic templates; `quality.yml` runs `pylint` on changed files against `origin/main`). Formatting is intentionally narrow at **line-length 70** (both black and ruff) — match it.
