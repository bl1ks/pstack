# pstack on Devin

Run pstack's orchestration workflows on the Devin CLI. Everything below was verified against `devin 3000.6.14`; the Devin notes inside the skills stay generic, this file is the full reference.

## Install

Three ways, pick one:

```bash
# A. Plugin install (namespaced skills: /pstack:how, profile pstack:poteto-agent)
devin plugins install --local /path/to/pstack        # local checkout, this machine only
devin plugins install <owner>/<repo>                  # from git, recorded in your manifest

# B. User-level skills + agents (unnamespaced: /how, profile poteto-agent)
ln -s /path/to/pstack/skills/* ~/.config/devin/skills/
ln -s /path/to/pstack/agents/*.md ~/.config/devin/agents/

# C. Nix / home-manager — see the README section; it declaratively links the
#    repo's skills/ and agents/ into ~/.config/devin/.
```

Then start a session on a SWE model and run `/setup-pstack` once to write `~/.agents/pstack-models.md` (choose `swe-only`).

```bash
devin --model swe-2-max     # or --model swe
/pstack:poteto-mode         # or /poteto-mode for user-level installs
```

## Concept mapping

| pstack / Cursor | Devin |
| --- | --- |
| `Task` tool | `run_subagent` |
| `subagent_type: generalPurpose` | `profile: subagent_general` — inherits the parent model |
| `subagent_type: poteto-agent` | `profile: poteto-agent` (or `<plugin>:poteto-agent`) — bundled custom profile, `model: inherit`, `max-nesting: 3` |
| `subagent_type: Comment Sicko` | `profile: Comment Sicko` (or `<plugin>:Comment Sicko`) |
| `readonly: true` | `profile: subagent_explore` (router model) or `pstack-explore` (read-only, parent model) |
| `run_in_background: true` | `is_background: true` |
| parallel `Task` calls | parallel `run_subagent` calls in one block |
| `model:` per call | no equivalent — `run_subagent` takes a profile; see Model routing |
| `environment: "cloud"` | none — run locally in a git worktree |
| `AskQuestion` | `ask_user_question` |
| `~/.cursor/rules/pstack-models.mdc` | `~/.agents/pstack-models.md` (read by the skills) |
| `agent-transcripts/` | `~/.local/share/devin/cli/transcripts/<session>.json` (`devin list` names sessions) |
| `create-skill` | none — author `SKILL.md` per agentskills.io |
| `deslop`, `control-*` (`cursor-team-kit`) | not shipped — see the fallbacks in poteto-mode's Harness section |
| `/loop` | none — timed `sleep` in a shell |

## Model routing

Devin has no per-call model override for subagents. A spawn's model comes from its profile:

- `subagent_general` — always the **parent's** model. This is what `inherit-parent` and `auto` mean on Devin.
- `subagent_explore` — read-only, runs on the server-side **default subagent model** (router-selected), *not* the parent's.
- Custom profile — `model:` field if set, else the default subagent model. `model: inherit`, `model: auto`, and `model: inherit-parent` all resolve to the **parent's** model (verified, though only `inherit` is a documented Cursor-compatible value — that is why the bundled profiles use it).
- A concrete role slug in `~/.agents/pstack-models.md` means a generated custom profile that pins `model:` (`/setup-pstack` writes `~/.config/devin/agents/pstack-<slug>.md`).

Note: the `swe` family alias resolves to `swe-2-high`, not `swe-2-max`. To pin Max, write `swe-2-max`.

## SWE-only mode

The first-class Devin configuration. Run `/setup-pstack`, pick `swe-only`, and every role line in `~/.agents/pstack-models.md` becomes `inherit-parent`. Panel roles keep their entry count (`interrogate reviewers: inherit-parent, inherit-parent, inherit-parent, inherit-parent` still spawns four reviewers).

With the parent on a SWE model (`devin --model swe-2-max`, or `agent.model` in `~/.config/devin/config.json`), every spawn — delegates, explorers, reviewers, synthesizers — runs on that same SWE model. No non-SWE inference is invoked: `subagent_general` and all bundled profiles inherit the parent, and nothing routes to the default subagent model unless you choose `subagent_explore` deliberately.

```text
devin --model swe-2-max          # root agent: SWE-2 Max
└── run_subagent profile=subagent_general / poteto-agent / pstack-explore
        model: inherit → SWE-2 Max everywhere
```

## Semantics that change on a single model

pstack's `interrogate`, `arena`, `architect`, and `reflect` defaults assume *multi-model* diversity. In SWE-only mode the diversity is **independent-context**: each reviewer is a separate agent session with its own context window and no shared history, which still produces independent judgments. To recover more of the original signal, give reviewers different stances in their task text (correctness, architecture, minimalism, adversarial/security) — the prompt carries the diversity that the model no longer does.

If you want genuine multi-model panels on Devin, configure concrete slugs in `/setup-pstack`; it generates pinned profiles under `~/.config/devin/agents/`.

## Parallel writes

Devin subagents share the parent's working tree — there is no per-subagent isolation. Consequences, already encoded in the skills:

- Parallel **read-only** spawns (`how` explorers, `interrogate` reviewers, `why` investigators) are safe as-is.
- Parallel **writers** (`swarm` workers, `arena` candidates, feature delegates) must each get their own git worktree (`git worktree add`) or a provably disjoint set of files. `arena`'s per-candidate output paths and the `separate-before-serializing-shared-state` principle exist for exactly this reason — they are mandatory on Devin, not optional.
- A `poteto-agent` can itself spawn children (`max-nesting: 3`), which preserves playbook-internal delegation.

## Known limitations

- `environment: "cloud"` maps to local worktrees; there is no per-subagent cloud execution.
- `run_in_background` auto-denies unapproved tools — a background writer may stall on permissions; pre-approve or run foreground.
- Resumed subagents always run in the foreground.
- Plugin subagents load in the local CLI/Desktop only, not in Devin cloud sessions.
- `devin doctor`'s "custom subagent profiles" check only inspects user/project dirs — plugin-bundled profiles don't show there, but they are registered (visible in-session and via `devin plugins info`).
