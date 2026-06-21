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

**Session-open pre-flight:** run `doc_path_audit` before any action. Catches stale doc claims before they corrupt the session's map.

**Precondition A — Tests-first with real-output fixtures:**
An untested replacement script you blindly trust is worse than a sub-agent you'd sanity-check. Write tests first, run them (they should fail), implement the minimum to pass, verify until green.

**Precondition B — The JSON envelope:**
Every script returns `{success, data, metadata, errors}`. This makes it a drop-in sub-agent replacement — same call site, same contract, no context spent on raw output.

```json
{ "success": true, "data": { ... }, "metadata": { ... }, "errors": [] }
```

On failure: `success: false`, `data: null`, `errors: [{code, message}]`. Never raise for an expected failure.

**The scope trap:** a correct envelope around the wrong measurement is still wrong. Every tool's README carries a `does_not_measure` section.

### Rule 1 — Proactive API Automation

When the user provides an API key or secret, don't ask them to perform manual dashboard steps. Use the key to automate setup via REST APIs or CLIs. Always pick the most automated path.

---

## The three patterns

### Pattern 1 — Pre-flight Orientation ✅ implemented + enforced

**Problem:** The agent opens a session with a stale map. Acts confidently on it. No tripwire fires because none was set yet.

**Mechanism:** `doc_path_audit` runs at session start via a `SessionStart` hook. Extracts backtick-quoted path claims from `.md` files, checks each against disk, returns `{missing, present}`. Result is injected as `additionalContext` — the agent sees it before its first action, without being asked.

**What "enforced" means:** the hook fires at the infrastructure level via `settings.local.json`. The agent cannot start a session without the audit result already in context. Behavioral rules (CLAUDE.md instructions to "remember to run X") don't enforce — hooks do.

**Proven:** on PuranGPT, the hook correctly surfaced two stale references on first run, and reports clean on every subsequent session after the docs were fixed.

→ [`tools/python/doc_path_audit/`](tools/python/doc_path_audit/) | [`hooks/session-start.sh`](hooks/session-start.sh)

---

### Pattern 2 — Branch-the-Future

**Problem:** The agent hits an irreducibly empirical fork — K candidates where only running the real system reveals which is right. Standard methodology serializes: pick one, build it, discover it's wrong, unwind, repeat.

**Mechanism:** Write the verdict predicate first (tests-first, JSON envelope). Launch all K candidates in parallel git worktrees. Drive each to an observable verdict. Ingest only K one-line verdict envelopes — not K output streams. Promote the winner; discard the losers.

**The two novel primitives:**
- **Verdict-predicate-before-candidates** — the script declares the winner; the agent's context never sees the K competing streams
- **Silently-discarded speculative-next-task** — when P(next user ask) ≈ 1.0, run the next task in a background worktree while the user reads the current response. Hit: instant delivery. Miss: `git worktree remove`, never shown.

**Proven:** modeled a real 4-commit, 3-hour serial revert loop on PuranGPT and adjudicated it in milliseconds — picking the same winner the human eventually reverted to. See [`examples/purangpt-session.md`](examples/purangpt-session.md).

---

### Pattern 3 — Assumption Tripwires *(design)*

**Problem:** The agent makes a wrong assumption, unwinds, records it in FINDINGS.md — then pays the same cost again next session because the lesson is prose, not an executable gate.

**Mechanism:** Every wrong-direction unwind auto-compiles into a `PreToolUse` hook. The falsified belief becomes a hard check, not a note.

**Status:** not yet built. Pattern 1's hook proves the infrastructure works. The auto-compile-on-unwind mechanism is the next piece.

---

## How to use this in your project

### Quickest path — one slash command

Copy the install skill to your global Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
curl -o ~/.claude/skills/orchestrator-first.md \
  https://raw.githubusercontent.com/prashantpandey-creator/orchestrator-first/main/skills/orchestrator-first.md
```

Then in any Claude Code session, in the project you want to install into:

```
/orchestrator-first
```

Claude will copy the tools, wire the hook, patch `settings.local.json`, and confirm everything works. One command, any project.

---

### Manual path — three steps

### Step 1 — Drop in the rules

```bash
mkdir -p your-project/.claude/rules
curl -o your-project/.claude/rules/AGENTS.md \
  https://raw.githubusercontent.com/prashantpandey-creator/orchestrator-first/main/AGENTS.md
```

Claude Code auto-loads `.claude/rules/` at CLAUDE.md priority for every session under your project root.

If your project path contains spaces, symlink:
```bash
ln -s "$(pwd)/AGENTS.md" ".claude/rules/engineering.md"
```

### Step 2 — Wire the pre-flight hook (enforcement layer)

This is what makes the methodology enforced rather than advisory.

```bash
# Copy the tool and hook into your project
mkdir -p your-project/tools your-project/.claude/hooks
cp -r tools/python/doc_path_audit your-project/tools/
cp hooks/session-start.sh your-project/.claude/hooks/
chmod +x your-project/.claude/hooks/session-start.sh
```

Edit `your-project/.claude/hooks/session-start.sh` — change the `REPO_ROOT` line to point to your repo:
```bash
REPO_ROOT="$(cd "$(dirname "$0")/../.." 2>/dev/null && pwd)"
```

Add to `your-project/.claude/settings.local.json`:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"your-project/.claude/hooks/session-start.sh\""
          }
        ]
      }
    ]
  }
}
```

Now every session opens with the audit result already in context — no rule to remember, no behavioral reliance.

### Step 3 — Build your own tools from the template

For every recurring decision-tree task in your project:

```bash
cp -r tools/python/_template your-project/tools/your_tool_name
```

1. Write tests first in `test_check.py` — run them, watch them fail
2. Implement `run()` in `check.py` until tests pass
3. Update `README.md` with the `does_not_measure` section
4. Add a row to your `tools/README.md` registry

---

## What's in this repo

```
AGENTS.md                     # The rules — drop into .claude/rules/
hooks/
  session-start.sh            # SessionStart hook — enforcement layer
skills/
  orchestrator-first.md       # Install skill — copy to ~/.claude/skills/, then /orchestrator-first
tools/
  python/
    _template/                # Copy this to build a new tool
    doc_path_audit/           # Pre-flight: surface stale path claims in docs
examples/
  purangpt-session.md         # Annotated real session showing each pattern
```

---

## Implementation status

| Pattern | Status | Enforced via |
|---------|--------|-------------|
| Pre-flight Orientation | ✅ built + running | `SessionStart` hook in `settings.local.json` |
| Branch-the-Future | ✅ proven on real data | proof-of-concept only, not wired as hook |
| Assumption Tripwires | design only | `PreToolUse` hooks (not yet built) |

The key distinction: **rules load, hooks enforce.** AGENTS.md shapes behavior. The `SessionStart` hook is what makes Pattern 1 actually run every session regardless of whether the agent "remembers" to.

---

## Why this works

The methodology was developed on [PuranGPT](https://github.com/prashantpandey-creator/purangpt). The problems it solves are real:

- A stale `engine/query_engine.py` reference in docs would have corrupted a session's map — the hook catches it before the first action
- An SSE contract drift went undetected because the checker measured the wrong scope — the scope trap, now documented in every tool's `does_not_measure` section
- A 3-hour, 4-commit revert loop was adjudicated by a verdict script in milliseconds — Branch-the-future proven on real git history

The envelope shape (`{success, data, metadata, errors}`) is compatible with MCP servers, Anthropic/OpenAI tool calling, and LangGraph — retiring a sub-agent is a drop-in swap.

---

## License

MIT
