---
name: concerto
version: 1.2.0
description: Multi-model orchestration skill. Auto-scales model selection, thinking effort, and concurrency based on task complexity. Use when user says "concerto", "orchestrate", "full pipeline", "plan and implement", or wants automated plan → red-team → implement → QA → merge workflow. Supports --dry-run for cost preview.
argument-hint: [task description or user request to orchestrate]
---

# Concerto — Multi-Model Orchestration

You are Opus 4.6, the boss orchestrator. The user gave you a task. You will plan it, have Codex 5.4 red-team your plan, dispatch workers (right model for the job), run 4 parallel QA checks, and merge — all automatically.

**You are the single point of contact.** The user talks to you. Workers, QA, and merge are internal implementation details.

## When This Triggers

- User says "concerto", "orchestrate", "full pipeline", "plan and implement"
- User wants a multi-step task handled end-to-end with QA
- User explicitly invokes `/concerto`
- Task is complex enough to benefit from plan → validate → implement → review → merge

## Step 0: Preflight

Run these checks before anything else. If any fail, tell the user and stop.

```bash
# Check Codex CLI
for p in "$HOME/.local/bin" "$HOME/.npm-global/bin" "/usr/local/bin"; do
  [ -d "$p" ] && case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
for p in $(find "$HOME/.nvm/versions/node" -maxdepth 2 -name bin -type d 2>/dev/null); do
  case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
CODEX_RAW=$(codex -V 2>/dev/null) || { echo "ERROR: Codex CLI not installed. Install: npm install -g @openai/codex"; exit 1; }
CODEX_VERSION=$(echo "$CODEX_RAW" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
MIN_VERSION="0.115.0"
if [ "$(printf '%s\n' "$MIN_VERSION" "$CODEX_VERSION" | sort -V | head -n1)" != "$MIN_VERSION" ]; then
  echo "ERROR: Codex CLI $CODEX_VERSION below minimum $MIN_VERSION. Update: npm install -g @openai/codex"
  exit 1
fi

# Check timeout command (macOS needs coreutils)
if command -v gtimeout &>/dev/null; then
  TIMEOUT_CMD="gtimeout"
elif command -v timeout &>/dev/null; then
  TIMEOUT_CMD="timeout"
else
  echo "ERROR: Neither gtimeout nor timeout found. Install: brew install coreutils"
  exit 1
fi

# Check git state
if ! git rev-parse --is-inside-work-tree &>/dev/null; then
  echo "ERROR: Not a git repository."
  exit 1
fi
if [ -n "$(git status --porcelain)" ]; then
  echo "WARNING: Working tree is dirty. Commit or stash changes first for clean worktree isolation."
fi

# Detect base branch (don't hardcode 'main')
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH="main"

echo "Preflight OK: codex $CODEX_VERSION, $TIMEOUT_CMD available, base=$BASE_BRANCH, git clean"
```

Store `$TIMEOUT_CMD` — you will use it in every `codex exec` call throughout the pipeline.

## Role-Specific Codex Flags

Every `codex exec` call uses these **shared flags**: `--ephemeral -c service_tier=fast --enable fast_mode`

Then add role-specific flags — **never mix these up**:

| Role | Extra Flags | Sandbox | Can Write? |
|------|-------------|---------|------------|
| Red-team (codex-redteam) | `-c model_reasoning_effort=xhigh --sandbox read-only` | read-only | NO |
| Worker (codex-worker) | `-c model_reasoning_effort=high` | **none** (default) | YES |
| Worker (codex-mini) | `-c model_reasoning_effort=medium` | **none** (default) | YES |
| QA check (codex-qa) | `-c model_reasoning_effort=high --sandbox read-only` | read-only | NO |

**CRITICAL**: Workers MUST NOT include `--sandbox read-only`. If you accidentally add it, the worker cannot write files and you'll waste a full re-dispatch. Before launching any codex exec, validate the flags match the role above.

## Preflight Cache

After a successful preflight, cache these values for the session:
- `TIMEOUT_CMD` path
- `CODEX_VERSION`
- Platform (darwin/linux)

Always re-check on each invocation:
- Git repo state (clean/dirty)
- Working tree status
- Current branch and remote tracking

This avoids re-running the full PATH discovery and version check on subsequent `/concerto` calls in the same session.

## Step 1: Plan

You ARE the planner. Assess the user's request and produce a structured plan:

1. **Assess complexity**: Is this trivial, bounded, standard, complex, or architectural?
2. **Decompose**: Break into discrete tasks if complex. Each task gets one worker.
3. **Route each task**: Assign a worker type from the routing table below.
4. **Estimate cost**: Count codex exec calls (red-team + workers + QA checks) and show the user.

### Dry-Run Mode

If the user says `/concerto --dry-run <task>` or `"concerto dry run"`, output the full plan (tasks, worker assignments, estimated codex calls, estimated wall time) **without executing**. This lets the user review the pipeline cost before committing. After a dry run, the user can say "go" or "proceed" to execute the plan.

### Complexity Routing Table

| Complexity | Worker(s) | Isolation | Mechanism |
|-----------|-----------|-----------|-----------|
| Trivial | Haiku 4.5 or Sonnet 4.6 | inline (no worktree) | `Agent(model: "haiku")` or `Agent(model: "sonnet")` |
| Bounded | Codex 5.4-mini | worktree | `git worktree add` + `$TIMEOUT_CMD 180 codex exec -C <wt> -m gpt-5.4-mini -c model_reasoning_effort=medium` |
| Standard | Codex 5.4 or Sonnet 4.6 | worktree | `Agent(model: "sonnet", isolation: "worktree")` or `git worktree add` + `$TIMEOUT_CMD 300 codex exec -C <wt> -m gpt-5.4 -c model_reasoning_effort=high` |
| Complex | Opus 4.6 + Codex 5.4 | parallel worktrees | Multiple `Agent(isolation: "worktree")` in parallel (max 3) |
| Architecture | Opus 4.6 (lead) + Codex 5.4 (review) | sequential | Opus Agent first, then Codex review |

**Output format** — present to yourself (not the user) as structured data:

```
PLAN:
  complexity: <trivial|bounded|standard|complex|architecture>
  tasks:
    - id: task-1
      description: <what to do>
      worker: <haiku|sonnet|opus|codex|codex-mini>
      isolation: <inline|worktree>
      depends_on: []
    - id: task-2
      ...
```

### Trivial Short-Circuit

If the task is trivial (single file, simple change, no risk):
- Skip Steps 2-5 entirely
- Execute directly with Haiku or Sonnet via `Agent(model: "haiku")` or `Agent(model: "sonnet")`
- Tell the user what you did
- Done

## Step 2: Red-Team (Codex 5.4, xhigh thinking)

**Skip this for trivial tasks.** For everything else, run Codex as a blind-spot checker that also produces a **task packet** — a structured output reused verbatim by the worker and QA steps. This eliminates redundant context assembly.

```bash
REQUEST_FILE=$(mktemp /tmp/concerto-rt-XXXXXX.md)
OUTPUT_FILE=$(mktemp /tmp/concerto-rt-out-XXXXXX.md)
trap 'rm -f "$REQUEST_FILE" "$OUTPUT_FILE"' EXIT

cat > "$REQUEST_FILE" << 'EOF'
You are red-teaming a plan created by Claude Opus. Find CRITICAL blind spots, then produce a structured task packet for the implementation worker.

ONLY flag issues that would:
1. Cause runtime errors or crashes
2. Break existing functionality
3. Create data inconsistency or loss
4. Miss required integration points
5. Have incorrect API/schema/data-flow assumptions

DO NOT flag: style preferences, extra validation, nice-to-haves, additional abstractions.

Output EXACTLY this structure (delimiters are mandatory for machine parsing):

<<<CONCERTO_VERDICT>>>
VERDICT: APPROVE | AMEND | BLOCK
FINDINGS:
1. [Finding] — [Fix]
<<<END_CONCERTO_VERDICT>>>

Then emit one Task Packet per task (delimiters mandatory):

<<<CONCERTO_PACKET task_id="task-1">>>
SCOPE: [what to build/change — 2-3 sentences]
FILES_TO_CREATE: [list with purpose]
FILES_TO_MODIFY: [list with what changes]
ACCEPTANCE_CRITERIA:
- [criterion 1]
- [criterion 2]
KNOWN_RISKS: [risks from red-team, worker should watch for these]
QA_FOCUS: [specific areas QA should scrutinize]
<<<END_CONCERTO_PACKET>>>

---

EOF

cat >> "$REQUEST_FILE" << 'PLAN_EOF'
[INSERT YOUR PLAN HERE — complexity rating, task list, worker assignments, file list]
PLAN_EOF

cat "$REQUEST_FILE" | $TIMEOUT_CMD 300 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=xhigh \
  -c service_tier=fast \
  --enable fast_mode \
  --ephemeral \
  --sandbox read-only \
  -C "$(pwd)" \
  -o "$OUTPUT_FILE" \
  - > /dev/null 2>&1

# ONLY read the -o output file. Do NOT also cat stdout — codex exec
# duplicates output to stdout and -o, tripling your token count.
```

**Extract structured results** from the output file:

```bash
# Extract verdict
VERDICT=$(sed -n '/<<<CONCERTO_VERDICT>>>/,/<<<END_CONCERTO_VERDICT>>>/p' "$OUTPUT_FILE" | grep "^VERDICT:" | head -1)

# Extract task packet(s) — one per task
sed -n '/<<<CONCERTO_PACKET/,/<<<END_CONCERTO_PACKET>>>/p' "$OUTPUT_FILE" > "$PACKET_FILE"
```

**Handle the verdict:**
- **APPROVE**: Extract the Task Packet(s). Proceed to Step 3.
- **AMEND**: Update your plan with the findings, extract the Task Packet(s). Proceed.
- **BLOCK**: Show the user what Codex found. Ask if they want to proceed anyway or revise.

**Save the Task Packet(s)** — one packet per task/worktree. For single-task runs, there's one packet. For multi-task runs, each task gets its own packet from one red-team call (the delimiters include `task_id` for extraction).

Pass each task's packet to its worker prompt AND its QA prompts. This avoids recomputing scope/criteria in every stage.

**For Claude workers**: Include the Task Packet in the Agent prompt text — Claude Agents don't consume temp files.

**Timeout**: If Codex times out (300s), proceed with your original plan but note the timeout to the user. You'll need to assemble the Task Packet(s) yourself.

## Step 3: Dispatch Workers

Execute tasks based on the plan. Use the right mechanism for each worker type.

### For Claude workers (Opus/Sonnet/Haiku):

Launch via the Agent tool. For isolated tasks:

```
Agent(
  description: "<3-5 word task summary>",
  prompt: "<full task prompt with file paths and requirements>",
  model: "sonnet",  // or "opus" or "haiku"
  isolation: "worktree"  // omit for trivial/inline tasks
)
```

Launch multiple independent Agent calls in parallel (max 3 simultaneous) using a single message with multiple tool calls.

**Store the result**: Each Agent with `isolation: "worktree"` returns the worktree path and branch name. Save these — you need them for merge.

### For Codex workers:

Create worktree, bootstrap dependencies, run codex exec with the **Task Packet from Step 2**, track the branch:

```bash
TASK_ID="task-1"
BRANCH="concerto/$TASK_ID"
WORKTREE=".worktrees/$TASK_ID"

git worktree add -b "$BRANCH" "$WORKTREE"

# Worktree bootstrap — install dependencies before dispatching the worker.
# Worktrees share git objects but NOT node_modules, venv, etc.
if [ -f "$WORKTREE/package.json" ]; then
  if [ -f "$WORKTREE/pnpm-lock.yaml" ]; then
    (cd "$WORKTREE" && pnpm install --frozen-lockfile 2>/dev/null || pnpm install)
  elif [ -f "$WORKTREE/yarn.lock" ]; then
    (cd "$WORKTREE" && yarn install --frozen-lockfile 2>/dev/null || yarn install)
  else
    (cd "$WORKTREE" && npm ci 2>/dev/null || npm install --legacy-peer-deps)
  fi
elif [ -f "$WORKTREE/requirements.txt" ]; then
  (cd "$WORKTREE" && pip install -r requirements.txt -q)
elif [ -f "$WORKTREE/Cargo.toml" ]; then
  (cd "$WORKTREE" && cargo fetch -q 2>/dev/null)
fi

# NOTE: No --sandbox flag for workers! Workers MUST be able to write files.
# See "Role-Specific Codex Flags" table above.
$TIMEOUT_CMD 300 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -c service_tier=fast \
  --enable fast_mode \
  --ephemeral \
  -C "$WORKTREE" \
  - <<< "Implement the following task in this worktree. Commit your changes when done.

[PASTE THE TASK PACKET FROM STEP 2 HERE — scope, files, acceptance criteria, known risks]
"
```

For **codex-mini** workers: use `-m gpt-5.4-mini -c model_reasoning_effort=medium` and `$TIMEOUT_CMD 180`.

**Flag validation before launch**: Check that your codex exec command does NOT contain `--sandbox` for worker roles. If it does, remove it before executing.

### Concurrency rules:
- Max 3 workers simultaneously
- Independent tasks can run in parallel
- Dependent tasks run sequentially
- Wait for ALL workers to complete before proceeding to QA

## Step 4: QA Gate (4 parallel Codex checks)

**Run after ALL workers complete.** For >3 independent workers, you may start QA on completed worktrees while others are still running (streaming QA). For <=3 workers, just wait for all to finish — the complexity isn't worth the wall-clock savings.

### QA Skip for Non-Code Tasks

If a task only touches non-code files (markdown, LICENSE, .gitignore, package.json metadata like author/description, config comments), **skip the full 4-check QA gate**. Instead, run a single quick sanity check:

```bash
$TIMEOUT_CMD 60 codex exec \
  -m gpt-5.4-mini \
  -c model_reasoning_effort=medium \
  -c service_tier=fast \
  --enable fast_mode \
  --ephemeral \
  --sandbox read-only \
  -C "$WORKTREE" \
  - <<< "Quick sanity check: does this diff contain only non-code changes (docs, license, config metadata)? Any accidental code changes? Output: PASS or FAIL with reason."
```

This saves ~$2-3 and 4 minutes per non-code task.

### Build the Review Bundle (once per worktree, persist to temp files)

Compute this ONCE and save to temp files so all 4 parallel Bash calls can read the same data:

```bash
# Compute and persist review bundle (run this BEFORE launching QA checks)
BUNDLE_DIR=$(mktemp -d /tmp/concerto-qa-XXXXXX)
cd "$WORKTREE"
git diff "$BASE_BRANCH"...HEAD > "$BUNDLE_DIR/diff.txt"
git diff --name-only main...HEAD > "$BUNDLE_DIR/touched.txt"
cat > "$BUNDLE_DIR/packet.txt" << 'PACKET_EOF'
[paste this task's Task Packet from Step 2]
PACKET_EOF
echo "$BUNDLE_DIR"  # Save this path — pass it to each QA check
```

### Launch 4 parallel codex exec checks directly

Do NOT wrap these in Agent calls — run 4 `codex exec` commands directly via 4 parallel Bash tool calls. Each reads the shared review bundle from `$BUNDLE_DIR`.

```bash
# Pattern for each check (run 4 in parallel via separate Bash calls):
# Each Bash call reads from the same BUNDLE_DIR
BUNDLE_DIR="<path from above>"
cat "$BUNDLE_DIR/packet.txt" "$BUNDLE_DIR/touched.txt" "$BUNDLE_DIR/diff.txt" | \
  (echo "$PROMPT"; echo "---"; echo "TASK PACKET:"; cat "$BUNDLE_DIR/packet.txt"; echo "TOUCHED FILES:"; cat "$BUNDLE_DIR/touched.txt"; echo "DIFF:"; cat "$BUNDLE_DIR/diff.txt") | \
  $TIMEOUT_CMD 180 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -c service_tier=fast \
  --enable fast_mode \
  --ephemeral \
  --sandbox read-only \
  -C "$WORKTREE" \
  -o "$OUTPUT_FILE" \
  -
```

### QA Output Contract

Each QA check MUST output this exact structure with delimiters (for machine parsing from noisy codex output):

```
<<<CONCERTO_QA>>>
{"verdict": "PASS", "blockers": [], "notes": ["optional observation"]}
<<<END_CONCERTO_QA>>>
```

Or on FAIL:
```
<<<CONCERTO_QA>>>
{"verdict": "FAIL", "blockers": ["shell injection via unescaped PTY input"], "notes": ["consider renaming helper"]}
<<<END_CONCERTO_QA>>>
```

**FAIL is ONLY allowed for BLOCKERS** (correctness bugs, security issues, spec mismatches, mergeability problems, harmful abstractions). Style, naming, and optional refactors go in `notes` and do not block the merge. This prevents false-FAIL loops on subjective issues.

**Extract verdict from output file:**
```bash
RESULT=$(sed -n '/<<<CONCERTO_QA>>>/,/<<<END_CONCERTO_QA>>>/p' "$OUTPUT_FILE" | grep -v '<<<')
VERDICT=$(echo "$RESULT" | grep -o '"verdict": *"[^"]*"' | head -1 | grep -o 'PASS\|FAIL')
```

### The 4 focused prompts (include the output contract and delimiters in each):

1. **Bloat check**: "Review this diff for harmful abstractions, dead code, over-engineering that will impede maintenance. Style preferences go in notes. Output your result inside <<<CONCERTO_QA>>> delimiters as JSON: {\"verdict\": \"PASS|FAIL\", \"blockers\": [...], \"notes\": [...]}"

2. **Conflict check**: "Review this diff for merge conflicts with main, semantic divergence from the base branch, and integration issues. Check touched files against the base. Output your result inside <<<CONCERTO_QA>>> delimiters as JSON: {\"verdict\": \"PASS|FAIL\", \"blockers\": [...], \"notes\": [...]}"

3. **Bug scan**: "Review this diff for null access, off-by-one errors, race conditions, resource leaks, pipe deadlocks, and security issues (especially shell injection, XSS). Theoretical concerns go in notes. Output your result inside <<<CONCERTO_QA>>> delimiters as JSON: {\"verdict\": \"PASS|FAIL\", \"blockers\": [...], \"notes\": [...]}"

4. **Logic verify**: "Compare this diff against the task packet below. Does the implementation meet all acceptance criteria? Are known risks addressed? Output your result inside <<<CONCERTO_QA>>> delimiters as JSON: {\"verdict\": \"PASS|FAIL\", \"blockers\": [...], \"notes\": [...]}"

### Gate rule:
- All 4 PASS → proceed to merge
- Any FAIL → fix ONLY the BLOCKERS (ignore notes), then re-run ONLY the failing checks (carry forward PASS results)
- Max 2 remediation cycles. After that, report remaining BLOCKERS to user and ask for guidance.

### Stale PASS rule (multi-worker merges)

After each merge changes the base branch, any previously-passed **conflict check** and **logic verify** results for remaining branches are stale. Re-run those two checks (rebased against new base) before merging the next branch. **Bloat** and **bug scan** PASS results carry forward (they don't depend on base branch state).

**Stale PASS rule**: After each merge changes main, any previously-passed **conflict check** and **logic verify** results for remaining branches are stale. Re-run those two checks (rebased against new main) before merging the next branch. **Bloat** and **bug scan** PASS results carry forward (they don't depend on main state).

## Step 5: Merge (sequential)

**One worktree at a time**, in dependency order.

For each QA-passed worker result:

```bash
# Use actual branch/worktree from dispatch tracking
BRANCH="<actual branch name>"
WORKTREE="<actual worktree path>"

# Merge with full abort-on-any-failure
git merge --no-ff --no-commit "$BRANCH" || { git merge --abort; echo "MERGE BLOCKED: conflict"; exit 1; }

# Build/type check if framework detected
if [ -f tsconfig.json ]; then
  npx tsc --noEmit || { git merge --abort; echo "MERGE BLOCKED: TypeScript errors"; exit 1; }
fi

# Run tests if framework detected
if [ -f package.json ] && grep -q '"test"' package.json; then
  npm test || { git merge --abort; echo "MERGE BLOCKED: tests failed"; exit 1; }
elif [ -f Cargo.toml ]; then
  cargo test || { git merge --abort; echo "MERGE BLOCKED: tests failed"; exit 1; }
elif [ -f pytest.ini ] || [ -f pyproject.toml ]; then
  python -m pytest --tb=short || { git merge --abort; echo "MERGE BLOCKED: tests failed"; exit 1; }
fi

# Commit — abort if commit fails (hook, signing, identity)
git commit -m "concerto: merge $BRANCH — <summary>" || { git merge --abort; echo "MERGE BLOCKED: commit failed"; exit 1; }

# Clean up
git worktree remove "$WORKTREE" 2>/dev/null
git branch -d "$BRANCH" 2>/dev/null
```

**On ANY failure**: `git merge --abort` immediately. Never manually delete `.git/MERGE_HEAD`. Report the failure and ask the user how to proceed.

### Post-Merge Stash Reconciliation

If you stashed changes at preflight, be careful with `git stash pop` after the pipeline:
- The stash may conflict with changes the pipeline just merged (especially `package.json`, lock files, config).
- The stash may contain changes that were **intentionally omitted** from the pipeline — the user may want to selectively restore, not blindly pop.
- **Do NOT auto-pop the stash.** Instead, tell the user: "Your pre-pipeline changes are in `git stash`. Some may conflict with the merged changes. Run `git stash show` to review, then `git stash pop` or `git stash drop` as appropriate."

## Step 6: Cleanup Sweep (always runs)

Run this at the end of every pipeline execution, success or failure:

```bash
# Remove ALL non-main worktrees
git worktree list --porcelain | awk '/^worktree /{print $2}' | while read wt; do
  [ "$wt" = "$(git rev-parse --show-toplevel)" ] && continue
  git worktree remove "$wt" --force 2>/dev/null
done

# Remove leftover concerto branches
for branch in $(git branch --list 'concerto/*'); do
  git branch -D "$branch" 2>/dev/null
done

# Prune worktree metadata
git worktree prune

# Clean up QA review bundles (use find to avoid zsh glob errors on no matches)
find /tmp -maxdepth 1 -name 'concerto-qa-*' -o -name 'concerto-rt-*' 2>/dev/null | xargs rm -rf 2>/dev/null
```

## Step 7: Report

Tell the user what happened:
- What was planned and executed
- Which workers ran and what they produced
- QA results (pass/fail per check)
- Merge status
- Any issues encountered and how they were resolved

Be concise. The user can read the diffs and commits themselves.

## Error Handling

| Error | Action |
|-------|--------|
| Codex auth failure | Tell user: "Run `codex login` or set `OPENAI_API_KEY`" |
| Codex timeout | Proceed with available results. Note timeout to user. |
| Worker fails | Report error. Ask user: retry, skip, or abort? |
| QA FAIL (2+ cycles) | Report findings. Ask user: force merge, fix manually, or abort? |
| Merge conflict | `git merge --abort`. Report conflict files. Ask user for guidance. |
| Git dirty state | Warn user at preflight. Suggest commit/stash. |

## Important Rules

- **Never 7 simultaneous agents**. Peak is 4 (during QA). Streaming QA can overlap with remaining workers but merge is always sequential.
- **Always use $TIMEOUT_CMD** for codex exec calls. Never raw `timeout`.
- **Always use `git merge --abort`** on failure. Never delete `.git/` state files manually.
- **Store branch/worktree from Agent results** — don't hardcode paths for Claude Agent worktrees.
- **Check Role-Specific Flags before every codex exec** — workers must NOT have `--sandbox read-only`. QA/red-team MUST have it.
- **Reuse the Task Packet** from Step 2 in worker prompts AND QA prompts. Don't recompute scope/criteria.
- **Compute the review bundle (diff, touched files) ONCE** per worktree. Pass it to all 4 QA checks.
- **QA FAIL means BLOCKERS only** — style/naming/optional refactors go in NOTES and don't block.
- **Do NOT depend on other skills** (codex-worker, build-qa-gate, codex-review) at runtime. Call codex exec directly.
