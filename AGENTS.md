# AGENTS.md — working in the codeit repo

Guidance for AI coding agents working on this repository. Human contributors should read
[README.md](README.md) first; this file assumes it.

> **Note:** this file is instructions for agents *editing codeit*. It is not a prompt for
> codeit itself — codeit's own system prompts live in `prompts/`. Do not conflate them, and
> do not "improve" `prompts/` templates as a side effect of an unrelated task.

---

## What this project is

A from-scratch agentic coding harness for locally-hosted LLMs. **The point is pedagogical:**
the author is building it to understand how agent harnesses work end to end. This changes what
"good" means here in one important way — see Constraints below.

## Constraints

**1. No agent frameworks.** No LangChain, LlamaIndex, AutoGen, CrewAI, smolagents, or
`openai-agents`. The loop, the tool dispatch, the retries, and the context management are the
deliverable. Importing a framework that provides them defeats the entire project.

Allowed and encouraged: `pydantic`, `httpx`, `typer`/`argparse`, `rich`, `tree-sitter`,
`pytest`, `unidiff`, plain `subprocess`. OpenAI-compatible HTTP calls to vLLM/Ollama are
fine — that is a wire protocol, not a framework. **If a dependency would implement a concept
named in the README's phase list, do not add it; ask.**

**2. Do not abstract ahead of the phases.** The README is sequenced deliberately. Building
Phase 4 machinery during Phase 2 produces code the author has not yet earned the context to
understand. Prefer the simplest thing that satisfies the current phase's exit criteria.

**3. 6GB VRAM is a hard budget.** Anything that grows the context window per loop iteration
is a correctness issue, not a performance one. Read the hardware budget table in the README
before touching the context manager, prompt templates, or tool output formatting.

**4. Explain, don't just implement.** When adding a non-obvious mechanism, leave a short
comment on *why* this approach over the alternative. The author is reading this code to learn
from it. Comments that restate the code are noise; comments that capture a design tradeoff
are the value.

---

## Environment

- **Python 3.12.** Not 3.14 (the system default here) — vLLM wheels lag new CPython releases.
  Work inside the project venv; if it does not exist, say so rather than falling back to
  system Python.
- Target GPU: RTX 3050 Laptop, 6GB.
- vLLM and Ollama are **not assumed to be running.** Never make a test or a default code path
  depend on a live model server. Model-backed tests go behind a marker (`@pytest.mark.live`)
  and are excluded by default.

## Commands

The scaffold may not exist yet; check before assuming. Once Phase 0 lands:

```bash
pytest -m "not live"          # unit tests — the default; must pass without a model server
pytest -m live                # requires vLLM or Ollama running
ruff check . && ruff format . # lint + format
codeit edit "<instruction>"   # single-task run
codeit eval                   # benchmark suite
```

---

## Conventions

**Layout** follows the README's repo structure. Put new code in the module that owns the
concept; if nothing owns it, that is worth a conversation, not a new top-level package.

**Typing.** Type-annotate public functions. Pydantic models for anything crossing the
model boundary (actions, tool results, trace records) — they are the schema *and* the
grammar source for constrained decoding, so they are not merely documentation.

**Errors.** Tool failures are *data*, not exceptions: a tool returns a result object with
`ok: bool` and an error message the model can read and act on. Reserve exceptions for harness
bugs. A traceback escaping the loop is a bug in the loop.

**Every loop step gets traced.** If you add a step, a retry, or a branch, it must emit a trace
record. Untraced control flow is invisible to the eval and to the writeup, which makes it
worse than no feature at all.

**Prompts** are versioned template files in `prompts/`, loaded by name. No prompt strings
inline in harness code — the eval needs to attribute results to a prompt version.

---

## Things that will bite you

- **`eval/tasks/` contains git repo fixtures.** They have intentionally broken code and
  failing tests — that is the point. Never "fix" a failing test under `eval/tasks/`. Never
  run `git` commands from inside a fixture directory expecting them to hit the outer repo.
- **`traces/` is gitignored and can get large.** Do not commit traces. Do not delete the
  directory; other runs may be reading from it.
- **Edit strategies are an eval axis, not dead code.** `udiff` and `whole_file` may look
  redundant next to `search_replace`. They are the independent variable in the Phase 5
  ablation. Do not consolidate them.
- **`run_command` is accident containment, not a security sandbox.** It is not
  adversary-resistant and the README says so. Do not add code or docs claiming it is
  sandboxed, and do not weaken its existing guards (no shell, argv lists, timeout, cwd
  pinning, output caps, env scrubbing).
- **Benchmark numbers must come from trace files on disk**, never from a hand-assembled
  table. If you change the report generator, regenerate rather than editing results.

## Scope

Small, reviewable changes tied to a phase. This is a learning project on a two-month
timeline — a large unsolicited refactor costs the author more in reading time than it saves.
If a task reveals work belonging to a later phase, note it and stop; don't pull it forward.
