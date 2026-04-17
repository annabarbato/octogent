# PreSnap IQ Doc Integration Plan for Octogent

**Status:** Plan only. No code or config changes in this commit.
**Author:** Claude Code audit, branch `claude/audit-presnap-iq-docs-UHmiG`.
**Goal:** Every octogent-dispatched agent that works on the `nextdown_oracle`
(PreSnap IQ) repo must read `PRD.md` §1 and §3 before starting, consult
`PLAY_FACTS_SCHEMA.md` when touching data shape, and check `ACTION_PLAN.md` for
the task queue.

This document is repo-truthful to octogent as it exists today. It does not
modify octogent core unless required. Citations are file path + line number so
you can verify.

---

## 1. Octogent's Agent-Dispatch Architecture (what exists today)

Octogent launches Claude Code sessions as shell-wrapped PTY processes. It does
**not** use Claude Code's system-prompt flags, hooks, or settings.json. It
injects the initial prompt as **plain text written into the PTY's stdin** after
Claude Code boots. Relevant files:

- `apps/api/src/terminalRuntime/sessionRuntime.ts:66-86` — spawns PTY via
  `node-pty`, cwd resolved per tentacle (worktree or shared), env inherits
  parent + `TERM`, `COLORTERM`, `OCTOGENT_SESSION_ID`.
- `apps/api/src/terminalRuntime/sessionRuntime.ts:318-363` — bootstrap sequence:
  4-second delay → write `claude\r` → 4-second delay → paste `initialPrompt`
  wrapped in bracketed-paste escape codes → send `\r`. **That is the entire
  context delivery mechanism.** Claude Code sees the prompt as if a human pasted
  it.
- `apps/api/src/createApiServer/terminalRoutes.ts:63-258` — the
  `POST /api/terminals` handler. Callers can supply:
  - `promptTemplate` (name under `prompts/`) + `promptVariables` (substitution
    map), **or**
  - raw `initialPrompt` text, **or**
  - nothing, in which case a `tentacleId` triggers the default
    `tentacle-context-init` template (`terminalRoutes.ts:24-43`, `231-241`).
- `apps/api/src/prompts/promptResolver.ts:8-9` — template interpolation is
  `{{key}}` regex substitution. Unknown keys pass through verbatim.
- `apps/api/src/prompts/promptResolver.ts:98-112` — user prompts in
  `.octogent/prompts/` (the workspace) shadow built-ins in the repo's
  `prompts/` directory. This is the cleanest built-in extension point.
- `prompts/tentacle-context-init.md:1` — default bootstrap is a single line
  telling the agent to consult `{{tentacleContextPath}}` (which resolves to
  `.octogent/tentacles/<tentacleId>`). **Content is not inlined; only a path
  reference is given.**
- `prompts/swarm-worker.md:17-19` — swarm workers are told to read `CONTEXT.md`
  and any other `.md` files in the tentacle folder before writing code.
- `apps/api/src/terminalRuntime/worktreeManager.ts:68-85` — worktree mode
  creates `.octogent/worktrees/<tentacleId>/` on branch
  `octogent/<tentacleId>`. It does **not** copy extra files into the worktree;
  the worktree is just a git checkout of whatever repo octogent is running in.
- `apps/api/src/projectPersistence.ts:14-23` — `.octogent/project.json` exists
  but carries only `{version, projectId, displayName, createdAt}`. There is no
  "always-load instructions" field. Global `~/.octogent/` only holds a project
  registry.

Implication for our use case: **Claude Code starts in the target repo's
working directory. That repo's own `CLAUDE.md` (at its root) is auto-loaded by
Claude Code natively — that's a Claude Code behavior, not an octogent
behavior.** This is the most reliable delivery channel we already have.

---

## 2. Where the Three Docs Should Live

**Recommendation: the canonical copies live in `nextdown_oracle`, not in
octogent.**

Reasons:
1. The docs govern nextdown_oracle code. They should version with that code.
2. Contributors who work on nextdown_oracle without octogent (direct edits,
   other tools, CI) still need to see them.
3. In both of octogent's workspace modes, Claude Code's cwd is a checkout of
   nextdown_oracle. If the docs are in that repo, they're already visible in
   the cwd — no path gymnastics.
4. Storing them in octogent's tentacle folder (`.octogent/tentacles/<id>/*.md`)
   works for octogent-launched agents, but creates a second source of truth
   that will drift.

Proposed path inside `nextdown_oracle`:

```
nextdown_oracle/
  docs/
    presnap-iq/
      PRD.md
      PLAY_FACTS_SCHEMA.md
      ACTION_PLAN.md
```

(Using `docs/presnap-iq/` rather than repo root keeps these grouped and
discoverable alongside any existing docs.)

**Octogent does not store these docs.** Octogent only points at them.

---

## 3. How to Make Every Agent Read Them

Three coordinated channels. Each reinforces the others.

### Channel A — `nextdown_oracle/CLAUDE.md` (primary, auto-loaded)

Claude Code auto-loads `CLAUDE.md` from the cwd when a session starts. This
happens regardless of whether octogent launched the session. Add a "Required
reading" section to `nextdown_oracle/CLAUDE.md` that says, in so many words:

> Before any work on this repo, read `docs/presnap-iq/PRD.md` §1 (Thesis) and
> §3 (Scope Fences). For data-shape changes, consult
> `docs/presnap-iq/PLAY_FACTS_SCHEMA.md`. For task assignment, check
> `docs/presnap-iq/ACTION_PLAN.md` before starting work.
>
> If the task contradicts these docs, stop and surface the contradiction — do
> not quietly drift.

This gets you ~80% coverage with zero octogent changes. Any tool that respects
CLAUDE.md (Claude Code natively; most agent harnesses) picks it up.

### Channel B — Custom octogent prompt templates (reinforces Channel A)

In the octogent workspace where PreSnap IQ work happens, add two user prompt
templates that shadow the built-ins:

```
.octogent/prompts/
  presnap-iq-worker.md          # shadow of swarm-worker.md, or new name
  presnap-iq-parent.md          # shadow of swarm-parent.md, or new name
```

These wrap (or replace) the swarm templates and add an explicit pre-flight
block at the top:

> Before reading the tentacle context or writing any code:
>
> 1. Read `docs/presnap-iq/PRD.md` §1 and §3 in full. If your task appears to
>    cross a Scope Fence from §3, stop and ask.
> 2. If your task touches Play, Game, Tendency, MacroTell, MicroTell,
>    Vulnerability, or any object defined in `PLAY_FACTS_SCHEMA.md`, read that
>    file before editing code or database schema.
> 3. Confirm your task exists under a `NOW` heading in
>    `docs/presnap-iq/ACTION_PLAN.md`. Respect the `[AUTO]` vs `[CONNOR-GATE]`
>    tag — `[CONNOR-GATE]` tasks must append to `tasks/connor_review_queue.md`
>    and wait, not merge.
>
> Do these three steps in order. Then continue with the assignment below.

The original swarm template content continues below the preamble via a normal
`{{todoItemText}}`-style body.

Keeping these as **user templates in `.octogent/prompts/` — not edits to the
built-in `prompts/*.md`** — means:
- No fork of octogent.
- The shadowing behavior in `promptResolver.ts:98-112` means user templates
  with the same filename as a built-in win automatically.
- Naming them differently (`presnap-iq-worker.md`) and selecting them by name
  in the caller is cleaner — it avoids silently overriding the built-in for
  non-PreSnap-IQ work that happens to share the same octogent install.
  **Preferred.**

### Channel C — Tentacle `CONTEXT.md` (reinforces Channel A+B)

Whatever tentacles you create for PreSnap IQ work (e.g.
`.octogent/tentacles/engine-vlm/`, `.octogent/tentacles/web-surfaces/`) should
have `CONTEXT.md` files whose top section says:

> This tentacle works on the nextdown_oracle repo (PreSnap IQ). Before doing
> anything, read the source-of-truth docs at
> `docs/presnap-iq/{PRD.md, PLAY_FACTS_SCHEMA.md, ACTION_PLAN.md}` in the
> target repo. These supersede any older guidance.

Since swarm workers are already instructed to read tentacle `CONTEXT.md`
(`prompts/swarm-worker.md:17-19`), this is automatic reinforcement. The
tentacle `CONTEXT.md` also carries the octogent-side orchestration rules
(Connor-gate flow, review-queue file location) that don't belong in the target
repo.

---

## 4. Concrete Changes Required

### 4a. In `nextdown_oracle` (not this repo — flag for execution)
- **New files:** `docs/presnap-iq/PRD.md`, `docs/presnap-iq/PLAY_FACTS_SCHEMA.md`,
  `docs/presnap-iq/ACTION_PLAN.md` — copies of the three attached source docs.
- **Edit:** `nextdown_oracle/CLAUDE.md` — add a "Required reading" section
  pointing at the three docs (or create the file if it doesn't exist).
- **Optional:** `nextdown_oracle/tasks/connor_review_queue.md` scaffold, since
  several `[CONNOR-GATE]` tasks require that file to exist.

### 4b. In the octogent workspace used for PreSnap IQ work
- **New files:** `.octogent/prompts/presnap-iq-worker.md` and
  `.octogent/prompts/presnap-iq-parent.md` (content sketched in §3 Channel B).
- **New tentacles (as created):** each gets a `CONTEXT.md` with the pointer
  block from §3 Channel C.
- **Caller convention:** when creating a terminal via `POST /api/terminals` or
  the UI for PreSnap IQ work, use `promptTemplate: "presnap-iq-worker"` (or
  `"presnap-iq-parent"` for coordinator terminals) instead of the default
  `swarm-worker` / `swarm-parent`. This is a per-spawn choice, not a global
  setting — see §5 for why that's a limitation.

### 4c. In octogent core (**not recommended for v1 — only if needed**)
Octogent has no per-tentacle or per-project "default prompt template" setting.
If you want agents to *automatically* pick `presnap-iq-worker` whenever they
spawn under a PreSnap IQ tentacle (rather than the caller remembering to pass
it), that requires a small core change:

- Add an optional field to the tentacle config (e.g.
  `.octogent/tentacles/<id>/config.json` with `defaultPromptTemplate: string`)
  or to the deck tentacle record.
- In `terminalRoutes.ts:231-241` — the branch that currently falls through to
  `tentacle-context-init` when no explicit prompt is given — check for a
  tentacle-level default before falling back.

**Defer this.** Start with the caller-convention approach in §4b. If pilot
usage shows operators consistently forget to pass the template, then promote
the tentacle-default field to octogent core. The three channels in §3 already
give defense-in-depth without it.

---

## 5. Where These Changes Belong (core vs project-specific)

| Change | Location | Type |
|---|---|---|
| The three doc files | `nextdown_oracle/docs/presnap-iq/` | Target-repo, project-specific |
| `CLAUDE.md` required-reading section | `nextdown_oracle/CLAUDE.md` | Target-repo, project-specific |
| Custom prompt templates | `.octogent/prompts/*.md` (user dir) | Octogent workspace, project-specific |
| Tentacle `CONTEXT.md` pointers | `.octogent/tentacles/<id>/CONTEXT.md` | Octogent workspace, project-specific |
| Tentacle default-prompt-template field | `apps/api/src/createApiServer/terminalRoutes.ts` + deck store | Octogent core, only if §4c is greenlit |

**Net: zero octogent core changes required for the v1 integration.** Everything
lives either in the target repo or in the user-extension directories octogent
already supports. The optional core change is a small enhancement worth
considering only if operator behavior proves unreliable.

---

## 6. Assumptions and Open Questions

These need your answer before execution:

1. **Where does octogent run relative to nextdown_oracle?** Is nextdown_oracle
   itself the octogent "project" (i.e. you run octogent inside the
   nextdown_oracle checkout, and `.octogent/` lives at
   `nextdown_oracle/.octogent/`), or is octogent a separate workspace that
   spawns worktrees pointing at a nextdown_oracle checkout elsewhere? **This
   matters because in case (a), the `.octogent/prompts/` and
   `nextdown_oracle/docs/presnap-iq/` paths coexist in one tree. In case (b),
   custom prompts live in the octogent workspace while docs live in a
   different repo the worktrees check out.** My plan assumes case (a) (same
   tree). If (b), the plan still works but path conventions shift slightly.

2. **Is `CLAUDE.md` already present at `nextdown_oracle/` root?** If yes, we
   add a section. If no, we create the file. Your earlier audit request
   implied one exists but I cannot verify without repo access.

3. **Do you want the custom prompt to require reading the three docs
   *literally every time*, or only the first time in a session?** Claude Code
   sessions are single-use (no cross-session memory in the PTY), so "every
   time" is effectively automatic — but if you'd rather the prompt be short
   and rely on `CLAUDE.md` alone, say so and I'll slim Channel B to a
   one-liner.

4. **Should `[CONNOR-GATE]` enforcement live in the prompt or as an octogent
   feature?** The prompt can *instruct* agents to stop, but nothing in
   octogent prevents a misbehaving agent from pushing. If you want a hard
   gate, that's a separate design question — hooks on commit/push, CODEOWNERS,
   branch protection — and does not belong in this integration plan.

5. **Swarm-parent template treatment.** The parent coordinator spawns workers
   and needs the same reading. The simplest approach is a sibling
   `presnap-iq-parent.md` template. Confirm you want parent coverage, or if
   only workers need it.

6. **Prompt length budget.** `initialPrompt` is pasted into the PTY verbatim.
   Very long prompts cause terminal lag and clutter the user's first-paint
   view. My plan keeps the preamble under ~15 lines and references file paths
   rather than inlining doc content. Confirm that's acceptable, or say if you
   want doc excerpts inlined (at the cost of staleness and length).

7. **Non-PreSnap-IQ work in the same octogent install.** If this octogent
   workspace is dedicated to PreSnap IQ, we could shadow the built-in
   `swarm-worker.md` directly (same filename) and skip the
   `presnap-iq-worker.md` naming. If the octogent install may orchestrate
   other projects in parallel, keep the distinct name. My plan assumes the
   distinct name.

---

## 7. What I Recommend You Do Next

1. Answer open questions 1–7 above.
2. Approve or adjust this plan.
3. Then I execute in this order: (a) add docs + CLAUDE.md edit in
   nextdown_oracle; (b) add custom prompt templates in `.octogent/prompts/`;
   (c) draft tentacle `CONTEXT.md` pointers as tentacles get created.
4. Smoke test: spawn one swarm worker for a simple PreSnap IQ task, confirm
   the agent mentions the three docs in its first response.

End of integration plan.
