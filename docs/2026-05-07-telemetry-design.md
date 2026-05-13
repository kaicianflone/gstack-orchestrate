# parallel-orchestrate telemetry & timeline integration

**Date:** 2026-05-07
**Skill:** `parallel-orchestrate` (target version: 1.6.0)
**Status:** approved design, ready for plan

---

## Goal

Wire `/parallel-orchestrate` into the existing gstack telemetry/timeline infrastructure so its performance can be tracked alongside `/ship`, `/review`, `/qa`. Update the README to reflect that the skill is now in regular, no-issue use.

---

## Why

The README explicitly flags this as a known gap:

> "**No telemetry/learnings hooks.** Sibling gstack skills (`/ship`, `/review`, `/qa`) log to `~/.gstack/analytics/` and `~/.gstack/projects/$SLUG/learnings.jsonl`. This skill doesn't yet."

The user is now using parallel-orchestrate consistently with no issues. Without telemetry hooks, the orchestrator is invisible to the existing analytics dashboards (`gstack-analytics`), security audits, and timeline read-back tools. Adding the standard preamble/epilogue plus per-wave timeline events brings it to parity with siblings AND yields the most operationally useful signal: which waves are slow and how often fix-ups fire.

---

## Scope

In:
1. Add the standard sibling-skill telemetry preamble (matches `/ship`, `/review`, `/qa`).
2. Add per-wave timeline events (`wave_dispatched`, `wave_completed`).
3. Enrich the existing Phase 3.5 `state.jsonl` checkpoint with `duration_s` and `task_count` (additive, backward-compatible with resume).
4. Add the standard sibling-skill telemetry epilogue.
5. Update `README.md` to remove WIP/known-issues caveats that no longer apply, document where telemetry lands, and bump status to "stable, in regular use."
6. Bump `version: 1.5.1` → `version: 1.6.0` in SKILL.md frontmatter.

Out:
- Per-task analytics records (already captured in `$_STATE_DIR/results/*.json`).
- Changes to subagent prompts.
- Any change to the shred/dispatch/verify/integrate phase logic itself.
- Adding a `gstack-learnings-log` integration. (Operational learnings are a fit, but not load-bearing for "performance tracking" — defer to a follow-up if useful.)

---

## Architecture

### Data flow

```
PRE-FLIGHT (existing Phase 0)
   │
   ├─▶ NEW: telemetry preamble — sets _TEL, _TEL_START, _SESSION_ID
   │       writes .pending-$_SESSION_ID
   │       fires gstack-timeline-log {event:"started"} (bg)
   │       (vars persisted to env.sh in 0.6 alongside existing ones)
   ▼
PHASE 1 (shred + approval) — unchanged
   │
   ▼
PHASE 2 — DISPATCH (per wave)
   │
   ├─▶ NEW: stamp _WAVE_START at wave entry
   ├─▶ existing 2.1 integration-point detection
   ├─▶ NEW: gstack-timeline-log {event:"wave_dispatched", wave:N, tasks:M} (bg)
   ├─▶ existing 2.2 single-message Agent dispatch
   ▼
PHASE 3 — VERIFY EACH WAVE
   │
   ├─▶ existing 3.1–3.4 aggregate / block-on-fail / integrate / wave-suite
   ├─▶ MODIFIED 3.5 checkpoint:
   │       state.jsonl entry now includes duration_s + task_count
   │       AND emits gstack-timeline-log {event:"wave_completed", wave:N, success:X, failed:Y, fixups:Z, duration_s:D} (bg)
   ▼
PHASE 4 — FINAL INTEGRATION & HANDOFF
   │
   ├─▶ existing 4.1–4.4
   ├─▶ NEW: telemetry epilogue (BEFORE 4.5 handoff)
   │       computes _TEL_DUR
   │       removes .pending-$_SESSION_ID
   │       fires gstack-timeline-log {event:"completed", outcome, duration_s} (bg)
   │       appends to skill-usage.jsonl (gated on _TEL != off)
   │       fires gstack-telemetry-log (bg, gated on _TEL != off)
   ├─▶ existing 4.5 handoff to /ship
```

### env.sh additions

Phase 0.6 currently writes:
```
_REPO_ROOT _BASE _BASE_SHA _BRANCH _BRANCH_SAFE _STATE_DIR SLUG STACK TEST_CMD LINT_CMD _COMMIT_FORMAT
```

Add three exports so the epilogue (which runs in a fresh shell after Phase 4) can read them:
```
_TEL _TEL_START _SESSION_ID
```

### state.jsonl schema delta

Current entry shape (Phase 3.5):
```json
{"ts":"2026-05-07T10:00:00Z","wave":"1","head":"<sha>","status":"completed"}
```

New shape (additive — Phase 1.1 resume logic only reads `head`, `wave`, `status`, so older entries remain compatible):
```json
{"ts":"2026-05-07T10:00:00Z","wave":"1","head":"<sha>","status":"completed","duration_s":312,"task_count":4,"success":4,"failed":0,"fixups":0}
```

### Outcome mapping (epilogue)

The epilogue fires from one of these terminal states:
| Phase reached | OUTCOME |
|---|---|
| Phase 4 cleared, handoff offered | `success` |
| User picked Abort at a user-driven gate (1.1-C, 1.4-D, 3.2-C) | `abort` |
| Fix-up retries exhausted (3.4-B after a second failure, or 4.1-B after 2 cleanup passes) | `error` |
| Phase 0 pre-flight failure (no repo, on base branch, dirty tree, freeze active, missing tooling) | `error` |

Implementation: a single shell variable `_OUTCOME=success` is initialized at preamble entry. Each abort/error gate sets it before triggering the epilogue. The epilogue reads `$_OUTCOME` and passes it to both `gstack-timeline-log` and `gstack-telemetry-log`.

For the abort cases that exit mid-skill (Phase 1.1-C, 1.4-D, 3.2-C, 3.4-B, 4.1-B), the epilogue must still run before the skill returns control. Add a small "emit telemetry then stop" subroutine and call it from each abort branch instead of the current bare "stop" instruction.

### Telemetry sites — exact insertion points in SKILL.md

| Site | Where | What |
|---|---|---|
| Preamble | Between frontmatter and `## PHASE 0 — PRE-FLIGHT` | New `## TELEMETRY PREAMBLE` section, ~25 lines mirroring `/review` lines 50-74 + 86 |
| env.sh | Phase 0.6 heredoc | Add three more `export` lines |
| Wave start stamp | Phase 2.1, top of the source-and-detect bash block | `_WAVE_START=$(date +%s)` |
| wave_dispatched | End of Phase 2.1 (before 2.2 dispatch) | One bash line: `gstack-timeline-log '{...}' &` |
| wave_completed + state.jsonl | Phase 3.5 (replace the existing `jq` line) | Enriched `jq -nc` plus a `gstack-timeline-log` call |
| Outcome var | Preamble (init) + each abort/error branch (set before stop) | `_OUTCOME=...` |
| Epilogue | New `## TELEMETRY EPILOGUE` section between Phase 4.4 and 4.5 | ~15 lines mirroring `/review` lines 700-714 |

---

## Failure modes & resilience

- **Telemetry off:** all three loggers (`gstack-telemetry-log`, `gstack-timeline-log`, `gstack-learnings-log`) already no-op or write only local fields when `telemetry: off`. No skill behavior changes.
- **`bun` not installed:** `gstack-timeline-log` validates input via `bun -e` and silently skips on validation failure. No skill behavior changes.
- **`gstack-config` missing:** preamble defaults `_TEL` to `off` via `|| true`. No skill behavior changes.
- **Background loggers slow:** all logger calls use `&` so they cannot block the orchestrator. Already the convention in sibling skills.
- **Abort path skips epilogue:** mitigated by the "emit telemetry then stop" subroutine wired into every abort branch.
- **Resume across telemetry boundaries:** resume from a prior wave does not re-fire its `wave_completed` event (idempotency preserved by Phase 1.1's head-match validation — resumed wave starts at N+1).

---

## README changes (concrete)

1. Header `## Status: WIP. v1.5...` → `## Status: stable, in regular use. v1.6.`
2. "What's working" table — add row: `Telemetry + timeline events (skill_run + per-wave) | ✓`
3. "What's NOT working yet" — remove these two items:
   - "Never tested on a real plan."
   - "No telemetry/learnings hooks."
4. New section after "Architecture in 5 bullets":
   ```markdown
   ## Where the data lives

   - Skill-level analytics: `~/.gstack/analytics/skill-usage.jsonl`
   - Per-wave timeline: `~/.gstack/projects/<slug>/timeline.jsonl`
   - Per-task results (already): `~/.gstack/projects/<slug>/orchestrate/<branch>/results/*.json`
   - Resume state with per-wave duration: `~/.gstack/projects/<slug>/orchestrate/<branch>/state.jsonl`

   Query with `gstack-analytics` (skill-usage), `gstack-timeline-read` (timeline), or `jq` directly.
   ```

---

## Testing

Manual verification (the skill has no unit tests today; mirror that):

1. **Off-mode:** run with `telemetry: off`. Confirm `~/.gstack/analytics/skill-usage.jsonl` does not gain a row, but `timeline.jsonl` still does (timeline is local-only and runs regardless of telemetry tier).
2. **Anonymous-mode:** run with `telemetry: anonymous`. Confirm `skill-usage.jsonl` gains both a start row and an end row, no `installation_id`.
3. **Per-wave events:** confirm one `wave_dispatched` and one `wave_completed` per wave land in `timeline.jsonl`, with `duration_s` populated and matching the diff between events.
4. **Abort path:** run, abort at the Phase 1.4 approval gate. Confirm `skill-usage.jsonl` end row has `outcome: abort`.
5. **Resume path:** complete wave 1, kill the skill, re-run, pick "Resume." Confirm wave 1 does not get a duplicate `wave_completed` event.

Document these as a manual smoke checklist in the implementation plan rather than as automated tests.

---

## Migration / backward compatibility

- Existing `state.jsonl` files: older entries lack `duration_s` and `task_count`. Resume logic at Phase 1.1 reads only `head`, `wave`, and `status` (already), so older entries continue to work.
- Existing in-flight runs at upgrade time: there are none (the skill runs to completion in a single session). No migration needed.

---

## Open questions

None at this point. The user approved the recommended scope ("Skill-level + per-wave events").
