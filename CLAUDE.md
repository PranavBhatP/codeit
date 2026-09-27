# CLAUDE.md

Project guidance for Claude Code working in this repository.

@AGENTS.md

---

## Claude Code specifics

The shared guidance in AGENTS.md above is the substance — constraints, conventions, and the
list of things that will bite you. It applies in full. A few additions for this harness:

**Read [README.md](README.md) before non-trivial work.** It is a phased plan with per-phase
exit criteria, and the phase sequencing is a deliberate pedagogical constraint rather than a
rough sketch. Anchor changes to a phase.

**Plan before implementing anything that spans modules.** The loop, context manager, and
action schema are tightly coupled by design; a change to one usually implies the others.
Surface that coupling in a plan before writing code.

**Don't use subagents or workflows to write harness code.** Parallel agents produce more code
than the author can read, which is the failure mode this project is specifically trying to
avoid. Subagents are fine for read-only exploration — reading a reference implementation like
Aider or OpenHands, or surveying how something is done across files.

**Prefer verification over assertion.** Claims about local model behaviour, token counts, VRAM
headroom, or benchmark results should come from an actual run with a trace file, not from
inference. "This should fit in 6GB" is a hypothesis; `nvidia-smi` during a run is a result. If
no model server is available to check, say the claim is unverified.

**Reference implementations worth consulting** when a design decision is genuinely open:
Aider (edit formats, repo map), OpenHands (event stream, sandboxing). The README's Phase 5
calls for comparing codeit's choices against them, so noting a divergence and its reason as
you go is directly useful later.
