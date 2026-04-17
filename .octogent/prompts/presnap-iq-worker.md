# PreSnap IQ Worker — Pre-flight

You are starting a Claude Code session against the `nextdown_oracle` repository. That is the PreSnap IQ codebase.

Before you touch any code, file, or command:

## 1. Read the required docs

Read these four files in this order. Do not skim.

1. `CLAUDE.md` (repo root) — binding rules for this repo, required reading, anti-drift list.
2. `docs/PRD.md` — product source of truth. Read §1 (Thesis), §3 (Scope Fences), §9 (Anti-Drift Rules) at minimum. Read the full doc for any task tagged `[CONNOR-GATE]` or spanning multiple systems.
3. `docs/PLAY_FACTS_SCHEMA.md` — the data contract. Read in full if your task touches plays, games, tendencies, tells, vulnerabilities, the DB, or any Python/TypeScript type involving football data.
4. `docs/ACTION_PLAN.md` — the task queue. Find your task under NOW.

If any of those files are missing, stop immediately and report — do not proceed.

## 2. Locate your task

- Your task must be listed in `docs/ACTION_PLAN.md` under NOW.
- If you cannot find it, stop and ask the human which task you are working on.
- Do not invent a task. Do not combine tasks. Do not expand scope.

## 3. Check the task tag

- `[AUTO]` → you may commit and push to a feature branch without human review. Do not merge to main.
- `[CONNOR-GATE]` → complete the work, then append to `tasks/connor_review_queue.md` with: task ID, what changed, sample output or screenshot, PR link. Mark the task `ready-for-review`. Do not merge. Wait.

## 4. Move the task to IN PROGRESS

Before starting, move your task from the NOW section to the IN PROGRESS section at the top of `docs/ACTION_PLAN.md`. Note your session start time.

## 5. Stop-and-ask triggers

During work, stop and ask the human if any of these happen:

- Your task requires a product decision the PRD does not answer.
- Your task would cross a Scope Fence listed in `docs/PRD.md §3`.
- Your task needs a schema change — propose the change against `docs/PLAY_FACTS_SCHEMA.md` first, then wait for approval before building.
- You find code in the repo that contradicts the docs in a way that suggests the docs might be wrong (not just the repo being stale).
- The task appears larger than the `scope:` field on it indicates.

Ask questions in small rounds. One topic at a time. Do not produce overwhelming question dumps.

## 6. Anti-drift — binding

Full list in `CLAUDE.md` and `docs/PRD.md §9`. These are the ones most easily violated in a code change:

- Never expose the term "SVS" in coach-facing strings. Plain English only.
- Never preamble AutoScout responses with "Coach,". Known prior bug.
- Never display Unknown formations to coaches. Route to internal review queue.
- Never use internal enum codes (`FORM_TRIPS_R`) in UI strings. Translate via presentation layer.
- Never claim micro-tells are Core. Beta label until Connor-approved graduation.
- Never treat GNN retraining, homography, or pose estimation as v1 critical path.
- Never build features assuming Hudl goes away. We coexist.

## 7. When done

- `[AUTO]`: commit, push to feature branch, open PR. Append completion entry to `tasks/completed.md`.
- `[CONNOR-GATE]`: append to `tasks/connor_review_queue.md` as described in §3. Do not merge.
- If you learned something that should inform future agents, append to `tasks/lessons.md`.

## 8. Reporting back

When you finish or when you stop for a question, report back with:

- What you did (specific files changed, tests added)
- What tests pass
- What remains
- Any questions or concerns

Do not produce long narratives. Terse and specific beats verbose and vague.

---

**Now read `CLAUDE.md` and `docs/ACTION_PLAN.md`, identify your task, and report back with what you're about to do before doing it.**
