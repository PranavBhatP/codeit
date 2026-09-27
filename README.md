# codeit — Project Plan

**A self-built agent harness for local-LLM-powered code editing**

---

## Overview

**codeit** is a from-scratch agentic coding assistant harness (no LangChain/AutoGPT-style
frameworks) that lets a locally-hosted LLM autonomously read, search, edit, and verify changes
to a codebase through a bounded action loop. The project measures how reliably small local
models complete bounded code-editing tasks, and how much of that reliability comes from the
*harness* rather than the model.

**Tech stack:** Python 3.12 (core harness), vLLM + Ollama (model serving), Pydantic (action
schema + grammar-constrained decoding), git (edit application/rollback), `tree-sitter`
(code-aware search), optional Go layer (scheduling proxy).

**Timeline:** 8 weeks.

### Hardware budget (drives most design decisions)

Target machine: **RTX 3050 Laptop, 6GB VRAM**. This is the central constraint of the project.

| | weights | usable ctx | notes |
|---|---|---|---|
| Qwen2.5-Coder-7B-Instruct, Q4_K_M (Ollama) | ~4.7GB | ~8–16k | partial CPU offload; slow but works |
| Qwen2.5-Coder-7B-Instruct-AWQ (vLLM) | ~5.2GB | very tight | needs low `--max-model-len`, may not fit |
| Qwen2.5-Coder-3B-Instruct-AWQ (vLLM) | ~2.2GB | 32k+ | the fast path; primary vLLM target |

Rough KV-cache planning math (fp16, GQA): 7B ≈ **57KB/token**, 3B ≈ **36KB/token**. So 16k of
7B context is ~0.9GB — which, on top of 4.7GB of weights, is the whole card.

**Consequence:** the agent cannot hold a repo in context. Context budgeting, retrieval, and
observation truncation are load-bearing components, not Phase 3 polish. Treat the context
window as a managed resource with an explicit accounting layer.

---

## Phase 0 — Scaffold (Days 1–2)

**Goal:** Nothing interesting, done once, correctly.

- `git init`; Python **3.12** venv (vLLM wheels lag new CPython releases — 3.14 will not work).
- `pyproject.toml`, ruff + pytest, `codeit` console entry point.
- **Trace logging first.** A `Trace` object writing newline-delimited JSON per loop step:
  step index, prompt token count, raw completion, parsed action, tool result, wall time,
  tokens/sec. Everything downstream (debugging, eval, writeup) reads these files.

**Exit criteria:** `codeit --version` runs; a stub loop writes a well-formed trace file.

---

## Phase 1 — Model Serving & Action Schema (Weeks 1–2)

**Goal:** A local model that reliably emits schema-valid actions.

- Stand up vLLM and Ollama behind one interface:
  `complete(messages, schema=None) -> Completion` (text + token counts + timing).
  Both speak OpenAI-compatible HTTP; prefer that over Python SDKs to keep the seam thin.
- Define the action schema as Pydantic models: `read_file`, `search`, `edit_file`,
  `run_command`, `done`. One action per turn — easier to trace, and small models handle it
  far better than multi-action batches.
- **Constrain generation rather than parsing it after the fact:** vLLM guided JSON
  (xgrammar backend) and Ollama's schema `format`. Malformed output stops being the common
  case. Keep a tolerant text parser as a documented fallback path — and measure how often
  each path is taken, since Ollama/vLLM constraint support differs and that becomes an
  eval dimension.
- Prompt assembly lives in `prompts/` as versioned templates, not f-strings inline in the loop.

**Exit criteria:** For each of the five action types, the model emits a schema-valid call
that round-trips through the parser, across both backends, logged to a trace.

---

## Phase 2 — File Tools, Edit Application & Minimal Eval (Weeks 3–4)

**Goal:** Actions that do real work, safely — and a way to score them.

- `read_file` (with line-range windowing and a hard output cap) and `search` (grep-based).
- `edit_file` as a **pluggable edit strategy**, applied inside a git-tracked working copy:
  - `search_replace` — exact-match old/new blocks. **Default.**
  - `udiff` — unified diff via git apply.
  - `whole_file` — full rewrite. The dumb baseline; often wins on small models.
  Line numbers and hunk headers are a precision tax a 7B model pays badly, so do not assume
  udiff is the right default. Which format survives contact with a local model is an
  *empirical question this project answers* (Phase 5).
- `run_command` with an explicit, honest threat model: **accident containment, not adversarial
  isolation.** argv lists (no shell), hard timeout, cwd pinned to the working copy, output
  truncated, environment scrubbed, command deny-list. Document what this does *not* stop.
- Rollback via `git stash` / `git checkout` — every task starts from a clean tree and any
  failed edit or failed verification reverts.
- **Minimal eval runner (3 tasks).** Needed now so Phase 3's retry loop has a baseline to
  beat. Each task = git fixture + instruction + verification command.

**Exit criteria:** The agent applies an edit to a real repo with `git diff` showing exactly
the intended change; a failed edit leaves the tree clean; `codeit eval` scores 3 tasks.

---

## Phase 3 — Verification Loop & Retrieval (Weeks 5–6)

**Goal:** Close the loop — the agent checks its own work and retries on failure.

- Run tests/linter after each edit; feed truncated pass/fail output back as an observation.
- Bounded retry with an explicit budget: max iterations, max tokens, max wall time.
  Budget state is visible in the prompt so the model knows it is running out of room.
- **Context manager** (the real work of this phase, given the 6GB budget): token accounting
  per message, observation truncation with head/tail retention, eviction of stale tool
  results, and a running summary of what has already been tried. Measure how often tasks
  fail from context exhaustion versus reasoning failure — this distinction is the most
  interesting number in the project.
- Code-aware retrieval: `tree-sitter` symbol/definition search replacing plain grep.
  Embeddings only if time allows — symbol search is likely to beat them on this task shape.

**Exit criteria:** The agent fails a test, reads the failure, revises its edit, and passes on
a later attempt, with no manual intervention, and the trace shows each step.

---

## Phase 4 — Robustness Pass (Week 7)

**Goal:** Handle the failure modes that actually show up.

- Pre-apply edit validation: reject non-matching search blocks, empty edits, paths outside
  the working copy, and (for Python targets) edits that fail to parse.
- Loop guards: no-progress detection (repeated identical actions), oscillation detection,
  hard caps on every budget.
- Edge cases: multi-file changes, partial application, ambiguous instructions, model asking
  to read a file that does not exist.
- **Failure taxonomy.** Tag every failed trace with a category (bad edit format, context
  exhaustion, wrong file, hallucinated API, gave up early, loop). This table is the
  centerpiece of the writeup — it is the part that shows systems judgment rather than
  prompt luck.

**Exit criteria:** Every catalogued bad-input case produces a clear error and a clean tree.
No hangs, no corrupted state, no partial edits left behind.

---

## Phase 5 — Evaluation & Writeup (Week 8)

**Goal:** A defensible benchmark, not a demo.

- **12–15 self-authored tasks** with deterministic test oracles (type-hint additions, bug
  fixes with a failing test, helper extraction, small refactors, docstring/API changes).
  Self-authored over SWE-bench-lite deliberately: SWE-bench resolve rates for quantized
  7B-class models are near the noise floor, so the results would say nothing about the
  harness. Note the tradeoff — self-authored tasks are not comparable to published numbers.
- Two eval sweeps:
  - **Model/backend:** 3B-AWQ/vLLM vs. 7B-Q4/Ollama vs. (if it fits) 7B-AWQ/vLLM.
    Quantization confounds model-size comparisons — control for it or state it.
  - **Harness ablation** (hold the model fixed, vary the harness): edit format ×
    {search_replace, udiff, whole_file}, retrieval × {grep, tree-sitter}, retry budget ×
    {1, 3, 6}, constrained decoding × {on, off}. Cheaper than the model sweep and more
    informative — it isolates the contribution of *your* work.
- Report: success rate (pass@1), iterations-to-success, tokens per task, latency and
  tokens/sec per backend, failure taxonomy breakdown.
- Writeup: architecture, results tables, and design decisions compared against Aider
  (edit formats, repo map) and OpenHands (sandboxing, event stream) — including where
  codeit deliberately diverges and why.

**Exit criteria:** `codeit eval --sweep` reproduces every table in the README from trace
files on disk.

---

## Deliverables Checklist

- [ ] CLI (`codeit edit "<instruction>"`, `codeit eval`)
- [ ] Action schema + constrained decoding + fallback parser
- [ ] Tools: read / search / edit / run / done
- [ ] Pluggable edit strategies (search_replace / udiff / whole_file)
- [ ] Context manager with token accounting
- [ ] Verification + bounded retry loop
- [ ] Trace logging (JSONL) for every run
- [ ] Benchmark suite + eval runner + sweep support
- [ ] Failure taxonomy table
- [ ] README with architecture + results + design-decision writeup
- [ ] (Stretch) Go scheduling proxy in front of vLLM

### On the stretch goal

The original plan proposed a Go/gRPC layer for parallel multi-agent task splitting. On a
single 6GB GPU, parallel agents contend for the same weights and KV cache, so there is no
speedup to demonstrate — the result would be a negative one for uninteresting reasons.

A better target for the same amount of Go, on a bottleneck this project will have already
measured: a **scheduling proxy in front of vLLM** — prefix-cache-aware routing (agent loops
share long system-prompt prefixes, so routing by prefix is a real win), admission control
under VRAM pressure, request queueing with per-task budgets, and metrics. Same systems
depth, aimed at a bottleneck the traces prove exists.

---

## Repo Structure

```
codeit/
├── harness/          # loop, budgets, context manager, trace logging
├── llm/              # vLLM + Ollama clients behind one interface
├── actions/          # Pydantic action schema + parser/fallback
├── tools/            # read / search / edit / run implementations
│   └── edits/        # pluggable edit strategies
├── prompts/          # versioned prompt templates
├── eval/
│   ├── tasks/        # benchmark fixtures (git repos + oracles)
│   ├── runner.py
│   └── report.py
└── cli.py
tests/                # harness unit tests (not the benchmark)
traces/               # JSONL run logs (gitignored)
```
