---
feature: compose-next
status: draft
updated: 2026-07-22
branch: compose-next
predecessor: compose-slim (draft PR #1850)
---

# Compose Next

## Report

**What this ships** - One self-contained builtin skill named `compose-next`, invoked from Build as `/compose-next`. It bundles the compact compose workflow (spec, worktrees, dependency-ordered implementation, verification, review, feature-document finalize, finish) into a single skill load. Legacy Compose keeps working unchanged and is marked deprecated.

**Compatibility posture** - Legacy Compose (`compose` agent, `compose:*` private skills, `compose.txt` prompt, existing `compose.js` workflow) remains functional. `compose-next` is additive. No workflow rewrite, no `compose:` skill deletion, no `compose.txt` change, no plan-mode change, no permission-preset change. Removal of Legacy Compose is a later, separate PR gated on release observation of the dual path.

**Explicit non-goals for this PR** - Do not introduce structured `Skill.Info.scope`, `Permission.evaluateSkill`, or `ScanMeta` scanning. The `compose:*` name-prefix heuristic that already exists on `main` at six sites is left in place; it will disappear alongside Legacy Compose in the same removal PR, at which point no general scope mechanism is required either. Keeping the scope refactor out of this PR is the whole reason `compose-next` fits as a small, additive compatibility step.

## [S1] Problem

Legacy Compose bundles three concerns into one agent mode: permission policy (what tools may be used), workflow curriculum (fourteen internal skills orchestrating brainstorm, plan, tdd, review, merge, ...), and UI state (Tab-cycle entry, status bar, dialog filtering). The curriculum was necessary for weaker models; stronger Fable/Sol-class models internalize most of it and benefit more from one compact executable contract than from fourteen orchestrated skills.

Draft PR #1850 (`compose-slim`) validated this experimentally by consolidating fourteen skills into three (`compose-grill`, `compose-spec`, `compose-dev`), reducing `compose.txt`, and introducing a structured `scope` field with `Permission.evaluateSkill`. The experiment worked, but it replaces too much of Legacy Compose at once and cannot ship as-is: existing users depend on the current Compose agent, and rewriting the workflow while migrating discovery mechanism in the same PR is an unnecessarily large blast radius.

Compose Next is the compatibility-first successor. It carries only the additive product surface (one new user-selectable skill) and the minimum discovery adjustment required to make it user-visible while keeping it out of routine model auto-discovery. Everything else — including the eventual removal of `compose:*` name-prefix filtering — is deferred to the Legacy-Compose-removal PR.

## [S2] Design

### One self-contained builtin skill

Add one skill file:

```text
packages/opencode/src/skill/builtin/.bundle/compose-next/SKILL.md
```

Canonical name: `compose-next`. Bundle root: builtin. Not prefixed with `compose:`; not scoped to the Compose agent. It is a normal builtin capability whose consumer is any primary agent (in practice, Build) that explicitly loads it.

The skill body is a single executable contract, in this order:

1. **Grill** - resolve genuine user decisions (question tool with concrete options); apply Never-Ask to a single decision only; do not batch later decisions under one grant.
2. **Spec** - create or amend a feature document at `<compose_docs_dir>/spec/<feature>.md` when the work warrants one; keep design, tasks, and delivery report in that one document.
3. **Implement** - proceed in dependency order; use test-first where applicable; own a worktree explicitly if isolation is needed; do not spawn parallel edits into the same worktree.
4. **Verify** - run verification and produce a compact PASS/FAIL/PRE-EXISTING summary; verification must complete before review is dispatched.
5. **Review** - dispatch one fresh reviewer with spec path, worktree path, base/head SHAs, diff coordinates, and the compact verification summary; the reviewer reuses that summary rather than duplicating heavy E2E commands without cause.
6. **Finalize** - update the feature document (report, journey log, verification evidence) before branch completion.
7. **Finish** - explicit merge / PR / keep-branch / discard, with worktree ownership stated; destructive actions never auto-approve.

The three slim experimental skills are source material for this document; they are not carried into the production bundle.

### User-visible, model-undiscovered

Discovery uses the existing `Skill.all()` versus `Skill.available(agent)` split; no new mechanism is added.

- `Skill.all()` includes `compose-next`. Command registration and the app skills endpoint already resolve from `all()`, so `/compose-next` slash autocomplete and explicit invocation work in Build without further wiring.
- Default agent skill permission adds an exact `compose-next: deny` rule at `packages/opencode/src/agent/agent.ts` alongside the existing `"compose:*": "deny"`. `Skill.available(agent)` therefore omits `compose-next` from `available_skills` and from the skill tool description surface.
- `packages/opencode/src/tool/skill-search.ts` currently reads `Skill.all()`; it must switch to `Skill.available(agent)` so `compose-next` (and any future permission-hidden skill) is not returned by BM25 search or auto-load. This is a general correctness fix that PR #1850 already documented as independent of the scope refactor.
- `SkillTool.execute()` (via `Skill.get()`) stays permissive. If a model guesses the exact name `compose-next` it may invoke it; this is behavior guidance, not a security boundary. Do not add activation state, invoke-time refusal, model allowlists, or new visibility schema.

Because the new skill is not named `compose:*` and lives in the builtin bundle, none of the six existing `startsWith("compose:")` sites accidentally hide it:

- `skill/search.ts:65` (name-prefix filter) — no match; unaffected.
- `skill/localized-alias.ts:8` (alias suppression) — no match; `compose-next` produces localized aliases like any other builtin.
- `cli/.../dialog-skill.tsx:29` (skill dialog filter) — no match; appears in the dialog.
- `cli/.../autocomplete.tsx:388` (slash autocomplete filter outside Compose) — no match; `/compose-next` shows in Build autocomplete.
- `agent/agent.ts:110` default agent `"compose:*": "deny"` — no match; a separate exact `compose-next: deny` rule handles model discovery.
- `agent/agent.ts:219` compose agent `"compose:*": "allow"` — no match; Compose agent does not auto-allow `compose-next`, which is intended (Compose Next is not a Compose-mode internal).

### Legacy Compose deprecation surface

Compose agent stays enabled, keeps its private skills, its prompt injection, and `compose.js`. It is not removed from the Tab-cycle in this PR; the dual-path release window requires it to remain reachable exactly the way users know it today.

Three additive deprecation touchpoints:

1. **Agent description and opening prompt** — single deprecation line appended to `agent.ts` compose block description and to `packages/opencode/src/session/prompt/compose.txt`:

   > Legacy Compose is deprecated but remains available for compatibility. With a Fable/Sol-class model, switch to Build and run `/compose-next`.

2. **Home tips — context-aware lock on Compose** — one new tip `tui.tips.compose_next` recommending Build + `/compose-next` is added to the rotating home tips pool (`packages/opencode/src/cli/cmd/tui/feature-plugins/home/tips-view.tsx`) with weight `50`, same tier as `multi_skills` / `free_models`. Additionally, when the current agent is `compose` and the Tips component is mounted (home only), the rotation is **locked** to `tui.tips.compose_next`: the interval timer is cleared and the tip key is fixed to that entry. When the agent changes away from `compose`, the interval restarts and normal weighted rotation resumes. Tab-switching into Compose from home therefore visibly and immediately shows the recommendation in place, without adding any new UI element. Tab-switching in a session view does not affect this — the Tips component only renders on home.

3. **New-session toast on Compose creation** — when a **new** compose session is created (the `sdk.client.session.create({...})` call site while `agent.name === "compose"`), fire one info toast recommending Compose Next:

   > Legacy Compose is deprecated. For Fable/Sol-class models, switch to Build and run `/compose-next`.

   Trigger location: at the successful `session.create` return in `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` around L1101, gated by `agent.name === "compose"`. Trigger only on the create branch — resuming an existing compose session, Tab-switching into an already-running compose session, or re-rendering the view does not toast. "Entering Compose" is defined narrowly as "a new compose session came into existence," not "the compose agent became active." A single conditional at the create site is sufficient; no persistent has-seen flag, no session-scoped counter.

The tip lock and the toast are two independent signals for two different moments and are intentionally allowed to co-occur: the tip is a passive background hint on the home view while the user considers the choice; the toast is the confirming reminder at the exact moment the user commits by creating the first compose session. By the time the toast fires, the user has already left the home route (a session route has just been opened), so the tip is unmounted and the two never render on screen at the same instant.

This is guidance text, not runtime model detection. No hard model-ID list. Users choose either path.

### i18n coverage

All user-facing strings introduced by this PR ship translations for every locale under `packages/opencode/src/cli/cmd/tui/i18n/` (`en`, `es`, `fr`, `ja`, `ru`, `zh`, `zht`). Missing a locale is a review-blocking gap.

Keys added:

- `tui.tips.compose_next` — home tip body recommending Build + `/compose-next`.
- `tui.toast.compose_deprecated` — toast body fired on new compose session creation.
- `tui.skill.compose-next.description` — description shown in skill dialog / autocomplete / command palette. Naming follows the existing `tui.skill.<name>.description` convention.

The Compose agent description line appended in `agent.ts` is inline English today and stays inline English in this PR — the localized surface for users is the tip and the toast, both keyed above.

### Why no scope mechanism in this PR

On `main` today, `compose:` is not a runtime namespace — it is literally the leading segment of each Legacy Compose skill's frontmatter `name` (`compose:brainstorm`, `compose:tdd`, ...). The six `startsWith("compose:")` sites are therefore self-consistent with the legacy bundle content, and they will disappear in one PR when the bundle is deleted.

Adding `Skill.Info.scope` + `evaluateSkill` + `ScanMeta` now would introduce a general mechanism whose only consumer is a subsystem scheduled for removal. It also enlarges this PR's blast radius (schema, scanner, permission evaluator, six migration sites, tests) without user-visible benefit. Compose Next needs exactly one thing beyond `main`: a way to keep the new skill out of default-agent discovery. Exact-name permission handles that in one line.

If a future skill genuinely needs namespace-level gating, the scope refactor can be revisited then, informed by an actual second use case.

## [S3] Implementation

### Files

Add:

- `packages/opencode/src/skill/builtin/.bundle/compose-next/SKILL.md`
- `packages/opencode/test/skill/compose-next-discovery.test.ts`
- `packages/opencode/test/tool/skill-search-hidden.test.ts` (or extend an existing search test)

Modify:

- `packages/opencode/src/agent/agent.ts` — add exact `"compose-next": "deny"` to the default agent's `skill` permission ruleset (adjacent to the existing `"compose:*": "deny"`). Append the deprecation line to the Compose agent block's `description`. Do not add any skill rule to the Compose agent.
- `packages/opencode/src/session/prompt/compose.txt` — append the same deprecation line as a single-line edit; do not rewrite the file.
- `packages/opencode/src/tool/skill-search.ts` — resolve current agent from context and source the searchable list from `Skill.available(agent)` instead of `Skill.all()`. Update the tool description string accordingly if needed.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` — at the successful `sdk.client.session.create` return (~L1101), when `agent.name === "compose"`, fire `toast.show({ message: t("tui.toast.compose_deprecated"), variant: "info" })`. Do not touch the resume branch, the orchestrator branch, or any switch/Tab path.
- `packages/opencode/src/cli/cmd/tui/feature-plugins/home/tips-view.tsx` — add `"tui.tips.compose_next"` to `TIP_KEYS` and to `PRIORITY_WEIGHTS` with weight `50`. Add a reactive lock: when `local.agent.current()?.name === "compose"`, clear the rotation interval and set the tip key to `tui.tips.compose_next`; when it changes back, restart the interval and resume weighted picking. The lock lives entirely inside the Tips component and does not need any state outside home.
- All seven i18n locale files under `packages/opencode/src/cli/cmd/tui/i18n/` (`en.ts`, `es.ts`, `fr.ts`, `ja.ts`, `ru.ts`, `zh.ts`, `zht.ts`) — add three keys each: `tui.tips.compose_next`, `tui.toast.compose_deprecated`, `tui.skill.compose-next.description`.

Do not touch:

- `packages/opencode/src/skill/index.ts` scanner or `Info` schema.
- `packages/opencode/src/skill/search.ts` `startsWith("compose:")` filter.
- `packages/opencode/src/permission/evaluate.ts` (no `evaluateSkill`).
- `packages/opencode/src/skill/compose/**` bundle contents.
- `packages/opencode/src/workflow/builtin/compose.js`.
- `packages/opencode/src/skill/localized-alias.ts`, `dialog-skill.tsx`, `autocomplete.tsx` legacy filters.
- Tab-cycle order in `local.agent.move` / `app.tsx` agent registration order.

### Skill content composition

Source material comes from the compact contracts on `compose-slim`:

- `compose-slim:packages/opencode/src/skill/compose/.bundle/compose-grill/SKILL.md` → sections on question-tool shapes and Never-Ask scope.
- `compose-slim:packages/opencode/src/skill/compose/.bundle/compose-spec/SKILL.md` → single-document `<feature>.md` invariant.
- `compose-slim:packages/opencode/src/skill/compose/.bundle/compose-dev/SKILL.md` → worktree, verification-before-review, review coordinate, finish rules.

These are copied via `git checkout origin/compose-slim -- <path>` into a scratch location, then hand-merged into one SKILL.md preserving executable contract text (tool shapes, ordering constraints, review coordinates) and dropping cross-skill coordination language ("this skill hands off to the next"). Rationale prose that does not carry executable content is dropped.

### Tests

- Discovery: `compose-next` appears in `Skill.all()`; `compose-next` is absent from `Skill.available(defaultAgent)`; `compose-next` is absent from `Skill.available(composeAgent)` (Compose Next is not a Compose-mode internal).
- Slash surface: command registry / app skills endpoint returns `compose-next` (Build slash autocomplete works).
- Skill search: `skill_search` on a query that would otherwise match `compose-next` does not return it under the default agent.
- Legacy invariants preserved: existing `compose:*` filter tests remain green unchanged.
- i18n: a light test asserts each of the three new keys is present in all seven locale files (mirror any existing `i18n` completeness test if one exists; if not, add a small one).
- Tip lock: on the home view with `agent.name === "compose"`, the current tip key is `tui.tips.compose_next` and no rotation occurs; changing the agent back to a non-compose primary restores rotation.
- Toast trigger: the toast fires only on the create branch when `agent.name === "compose"`; resume, Tab, and non-compose creation paths do not toast.

### Verification

From `packages/opencode`:

- `bun test test/skill/compose-next-discovery.test.ts test/tool/skill-search-hidden.test.ts` (new).
- `bun test test/agent test/skill test/permission test/tool` (regression band relevant to the touched files).
- `bun typecheck` (workspace-level from package dir).
- `git diff --check`.

Draft PR opens in Ready state (not Draft) since this is the successor implementation, not a further experiment.

## [S4] Migration and PR sequencing

1. **This PR (P1)** - Compose Next additive; Legacy Compose deprecated but functional.
2. **Draft PR #1850 closure** - After this PR opens, update PR #1850 body with this PR's URL and close #1850 as superseded by the compatibility route. The experiment graduated; it did not fail.
3. **Dual-path release window** - Observe: task completion rate, user intervention rate, skipped spec/report/review, duplicate heavy verification, context/token cost, fallback-to-Legacy rate, third-party model behavior near Fable/Sol capability.
4. **Legacy Compose removal PR (later)** - Remove `compose` agent, `compose:*` bundle, `compose.txt`, all six `startsWith("compose:")` sites, and add `/compose` as an alias of `compose-next`. Only proceed after Fable/Sol-class capability is broadly available and the dual path has been observed without material regression.
5. **Separate later work** - Plan-mode dissolution and Tab permission presets are independent roadmap items with their own specs.

## [S5] Out of scope

- Deleting or renaming any `compose:*` skill.
- Any change to `compose.js`, `compose.txt` content beyond the single deprecation line, or the Compose agent's permission set.
- Introducing `Skill.Info.scope`, `Permission.evaluateSkill`, or `ScanMeta` scanning.
- Hard model-ID gating for `/compose-next`.
- Treating model-undiscoverability as a security boundary.
- Removing Legacy Compose in this PR.
- Removing Legacy Compose from the Tab-cycle in this PR.
- Toasting on Tab-switch into Compose or on any resume of an existing compose session.
- Locking the tip anywhere except on home (session views do not render Tips).
- Persistent "user has seen this deprecation" state.
- i18n-keying the Compose agent description text; that path is broader than this PR.
- Migrating existing Compose feature documents.
- Plan-mode dissolution and Tab permission-preset changes.

## Tasks

- [ ] T1: author `packages/opencode/src/skill/builtin/.bundle/compose-next/SKILL.md` by hand-merging compact contracts from `origin/compose-slim` three slim skills — acceptance: skill validator reports 0 errors, 0 warnings; single-file self-contained load; executable contracts for grill / spec / worktree / verify / review / finalize / finish present (covers: S2)
- [ ] T2: add exact `"compose-next": "deny"` to default agent skill permission — acceptance: `Skill.available(defaultAgent)` omits `compose-next`; `Skill.all()` includes it; test asserts both (covers: S2, S3)
- [ ] T3: switch `tool/skill-search.ts` to `Skill.available(agent)` — acceptance: search over a query matching `compose-next` under the default agent returns no result; existing search tests remain green (covers: S2, S3)
- [ ] T4: append deprecation line to Compose agent `description` in `agent.ts` and to `compose.txt` opening prompt — acceptance: single-line addition in each; Compose agent behavior otherwise unchanged; existing Compose tests green (covers: S2)
- [ ] T5: add home tip `tui.tips.compose_next` (weight `50`) to `tips-view.tsx` and implement the compose-agent lock (clear interval + fix key on entry, restart on exit); add key to all seven locale files — acceptance: tip enters the rotation on home; switching to the Compose agent from home immediately shows the compose_next tip and stops rotation; switching away restarts rotation; all seven locales carry the key (covers: S2)
- [ ] T6: fire deprecation toast on new compose session creation in `component/prompt/index.tsx`; add `tui.toast.compose_deprecated` to all seven locales — acceptance: toast fires only on the create branch when `agent.name === "compose"`; resume and Tab paths do not toast (covers: S2)
- [ ] T7: add `tui.skill.compose-next.description` to all seven locales — acceptance: skill dialog / autocomplete render the localized description (covers: S2)
- [ ] T8: verification band pass — acceptance: relevant `bun test` bands and `bun typecheck` and `git diff --check` all pass from `packages/opencode` (covers: S3)
- [ ] T9: open the PR (Ready, not Draft); update PR #1850 body with its URL and close #1850 as superseded — acceptance: successor URL recorded on #1850; closure message frames the experiment as graduated (covers: S4)
