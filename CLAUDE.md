# Rules

- NEVER write rules as "do X". Always phrase rules as "NEVER do Y" to clearly define what to avoid.

# Git Workflow

- NEVER commit directly to main.
- NEVER use non-conventional commit formats. See .claude/rules/conventional-commits.md
- NEVER leave commits unpushed.
- NEVER use raw git for remote operations when a CLI is available for the remote platform (e.g., `gh` for GitHub, `az repos` for Azure DevOps).
- NEVER rely on global git email. Before committing, check `git config --local user.email`. If not set, prompt the user.
- NEVER open PRs as ready. Always open as draft (e.g., `gh pr create --draft`).
- NEVER enable auto-merge on draft PRs. Enable auto-merge only after the PR is marked as ready (e.g., `gh pr ready` then `gh pr merge --auto --merge`).
- NEVER enable auto-merge on PRs from external contributors. Only repo admins/owners may use auto-merge.
- NEVER use squash or rebase merges. Always use regular merge commits (`--merge`).

# Repo Setup

- NEVER leave the default branch unprotected. Require PRs (no direct pushes) with at least one approving review. Admins may bypass.
- NEVER leave auto-merge disabled on a new repo.
- NEVER leave auto-delete of merged branches disabled on a new repo.

# Superpowers

- NEVER store plans and design specs in `docs/plans/`. Store plans in `docs/superpowers/plans/` and design specs in `docs/superpowers/specs/`.
- NEVER default plans to sequential execution. Optimize for parallelization.
- NEVER dispatch parallel subagents into the same worktree. Each subagent MUST work in its own isolated worktree (via `using-git-worktrees`).
- NEVER use `isolation: "worktree"` when the parent is on a non-main branch — it branches from main, not the parent branch. Create worktrees manually from the current branch (via `using-git-worktrees`) instead.

---

# daana-modeler

daana-modeler is a Claude Code plugin for the Daana data platform. It provides six skills for building, querying, and transforming DMDL data models.

## Repository Structure

- **`.claude-plugin/`** — Plugin manifests
  - `plugin.json` — Plugin metadata (name: `daana`)
  - `marketplace.json` — Marketplace catalog for plugin discovery
- **`skills/focal/SKILL.md`** — Shared Focal foundation skill (`/daana-focal`)
  - `skills/focal/references/` — Focal framework, bootstrap, connections, dialect references
- **`skills/model/SKILL.md`** — Model interview skill (`/daana-model`)
  - `skills/model/references/` — Model schema, examples, source format references
- **`skills/map/SKILL.md`** — Mapping interview skill (`/daana-map`)
  - `skills/map/references/` — Mapping schema, examples, source format references
- **`skills/query/SKILL.md`** — Data query skill (`/daana-query`)
  - `skills/query/references/` — Ad-hoc query patterns
- **`skills/uss/SKILL.md`** — Unified Star Schema generator (`/daana-uss`)
  - `skills/uss/references/` — USS SQL patterns, worked examples
- **`skills/star/SKILL.md`** — Star schema generator skeleton (`/daana-star`)
  - `skills/star/references/` — Dimension patterns, fact patterns
- **`docs/superpowers/specs/`** — Design specifications
- **`docs/superpowers/plans/`** — Implementation plans

## External References

- `external.lock` pins upstream repos. When exploring project context (e.g., start of brainstorming), fresh-clone each repo and compare HEAD against the pinned commit to detect new upstream changes.
