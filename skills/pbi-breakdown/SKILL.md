---
name: pbi-breakdown
description: Split ONE sprint PBI into reliable, artifact-anchored dev tasks for a chosen team slice (FE/BE/MB), each with a time estimate, output as a markdown checklist in the team naming pattern. Use when the user pastes a PBI, story, ticket, or sprint item (text or screenshot) and wants it broken into tasks, or says "split this PBI", "break down this story", "make tasks from this", "/pbi-breakdown". Asks team/slice first, anchors every task to a named endpoint/table/field/component, flags missing specs instead of guessing, and estimates each task.
---

# PBI → Tasks

Turn one sprint PBI into a set of correctly-named, time-estimated dev tasks for ONE team slice. Reliability comes from four rules: **slice first, anchor to artifacts, flag gaps (never fabricate), self-check before output.**

## Naming pattern (every task)

```
[<system>][<team>] <work-type>: <part> (<level>) - <page> — <Xh>
```

Example: `[WEB][FE] UI: Inspection plan table (Filter by column) - Inspection plan page — 3h`

| Slot | Meaning | Examples |
|------|---------|----------|
| `system` | platform | WEB, API, MOBILE |
| `team` | who does it | FE, BE, MB |
| `work-type` | stage of work (closed list ↓) | Request model, Validation, Persist, UI |
| `part` | box / component / table / field / feature | Inspection plan table |
| `level` | scope of that part (optional, in parens) | Filter by column |
| `page` | page / screen / endpoint / module | Inspection plan page |
| `Xh` | estimate | 3h, or `2–4h ⚠` if uncertain |

**`page` for BE** (no screen exists): use the endpoint or module name **straight from the PBI** — the route (`DraftCompleteWorkOrder`), the resource group (`WorkOrderCustody`), or the feature name. This is naming a real named artifact, not inventing a screen — so it does *not* count as a fabricated `page`. Only freehanding a UI screen name nobody mentioned is banned. When two endpoints share one resource path (POST + GET on the same route), prefix the verb so titles stay distinct — `page` = `POST proof-of-delivery` vs `GET proof-of-delivery`.

**`page` for a sub-screen** (FE tab, MB bottom sheet, modal, dialog, drawer): these aren't full pages — use the **host page/screen the PBI names** (e.g. a custody tab/sheet that lives on "Work Order Detail" → `page` = `Work Order Detail`). Put the sub-surface in `part`/`level`, not `page`. Don't fabricate a name for the sheet itself.

**system auto-map** (confirm only on conflict): BE→`API`, FE→`WEB`, MB→`MOBILE`.

**`work-type` is a closed list — pick from the team's stages, never freehand:**

| Team | Allowed work-types |
|------|--------------------|
| BE | `Request model` · `Validation` · `Persist` · `Read` · `Response` · `Migration` · `Auth` · `Upload` · `Integration` · `Queue` · `Job` · `Cache` · `Config` · `Test` |
| FE | `UI` · `Data mapping` · `State` · `Logic` · `Form` · `Integrate API` · `Routing` · `Auth` · `Translate` · `Test` |
| MB | `UI` · `Data mapping` · `State` · `Logic` · `Integrate API` · `Local store` · `Offline sync` · `Permission` · `Device` · `Auth` · `Translate` · `Test` |

**What each non-obvious label means** (use the *most specific* one that fits; don't stretch a generic verb when a precise label exists):

| Label | Team | Use for |
|-------|------|---------|
| `Read` | BE | a GET / query endpoint that *fetches* data (the read path) |
| `Response` | BE | shaping/serializing the return DTO of a *write* (POST/PUT) — pairs with that write, not a standalone fetch. Rule: GET endpoint → `Read`; the body a POST returns → `Response`. |
| `Migration` | BE | new table / column / DDL / data backfill |
| `Upload` | BE | **server-side write of a file/blob to external storage** (e.g. BE receives a file part → writes to Azure Blob → returns URL). `Upload` is the *storage write*, always its own task. NOT `Upload`: a client (FE/MB) *sending* a file — whether base64-in-body or a multipart file part — that's the network call, so it folds into the client's `Integrate API` task. Rule: who writes to storage gets `Upload`; who sends the bytes gets `Integrate API`. |
| `Integration` | BE | call a 3rd-party / external service |
| `Queue` | BE | publish/consume a message or event; enqueue async work |
| `Job` | BE | scheduled / background / cron job |
| `Cache` | BE | caching layer read/write/invalidation |
| `Config` | BE/any | feature flag, env, config wiring |
| `State` | FE/MB | state container (store / bloc / provider / reducer) — distinct from `Logic` |
| `Form` | FE | form build + client-side field validation |
| `Routing` | FE | route / navigation / guard wiring |
| `Local store` | MB | on-device persistence (sqflite / Hive / secure storage) — **owns the storage**, incl. the durable pending-event queue table/box |
| `Offline sync` | MB | connectivity listener + flush/retry/dedup that drains what `Local store` holds — **the sync engine, not the storage**. Pair with `Local store` for an offline flow: one stores, one syncs. |
| `Permission` | MB | OS/device permission request (location / camera / storage) — **one task per distinct OS permission** (camera + location = two `Permission` tasks, not one) |
| `Device` | MB | sensor / hardware use (GPS capture, camera, QR scan, biometrics) — **one task per distinct sensor/capability** (photo capture + GPS capture = two `Device` tasks) |
| `Auth` | all | **user identity / session / access authorization** — NOT OS device permission (that's MB `Permission`) |

Same word every run → consistent task names. Need a label still not listed? Raise it as a question (Step 6), don't freehand one.

## Workflow

### Step 0 — Read all input, detect truncation
Parse PBI text + every image. If a screenshot is cut off (cropped rows, scrollbar, trailing blanks), do NOT assume you saw it all — mark the unseen region a **spec-gap** (Step 3).

### Step 1 — Slice first (mandatory, before inference)
Ask which slice this breakdown is for: **whole PBI, or just FE / BE / MB.** This is the first grill-me question (see Step 6) — ask it one-at-a-time, with a recommended slice. Decompose ONLY that slice. PBI text is context, not scope — do not split the whole feature when the user owns one slice.

**Re-run continuity:** if the user already ran another slice of the *same* PBI, ask for that prior output (or paste) before decomposing. Don't re-derive shared artifacts — reuse the earlier run's `🔗` flags as the join: this slice's `Integrate API` (FE/MB) hangs off the BE endpoint already flagged. No persistent state is kept; the prior list IS the memory.

### Step 2 — Build the change inventory (artifact-anchored)
Extract concrete named artifacts the slice touches: endpoints, tables, fields, components, pages. **Every task must trace to a named artifact.** No artifact → no task. Do not invent filler tasks (logging, migration, error states, tests) unless the PBI names them or the user asks.

**Implied-work triggers** (raise, don't auto-create): scan the inventory against these — when a trigger fires, ask it as a grill-me question (Step 6) anchored to the real artifact; create the task only if the user confirms, using the mapped label.
- new table / column / schema change on populated data → `Migration` task?
- new endpoint / role-gated or sensitive data → `Auth` task? (identity/access — not device permission)
- file/image written to external storage → `Upload` task? (not if it rides in the request body as base64)
- 3rd-party / external service call → `Integration` task?
- async / event / message / background work → `Queue` or `Job` task?
- offline capability / "must not lose data when no network" → MB `Local store` + `Offline sync` tasks?
- uses GPS / camera / QR / sensor → MB `Permission` (request access) + `Device` (capture) tasks?

This keeps "flag before guess" — surface the maybe-task as a question, never silently slip it in.

### Step 3 — Apply the stage template per artifact
Walk the team's **default stages** (the common path) per artifact, mark applies / N-A:
- **BE:** request model → validation → persist → response/read → tests*(opt)*
- **FE:** UI → data mapping → logic & action → integrate API → translate
- **MB:** UI → data mapping → logic & action → integrate API → translate

The default walk is just the spine — the full closed list above carries more labels for work the feature actually needs. Pull them in when the artifact calls for it: BE `Queue`/`Integration`/`Cache`/`Job`, FE `State`/`Form`/`Routing`, MB `Local store`/`Offline sync`/`Permission`/`Device`. e.g. an offline mobile flow adds `Local store` + `Offline sync` tasks; a GPS feature adds `Permission` (request access) + `Device` (capture) — separate from `Logic`.

One task per (artifact × applicable stage). The stage name **is** the task's `work-type` — use the exact closed-list label above. Stages are a starting point — propose, let the user trim. **Tests stage: off by default — ask once "include test tasks?"**

**One work-type per task — never bundle concerns.** A task carries exactly one label. Don't fold a second concern into another's task — that hides effort and wrecks the estimate. In particular, **external integration is always its own task, never merged into `Persist`**: writing a file/image to blob or external storage → separate `Upload` task; calling a 3rd-party API → its own task; queue/event publish → its own task. Rule of thumb: if you catch yourself writing "X **+** Y" in a task title, it's two tasks.

When an artifact is named but its detail is missing (fields unknown, behavior unstated, image truncated): create the task, tag `⚠ spec-gap: <what's missing>`, and raise it in Step 6. **Never fabricate field lists or behavior.**

Two distinct kinds of gap — keep them separate:
- `⚠ spec-gap: <what's missing>` — **information we don't have yet** (truncated image, unlisted fields, unspecified layout). Resolves by someone *providing* the info.
- `❓ open-decision: <the undecided rule>` — **a business rule deliberately deferred** ("self-handover allowed? ยังไม่สรุป"). Resolves by someone *deciding*, not by data. Attach it to the task whose behavior it changes (often `Logic`/`Validation`); proceed on the recommended default and flag it. If it spans no single task, list it in the output's `**Open decisions:**` block.

### Step 4 — Estimate the REAL effort (honest hours, no cap; ~3h is the comfort zone, not a ceiling)
Estimate the time the task **actually** takes — count the real work: external integrations, first-time setup, error handling, the lot. Do NOT shrink a number to fit a band. Default anchors (a floor for clean, familiar work):

- **BE:** DTO/request model 1–2h · validation 2–3h · persist (delete-then-insert / upsert) 3–4h · response/read 2h · `Migration` (new table + FK) 2–3h · `Upload` (blob write, 1 file) 2–3h · `Integration` (3rd-party call, fail handling) 3–4h · `Queue`/`Job` 3–4h · `Cache` 2h · `Auth` (guard) 2h
- **FE:** UI layout 2–3h · data mapping 2h · `State` 2h · logic 2h · `Form` (+ client validation) 2–3h · integrate API 2h · `Routing` 1–2h · translate 1h
- **MB:** UI 2–3h · data mapping 2h · `State` 2h · logic 2h · integrate API 2h · `Local store` 3h · `Offline sync` 3–4h · `Permission` (per OS permission) 2h · `Device` (per sensor) 2h · translate 1h

These are floors for familiar work; raise them for first-time setup or extra error paths.
- Clean task → point estimate `— 3h`.
- **Any task carrying `⚠ spec-gap` → must be a range, never a point** (the gap is uncertainty; a flat number hides it) `— 2–4h ⚠`.
- **Real effort > 5h → keep the honest number, add `⚠ over 5h — consider splitting`.** Don't force the split and don't fake a smaller estimate; warn and let the user decide. Split only when there's a natural seam.
- Too small (<1h)? Decide by concern, not size:
  - Same stage/concern as a neighbor task (e.g. an empty-state on the component being built) → **fold into that task as a `→ done:` note**, don't make a standalone task.
  - A distinct concern of its own (different work-type) → **keep it standalone even if <1h** — never bundle concerns just to hit an hour. `Translate` always stands alone (the i18n pass is its own concern).
  - Tie-breaker so two runs match: if it maps to its own closed-list work-type, it's standalone; if it's a sub-part of another task's work-type, it folds.

**Anchor override:** the numbers above are defaults. If the user gives team-specific effort (e.g. "our persist is 5h"), use their value this run instead of the default. No actuals/velocity lookup — defaults + explicit override only.

**Estimate scope = developer coding effort only.** The hours cover writing + locally verifying the code. They do NOT include code review, PR rework, QA, deploy, or meetings. State this when handing off so capacity planning adds its own overhead buffer on top.

### Step 4b — Order, dependencies, cross-slice flags
Tasks ship in sequence, not as a flat bag. Capture two link types:

- **Intra-slice (within this run):** later stage of an artifact depends on its earlier stage — `Validation` after `Request model`, `Persist` after `Validation`. Number tasks per slice and tag the dependent one `↳ after #n`. Independent artifacts run in parallel (no tag).
- **Cross-slice (other teams):** when this slice produces something another slice consumes, flag it — `🔗 blocks FE` / `🔗 blocks MB` on the producing task, or `🔗 blocked-by BE` when this slice waits on another. Canonical case: a BE endpoint blocks the FE/MB `Integrate API` stage. State it even though the other slice isn't decomposed this run — it's the handoff contract.

Don't fabricate links — only chains the artifact/stage logic actually forces.

### Step 5 — Self-check gate (block output until pass or asked)
1. Every task traces to a named artifact.
2. Every task carries an honest estimate. >5h is allowed but MUST carry `⚠ over 5h — consider splitting` (never a faked-small number). Spec-gap unknowable size → range + same warning.
3. No duplicate / overlapping tasks.
4. Slice coverage complete — each artifact has its applicable stages.
5. All spec-gaps flagged.
6. Every `work-type` is from the closed list (no freehand labels).
7. Intra-slice deps ordered (`↳ after #n`); cross-slice producers/consumers flagged (`🔗`).
8. Implied-work triggers checked (migration / auth / backfill raised if fired).
9. If capacity was given, total compared and over-capacity flagged.
10. No bundled task — one work-type each; external integration (upload/3rd-party/queue) split out, no "X + Y" titles.
Any fail → fix, or ask.

### Step 6 — Ask via grill-me (step-by-step, one question at a time)
Invoke the **grill-me** skill (`Skill` tool, `skill: "grill-me"`) to drive ALL clarifying. Do NOT batch with AskUserQuestion — grill-me asks one question at a time, recommends an answer per question, and explores the input/codebase before asking when it can.

Walk the clarify tree one branch at a time:
1. **Slice** — whole PBI or FE / BE / MB (this is the Step 1 question; ask it first via grill-me).
2. **Inferable gaps** — page/box names, work-type confirm, tests-in-scope (default off).
3. **Implied-work triggers** — only the ones that fired in Step 2 (migration / auth / backfill). Create the task only on a yes.
4. **Spec-gaps** (`⚠`) — each artifact whose fields/behavior are unknown.
5. **Open decisions** (`❓`) — each deferred business rule; offer the recommended default, let the user decide or keep the default.
6. **Capacity** — "how much sprint capacity is left? (skip to leave it unchecked)". Skip → just show total. Given → compare in Step 7.
7. **Final total confirm.**

Resolve each branch before the next; later questions may depend on earlier answers (e.g. slice narrows which artifacts get grilled).

### Step 7 — Output, then stop
Numbered checklist grouped by team, estimate per task, intra-slice deps (`↳ after #n`), cross-slice flags (`🔗`), per-team subtotal + grand total. Add a `→ done:` sub-line **only when it says more than the task title already does** (logic, rules, delete-then-insert behavior) — skip it for self-evident tasks:

```markdown
## [API][BE] — 10–12h
1. [ ] [API][BE] Request model: SparepartList (10 fields) - DraftCompleteWorkOrder — 2h
2. [ ] [API][BE] Validation: SparepartList (required + RemarkMaterialType fix 'SPARE PART') - DraftCompleteWorkOrder — 3h  ↳ after #1
   → done: all 10 fields required-checked; RemarkMaterialType forced to 'SPARE PART'
3. [ ] [API][BE] Persist: SparepartList → spare_part_requests (delete-then-insert) - DraftCompleteWorkOrder — 3h  ↳ after #2
   → done: existing rows for the work order deleted, then new list inserted in one tx
4. [ ] [API][BE] Persist: refrigerantList → maintenance_refrigerants (delete-then-insert) - DraftCompleteWorkOrder — 2–4h ⚠ spec-gap: refrigerant fields not in image

**Cross-slice:** #1–4 (DraftCompleteWorkOrder request) 🔗 blocks FE & MB `Integrate API`.

**Total: 10–12h**
```

**Open-decisions block** (if any `❓` deferred rules): list them under `**Open decisions:**` after the tasks, each with the default you proceeded on — so the deferred call stays visible, not buried in one task line.

**Cross-task field** (a field set in one task but carried in another, e.g. `gps_denied` set in `Permission`, sent via `Data mapping`): note it in the `→ done:` of **both** tasks so the thread isn't lost between them.

**Capacity line** (only if capacity was given in Step 6): append after total. Within → `✅ fits: 12h ≤ 16h left`. Over → `⚠ over capacity: 18h > 16h left` + offer options (defer which task / split the PBI) — never drop tasks yourself.

**Large-slice signal** (even with no capacity given): if one slice's total exceeds ~30h (roughly a single dev's sprint), append `⚠ large slice (~Xh) — may need >1 dev or splitting across sprints`. A heads-up, not a forced split.

Output is the list only. Don't push to Azure DevOps unless asked — then hand to `azure-devops-tasks` with this field map:

| Task slot | Azure DevOps field |
|-----------|--------------------|
| full task line (`[API][BE] … - page`) | **Title** |
| `Xh` estimate | **Remaining Work** + **Estimated Effort (hrs.)** (range → upper bound; strip `⚠`) — Tasks use hours, never Story Points |
| the source PBI | **Parent** (link task under it) |
| `↳ after #n` | **Predecessor** link (or note in Description if links unavailable) |
| `🔗 blocks/blocked-by` | **Description** note + tag |
| `⚠ spec-gap` | **Description** + tag `spec-gap` (don't create until resolved unless user says so) |
| acceptance / "done-when" | **Description** — no separate AC field on Tasks, fold it in |
| Area Path | default `Your Project` (inherit from parent PBI) |
| Iteration | **ALWAYS ASK which sprint** — azure-devops-tasks never assumes current |
| team (`BE`/`FE`/`MB`) | board column / inherit; estimates already carry the effort |

## Rules

- One PBI → many tasks, but ONE slice per run.
- Slice before inference. Artifact before task. Flag before guess. Check before output.
- Estimate the REAL effort, no cap (~3h comfort zone, not a ceiling); never shrink a number to fit. >5h → keep honest + `⚠ over 5h — consider splitting`. Every task carries an `— Xh` estimate. Anchors are defaults — honor a user override.
- Never invent a `page`, box, field, or behavior. Unknown → `⚠ spec-gap` + ask.
- `work-type` from the closed list only — never freehand a stage label.
- One work-type per task; never bundle concerns. External integration (blob/upload, 3rd-party, queue) = its own task, never folded into `Persist`. "X + Y" in a title = two tasks.
- Estimates = dev coding effort only (exclude review/QA/deploy/meetings); say so on handoff.
- Order tasks; tag intra-slice deps `↳ after #n` and cross-slice handoffs `🔗`. Don't fabricate links.
- Implied work (migration / auth / backfill) → raise as a question, never auto-create.
- `→ done:` sub-line only when it adds info past the title. Capacity over → warn + offer options, never silently drop tasks.

See [EXAMPLES.md](EXAMPLES.md) for a full worked run.
