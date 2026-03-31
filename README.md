# Concerto

Multi-model orchestration skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Auto-scales model selection, thinking effort, and concurrency based on task complexity.

One command. Full pipeline: **Plan > Red-team > Implement > QA > Merge**.

## What It Does

Concerto turns Claude Code into a multi-model orchestrator. You describe a task, and it:

1. **Plans** — Opus 4.6 assesses complexity and decomposes the work
2. **Red-teams** — Codex 5.4 (xhigh thinking) checks the plan for blind spots
3. **Dispatches workers** — Routes each sub-task to the right model (Haiku for trivial, Sonnet for standard, Codex for precision work, Opus for architecture)
4. **Runs QA** — 4 parallel quality checks (bloat, conflicts, bugs, logic) before any merge
5. **Merges** — Sequential, safe merges with automatic conflict handling and cleanup

Trivial tasks short-circuit the pipeline and execute directly — no unnecessary overhead.

## Prerequisites

| Requirement | Install |
|---|---|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | `npm install -g @anthropic-ai/claude-code` |
| [Codex CLI](https://github.com/openai/codex) >= 0.115.0 | `npm install -g @openai/codex` |
| GNU timeout | macOS: `brew install coreutils` (provides `gtimeout`). Linux: built-in. |
| Git | Any modern version |

**Authentication:**
- Claude Code must be authenticated (Anthropic API key or `claude login`)
- Codex CLI must be authenticated (`codex login` or `OPENAI_API_KEY` env var)

## Installation

### 1. Copy the skill file

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills/concerto

# Copy the skill definition
cp SKILL.md ~/.claude/skills/concerto/SKILL.md
```

### 2. Verify installation

Open Claude Code in any git repository and type:

```
/concerto add a health check endpoint to this API
```

Claude Code will detect the skill and run the full pipeline.

## Usage

### Trigger phrases

The skill activates when you say any of:
- `/concerto <task>` — explicit invocation
- `"concerto: <task>"` — keyword trigger
- `"orchestrate <task>"` — keyword trigger
- `"full pipeline: <task>"` — keyword trigger
- `"plan and implement <task>"` — keyword trigger

### Examples

```
/concerto refactor the auth middleware to use JWT tokens

/concerto add pagination to the /users endpoint and update the tests

/concerto --dry-run migrate the database layer from Prisma to Drizzle

plan and implement a rate limiter for the API with Redis backing
```

### Dry-Run Mode

Use `--dry-run` to preview the plan (tasks, workers, estimated codex calls, wall time) without executing:

```
/concerto --dry-run add SSO login with SAML support
```

Review the plan, then say "go" or "proceed" to execute.

### What happens behind the scenes

```
USER MESSAGE
  |
  +-- PREFLIGHT: Check codex CLI, timeout cmd, git state
  |
  +-- PLAN (Opus 4.6): Assess complexity, decompose tasks
  |
  +-- RED-TEAM (Codex 5.4 xhigh, 5min timeout): Blind spot check
  |     Output: Task Packet (scope, files, acceptance criteria, risks)
  |
  +-- ROUTE by complexity:
  |     Trivial   -> Haiku/Sonnet inline (no worktree)
  |     Bounded   -> Codex 5.4-mini in worktree
  |     Standard  -> Codex 5.4 or Sonnet in worktree
  |     Complex   -> Parallel workers (max 3 simultaneous)
  |     Architect -> Opus lead + Codex review
  |
  +-- QA GATE (4 parallel Codex checks):
  |     1. Bloat check (dead code, over-engineering)
  |     2. Conflict check (merge safety)
  |     3. Bug scan (null access, races, leaks)
  |     4. Logic verify (matches spec, edge cases)
  |     All 4 must PASS. Max 2 remediation cycles.
  |
  +-- MERGE: Sequential, one branch at a time
  |     git merge --no-ff --no-commit -> test -> commit
  |     Any failure: git merge --abort (never manual .git cleanup)
  |
  +-- CLEANUP: Remove all worktrees and concerto/* branches
  |
  +-- REPORT: Summary of what was done
```

## Model Catalog

| Role | Model | Thinking | Max Concurrent |
|------|-------|----------|----------------|
| Boss / Planner | Opus 4.6 | max | 1 |
| Architecture worker | Opus 4.6 | high | 2 |
| Balanced worker | Sonnet 4.6 | medium | 3 |
| Grunt / boilerplate | Haiku 4.5 | low | 3 |
| Red-team reviewer | Codex 5.4 (gpt-5.4) | xhigh | 1 |
| Precision worker | Codex 5.4 (gpt-5.4) | high | 2 |
| Fast patch worker | Codex 5.4-mini (gpt-5.4-mini) | medium | 3 |
| QA gate check | Codex 5.4 (gpt-5.4) | high | 4 |

**Peak simultaneous agents: 4** (during QA phase). Never more.

## File Structure

```
.
+-- SKILL.md        # The skill definition (install this)
+-- SPEC.md         # Detailed technical specification
+-- README.md       # This file
+-- .gitignore
```

- **`SKILL.md`** is the only file needed at runtime. It contains the full orchestration logic as a Claude Code skill definition with frontmatter metadata.
- **`SPEC.md`** is the design document covering architecture decisions, model catalog, capacity limits, and the merge/cleanup protocol in detail.

## How It Works (for AI agents)

If you are a coding agent (Claude, Codex, etc.) and want to understand this skill:

1. **`SKILL.md`** is the complete runtime specification. It is a markdown file with YAML frontmatter that Claude Code loads as a skill. Read it top-to-bottom.

2. The skill follows a strict **7-step pipeline**: Preflight > Plan > Red-team > Dispatch > QA > Merge > Cleanup. Each step has explicit inputs, outputs, and error handling.

3. **Worker isolation** uses git worktrees. Claude models use `Agent(isolation: "worktree")`. Codex models use `git worktree add` + `codex exec -C <worktree>`.

4. **The Task Packet** (produced in Step 2 by the red-team) is the central data structure — it flows from red-team to worker prompt to QA prompt. This avoids recomputing scope/criteria at each stage. Task Packets are wrapped in `<<<CONCERTO_PACKET>>>` delimiters for machine extraction.

5. **All structured outputs use delimiters** for reliable parsing from noisy codex output. Red-team verdicts use `<<<CONCERTO_VERDICT>>>`, QA results use `<<<CONCERTO_QA>>>` with JSON payloads. Extract with `sed -n '/<<<CONCERTO_QA>>>/,/<<<END_CONCERTO_QA>>>/p'`. Only BLOCKERS can cause a FAIL — style/naming go in `notes` and never block.

6. **Worktree bootstrap**: For Node.js/Python/Rust projects, the skill installs dependencies in each worktree before dispatching the worker. Non-code tasks (docs, license, config metadata) skip the full 4-check QA gate.

7. **Timeouts are enforced at the bash level** via `gtimeout` (macOS) or `timeout` (Linux), not via Codex agent config. Every `codex exec` call is wrapped with `$TIMEOUT_CMD <seconds>`.

8. **Cleanup always runs** — even on failure. It removes all non-main worktrees and `concerto/*` branches.

## Customization

The skill is a single markdown file. To customize:

- **Change model routing** — Edit the Complexity Routing Table in Step 1
- **Adjust timeouts** — Modify the `$TIMEOUT_CMD` values in Steps 2-4
- **Add/remove QA checks** — Edit the 4 prompts in Step 4
- **Change concurrency limits** — Adjust "max 3 workers" / "4 QA checks" limits

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `ERROR: Codex CLI not installed` | `npm install -g @openai/codex` |
| `ERROR: Codex CLI below minimum` | `npm install -g @openai/codex` (update) |
| `ERROR: Neither gtimeout nor timeout found` | macOS: `brew install coreutils` |
| `ERROR: Not a git repository` | Run from inside a git repo |
| `WARNING: Working tree is dirty` | Commit or stash changes first |
| Codex auth failure | Run `codex login` or set `OPENAI_API_KEY` |
| QA stuck in FAIL loop | After 2 cycles, Concerto asks you for guidance |

## License

MIT
