# Upstream sync and fork policy

This repo is [`backnotprop/pstack`](https://github.com/backnotprop/pstack) plus a thin Devin adapter. `backnotprop/pstack` itself mirrors [`cursor/plugins/pstack`](https://github.com/cursor/plugins/tree/main/pstack) — see `MIRROR.md` for that sync procedure.

## Branches

- `upstream` — Cursor's `plugins/pstack` files exactly, no local edits (from `backnotprop/pstack`).
- `main` — `upstream` plus the mirror's harness-neutral edits (tracks `backnotprop/pstack`).
- `devin` — `main` plus the Devin adapter commits. This is the branch to install.

## Devin-specific changes

All Devin work lives on the `devin` branch, in these commits:

1. `Add Devin plugin metadata` — `.devin-plugin/plugin.json` (Devin's manifest path; `.cursor-plugin/plugin.json` is not read by Devin).
2. `Adapt agents for Devin custom subagent profiles` — `model: inherit` and `max-nesting: 3` on `agents/poteto-agent.md`, `model: inherit` on `agents/comment-sicko.md`, new `agents/pstack-explore.md`.
3. `Add Devin harness mapping to skills` — Devin clauses in the "Other harnesses" / "Harness" blocks of `skills/**/SKILL.md`, `skills/poteto-mode/playbooks/{orchestrate,multi-phase-plan}.md`, and SWE-only mode in `skills/setup-pstack/SKILL.md`.
4. Docs: `UPSTREAM.md`, `docs/devin.md`, and the Devin section at the top of `README.md`.

## Sync procedure

```bash
# 1. Update the mirror's upstream branch per MIRROR.md, then:
git fetch origin
git switch main && git merge --ff-only origin/main   # main tracks the mirror
git switch devin && git merge main                   # or: git rebase main

# 2. Resolve conflicts.
#    Conflict-prone files (edited by both the mirror and this adapter):
#      skills/poteto-mode/SKILL.md        (Harness section)
#      skills/{how,why,interrogate,reflect,swarm,arena,setup-pstack}/SKILL.md
#      skills/no-comments/SKILL.md
#      skills/{recall,show-me-your-work,automate-me,create-verification-skill}/SKILL.md
#      agents/poteto-agent.md, agents/comment-sicko.md
#      README.md
#    Rule: keep the upstream/mirror wording, re-apply the Devin clause on top.
#    Each Devin edit is a self-contained sentence or bullet — re-attach it
#    next to the same anchor text.

# 3. Sanity-check after a sync:
grep -rn "run_subagent" skills/ agents/ | wc -l   # Devin mentions still present
devin plugins install --local . -y                # plugin still loads
```

## Rules for Devin edits

- Never fork-rewrite a skill's core prose. Devin support goes in the existing "Other harnesses" blocks or as a standalone bullet — one clause per file.
- `model: inherit` is the only model value written into shared files; it is Cursor's documented default and resolves to the parent model in Devin. Never write a Devin-only slug (e.g. `swe-2-max`) into `skills/` or `agents/`.
- New Devin-only files (`.devin-plugin/`, `docs/devin.md`, `UPSTREAM.md`) never conflict with upstream — upstream does not ship those paths.
