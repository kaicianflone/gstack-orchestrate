# Implementation Plan: gstack-orchestrate telemetry & timeline integration

**Date:** 2026-05-07
**Spec:** [`2026-05-07-telemetry-design.md`](./2026-05-07-telemetry-design.md)
**Eng review:** completed 2026-05-07 (all clear, 3 review-driven refinements)
**Target version:** SKILL.md frontmatter `1.5.1` → `1.6.0`

---

## Conventions

- **Repo:** `~/.claude/skills/gstack-orchestrate` (independent git repo, currently on `main`)
- **Working branch:** `feat/telemetry-hooks`
- **Files modified:** 2 (`SKILL.md`, `README.md`)
- **Execution mode:** sequential, direct edits. `/gstack-orchestrate` is incompatible by its own design (<4 parallel tasks → falls under its "use /autoplan instead" rule).
- **Smoke verification:** manual checklist at the end. No automated tests (sibling-skill convention).

---

## Task list (sequential)

### T1 — Add telemetry preamble section

**File:** `SKILL.md`
**Where:** Insert a new `## TELEMETRY PREAMBLE` section between the frontmatter (ends ~line 28) and `## PHASE 0 — PRE-FLIGHT` (starts ~line 54).

**Content:** mirror `/review/SKILL.md:50-86` substituting `gstack-orchestrate` for `review`. Keep these elements:
- Read `_TEL` from `gstack-config get telemetry`
- Stamp `_TEL_START=$(date +%s)`
- Compute `_SESSION_ID="$$-$(date +%s)"`
- Initialize `_OUTCOME=success` (default — abort/error gates override)
- Write start row to `~/.gstack/analytics/skill-usage.jsonl` (gated on `_TEL != off`)
- Finalize stale `.pending-*` markers (sibling pattern lines 66-74)
- Background-fire `gstack-timeline-log {event:"started"}` (sibling pattern line 86)

### T2 — Persist telemetry vars in env.sh

**File:** `SKILL.md`
**Where:** Phase 0.6 heredoc (currently lines ~157-174).

**Edit:** add three exports inside the heredoc:
```bash
export _TEL="$_TEL"
export _TEL_START="$_TEL_START"
export _SESSION_ID="$_SESSION_ID"
export _OUTCOME="$_OUTCOME"
```

(Four total — `_OUTCOME` needs to persist too so abort/success branches can update it from later Bash calls.)

### T3 — Stamp wave start time at Phase 2.1

**File:** `SKILL.md`
**Where:** Phase 2.1 bash block (currently around lines 345-353).

**Edit:** add `_WAVE_START=$(date +%s)` as the first line after `source "<env.sh>"`. Persist it to env.sh too OR re-stamp at Phase 3.5 read-time. Recommendation: **re-stamp at Phase 3.5 by reading from env.sh-style ad-hoc file** — write `_WAVE_START` to `$_STATE_DIR/wave-N.start` so it survives shell calls.

Concrete approach:
```bash
echo "$(date +%s)" > "$_STATE_DIR/wave-$_WAVE_NUM.start"
```
Phase 3.5 reads it back.

### T4 — Emit `wave_dispatched` event after Phase 2.1

**File:** `SKILL.md`
**Where:** End of Phase 2.1, before Phase 2.2's "Dispatch all subagents" header.

**Edit:** add a single backgrounded bash line:
```bash
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"gstack-orchestrate","event":"wave_dispatched","branch":"'"$_BRANCH"'","wave":"'"$_WAVE_NUM"'","tasks":"'"$_WAVE_TASK_COUNT"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

`_WAVE_NUM` and `_WAVE_TASK_COUNT` come from the orchestrator's wave-iteration logic.

### T5 — Enrich Phase 3.5 checkpoint with duration + counts AND emit `wave_completed` event

**File:** `SKILL.md`
**Where:** Phase 3.5 (currently lines 569-574). Replace the existing `jq -nc` line.

**Edit:**
```bash
_WAVE_START=$(cat "$_STATE_DIR/wave-$_WAVE_NUM.start" 2>/dev/null || echo "$(date +%s)")
_WAVE_END=$(date +%s)
_WAVE_DUR=$(( _WAVE_END - _WAVE_START ))

jq -nc \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --arg wave "$_WAVE_NUM" \
  --arg head "$(git rev-parse HEAD)" \
  --argjson duration "$_WAVE_DUR" \
  --argjson tasks "$_WAVE_TASK_COUNT" \
  --argjson success "$_WAVE_SUCCESS_COUNT" \
  --argjson failed "$_WAVE_FAILED_COUNT" \
  --argjson fixups "$_WAVE_FIXUP_COUNT" \
  '{ts:$ts,wave:$wave,head:$head,status:"completed",duration_s:$duration,task_count:$tasks,success:$success,failed:$failed,fixups:$fixups}' \
  >> "$_STATE_DIR/state.jsonl"

~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"gstack-orchestrate","event":"wave_completed","branch":"'"$_BRANCH"'","wave":"'"$_WAVE_NUM"'","duration_s":"'"$_WAVE_DUR"'","success":"'"$_WAVE_SUCCESS_COUNT"'","failed":"'"$_WAVE_FAILED_COUNT"'","fixups":"'"$_WAVE_FIXUP_COUNT"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

(The aggregate counts `_WAVE_SUCCESS_COUNT`, `_WAVE_FAILED_COUNT`, `_WAVE_FIXUP_COUNT` already exist conceptually in Phase 3.1's aggregate loop — surface them as variables before reaching 3.5.)

### T6 — Add telemetry epilogue section

**File:** `SKILL.md`
**Where:** Insert a new `## TELEMETRY EPILOGUE` section between Phase 4.4 ("Final report", currently ends ~line 624) and Phase 4.5 ("Handoff", currently starts ~line 626).

**Content:** mirror `/review/SKILL.md:700-714` substituting `gstack-orchestrate` for `SKILL_NAME` and reading `$_OUTCOME` instead of hardcoding `OUTCOME`.

```bash
source "<env.sh>"
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"gstack-orchestrate","event":"completed","branch":"'$(git branch --show-current 2>/dev/null || echo unknown)'","outcome":"'"$_OUTCOME"'","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"gstack-orchestrate","duration_s":"'"$_TEL_DUR"'","outcome":"'"$_OUTCOME"'","browse":"false","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
if [ "$_TEL" != "off" ] && [ -x ~/.claude/skills/gstack/bin/gstack-telemetry-log ]; then
  ~/.claude/skills/gstack/bin/gstack-telemetry-log \
    --skill "gstack-orchestrate" --duration "$_TEL_DUR" --outcome "$_OUTCOME" \
    --used-browse "false" --session-id "$_SESSION_ID" 2>/dev/null &
fi
```

### T7 — Wire `_OUTCOME` updates at the 5 abort gates + 1 success gate (D2 from review)

**File:** `SKILL.md`

For each gate, before the orchestrator stops/exits, write `_OUTCOME=abort` (or `error`) into env.sh so the epilogue picks it up. Concrete sed-style approach:

```bash
sed -i.bak 's/^export _OUTCOME=.*/export _OUTCOME="abort"/' "$_STATE_DIR/env.sh" && rm -f "$_STATE_DIR/env.sh.bak"
```

**Six concrete edit sites** (line ranges from current SKILL.md):

| # | Phase | Current line range | Reason | New `_OUTCOME` value |
|---|-------|-------------------|--------|---------------------|
| 1 | 1.1 option C "Abort" | ~217-226 | resume aborted | `abort` |
| 2 | 1.4 option D "Abort" | ~326-331 | shred plan rejected | `abort` |
| 3 | 3.2 option C "Abort" | ~528-535 | wave failure not retryable | `abort` |
| 4 | 3.4 option B "abort" | ~567 | suite failing after fix-up retries | `error` |
| 5 | 4.1 option B "abort (preserve state)" | ~591-592 | review issues not addressable | `error` |
| 6 | 4.5 handoff (Skill stays) | ~626-630 | implicit success, no edit needed | `success` (already default from preamble) |

Sites 4 and 5 are `error` (unrecoverable), sites 1-3 are `abort` (user-driven). Site 6 inherits the preamble's default `success`.

Each abort branch also calls the epilogue subroutine (T6) before stopping. Concrete pattern at each site: append the epilogue inline OR wrap the epilogue bash into a re-usable section the abort branches reference.

**Implementation choice:** keep the epilogue inline at Phase 4 AND copy-paste-modify it (with `_OUTCOME=abort/error`) into each abort branch. The duplication is intentional and matches sibling-skill style — sibling skills also inline their epilogue. A shared helper would be the first such pattern in any gstack skill.

### T8 — Bump SKILL.md frontmatter version

**File:** `SKILL.md`
**Where:** Line 3 (`version: 1.5.1`)

**Edit:** `version: 1.5.1` → `version: 1.6.0`

### T9 — Rewrite README.md

**File:** `README.md`

**Edits:**
1. Status line (line 3): `> **Status:** WIP. v1.5. Runs end-to-end on a synthetic repo. Not yet exercised on a real project plan.` → `> **Status:** Stable, in regular use. v1.6.`
2. "What's working" table — add row: `| Telemetry + per-wave timeline events | ✓ |`
3. "What's NOT working yet" — remove these two bullets:
   - "Never tested on a real plan."
   - "No telemetry/learnings hooks."
4. Insert a new section after "Architecture in 5 bullets":
   ```markdown
   ## Where the data lives

   - **Skill-level analytics:** `~/.gstack/analytics/skill-usage.jsonl` — one start row + one completion row per orchestrate run, with `outcome` (success/abort/error) and `duration_s`.
   - **Per-wave timeline:** `~/.gstack/projects/<slug>/timeline.jsonl` — `wave_dispatched` (wave + task count) and `wave_completed` (duration + success/failed/fixup counts) events per wave, plus the standard `started` / `completed` skill events.
   - **Per-task results (already there):** `~/.gstack/projects/<slug>/orchestrate/<branch>/results/*.json` — what each subagent did.
   - **Resume state with per-wave duration:** `~/.gstack/projects/<slug>/orchestrate/<branch>/state.jsonl` — head, wave, duration, counts. Older entries (pre-1.6) lack the count/duration fields; resume logic only reads `head`/`wave`/`status` so it stays compatible.

   Query with `gstack-analytics` (skill-usage), `gstack-timeline-read` (timeline), or `jq` directly.
   ```
5. Drop the closing "Try it on a real plan next" section since the skill is now in regular use.

---

## Smoke checklist (post-edit, manual)

Run on a real parallelizable plan in a real project. Tick each:

1. **Off-mode:** `gstack-config set telemetry off`, run orchestrate end-to-end. Confirm:
   - `~/.gstack/analytics/skill-usage.jsonl` does NOT gain a row
   - `~/.gstack/projects/<slug>/timeline.jsonl` DOES gain rows (timeline is local-only, runs regardless of tier)
2. **Anonymous-mode:** `gstack-config set telemetry anonymous`, run again. Confirm `skill-usage.jsonl` gains start + end rows; no `installation_id` set.
3. **Community-mode + per-wave events:** `gstack-config set telemetry community`, run a 2-wave plan. Confirm `timeline.jsonl` has exactly:
   - 1 × `started`
   - 2 × `wave_dispatched` (one per wave, with correct `tasks` count)
   - 2 × `wave_completed` (with `duration_s` matching the diff between paired events)
   - 1 × `completed` (with `outcome:success`)
4. **Abort path:** start orchestrate, abort at Phase 1.4 approval gate (option D). Confirm:
   - `skill-usage.jsonl` end row has `outcome:abort`
   - timeline `completed` event has `outcome:abort`
5. **Resume path:** complete wave 1, kill the skill, re-run, pick "Resume." Confirm wave 1 does NOT get a duplicate `wave_completed` event (head-match validation works).
6. **`_OUTCOME=error` (D3 from review):** create a wave whose only task has a deliberate, persistent failure (e.g., a syntax error subagents can't fix). Run with the orchestrator allowed to attempt 2 fix-ups. After both retries fail, pick option B (abort). Confirm:
   - `skill-usage.jsonl` end row has `outcome:error`
   - timeline `completed` event has `outcome:error`

A learning-log entry is justified IF any scenario reveals a structural bug worth remembering, not for routine first-time misses.

---

## Out of scope (per spec)

- Operational learnings hook (deferred to TODOS.md, P3).
- Per-task analytics records.
- Refactoring sibling-skill preamble duplication.
- Automated tests for telemetry hooks.

---

## Rollback

Single-commit revert. Resume state.jsonl files written by 1.6 in mid-flight runs are forward-compatible (extra fields ignored by 1.5 resume logic). No migration script needed.
