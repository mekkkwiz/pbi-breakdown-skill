# pbi-breakdown

[ไทย / Thai version](README.th.md)

An [Agent Skill](https://github.com/vercel-labs/skills) that splits **one** sprint PBI into reliable, artifact-anchored dev tasks for a chosen team slice (FE / BE / MB) — each task time-estimated and named in a consistent pattern, output as a markdown checklist.

## What it does

- **Slice first** — asks which team owns the work (FE / BE / MB); decomposes only that slice.
- **Artifact-anchored** — every task traces to a named endpoint / table / field / component. No artifact → no task.
- **Closed work-type vocab** — per-team label list (e.g. BE `Migration`/`Upload`/`Integration`, MB `Local store`/`Offline sync`/`Permission`/`Device`) so task names stay consistent across runs.
- **Flags gaps, never guesses** — `⚠ spec-gap` for missing info, `❓ open-decision` for deferred business rules.
- **Honest estimates** — real effort per task (no cap); warns when a task exceeds 5h.
- **Deps + cross-slice** — `↳ after #n` intra-slice ordering, `🔗` cross-team handoffs.
- **Step-by-step clarify** via the `grill-me` skill, then a markdown checklist ready for Azure DevOps handoff.

## Install

```bash
npx skills@latest add mekkkwiz/pbi-breakdown-skill
```

Then invoke in Claude Code with `/pbi-breakdown`, or just paste a PBI and ask to "break it down".

## Files

- `skills/pbi-breakdown/SKILL.md` — the skill
- `skills/pbi-breakdown/EXAMPLES.md` — full worked runs (BE / FE / MB)
