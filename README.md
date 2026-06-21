# Orchestrator-First

A methodology for Claude Code sessions that replaces decision-tree sub-agents with deterministic, tested, JSON-contract scripts — keeping raw tool output out of context and reserving the model for genuine judgment.

Built and proven on a real production codebase ([PuranGPT](https://purangpt.com)).

---

## The core idea

Every Claude Code session pays a context tax. The two biggest drains are:

1. **Reading raw tool output** — grep results, log dumps, file trees piped straight into context
2. **Spawning sub-agents to eyeball output and pick a branch** — when the decision was always a fixed parse/filter/reshape

Most "go look at the output and decide" tasks are pure decision trees. They don't need judgment — they need a script.

**Rule 0 — Orchestrator-First:** before spawning a sub-agent or ingesting raw tool output, ask: *is this a fixed decision tree?* If yes → call a deterministic script under `tools/`, consume only its `data` field. Sub-agents are last resort, for genuine judgment over unstructured content only.

---

## The two rules

### Rule 0 — Orchestrator-First

**The decision test** (run before every delegation):

> Is this a fixed decision tree / a parse-filter-reshape over predictable tool output?

- **YES** → call an existing `tools/` script (or build one), consume only `data`
- **NO — needs human-like judgment over novel/unstructured content** → only then is a sub-agent or direct reasoning warranted

**The escalation ladder** (climb only when the rung below can't do it):

1. Existing script in `tools/` — call it; extend it if a small change closes the gap
2. New script — it's a decision tree but no tool exists; build it (see template), then call it
3. Sub-agent — last resort, judgment only

**Session-open pre-flight:** run `doc_path_audit` before any action. Catches stale doc claims (files cited as existing that don't) before they corrupt the session's map.

**Precondition A — Tests-first with real-output fixtures:**
An untested replacement script you blindly trust is worse than a sub-agent you'd sanity-check. For every script: write tests first, run them (they should fail), implement the minimum to pass, verify until green.

**Precondition B — The JSON envelope:**
Every script returns `{success, data, metadata, errors}`. This makes the script a drop-in sub-agent replacement — same call site, same contract, no context spent on raw output.

```json
{ "success": true, "data": { ... }, "metadata": { ... }, "errors": [] }
```

On failure: `success: false`, `data: null`, `errors: [{code, message}]`. Never raise for an expected failure.

**The failure path:** when `success: false`, read `errors[]`, fix and rerun or fall back to raw output once. Never proceed as if the envelope were true.

**The scope trap:** a correct envelope around the wrong measurement is still wrong. Every tool's README carries a `does_not_measure` section. Validate scope before trusting the verdict.

### Rule 1 — Proactive API Automation

When the user provides an API key or secret, don't ask them to perform manual dashboard steps. Use the key to automate setup via REST APIs or CLIs. Always pick the most automated path.

---

## The three patterns

### Pattern 1 — Pre-flight Orientation

**Problem:** The agent opens a session with a stale map (doc claims a file that no longer exists, cites a wrong module, points at a deleted path). Acts confidently on the wrong map. No tripwire fires because none was set yet.

**Mechanism:** Run `doc_path_audit` at session start. It extracts backtick-quoted path claims from `.md` files and checks each against disk. Returns `{missing, present}`. Any `.py/.ts/.md` entry in `missing` is a stale claim — verify before acting.

**Cost:** sub-second. **Payoff:** prevents acting on a stale map before any other pattern applies.

→ [`tools/python/doc_path_audit/`](tools/python/doc_path_audit/)

---

### Pattern 2 — Branch-the-Future

**Problem:** The agent hits an irreducibly empirical fork — K candidates where only running the real system reveals which is right. Standard methodology serializes: pick one, build it, discover it's wrong, unwind, repeat. Each cycle pays the full wrong-direction cost plus a human round-trip.

**Mechanism:** Write the verdict predicate first (tests-first, JSON envelope). Launch all K candidates in parallel git worktrees. Drive each to an observable verdict via a background loop. Ingest only K one-line verdict envelopes — not K output streams. Promote the winner; discard the losers.

**The two novel primitives:**
- **Verdict-predicate-before-candidates** — tests-first applied to adjudication, not code. The script declares the winner; the agent's context never sees the K competing streams.
- **Silently-discarded speculative-next-task** — when P(next user ask) ≈ 1.0 and the predicate already exists, run the next task in a background worktree while the user reads the current response. Hit: instant delivery. Miss: `git worktree remove`, never shown.

**Proven on real data:** the proof-of-concept in [`examples/`](examples/purangpt-session.md) modeled a real 4-commit, 3-hour serial revert loop and adjudicated it in milliseconds — picking the same winner the human eventually reverted to.

---

### Pattern 3 — Assumption Tripwires *(design, not yet implemented)*

**Problem:** The agent makes a wrong assumption, unwinds, records it in FINDINGS.md — then pays the same cost again in the next session because the lesson is prose a human reads, not an executable gate.

**Mechanism:** Every wrong-direction unwind auto-compiles into a `PreToolUse` hook that fires before the next action of the same class. The falsified belief becomes a hard pre-flight check, not a note.

**Status:** Claude Code's `PreToolUse` hooks exist today. The pilot would scope to one assumption class (file-path claims from docs). Not yet built — `doc_path_audit` covers the pre-flight half; the auto-compile-on-unwind half is the remaining piece.

---

## How to use this in your project

### Option A — Drop in AGENTS.md (recommended)

Copy [`AGENTS.md`](AGENTS.md) into your project:
```
your-project/.claude/rules/AGENTS.md
```

Claude Code auto-loads files under `.claude/rules/` at the same priority as `CLAUDE.md`. The rules apply to every session at or under your project root.

If your project path contains spaces, symlink instead:
```bash
mkdir -p .claude/rules
ln -s "$(pwd)/AGENTS.md" ".claude/rules/engineering.md"
```

### Option B — Add the pre-flight tool

Copy `doc_path_audit` into your repo and add one line to your `CLAUDE.md`:

```bash
cp -r tools/python/doc_path_audit your-project/tools/
cp -r tools/python/_template your-project/tools/   # for building more tools later
```

Add to your `CLAUDE.md`:
```
## Session-open pre-flight
Run `python -m tools.doc_path_audit.check --json` before any action.
Read data.missing — any .py/.ts/.md entry is a stale path claim.
```

### Option C — Build your own tools from the template

For every recurring decision-tree task in your project, build a tool:

```bash
cp -r tools/python/_template your-project/tools/your_tool_name
```

1. Write tests first in `test_check.py` — run them, watch them fail
2. Implement `run()` in `check.py` until tests pass
3. Update `README.md` with the descriptor and `does_not_measure` section
4. Add a row to your `tools/README.md` registry

---

## What's in this repo

```
AGENTS.md                     # The rules — drop into .claude/rules/
tools/
  python/
    _template/                # Copy this to build a new tool
    doc_path_audit/           # Pre-flight: surface stale path claims in docs
examples/
  purangpt-session.md         # Annotated real session showing each pattern
```

---

## Why this works

The methodology was developed on [PuranGPT](https://github.com/prashantpandey-creator/purangpt). The problems it solves are real, documented in [`examples/purangpt-session.md`](examples/purangpt-session.md):

- A stale `engine/query_engine.py` reference in docs would have corrupted a session's map — `doc_path_audit` catches it before any action
- An SSE contract drift went undetected because the checker was scoped to one generator — the scope trap, documented in the example
- A 3-hour, 4-commit revert loop on a streaming renderer was adjudicated by a verdict script in milliseconds — Branch-the-future proven on real git history

The envelope shape (`{success, data, metadata, errors}`) is compatible with MCP servers, Anthropic/OpenAI tool calling, and LangGraph — retiring a sub-agent is a drop-in swap.

---

## License

MIT
