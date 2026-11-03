---
name: antigravity-agents
description: Proactively delegate routine coding, frontend HTML/CSS, Tailwind components, boilerplate, code review, and research jobs to Google Antigravity CLI (agy) sub-agents. Automatically trigger this skill whenever a task involves routine frontend implementation, UI components, repetitive boilerplate, or heavy analysis, without waiting for the user to explicitly ask for Antigravity or delegation. Act as the orchestrator: plan the architecture, delegate mechanical tasks to Antigravity, verify the returned code, and integrate it. Includes automatic quota guard with intelligent fallback to internal thinking.
---

# Antigravity CLI Sub-Agents

Google Antigravity CLI (`agy`) is an autonomous terminal coding agent powered by Gemini and other models that can inspect repositories, generate code, edit files, and run commands. This skill turns Claude into an orchestrator with full control over the Antigravity sub-agent engine: model selection, reasoning effort, session memory, execution modes, directory scopes, and automated quota monitoring.

## Automatic Proactive Delegation Reflex

As an orchestrator (doctor / architect), do not burn expensive primary context window tokens or rate limits on repetitive typing. **Automatically delegate routine tasks without waiting for explicit user instructions.**

### Tasks to Delegate Proactively:
- Routine frontend components (React/Vue/HTML/CSS, Tailwind cards, navbars, forms, tables).
- Repetitive boilerplate, DTO interfaces, data mapping, and standard utility functions.
- Unit test suite generation, mock data creation, and lint/formatting sweeps.
- Independent code audits, security scans, and research sweeps.

### Tasks to Keep in Claude:
- High-level architectural planning, system design, and database schema decisions.
- Security-critical auth logic, payment workflows, and business invariants.
- Final code verification, build validation, and integration into the codebase.

## Quota Guard and Fallback Protocol

Antigravity operates on your Google account quotas. If Antigravity hits a rate limit, capacity exhaustion, or is temporarily unavailable, follow this decision tree:

1. **Detection:** The runner script `agy-delegate` automatically detects rate limits (429, resource exhausted, quota exceeded) and exits with code `42` plus the tag `[ANTIGRAVITY_QUOTA_EXHAUSTED]`.
2. **For Quick / Medium Tasks (Under ~150 lines):**
   - **Think inside yourself and fall back automatically:** Do not block the user. Solve the task directly within your own context.
   - Note to user: *"Antigravity kotası geçici olarak yoğun olduğundan görevi kendi içimde doğrudan tamamladım."*
3. **For Large / Heavy Tasks (Massive refactors, 10+ files, large repo scans):**
   - **Ask the user:** Stop and present the situation clearly:
     *"Antigravity kotası şu an limit aşımında görünüyor. İki seçeneğimiz var: (1) Görevi Claude olarak kendi context'imde parça parça yürütmek, (2) Bir süre bekleyip Antigravity kotası açılınca devam etmek. Nasıl ilerleyelim?"*

## Resolving the Binary

After install, `agy` is normally on your `PATH`. If a tool-spawned shell cannot find it (some shells do not inherit a freshly updated PATH), fall back to the full path:

- **Windows:** `%LOCALAPPDATA%\agy\bin\agy.exe` (in Bash-style shells: `"$LOCALAPPDATA/agy/bin/agy.exe"`)
- **macOS / Linux:** `~/.local/bin/agy-delegate`, `~/.local/bin/agy` or `which agy`

**Critical: Always close stdin.** `agy` blocks indefinitely (0% CPU, no output, even in print mode) when stdin is an open pipe, which is what non-interactive tool shells provide. Always launch jobs via `agy-delegate` (which handles this automatically) or append `</dev/null`.

## Preflight Verification (Once per Session)

Before the first job of a session, verify authentication with a quick probe:

```bash
agy-delegate --timeout 60s "Reply with exactly: OK"
```

- Replies `OK`: Authenticated, proceed.
- Prints a sign-in URL or errors about credentials: Stop and prompt the user to run `agy` once in their own interactive terminal to complete the one-time Google OAuth browser flow.

## The Orchestrator Control Panel

When delegating a task to Antigravity, actively calibrate the following parameters based on the task scope:

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

Use the bundled runner script (`agy-delegate`):

```bash
# Example: Fast frontend component generation with low reasoning effort
agy-delegate \
  --model gemini-3.8-flash-high \
  --effort low \
  --mode plan \
  --timeout 5m \
  "Write a responsive React Tailwind Card component for product listings. Output pure TSX code."

# Example: In-depth security audit with max reasoning effort
agy-delegate \
  --model gemini-4-argon-high \
  --effort max \
  --sandbox \
  --timeout 15m \
  --log-file /tmp/security-audit.log \
  "Perform a security review of authentication handlers in src/auth/. Report findings with severity and line numbers."
```

## Verification Protocol

Never accept a sub-agent's output on faith:

1. **Read-Only Jobs:** Check `git status --porcelain` to verify no files were touched. Read the output log, evaluate the findings critically, and synthesize conclusions for the user.
2. **Write Jobs:** Run `git diff` to inspect every modified line. Execute the project's verification suite (`npm test`, `npx tsc --noEmit`, `cargo check`, etc.). If errors exist, send a targeted follow-up using `--conversation <id>` or fix the trivial issues directly.
3. **Report:** Clearly report what Antigravity executed, what verification steps passed, and what modifications you made.
