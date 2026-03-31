# Concerto — Skill Specification (Draft)

## Overview

Concerto is a Claude Code skill that orchestrates multi-model, multi-agent workflows. It auto-scales model selection, thinking effort, and concurrency based on task complexity — functioning like a mechanism that scales up and down.

## Design Principles

1. **Boss First**: Opus 4.6 is the top-level orchestrator. The user talks to one agent.
2. **Blind Spot Check**: Every non-trivial task gets red-teamed by Codex 5.4 (xhigh, bash timeout enforcement) before workers are dispatched.
3. **Right Model for the Job**: Don't use Opus when Haiku will do. Route by complexity.
4. **Isolated Execution**: Workers run in git worktrees via `Agent(isolation: "worktree")` for Claude models, or `git worktree add` + `codex exec -C` for Codex workers. Trivial tasks run inline (no worktree).
5. **Gate Before Merge**: 4 parallel QA checks (each a focused Claude Agent calling codex exec) validate before any merge. All 4 must pass.
6. **One Lane Merges**: Merge sequentially via `git merge --no-ff --no-commit` from worktree branches. Abort path: `git merge --abort` on conflict, never manual `.git/` cleanup.
7. **Sequential Phases**: Workers complete before QA begins. QA completes before merge begins. Never 7 simultaneous agents — max 3 workers OR max 4 QA agents at any time.
8. **Bash Timeout Enforcement**: All Codex timeouts enforced via `gtimeout` (macOS/coreutils) or `timeout` (Linux) wrapping codex exec calls. Claude Agent workers use the Bash tool's built-in timeout parameter. Do not rely on `agents.job_max_runtime_seconds` (only applies to `spawn_agents_on_csv`).
9. **Cleanup on Failure**: Every failure path (timeout, QA fail, merge conflict, re-dispatch) must clean up worktrees and branches. A cleanup sweep runs at pipeline end regardless of outcome.

## Pipeline Stages

### Stage 1: Plan (Opus 4.6, max effort)
- Receive user request
- Assess complexity (trivial / bounded / complex)
- Decompose into discrete tasks if complex
- Output: task list with complexity ratings

### Stage 2: Red-Team (Codex 5.4, xhigh thinking, 5-min timeout)
- Review the plan for blind spots, risks, edge cases
- Challenge architectural assumptions
- Output: amended plan or approval with notes

### Stage 3: Route & Dispatch
Based on assessed complexity:

| Complexity | Worker(s) | Thinking | Isolation | Mechanism |
|-----------|-----------|----------|-----------|-----------|
| Trivial | Haiku 4.5 or Sonnet 4.6 | low/medium | inline (no worktree) | `Agent(model: "haiku")` or `Agent(model: "sonnet")` |
| Bounded | Codex 5.4-mini | medium | worktree | `git worktree add` + `$TIMEOUT_CMD 180 codex exec -C <wt> -m gpt-5.4-mini -c model_reasoning_effort=medium` |
| Standard | Codex 5.4 or Sonnet 4.6 | high | worktree | `Agent(model: "sonnet", isolation: "worktree")` or `git worktree add` + `$TIMEOUT_CMD 300 codex exec -C <wt> -m gpt-5.4 -c model_reasoning_effort=high` |
| Complex | Opus 4.6 + Codex 5.4 | max/high | parallel worktrees | Multiple `Agent(isolation: "worktree")` calls in parallel (max 3 simultaneous) |
| Architecture | Opus 4.6 (lead) + Codex 5.4 (review) | max/xhigh | sequential | Opus Agent first, then `$TIMEOUT_CMD 300 codex exec` review |

**Isolation rules:**
- Claude models (Opus/Sonnet/Haiku): Use `Agent(isolation: "worktree")` — Claude Code creates and manages the worktree automatically. Agent returns worktree path and branch on completion.
- Codex models: Use bash `git worktree add -b concerto/<task-id> .worktrees/<task-id>`, then `$TIMEOUT_CMD <seconds> codex exec -C .worktrees/<task-id>`. Clean up with `git worktree remove` after merge or on failure.
- Trivial tasks: Run inline, no isolation needed. Direct Agent call or `$TIMEOUT_CMD <seconds> codex exec` in current directory (no worktree).

### Stage 4: QA Gate (4 parallel Codex checks)

**Implementation**: 4 parallel Claude Code `Agent` calls, each running a focused `codex exec` review against the worktree diff. NOT build-qa-gate (which only does 1 review). NOT codex-review multi-agent (which is 2 sub-agents, not 4 checks).

Each QA agent:
```bash
$TIMEOUT_CMD 180 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -c service_tier=fast \
  --enable fast_mode \
  --ephemeral \
  --sandbox read-only \
  -C <worktree-path> \
  -o <output-file> \
  - <<< "<focused-prompt>"
```

The 4 checks with their focused prompts:
1. **Bloat check** — "Review this diff for unnecessary abstractions, dead code, over-engineering, premature generalization. Output: PASS or FAIL with specific findings."
2. **Conflict check** — "Review this diff for merge conflicts with main, semantic divergence from the base branch, and integration issues. Output: PASS or FAIL with specific findings."
3. **Bug scan** — "Review this diff for null access, off-by-one errors, race conditions, resource leaks, and security issues. Output: PASS or FAIL with specific findings."
4. **Logic verify** — "Compare this diff against the task spec below. Does the implementation match? Are edge cases handled? Output: PASS or FAIL with specific findings."

**Gate rule**: All 4 must output PASS. Any FAIL blocks the merge and reports findings back to the boss for remediation. After remediation, the failing check(s) re-run (not all 4).

### Stage 5: Merge (Boss orchestrated, sequential)

**One worktree at a time**, in dependency order. Use the actual branch name and worktree path returned by the worker — do NOT hardcode paths.

For **Codex workers**: branch is `concerto/<task-id>`, worktree is `.worktrees/<task-id>` (we created these).
For **Claude Agent workers**: branch and worktree path come from the Agent tool result. Store these when the Agent completes.

```bash
# Variables set per-worker from dispatch/completion tracking:
# BRANCH=<actual branch name>
# WORKTREE=<actual worktree path>

# Merge with full abort-on-any-failure wrapper
git merge --no-ff --no-commit "$BRANCH" || { git merge --abort; echo "MERGE BLOCKED: conflict"; exit 1; }

# Run tests if framework detected
if [ -f package.json ] && grep -q '"test"' package.json; then
  npm test || { git merge --abort; echo "MERGE BLOCKED: tests failed"; exit 1; }
fi

# Commit — abort merge state if commit itself fails (hook, signing, identity)
git commit -m "concerto: merge $BRANCH — <summary>" || { git merge --abort; echo "MERGE BLOCKED: commit failed"; exit 1; }

# Clean up worktree and branch (only after successful commit)
git worktree remove "$WORKTREE" 2>/dev/null
git branch -d "$BRANCH" 2>/dev/null
```

**Abort path**: On ANY failure (conflict, test, commit hook, signing), `git merge --abort` immediately. Never manually delete `.git/MERGE_HEAD` or `.git/CHERRY_PICK_HEAD`. Report the failure to the boss agent for manual resolution or re-dispatch.

### Stage 6: Cleanup Sweep (always runs)

Runs at pipeline end regardless of success/failure. Covers BOTH Codex-managed worktrees (`.worktrees/`) and Claude Agent-managed worktrees (arbitrary paths returned by Agent tool):

```bash
# Remove ALL non-main worktrees (covers both Codex and Claude Agent worktrees)
git worktree list --porcelain | awk '/^worktree /{print $2}' | while read wt; do
  # Skip the main working tree
  [ "$wt" = "$(git rev-parse --show-toplevel)" ] && continue
  git worktree remove "$wt" --force 2>/dev/null
done

# Remove any leftover concerto branches
for branch in $(git branch --list 'concerto/*'); do
  git branch -D "$branch" 2>/dev/null
done

# Prune worktree metadata for any already-deleted directories
git worktree prune
```

This prevents worktree/branch leaks from timeouts, QA failures, re-dispatches, or any other incomplete pipeline run.

### Stage 7 (Phase 2): Arbiter (Grok 4.2)
- Top-level referee when Opus and Codex disagree on architecture/engineering
- Web search capability for current best practices
- Called before workers, not after

## Model Catalog

| Key | Model | Invocation | Effort | Role | Max Simultaneous |
|-----|-------|-----------|--------|------|-----------------|
| opus-boss | Opus 4.6 | Top-level Claude Code session (model: opus) | max | Boss, plan, orchestration | 1 |
| opus-worker | Opus 4.6 | `Agent(model: "opus", isolation: "worktree")` | high | Architecture worker | 2 |
| sonnet-worker | Sonnet 4.6 | `Agent(model: "sonnet", isolation: "worktree")` | medium | Balanced worker | 3 |
| haiku-grunt | Haiku 4.5 | `Agent(model: "haiku")` | low | Boilerplate, formatting | 3 (inline, no worktree) |
| codex-redteam | gpt-5.4 | `$TIMEOUT_CMD 300 codex exec -m gpt-5.4 -c model_reasoning_effort=xhigh --sandbox read-only` | xhigh | Blind spot check | 1 |
| codex-worker | gpt-5.4 | `$TIMEOUT_CMD 300 codex exec -m gpt-5.4 -c model_reasoning_effort=high -C <worktree>` | high | Precision implementation | 2 |
| codex-mini | gpt-5.4-mini | `$TIMEOUT_CMD 180 codex exec -m gpt-5.4-mini -c model_reasoning_effort=medium -C <worktree>` | medium | Fast patches | 3 |
| codex-qa | gpt-5.4 | `$TIMEOUT_CMD 180 codex exec -m gpt-5.4 -c model_reasoning_effort=high --sandbox read-only` | high | QA gate check (4 parallel) | 4 |
| grok-arbiter | Grok 4.2 | `/grok-search` skill | — | Architecture arbiter (Phase 2) | 1 |

**Common flags for all codex exec calls**: `--ephemeral -c service_tier=fast --enable fast_mode`

**Note**: Model names in the "Model" column are prose labels. The "Invocation" column shows the actual CLI/API call. Claude models use Claude Code Agent tool parameters. Codex models use codex CLI flags.

## Skill Dependencies & Integration

Concerto is a **standalone orchestration skill** — it implements its own pipeline rather than wrapping existing skills. Existing skills inform the design but are not runtime dependencies for the core pipeline.

**Referenced but NOT required at runtime:**
- `codex-worker` — Concerto uses raw `codex exec` + `git worktree` instead. codex-worker's Single mode doesn't isolate, and its merge flow (cherry-pick) is unsafe.
- `build-qa-gate` — Concerto implements its own 4-check QA gate. build-qa-gate only runs 1 review.
- `codex-review` — Concerto calls `codex exec` directly with focused prompts. codex-review's multi-agent is 2 sub-agents, not the 4 checks we need.

**Optional skills (advisory, not pipeline-critical):**
- `grok-search` — for Grok-powered web search (Phase 2 arbiter)
- `gemini-advisor` / `kimi-advisor` / `minimax-advisor` — optional advisory calls

**Preflight requirements:**
- `command -v codex` — Codex CLI >= 0.115.0
- `codex exec` must work (user authenticated via `codex login` or `OPENAI_API_KEY`)
- `command -v gtimeout || command -v timeout` — GNU timeout required (macOS: `brew install coreutils` for `gtimeout`; Linux: built-in `timeout`)
- Git repository with clean working tree
- Claude Code with Agent tool support (isolation: "worktree" parameter)

**Timeout helper** (set once at pipeline start):
```bash
if command -v gtimeout &>/dev/null; then
  TIMEOUT_CMD="gtimeout"
elif command -v timeout &>/dev/null; then
  TIMEOUT_CMD="timeout"
else
  echo "ERROR: Neither gtimeout nor timeout found. Install coreutils: brew install coreutils"
  exit 1
fi
```
Then use `$TIMEOUT_CMD 300 codex exec ...` throughout.

## Capacity Limits & Timeouts

**Phases are sequential** — workers drain before QA starts, QA drains before merge starts. These are NOT simultaneous:

| Phase | Max Simultaneous | Timeout per Agent | Enforcement |
|-------|-----------------|-------------------|-------------|
| Red-team | 1 | 300s | `$TIMEOUT_CMD 300 codex exec ...` |
| Workers (Codex 5.4) | 2 | 300s | `$TIMEOUT_CMD 300 codex exec ...` |
| Workers (Codex mini) | 3 | 180s | `$TIMEOUT_CMD 180 codex exec ...` |
| Workers (Claude) | 3 | 600s | Bash tool timeout parameter (600000ms) |
| QA | 4 | 180s | `$TIMEOUT_CMD 180 codex exec ...` |
| Merge | 1 | 120s | Sequential, boss-controlled |

**Peak simultaneous agents**: 4 (during QA phase). Never 7.

**Timeout enforcement**:
- Codex exec calls: `$TIMEOUT_CMD <seconds> codex exec ...` (bash-level enforcement)
- Claude Agent calls: Bash tool `timeout` parameter (max 600000ms / 10 min)
- Do NOT rely on `agents.job_max_runtime_seconds` (only applies to Codex's `spawn_agents_on_csv`, not general multi_agent or standalone exec calls)

## Success Criteria

- Single invocation handles the full lifecycle: plan → validate → implement → review → merge
- No manual model selection required — concerto picks the right model
- Cost-efficient — trivial tasks don't burn premium model tokens
- Auditable — every stage produces typed output that can be reviewed
