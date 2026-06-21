---
name: orchestrator-first
description: Install the Orchestrator-First methodology into the current project. Sets up the pre-flight hook, doc_path_audit tool, and AGENTS.md rules. Use when the user says "install orchestrator-first", "set up the methodology", or "wire the pre-flight hook".
tools: Bash, Read, Write, Edit
---

# Orchestrator-First Installer

Install the Orchestrator-First methodology into the current project. This wires:
1. `AGENTS.md` rules into `.claude/rules/` (behavioral layer)
2. `doc_path_audit` tool into `tools/` (the pre-flight script)
3. `session-start.sh` hook into `.claude/hooks/` (enforcement layer)
4. `SessionStart` entry in `.claude/settings.local.json` (makes it fire)

## Workflow

### Step 1 — Detect current project

```bash
pwd
ls -la .claude/ 2>/dev/null || echo "no .claude dir yet"
ls -la tools/ 2>/dev/null || echo "no tools dir yet"
cat .claude/settings.local.json 2>/dev/null || echo "no settings.local.json yet"
```

Note the current working directory as `PROJECT_ROOT`. This is where everything gets installed.

### Step 2 — Locate the orchestrator-first repo

Check common locations:
```bash
ls ~/projects/orchestrator-first/ 2>/dev/null || \
ls ~/orchestrator-first/ 2>/dev/null || \
echo "not found locally"
```

If not found locally, clone it:
```bash
git clone https://github.com/prashantpandey-creator/orchestrator-first ~/orchestrator-first
```

Set `REPO` to the path where the repo exists.

### Step 3 — Create directory structure

```bash
mkdir -p .claude/rules .claude/hooks tools/doc_path_audit tools/_template
```

### Step 4 — Copy AGENTS.md

```bash
cp "$REPO/AGENTS.md" .claude/rules/AGENTS.md
```

If the project path contains spaces, create a symlink instead:
```bash
ln -sf "$(pwd)/.claude/rules/AGENTS.md" .claude/rules/engineering.md
```

### Step 5 — Copy doc_path_audit tool

```bash
cp "$REPO/tools/python/doc_path_audit/check.py" tools/doc_path_audit/check.py
cp "$REPO/tools/python/doc_path_audit/test_check.py" tools/doc_path_audit/test_check.py
cp "$REPO/tools/python/doc_path_audit/README.md" tools/doc_path_audit/README.md
touch tools/doc_path_audit/__init__.py tools/__init__.py

# Also copy the template for building future tools
cp "$REPO/tools/python/_template/check.py" tools/_template/check.py
cp "$REPO/tools/python/_template/test_check.py" tools/_template/test_check.py
cp "$REPO/tools/python/_template/README.md" tools/_template/README.md
touch tools/_template/__init__.py
```

### Step 6 — Copy and configure the hook

```bash
cp "$REPO/hooks/session-start.sh" .claude/hooks/session-start.sh
chmod +x .claude/hooks/session-start.sh
```

Now patch the `REPO_ROOT` line in `.claude/hooks/session-start.sh` to point at the current project:

Read `.claude/hooks/session-start.sh`. Find the line:
```bash
REPO_ROOT="$(cd "$(dirname "$0")/../../purangpt" 2>/dev/null && pwd)"
```

Replace it with:
```bash
REPO_ROOT="$(cd "$(dirname "$0")/../.." 2>/dev/null && pwd)"
```

This makes `REPO_ROOT` resolve to the project root (two levels up from `.claude/hooks/`).

Also check what Python command is available:
```bash
which python3 || which python
```

If the project uses a venv, find it:
```bash
ls venv/bin/python 2>/dev/null || ls .venv/bin/python 2>/dev/null || echo "no venv"
```

Patch the run line in `session-start.sh` to use the right Python:
- If venv exists at `venv/`: keep `venv/bin/python -m tools.doc_path_audit.check --json`
- If venv at `.venv/`: change to `.venv/bin/python -m tools.doc_path_audit.check --json`
- If no venv: change to `python3 -m tools.doc_path_audit.check --json`

### Step 7 — Wire the hook into settings.local.json

Read the current `.claude/settings.local.json` (or start with `{}`).

Add the `hooks` block. The full PROJECT_ROOT path must be used in the command (no `~` or relative paths — Claude Code resolves hook commands from the binary's directory, not the project root).

Merge in:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"PROJECT_ROOT/.claude/hooks/session-start.sh\""
          }
        ]
      }
    ]
  }
}
```

Where `PROJECT_ROOT` is the absolute path from Step 1. Preserve any existing keys (permissions, etc).

Write the merged result back to `.claude/settings.local.json`.

### Step 8 — Test it

Run the hook directly to verify it produces valid JSON:
```bash
bash .claude/hooks/session-start.sh
```

Expected output shape:
```json
{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "..."}}
```

If it outputs an error or empty string, diagnose:
- Check Python path is correct
- Check `tools/doc_path_audit/` files are present
- Check `REPO_ROOT` resolves correctly: `cd .claude/hooks && cd ../.. && pwd`

### Step 9 — Report

Tell the user:
- What was installed and where
- What the hook reported on first run (clean or stale paths found)
- That the methodology is now **enforced** — every new session under this project will open with the pre-flight audit result already in context
- Next step: start a new Claude Code session in this project and confirm the `SessionStart hook additional context` appears at the top of the system context

## Important notes

- Do NOT modify `~/.claude/settings.json` (global) — only the project-level `settings.local.json`
- If `settings.local.json` already has a `hooks.SessionStart` entry, merge rather than overwrite
- The hook silently exits if `tools/doc_path_audit/` is missing — it will never break a session
- Cross-repo path references and historical prose mentions are expected noise in the audit output — only `.py/.ts/.md` entries that are genuine stale claims need attention
