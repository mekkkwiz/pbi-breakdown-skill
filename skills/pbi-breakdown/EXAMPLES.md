# pbi-breakdown — Worked Example

Full run on a real PBI. Shows slice-first → artifact inventory → stage template → estimate → spec-gap → self-check → output.

## Input

PBI (text + screenshot):

> **Related Module / Page:** หน้ารายการงานของช่าง (Technician work list)
> **Objective:** Technician + engineer fill CPSS usage cost into CM repair notification, save draft for closing work. Fetch CM the technician may view, grouped by Status.
> **Test data:** Plant 4089

Second image (form-field spec, partly cut off): a `SparepartList` block with 10 named fields (SparepartReqType, ComponentCode, ReqQuantity, QuantityUnit, IssuedPlant, IssuedLocation, RefIssuedItem, RemarkText, RemarkMaterialType='SPARE PART', item), and a `refrigerantList` block pointing at table `maintenance_refrigerants` — **its fields are below the fold (truncated).**

## Step 0 — Read + truncation
Second image is cut off under `refrigerantList`. Its fields = **unseen → spec-gap.**

## Step 1 — Slice first
Ask: *whole PBI, or FE / BE / MB?* → **User: BE only.**
The objective mentions a UI cost-form, but that's the FE slice — NOT this run. Decompose BE only.

## Step 2 — Change inventory (BE slice)
Concrete artifacts the user names:
- Endpoint: `POST /api/v2.0/WorkOrder/DraftCompleteWorkOrder` (existing — add to request).
- Artifact 1: request list `SparepartList` → table `spare_part_requests` (10 fields, known).
- Artifact 2: request list `refrigerantList` → table `maintenance_refrigerants` (fields unknown — spec-gap).

No artifact for response/read (user said "request only") → those stages N-A. No tests asked → tests stage off.

## Step 3 — Stage template (BE) per artifact

| Stage | SparepartList | refrigerantList |
|-------|---------------|-----------------|
| request model | ✅ | ✅ |
| validation | ✅ | ✅ |
| persist (delete-then-insert) | ✅ | ✅ |
| response/read | N-A (request only) | N-A |
| tests | off (not asked) | off |

→ 6 tasks.

## Step 4 — Estimates
- Request model: DTO change → 2h each.
- Validation SparepartList: 10 required fields + `RemarkMaterialType` fix 'SPARE PART' → 3h. refrigerantList validation: fields unknown → `2–4h ⚠`.
- Persist (delete-then-insert): 3h each; refrigerant persist unknown shape → `2–4h ⚠`.

## Step 5 — Self-check gate
1. Each task ↔ named artifact ✅  2. Honest estimates, none >5h ✅  3. No dupes ✅  4. Coverage: both artifacts have model+validation+persist ✅  5. Spec-gaps flagged (refrigerant fields) ✅  6. work-types all from closed list ✅  7. deps ordered + cross-slice flagged ✅ → pass.

## Step 4b — Order, deps, cross-slice
- Per artifact: `Validation` after `Request model`, `Persist` after `Validation`. SparepartList chain and refrigerantList chain are independent → run in parallel.
- Cross-slice: the whole `DraftCompleteWorkOrder` request 🔗 **blocks** FE & MB `Integrate API` (they call this endpoint). Flag it though FE/MB aren't decomposed this run.

## Step 6 — grill-me (step-by-step)
Invoke `Skill grill-me`. Slice already settled in Step 1. Remaining branch is the one spec-gap, asked alone:
*"refrigerant fields aren't in the image — paste them, or confirm 'mirror all columns of maintenance_refrigerants'?"* → then confirm the grand total. One question at a time; no batch.

## Step 7 — Output

```markdown
## [API][BE] — 15–17h
1. [ ] [API][BE] Request model: SparepartList (10 fields) - DraftCompleteWorkOrder — 2h
2. [ ] [API][BE] Request model: refrigerantList (mirror maintenance_refrigerants) - DraftCompleteWorkOrder — 2h ⚠ spec-gap: confirm fields
3. [ ] [API][BE] Validation: SparepartList (required + RemarkMaterialType fix 'SPARE PART') - DraftCompleteWorkOrder — 3h  ↳ after #1
   → done: all 10 fields required-checked; RemarkMaterialType forced to 'SPARE PART'
4. [ ] [API][BE] Validation: refrigerantList (required fields) - DraftCompleteWorkOrder — 2–4h ⚠ spec-gap: fields unknown  ↳ after #2
5. [ ] [API][BE] Persist: SparepartList → spare_part_requests (delete-then-insert) - DraftCompleteWorkOrder — 3h  ↳ after #3
   → done: existing rows for the work order deleted, then new list inserted in one tx
6. [ ] [API][BE] Persist: refrigerantList → maintenance_refrigerants (delete-then-insert) - DraftCompleteWorkOrder — 2–4h ⚠ spec-gap: row shape  ↳ after #4

**Cross-slice:** #1–6 (DraftCompleteWorkOrder request) 🔗 blocks FE & MB `Integrate API`.

**Total: 15–17h**
```

---

# FE slice — same PBI, abbreviated run

User comes back: *"now do the FE slice."* Step 1 re-run rule → ask for the BE output above, reuse its `🔗` flag as the join.

**Change inventory (FE):** the cost form on หน้ารายการงานของช่าง — component `SparepartTable` (add/edit rows), component `RefrigerantTable` (⚠ spec-gap: columns unknown), and the existing draft-save action that calls `DraftCompleteWorkOrder`.

**Stage template (FE)** per component:

| Stage | SparepartTable | RefrigerantTable |
|-------|----------------|------------------|
| UI | ✅ | ✅ |
| Data mapping | ✅ | ✅ |
| Logic | ✅ (add/remove row, qty validation) | N-A (no rules given) |
| Integrate API | ✅ shared with RefrigerantTable — one task | — |
| Translate | ✅ (TH labels) | merged into SparepartTable task |

**Output:**

```markdown
## [WEB][FE] — 11–13h
1. [ ] [WEB][FE] UI: SparepartTable (editable rows) - Technician work list — 3h
2. [ ] [WEB][FE] UI: RefrigerantTable - Technician work list — 2–3h ⚠ spec-gap: columns not in image
3. [ ] [WEB][FE] Data mapping: SparepartList ↔ form state - Technician work list — 2h  ↳ after #1
4. [ ] [WEB][FE] Logic: SparepartTable (add/remove row + qty required) - Technician work list — 2h  ↳ after #1
   → done: blank qty blocks save; removed row drops from payload
5. [ ] [WEB][FE] Integrate API: draft save → DraftCompleteWorkOrder - Technician work list — 2h  ↳ after #3
   🔗 blocked-by BE #1–6 (request must accept SparepartList + refrigerantList first)
6. [ ] [WEB][FE] Translate: TH labels (Spare part / Refrigerant) - Technician work list — 1h

**Total: 11–13h**
```

# MB slice — full run (different PBI, to show the mobile-only labels)

The SparepartList PBI above has no offline/sensor work, so it won't exercise the MB labels. Here's a second, mobile-heavy PBI run end-to-end — it's where `Local store` · `Offline sync` · `Permission` · `Device` earn their place.

## Input
> **PBI:** Asset custody handover (mobile). After a technician scans an asset QR, open a "รับมอบ / ส่งมอบ" bottom sheet on the **Work Order Detail** screen: pick from/to user, sign (reuse the signature pad from หน้าใบส่งงาน), auto-capture GPS, confirm → `POST /api/v1/work-orders/{woId}/custody`.
> - AC4: no network → event must not be lost; queue + sync on reconnect.
> - AC5: GPS denied → lat/lng null + `gps_denied=true`.
> - Note: self-handover (รับเอง-ส่งเอง) **ยังไม่สรุป**. Payload tail `…` not fully listed.
> *(BE slice already ran: produced `GET + POST /…/custody`, POST flagged blocking.)*

## Step 0 — truncation
No image. Payload tail `…` = unlisted fields → **spec-gap** on the payload. Self-handover = **open-decision** (deferred rule, not missing data).

## Step 1 — slice + re-run
Slice = **MB**. BE ran → reuse its `🔗`; MB consumes the **POST** only (it sends events, doesn't read the timeline). `Integrate API` → `🔗 blocked-by BE`.

## Step 2 — inventory + implied-work triggers
Artifacts: custody bottom sheet, signature pad (reuse), GPS, GPS permission, offline queue, POST endpoint, TH labels.

| Trigger fired | → label(s) raised |
|---------------|-------------------|
| offline / "must not lose data" (AC4) | MB `Local store` + `Offline sync` (one stores, one syncs) |
| uses GPS + QR (sensor) | MB `Permission` (request GPS) + `Device` (capture GPS; QR scan = reuse, raise) |
| signature image | NOT `Upload` — it's `signature_base64` in the body, not a blob write |

## Step 3 — stage table (MB)

| Label | task? |
|-------|-------|
| UI | ✅ bottom sheet (signature pad folded — reuse, `→ done`) |
| Data mapping | ✅ form ↔ payload (⚠ spec-gap: tail `…`) |
| Logic | ✅ from/to select (❓ open-decision: self-handover) |
| Permission | ✅ GPS access + `gps_denied` flag |
| Device | ✅ GPS capture |
| Local store | ✅ durable pending-event queue |
| Offline sync | ✅ connectivity listener + flush/retry |
| Integrate API | ✅ POST custody (🔗 blocked-by BE) |
| Translate | ✅ TH labels (stands alone) |

## Step 7 — Output

```markdown
## [MOBILE][MB] — 21h
1. [ ] [MOBILE][MB] UI: Custody bottom sheet (รับมอบ/ส่งมอบ) - Work Order Detail — 3h
   → done: from/to pickers + reused signature pad (from ใบส่งงาน) + GPS readout + confirm
2. [ ] [MOBILE][MB] Permission: GPS location (denied → null + gps_denied=true) - Work Order Detail — 2h
   → done: requests location; on deny, gps_denied=true carried into payload (see #4)
3. [ ] [MOBILE][MB] Device: GPS auto-capture (gps_lat/gps_lng) - Work Order Detail — 2h  ↳ after #2
4. [ ] [MOBILE][MB] Data mapping: form ↔ POST payload (from/to/asset/signature_base64/gps_lat/gps_lng/gps_denied) - Work Order Detail — 2h ⚠ spec-gap: payload tail "…"  ↳ after #1
   → done: gps_denied (set in #2) included in payload
5. [ ] [MOBILE][MB] Logic: from/to selection + confirm assembles event - Work Order Detail — 2–3h ❓ open-decision: self-handover allowed? (default: allow)  ↳ after #1
6. [ ] [MOBILE][MB] Local store: durable pending-custody-event queue - Work Order Detail — 3h
   → done: unsent events survive app close / no network
7. [ ] [MOBILE][MB] Offline sync: connectivity listener + flush/retry pending events - Work Order Detail — 4h  ↳ after #6
   → done: on reconnect, queued events POST in order; failures retried, none lost (AC4)
8. [ ] [MOBILE][MB] Integrate API: POST custody → POST /api/v1/work-orders/{woId}/custody - Work Order Detail — 2h  ↳ after #4
   🔗 blocked-by BE POST /api/v1/work-orders/{woId}/custody
9. [ ] [MOBILE][MB] Translate: TH labels (รับมอบ / ส่งมอบ) - Work Order Detail — 1h

**Cross-slice:** #8 🔗 blocked-by BE POST /…/custody. Offline queue (#6–7) holds events until reachable.
**Open decisions:** self-handover (from==to) — proceeded on "allow"; confirm with PO.

**Total: 21h**  (dev coding effort only — excludes review, QA incl. Android 8.x offline test, deploy)
```

**Why the mobile labels matter here:** `Local store` (#6) and `Offline sync` (#7) split the offline flow cleanly — without them both collapse into a vague `Logic` task and the real ~7h of offline work hides. `Permission` (#2) and `Device` (#3) keep OS-permission separate from sensor-capture, and neither is mislabeled `Auth`.

## What hardening prevented
- **Slice-first** stopped the run from emitting FE cost-form tasks (wrong scope).
- **Artifact-anchor** stopped phantom logging/migration/error-state tasks.
- **Spec-gap flag** stopped silently inventing refrigerant fields — surfaced as a question + range estimate instead.
- **Truncation detect** caught the cut-off image.
- **Cross-slice flag** surfaced the FE/MB ↔ BE endpoint handoff up front, so the FE/MB run later knows it's blocked on this one.
