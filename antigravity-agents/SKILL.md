---
name: antigravity-agents
description: Delegate coding, code review, analysis, and research jobs to Google Antigravity CLI (agy) sub-agents that run alongside your own work. Use this whenever the user asks to spin off, offload, or delegate a task to Antigravity, agy, Gemini, or an external sub-agent; wants a second opinion or independent review from a different model; wants intensive repo work (audits, large refactors, research sweeps, frontend boilerplate) run in the background while you keep working; or says things like "have Antigravity do it", "spin up a sub-agent for this", or "get more done in parallel". Also use it proactively when a task is a good fit for parallel delegation and the user has expressed a preference for using Antigravity workers.
---

# Antigravity CLI Sub-Agents

Google Antigravity CLI (`agy`) is an autonomous terminal coding agent powered by Gemini and other models that can inspect repositories, generate code, edit files, and run commands. This skill turns Claude into an orchestrator with full control over the Antigravity sub-agent engine: model selection, reasoning effort, session memory, execution modes, and directory scopes.

Treat an agy job like a specialized contractor: you define the exact technical contract, tune the thinking effort and model tier to match the task's complexity, launch the job non-interactively, continue your primary work, then inspect and verify the returned code before integrating it.

## Resolving the Binary

After install, `agy` is normally on your `PATH`. If a tool-spawned shell cannot find it (some shells do not inherit a freshly updated PATH), fall back to the full path:

- **Windows:** `%LOCALAPPDATA%\agy\bin\agy.exe` (in Bash-style shells: `"$LOCALAPPDATA/agy/bin/agy.exe"`)
- **macOS / Linux:** `~/.local/bin/agy` or `which agy`

**Critical: Always close stdin.** `agy` blocks indefinitely (0% CPU, no output, even in print mode) when stdin is an open pipe, which is what non-interactive tool shells provide. Always launch jobs with `</dev/null` appended, or use the bundled `bin/agy-delegate` runner script which handles this automatically.

## Preflight Verification (Once per Session)

Before the first job of a session, verify authentication with a quick probe:

```bash
agy -p "Reply with exactly: OK" --print-timeout 60s </dev/null
```

- Replies `OK`: Authenticated, proceed.
- Prints a sign-in URL or errors about credentials: Stop and prompt the user to run `agy` once in their own interactive terminal to complete the one-time Google OAuth browser flow.
- Hangs with no output: Stdin was left open; kill the process and ensure `</dev/null` is used.

## The Orchestrator Control Panel

When delegating a task to Antigravity, do not simply send an uncalibrated prompt. Actively calibrate the following parameters based on the task scope:

### 1. Reasoning Effort (`--effort`)

Antigravity supports fine-grained reasoning effort calibration. Choose the level according to the task type:

| Effort Level | Intended Tasks | Characteristics |
| :--- | :--- | :--- |
| `low` | Mechanical tasks, static HTML/CSS, Tailwind components, boilerplate, lint fixes | Lowest latency, immediate output, minimal thinking overhead |
| `medium` | Standard component logic, isolated functions, unit test suites, documentation | Balanced reasoning and speed (default) |
| `high` | Multi-component coordination, API integrations, data transformations | Deep analysis, edge case exploration |
| `max` | Architectural audits, security reviews, complex debugging, protocol logic | Exhaustive multi-step reasoning before code output |

### 2. Model Selection (`--model`)

Use the model slug that fits the task value and risk profile:

- `gemini-3.8-flash-high` / `gemini-3.8-flash-medium` (Recommended default): Blazing fast, highly capable for frontend implementation, boilerplate, and routine refactors.
- `gemini-4-argon-high` / `gemini-4-argon-medium`: Heavier reasoning, deep code review, and architectural analysis.
- `claude-opus-5-5-high` / `claude-sonnet-5-5-high`: Cross-model verification and second opinions.
- Run `agy models` at any time to list all currently active models.

### 3. Session and Memory Control (`--conversation`)

- **Isolated One-Shot Jobs:** Omit the flag. The job executes in a fresh context window and exits cleanly without lingering state.
- **Deterministic Multi-Turn Follow-Ups:** Pass `--conversation <id>` to resume an exact prior session. This allows you to review a sub-agent's work and send concrete revisions ("fix the type error in line 42") without re-briefing from scratch.
- Avoid naked `--continue` in parallel workflows, as "most recent" becomes ambiguous when multiple jobs run concurrently.

### 4. Execution Mode (`--mode`)

- `--mode plan`: Read-only architectural planning. The sub-agent analyzes the code and produces a structured plan without modifying files.
- `--mode accept-edits`: Direct execution. The sub-agent applies changes directly to the target files.

### 5. Directory Scoping (`--add-dir`)

Antigravity operates in the current working directory by default. When a sub-agent needs access to supplementary directories without exposing the entire disk, pass `--add-dir <path>` for each required path.

## Safety and Conflict Rules

An Antigravity job and your own concurrent edits must never touch the same working tree files at the same time:

| Job Type | Examples | Isolation Method |
| :--- | :--- | :--- |
| **Read-Only** | Code review, security audit, research, dependency scan | Run concurrently in the repository. Add `--sandbox` and instruct: "Do not modify any files." |
| **Write (Different Repo)** | Fix a bug in repo B while working in repo A | Safe to run concurrently from repo B's root directory. |
| **Write (Same Repo)** | Feature implementation, component generation, refactoring | Either (a) create a git worktree on an isolated branch (`git worktree add ../feature-worktree -b feature-branch`), or (b) run sequentially while you pause file edits, then review the diff. |

For write jobs, always check status first (`git status --porcelain`) and ensure uncommitted changes are stashed or committed before delegating.

## Launching Jobs

### Method A: Using the Bundled Runner Script (Recommended)

The repository includes a standalone Python runner (`bin/agy-delegate`) that automates stdin closure, flag validation, and execution logging:

```bash
# Example: Fast frontend component generation with low reasoning effort
./bin/agy-delegate \
  --model gemini-3.8-flash-high \
  --effort low \
  --mode plan \
  --timeout 5m \
  "Write a responsive React Tailwind Card component for product listings. Output pure TSX code."

# Example: In-depth security audit with max reasoning effort
./bin/agy-delegate \
  --model gemini-4-argon-high \
  --effort max \
  --sandbox \
  --timeout 15m \
  --log-file /tmp/security-audit.log \
  "Perform a security review of authentication handlers in src/auth/. Report findings with severity and line numbers."
```

### Method B: Direct CLI Execution

```bash
cd <repo-root> && agy -p "$(cat <<'EOF'
<self-contained task prompt>
EOF
)" --model gemini-3.8-flash-high --effort medium --print-timeout 10m --dangerously-skip-permissions --sandbox </dev/null > /tmp/agy-<jobname>.log 2>&1 &
```

Always append `</dev/null` and use `--dangerously-skip-permissions` in non-interactive tool environments to prevent interactive approval prompts from blocking the background process.

## Writing the Job Prompt

Sub-agents run in isolated context windows and do not see your conversation history. An effective prompt must contain:

1. **Concrete Objective:** Specific goal and scope (e.g., "Implement the UserProfile avatar upload handler", not "improve profiles").
2. **Path Anchors:** Explicit files, directories, and interfaces to inspect.
3. **Boundaries:** Explicit constraints (e.g., "Do not modify files outside src/components/", "Do not add new external npm dependencies").
4. **Output Contract:** Precise expected output (e.g., "Return only the complete TypeScript code block", or "Provide a markdown table of findings with File, Line, Issue, Fix").

## Verification Protocol

Never accept a sub-agent's output on faith:

1. **Read-Only Jobs:** Check `git status --porcelain` to verify no files were touched. Read the output log, evaluate the findings critically, and synthesize conclusions for the user.
2. **Write Jobs:** Run `git diff` to inspect every modified line. Execute the project's verification suite (`npm test`, `npx tsc --noEmit`, `cargo check`, etc.). If errors exist, send a targeted follow-up using `--conversation <id>` or fix the trivial issues directly.
3. **Report:** Clearly report what Antigravity executed, what verification steps passed, and what modifications you made.

## When NOT to Delegate

- Small, single-line edits that take less than 10 seconds to do yourself.
- Operations requiring interactive user decisions or sensitive production credentials.
- Irreversible destructive actions (production database migrations, live deployments, branch deletions).
