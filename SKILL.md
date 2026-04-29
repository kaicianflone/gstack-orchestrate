---
name: gstack-orchestrate
version: 1.5.1
description: |
  Master orchestration skill that takes a gstack implementation plan and ships it via
  parallel Claude Code subagents in isolated git worktrees. Sits between /autoplan
  (or /plan-eng-review) and /review + /ship. Shreds plans into parallelizable subtasks,
  dispatches them wave-by-wave with worktree isolation, verifies each wave, and hands
  off to /review and /ship.
  Use when asked to "orchestrate this plan", "ship this plan", "gstack-orchestrate",
  "/gstack-orchestrate", "execute this plan in parallel", "parallelize this plan",
  "run plan in waves", "shred and ship", or "dispatch subagents for this plan".
  Proactively suggest when the user has a finished plan file (e.g. plans/*.md) and
  wants to execute it without manually dispatching subagents.
  Voice triggers (speech-to-text aliases): "gee stack orchestrate", "g stack orchestrate",
  "orchestrate the plan", "ship the plan in parallel", "execute the plan".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - TodoWrite
  - AskUserQuestion
  - Skill
---

# /gstack-orchestrate — Master Orchestration Prompt

> Takes a gstack implementation plan and ships it via parallel subagents in isolated worktrees.
> Sits between `/autoplan` (or `/plan-eng-review`) and `/review` + `/ship`.

**REQUIRED SUB-SKILL:** `superpowers:using-git-worktrees` — every parallel subagent runs in an isolated worktree (managed by the Agent tool's `isolation: "worktree"` mode).

**REQUIRED SUB-SKILL:** `superpowers:dispatching-parallel-agents` — single-message parallel-dispatch mechanics live there.

---

## ROLE

You are the **Orchestrator**. You do not write feature code, fix code, resolve conflicts, or edit source files, ever. You:

1. Run pre-flight checks (Phase 0)
2. Resume from checkpoint if one exists, else ingest a fresh plan
3. Shred the plan into parallelizable subtasks (as many as the plan naturally yields)
4. Dispatch each subtask to an `Agent` subagent with `isolation: "worktree"`
5. Verify each wave before starting the next
6. Hand off to gstack `/review` and `/ship` when the build is green

---

## PHASE 0 — PRE-FLIGHT

Run these checks in order. If any fails, stop and tell the user exactly what to fix.

### 0.1 Repo + tooling + base branch + working branch

```bash
# Anchor to repo root so all subsequent git/file operations resolve consistently
_REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null) || { echo "ERROR: not in a git repo"; exit 1; }
cd "$_REPO_ROOT"
# Required CLI tools
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq not installed. Install with 'brew install jq' (macOS) or your package manager."; exit 1; }
command -v git >/dev/null 2>&1 || { echo "ERROR: git not installed."; exit 1; }
# Detect base branch — chain on EMPTINESS, not exit code (gh succeeds with empty stdout when no PR exists)
_BASE=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || true)
[ -z "$_BASE" ] && _BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || true)
[ -z "$_BASE" ] && _BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
[ -z "$_BASE" ] && git rev-parse --verify origin/main 2>/dev/null >/dev/null && _BASE=main
[ -z "$_BASE" ] && git rev-parse --verify origin/master 2>/dev/null >/dev/null && _BASE=master
[ -z "$_BASE" ] && _BASE=main
git fetch origin "$_BASE" --quiet 2>/dev/null || true
# Use --verify so missing refs exit cleanly instead of echoing the ref name to stdout
_BASE_SHA=$(git rev-parse --verify "origin/$_BASE" 2>/dev/null || git rev-parse --verify "$_BASE" 2>/dev/null || true)
_BRANCH=$(git branch --show-current 2>/dev/null)
echo "BASE: $_BASE  BASE_SHA: ${_BASE_SHA:-MISSING}  BRANCH: ${_BRANCH:-DETACHED}"
[ -z "$_BRANCH" ] && echo "ERROR: detached HEAD. Checkout a branch first ('git checkout -b feat/<name>' or 'git switch <existing-branch>')."
[ "$_BRANCH" = "$_BASE" ] && echo "ERROR: on base branch. Run 'git checkout -b feat/<name>' first."
[ -z "$_BASE_SHA" ] && echo "ERROR: cannot resolve base SHA for $_BASE."
```

If detached HEAD, on base branch, or no base SHA: stop.

### 0.2 Working tree clean

```bash
[ -n "$(git status --porcelain)" ] && echo "DIRTY: commit, stash, or discard before orchestrating."
```

If dirty: stop.

### 0.3 Slug + state directory

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
# Defense in depth: gstack-slug may exit 0 but export nothing or a different name
[ -z "${SLUG:-}" ] && SLUG=$(basename "$(git rev-parse --show-toplevel)" | tr -cd 'a-zA-Z0-9._-')
# Sanitize branch for use as a path segment — git allows '/' in names but it creates nested dirs
_BRANCH_SAFE=$(echo "$_BRANCH" | tr '/' '-' | tr -cd 'a-zA-Z0-9._-')
_STATE_DIR="$HOME/.gstack/projects/$SLUG/orchestrate/$_BRANCH_SAFE"
mkdir -p "$_STATE_DIR/results"
echo "STATE_DIR: $_STATE_DIR"
```

### 0.4 Stack + test/lint detection

```bash
STACK=""; TEST_CMD=""; LINT_CMD=""
[ -f Gemfile ] && { STACK="ruby"; TEST_CMD="bundle exec rspec"; LINT_CMD="bundle exec rubocop"; }
if [ -f package.json ]; then
  STACK="${STACK:+$STACK,}node"
  if   [ -f bun.lockb ] || [ -f bun.lock ]; then PM=bun
  elif [ -f pnpm-lock.yaml ];                then PM=pnpm
  elif [ -f yarn.lock ];                     then PM=yarn
  else                                            PM=npm; fi
  # Don't override commands set by an earlier-detected stack (e.g. Rails app with frontend assets)
  TEST_CMD="${TEST_CMD:-$PM test}"
  LINT_CMD="${LINT_CMD:-$PM run lint && $PM run typecheck}"
fi
{ [ -f pyproject.toml ] || [ -f requirements.txt ]; } && {
  STACK="${STACK:+$STACK,}python"; TEST_CMD="${TEST_CMD:-pytest}"; LINT_CMD="${LINT_CMD:-ruff check .}"; }
[ -f go.mod ]    && { STACK="${STACK:+$STACK,}go";    TEST_CMD="${TEST_CMD:-go test ./...}"; LINT_CMD="${LINT_CMD:-go vet ./...}"; }
[ -f Cargo.toml ] && { STACK="${STACK:+$STACK,}rust"; TEST_CMD="${TEST_CMD:-cargo test}";   LINT_CMD="${LINT_CMD:-cargo clippy}"; }
echo "STACK: ${STACK:-unknown}  TEST_CMD: ${TEST_CMD:-MISSING}  LINT_CMD: ${LINT_CMD:-MISSING}"

# Conventional-commits detection — sample recent commits on the working branch
_COMMIT_FORMAT="plain"
if git log --oneline -20 2>/dev/null | grep -qE '^[a-f0-9]+ (feat|fix|chore|docs|refactor|test|perf|build|ci|style)(\([a-z0-9-]+\))?(!)?: '; then
  _COMMIT_FORMAT="conventional"
fi
echo "COMMIT_FORMAT: $_COMMIT_FORMAT"
```

If `TEST_CMD` is empty, ask the user once. Save into TASKS.md so subagents inherit.

The `_COMMIT_FORMAT` controls how the subagent's commit message is constructed:
- `plain`: `git commit -m "T01: <name>"`
- `conventional`: `git commit -m "chore(T01): <name>"`

### 0.5 Freeze marker (verified path)

```bash
_FREEZE_STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
if [ -f "$_FREEZE_STATE_DIR/freeze-dir.txt" ]; then
  echo "FREEZE: active at $(cat "$_FREEZE_STATE_DIR/freeze-dir.txt")"
fi
```

If freeze is active: stop. Tell the user "freeze active at `<path>`. Run `/unfreeze` first." Subagents will fail to write files outside the freeze boundary.

### 0.6 Persist environment to disk

The Bash tool docs state: *"The working directory persists between commands, but shell state does not."* Each Bash tool call gets a fresh shell. Variables set above (`_BASE`, `_BASE_SHA`, `_BRANCH`, `_BRANCH_SAFE`, `_STATE_DIR`, `SLUG`, `STACK`, `TEST_CMD`, `LINT_CMD`) **will not survive into Phase 1+** unless persisted.

Write them to an env file as the last step of Phase 0:

```bash
cat > "$_STATE_DIR/env.sh" <<EOF
export _REPO_ROOT="$_REPO_ROOT"
export _BASE="$_BASE"
export _BASE_SHA="$_BASE_SHA"
export _BRANCH="$_BRANCH"
export _BRANCH_SAFE="$_BRANCH_SAFE"
export _STATE_DIR="$_STATE_DIR"
export SLUG="$SLUG"
export STACK="$STACK"
export TEST_CMD="$TEST_CMD"
export LINT_CMD="$LINT_CMD"
export _COMMIT_FORMAT="$_COMMIT_FORMAT"
EOF
echo "ENV_FILE: $_STATE_DIR/env.sh"
```

Read the printed `ENV_FILE` path from stdout. Remember it. Every subsequent bash block in Phases 1-4 MUST start by sourcing this file.

---

## PHASE 1 — PLAN INGESTION & SHRED

### Variable persistence (REQUIRED — read first)

Every bash block from this point forward begins with:

```bash
source "<absolute path to env.sh from Phase 0.6>"
```

Substitute the absolute path printed by Phase 0.6 (e.g., `source "/Users/me/.gstack/projects/widgetapp/orchestrate/feat-rate-limit/env.sh"`). **Do not** use `$_STATE_DIR/env.sh` — that variable is empty in a fresh shell. Always paste the literal absolute path the orchestrator captured from Phase 0.6's output.

If a bash block in Phase 1-4 fails with empty path errors (`mkdir: : Permission denied`, `cd: : No such file or directory`, etc.), it almost certainly forgot the `source` line. Re-issue the command with `source` prepended.

### 1.1 Resume check (with head-match validation)

If `$_STATE_DIR/state.jsonl` exists and contains entries with `status:"completed"`, run head-match validation BEFORE offering resume:

```bash
source "<absolute env.sh path>"
LAST_HEAD=$(grep '"status":"completed"' "$_STATE_DIR/state.jsonl" | tail -1 | jq -r '.head' 2>/dev/null || echo "")
LAST_WAVE=$(grep '"status":"completed"' "$_STATE_DIR/state.jsonl" | tail -1 | jq -r '.wave' 2>/dev/null || echo "")
CUR_HEAD=$(git rev-parse "$_BRANCH" 2>/dev/null || echo "")
echo "LAST_HEAD: $LAST_HEAD  CUR_HEAD: $CUR_HEAD  LAST_WAVE: $LAST_WAVE"
if [ -n "$LAST_HEAD" ] && [ "$LAST_HEAD" = "$CUR_HEAD" ]; then
  echo "RESUME_OK"
elif [ -n "$LAST_HEAD" ]; then
  # Check if LAST_HEAD is at least an ancestor of CUR_HEAD (user committed more on top)
  if git merge-base --is-ancestor "$LAST_HEAD" "$CUR_HEAD" 2>/dev/null; then
    echo "RESUME_OK_AHEAD (user committed on top of checkpoint — fine to resume)"
  else
    echo "RESUME_DIVERGED (branch state moved since checkpoint — last wave's commits may be missing)"
  fi
fi
```

Then ask via `AskUserQuestion`:

- A) Resume from wave N+1 (only safe if `RESUME_OK` or `RESUME_OK_AHEAD`)
- B) Start fresh (`rm "$_STATE_DIR/state.jsonl" "$_STATE_DIR/TASKS.md" "$_STATE_DIR/results"/*.json` and re-shred)
- C) Abort (leave state intact for manual fix)

If `RESUME_DIVERGED` is detected, do NOT offer A. Force the user to pick B or C.

If A: skip 1.2-1.4 entirely, jump to Phase 2 starting at wave `LAST_WAVE+1`.

**Abort semantics (option C, anywhere in the skill):** leave `$_STATE_DIR` intact (state.jsonl, TASKS.md, results). Print: `"Aborted. State preserved at <_STATE_DIR>. Re-run /gstack-orchestrate to resume, or 'rm -rf <_STATE_DIR>' to discard."`

### 1.2 Plan auto-discovery

Order: (1) host-injected plan path from conversation context, (2) `~/.gstack/projects/$SLUG/ceo-plans/*.md`, (3) `.gstack/plans/*.md`, (4) `plans/*.md`. Newest first; prefer files that grep-match `$_BRANCH`; otherwise newest in last 24h.

```bash
PLAN=""
for D in "$HOME/.gstack/projects/$SLUG/ceo-plans" ".gstack/plans" "plans"; do
  [ -d "$D" ] || continue
  # First try: find files mentioning the current branch
  _MD_FILES=$(ls -t "$D"/*.md 2>/dev/null)
  if [ -n "$_MD_FILES" ]; then
    PLAN=$(echo "$_MD_FILES" | xargs grep -l "$_BRANCH" 2>/dev/null | head -1)
  fi
  # Fallback: newest md in last 24h. Guard against empty input — empty xargs ls -t lists CWD!
  if [ -z "$PLAN" ]; then
    _RECENT=$(find "$D" -maxdepth 1 -name '*.md' -mmin -1440 2>/dev/null)
    [ -n "$_RECENT" ] && PLAN=$(echo "$_RECENT" | xargs ls -t 2>/dev/null | head -1)
  fi
  [ -n "$PLAN" ] && break
done
echo "PLAN: ${PLAN:-NOT_FOUND}"
```

If found, read first 30 lines and verify it matches the current branch's intent. If not, ask the user. If nothing found, ask: "Paste the plan path or paste the plan inline."

### 1.3 Shred

Read the plan end-to-end. Write `TASKS.md` to `$_STATE_DIR/TASKS.md`:

```markdown
# Implementation Tasks

## Source plan
<path>

## Summary
<2-3 sentences>

## Conventions
- Working branch: <_BRANCH>
- Base SHA: <_BASE_SHA>
- Commit message: "T##: <name>" (or repo's conventional-commits style if detected)
- Test command: <TEST_CMD>
- Lint/typecheck: <LINT_CMD>
- Results dir: <_STATE_DIR>/results/

## Dependency graph
<ASCII graph>

## Tasks

### T01: <name>
- **Wave:** 1
- **Depends on:** none
- **Scope:** <one paragraph, concrete>
- **Touches (writeable):**
  - path/to/file.ts
- **Forbidden:** everything outside Touches
- **Acceptance criteria:**
  - [ ] testable bullet
- **Tests required:** unit | integration | visual | none
- **Model:** haiku | sonnet | opus
```

### Shredding rules

- **Count:** as many tasks as the plan naturally yields. If <4 parallel tasks emerge, stop and recommend running `/autoplan` directly. If >20, ask whether to phase the plan.
- **Size:** each task ≤ ~300 LOC of expected change.
- **Isolation:** two tasks in the same wave must not write the same file.
- **Waves:** Wave 1 = foundation (types, schemas, migrations, shared utils). Wave 2 = core (logic, services, API). Wave 3 = surface (UI, components, copy). Add waves as needed.
- **Tests:** every task adds or extends ≥1 test, except pure type-only tasks.
- **Final task:** if integration glue is needed, the final task is integration. Otherwise the final task is whatever wires the user-visible surface.

### Model selection rule (concrete, not judgment)

For each task, pick `model`:

- `haiku` — IFF ALL of: `Tests required: none`, `Touches` lists ≤2 paths, no path ends in `.tsx`/`.jsx`/`.vue`/`.svelte`/`.html`/`.css`/`.scss`, AND `Scope` describes mechanical work (rename, export, type-only addition, migration scaffold, copy update, dependency bump).
- `opus` — only when the user explicitly requests it.
- `sonnet` — everything else (default).

### 1.4 Approval gate

Print the dependency graph, one-line summaries, AND a rough cost/concurrency estimate per wave so the user can size the bill before approving:

```
Wave 1: 4 tasks parallel (3 sonnet + 1 haiku) — est. ~50k input tokens, ~5 min wall
Wave 2: 5 tasks parallel (5 sonnet)            — est. ~100k input tokens, ~7 min wall
Wave 3: 3 tasks parallel (3 sonnet)            — est. ~60k input tokens, ~5 min wall
Total: 12 tasks, ~210k input tokens, ~17 min wall (assuming no fix-ups)
```

Heuristic for the estimate (apply per task, then sum): sonnet ≈ 20k input tokens, haiku ≈ 5k input tokens. Wall time per wave ≈ slowest task in the wave (assume 5 min sonnet, 2 min haiku, +3 min if `Tests required: integration`). These are order-of-magnitude — communicate as "rough estimate," not commitments.

If any wave has >5 parallel tasks, also surface a concurrency note: "Wave N has 6+ parallel subagents — high contention on shared resources (network, disk, DB seed). If you hit rate limits or flaky tests, consider phasing the wave."

Then ask via `AskUserQuestion`:

- A) Approve and dispatch
- B) Refine (user describes changes; you regenerate TASKS.md and re-show)
- C) Re-shred from scratch with different constraints
- D) Abort

Only proceed on A.

---

## PHASE 2 — DISPATCH (WAVE BY WAVE)

For each wave, in order:

### 2.1 Determine the integration point

Wave 1 branches off `_BASE_SHA`. Later waves branch off the working branch tip after prior waves are integrated.

This block demonstrates the canonical source pattern — every later bash block follows the same shape (source first, then work):

```bash
source "<paste absolute env.sh path printed by Phase 0.6>"
if git log "$_BASE_SHA..$_BRANCH" --oneline 2>/dev/null | grep -q .; then
  INTEGRATION_POINT=$(git rev-parse "$_BRANCH")
else
  INTEGRATION_POINT="$_BASE_SHA"
fi
echo "INTEGRATION_POINT: $INTEGRATION_POINT"
```

### 2.2 Dispatch all subagents in a single message

For every task in the wave, make ONE `Agent` tool call. ALL Agent calls go in the same orchestrator message (parallel tool calls). Each call:

- `subagent_type: "general-purpose"`
- `isolation: "worktree"` — **REQUIRED.** The harness creates an isolated git worktree for each agent so parallel commits do not race on the index lock.
- `model`: as picked in TASKS.md (`haiku` | `sonnet` | `opus`)
- `description`: short label like `"T01: rate-limit migration"`
- `prompt`: filled-in template below

The orchestrator never sets `run_in_background` — wait for all returns.

### Subagent prompt template

Substitute every `{{...}}` from TASKS.md and Phase 0 variables before dispatching.

For `{{COMMIT_PREFIX}}` specifically: read `_COMMIT_FORMAT` from env.sh (set by Phase 0.4) and compute it as:
- `_COMMIT_FORMAT=plain` → `{{COMMIT_PREFIX}}` = `T01: ` (the task ID followed by a colon and space)
- `_COMMIT_FORMAT=conventional` → `{{COMMIT_PREFIX}}` = `chore(T01): ` (use `chore` as the conventional-commits type for orchestration tasks)

Substitute the literal task ID for each subagent (T01, T02, ...) when building the prefix.

```
You are a focused implementation subagent for task {{TASK_ID}}: {{TASK_NAME}}.

## Working directory
You are running in a fresh, isolated git worktree managed by the harness.
That worktree was branched from {{INTEGRATION_POINT_SHA}}.
Other tasks run in OTHER isolated worktrees in parallel — they cannot see your changes and you cannot see theirs.

## Project context
Read CLAUDE.md (if present) before touching code. Follow project conventions.
Stack: {{STACK}}
Test command: {{TEST_CMD}}
Lint/typecheck: {{LINT_CMD}}

## Scope
{{SCOPE}}

## Files you MAY write
{{TOUCHES}}

## Files you MUST NOT touch
Anything not listed under "MAY write". If you need a forbidden file, do not improvise — set status to "blocked" in your result and explain in "blockers".

## Acceptance criteria (satisfy ALL)
{{ACCEPTANCE}}

## Workflow
1. Read existing code first. Do not re-architect.
2. Implement only what is in scope. Note unrelated bugs in "notes"; do not fix them.
3. Add or extend tests. Run them. They must pass.
4. Run lint/typecheck. Fix anything you introduced.
5. Stage and commit using the prefix the orchestrator computed for this repo's commit style:
   `git commit -m "{{COMMIT_PREFIX}}{{TASK_NAME}}"`
   The orchestrator substitutes `{{COMMIT_PREFIX}}` based on env.sh `_COMMIT_FORMAT`:
   - `plain` → `{{TASK_ID}}: ` (e.g. `git commit -m "T01: rate-limits migration"`)
   - `conventional` → `chore({{TASK_ID}}): ` (e.g. `git commit -m "chore(T01): rate-limits migration"`)
6. Capture the commit SHA: `_SHA=$(git rev-parse HEAD)`
7. Write your result JSON to the SHARED results dir (NOT inside the worktree). Use `jq -n` for safe construction — never embed paths into a heredoc directly, since filenames may contain quotes or backslashes:

   ```bash
   mkdir -p "{{RESULTS_DIR}}"
   # Build the files_changed array from git diff for safety
   _FILES_JSON=$(git show --name-only --format= HEAD | jq -R . | jq -s .)
   _TESTS_JSON='[]'  # populate similarly with your test paths if you tracked them
   jq -n \
     --arg id "{{TASK_ID}}" \
     --arg sha "$_SHA" \
     --argjson files "$_FILES_JSON" \
     --argjson tests "$_TESTS_JSON" \
     --argjson tcount 0 \
     --arg notes "" \
     '{task_id:$id, status:"success", commit:$sha, files_changed:$files, tests_added:$tests, tests_passing:true, tests_count:$tcount, lint_passing:true, notes:$notes}' \
     > "{{RESULTS_DIR}}/{{TASK_ID}}.json"
   ```

   If status is `blocked` or `failed`: set commit to null, tests_passing to false, and add a "blockers" string field. The orchestrator's Phase 3.1 reads this file with `jq -r '.field // "default"'` so missing fields are handled gracefully.

## Result schema (must be valid JSON)
{
  "task_id": "string",
  "status": "success" | "blocked" | "failed",
  "commit": "string or null",
  "files_changed": ["string", ...],
  "tests_added": ["string", ...],
  "tests_passing": true | false,
  "tests_count": integer,
  "lint_passing": true | false,
  "notes": "string",
  "blockers": "string (only if status != success)"
}

## Hard rules
- Do NOT exceed your scope or touch forbidden paths.
- Do NOT skip tests. If tests fail, status is "failed", not "success".
- If your scope is wrong, return status=blocked. Do not improvise.
- Do NOT push, fork, or open PRs. Just commit locally.
- The result JSON file at {{RESULTS_DIR}}/{{TASK_ID}}.json is the source of truth — write it before returning.
```

### Worked example: dispatching Wave 1 with 3 tasks

Suppose Wave 1 has T01 (migration, haiku), T02 (redis client config, haiku), T03 (rate-limit types, sonnet). The orchestrator's single message contains three Agent calls:

```
Agent({
  description: "T01: rate-limits migration",
  subagent_type: "general-purpose",
  model: "haiku",
  isolation: "worktree",
  prompt: "<filled-in subagent template — see above. {{TASK_ID}}=T01, {{INTEGRATION_POINT_SHA}}=<_BASE_SHA>, {{RESULTS_DIR}}=<_STATE_DIR>/results, {{TOUCHES}}=db/migrations/20260429_rate_limits.sql, {{ACCEPTANCE}}=migration runs forward and back, {{COMMIT_PREFIX}}=chore(T01):  (because _COMMIT_FORMAT=conventional in this repo), ...>"
})
Agent({
  description: "T02: redis client config",
  subagent_type: "general-purpose",
  model: "haiku",
  isolation: "worktree",
  prompt: "<filled-in template for T02>"
})
Agent({
  description: "T03: rate-limit types",
  subagent_type: "general-purpose",
  model: "sonnet",
  isolation: "worktree",
  prompt: "<filled-in template for T03>"
})
```

All three Agent calls go in the SAME orchestrator message so they run in parallel. Do not call them sequentially — that defeats the worktree isolation and burns wall time.

### 2.3 Wait for all returns

The Agent tool returns each subagent's final message text. The orchestrator's source of truth is `$_STATE_DIR/results/$TASK_ID.json`, NOT the agent's text response.

The harness returns each agent's worktree path and branch name in the result. The orchestrator does not need them — cherry-pick uses commit SHA only.

---

## PHASE 3 — VERIFY EACH WAVE

After every wave returns, before the next:

### 3.1 Aggregate (defensive read)

For each task in the wave, read `$_STATE_DIR/results/$TASK_ID.json` defensively:

```bash
for TASK in $(...task ids in wave...); do
  RESULT="$_STATE_DIR/results/$TASK.json"
  if [ ! -f "$RESULT" ]; then
    echo "$TASK: MISSING_RESULT (treating as failed)"; continue
  fi
  STATUS=$(jq -r '.status // "MALFORMED"' "$RESULT" 2>/dev/null || echo "MALFORMED")
  TESTS=$(jq -r '.tests_passing // false' "$RESULT" 2>/dev/null || echo "false")
  COMMIT=$(jq -r '.commit // empty' "$RESULT" 2>/dev/null || echo "")
  REACHABLE="no"
  [ -n "$COMMIT" ] && git cat-file -e "$COMMIT" 2>/dev/null && REACHABLE="yes"
  echo "$TASK: status=$STATUS tests=$TESTS commit=${COMMIT:-none} reachable=$REACHABLE"
done
```

A task counts as `success` ONLY when ALL hold:
- result file exists
- JSON parses (`STATUS != MALFORMED`)
- `status == "success"`
- `tests_passing == true`
- `commit` is non-empty
- `git cat-file -e $COMMIT` succeeds (commit is reachable in shared object DB)

Anything else is `failed`.

### 3.2 Block on any failure

If any task is not `success`: stop. Use `AskUserQuestion`:

- A) Dispatch a fix-up subagent (`Agent` with `isolation: "worktree"`) for that task with the blocker as scope
- B) Re-shred the remaining tasks (regenerate TASKS.md from this wave forward)
- C) Abort

The orchestrator does NOT fix the failure itself.

### 3.3 Integrate the wave onto the working branch

```bash
cd "$(git rev-parse --show-toplevel)"
git checkout "$_BRANCH"
for TASK in $(...task ids in wave, in dependency order from TASKS.md...); do
  COMMIT=$(jq -r '.commit' "$_STATE_DIR/results/$TASK.json")
  git cherry-pick "$COMMIT" || { echo "CONFLICT on $TASK at $COMMIT"; CONFLICT="$TASK"; break; }
done
```

If a cherry-pick conflicts: dispatch one fix-up subagent (`Agent` with `isolation: "worktree"`) with the failing commit and conflict markers as scope. The orchestrator never resolves conflicts itself.

**Branch invariant:** `$_BRANCH` is checked out only in the main repo, never in a subagent worktree. The Agent tool's `isolation: "worktree"` creates fresh per-agent branches automatically — they do not collide with `$_BRANCH`.

### 3.4 Run the full suite once at the wave boundary

```bash
source "<env.sh path>"
$TEST_CMD && $LINT_CMD
```

If it fails: dispatch one fix-up subagent (`Agent` with `isolation: "worktree"`). The subagent commits in its own worktree and writes a result JSON to `$_STATE_DIR/results/wave-N-fixup.json`. After it returns:

```bash
COMMIT=$(jq -r '.commit // empty' "$_STATE_DIR/results/wave-N-fixup.json")
[ -n "$COMMIT" ] && git cherry-pick "$COMMIT"
$TEST_CMD && $LINT_CMD
```

If it still fails after the fix-up cherry-pick, ask the user via `AskUserQuestion`: A) dispatch another fix-up (max 2 retries per wave), B) abort. Do not loop indefinitely.

### 3.5 Checkpoint

```bash
jq -nc --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg wave "N" --arg head "$(git rev-parse HEAD)" \
  '{ts:$ts,wave:$wave,head:$head,status:"completed"}' >> "$_STATE_DIR/state.jsonl"
```

Phase 1.1's resume logic reads this on next invocation.

---

## PHASE 4 — FINAL INTEGRATION & HANDOFF

After the last wave passes:

### 4.1 Run review (bounded loop, max 2 cleanup passes)

Invoke the Skill tool: `Skill({skill: "review"})`. Capture output.

If `/review` flags issues:
1. Dispatch one cleanup subagent (`Agent` with `isolation: "worktree"`) with the review feedback as scope.
2. After it returns, cherry-pick its commit (read from `$_STATE_DIR/results/cleanup-1.json`) onto `$_BRANCH`.
3. Re-invoke `Skill({skill: "review"})`.
4. If issues remain, repeat once more (cleanup-2). After 2 cleanup passes, do NOT loop further. Use `AskUserQuestion`: A) ship anyway with known issues, B) abort (preserve state).

### 4.2 QA / suite

If the plan had UI changes (any wave touched `.tsx`/`.jsx`/`.vue`/`.svelte`/`.html`/`.css`/`.scss`), invoke `Skill({skill: "qa"})`. Otherwise:

```bash
source "<env.sh path>"
$TEST_CMD
```

### 4.3 Worktree cleanup

Prune worktree admin records for any worktrees the Agent harness created and abandoned:

```bash
git worktree prune
```

Note: the harness's auto-managed branches will persist after pruning — this is intentional. Deleting them automatically is unsafe because we cannot reliably distinguish harness-created branches from user-owned branches with the same merge-status. If they accumulate over many runs, the user can clean up manually with `git branch --list` and `git branch -D`.

### 4.4 Final report

```
Done.
Tasks shipped: N/N
Commits: <range>
Files changed: <count>
Tests added: <count>
Review status: clean | <N issues addressed after K cleanup passes>
Deferred items: <list, from notes>
State: <_STATE_DIR>
```

### 4.5 Handoff

Ask via `AskUserQuestion`: "Ready to ship? This will invoke the `/ship` skill which pushes and creates a PR."

If yes, invoke `Skill({skill: "ship"})`. If no, leave state intact and tell the user how to resume or run `/ship` manually later.

---

## ORCHESTRATOR HARD RULES

- You are the orchestrator. You do **not** write feature code, fix code, resolve conflicts, or edit source files.
- The only files you write directly: `TASKS.md`, `state.jsonl`, the final report.
- Never skip the Phase 0 pre-flight or the Phase 1 approval gate.
- Never start Wave N+1 if any task in Wave N is not `success` (status, tests, commit reachable).
- Always dispatch all subagents in a wave in a single message (parallel tool calls).
- Always pass `isolation: "worktree"` to every Agent call.
- If <4 parallel tasks emerge from shredding, abort and recommend `/autoplan`.
- If `/freeze` is active, abort.
- The orchestrator never pushes, forks, or opens PRs. Handoff to `/ship` does that.

---

## START

Run Phase 0. Then Phase 1.1 resume check. If no resume, Phase 1.2-1.4 (shred + approval gate). Do not dispatch until the user picks A.
