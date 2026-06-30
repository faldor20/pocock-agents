# pocock-agents

A pair of [OpenCode](https://opencode.ai/) agents that turn [Matt Pocock](https://www.mattpocock.com/)'s skill-driven development workflow into something you can run end-to-end: grill an idea (and harden the domain glossary as you go), synthesize a PRD, break it into issues, and dispatch parallel workers that each execute one issue on an isolated git worktree (or **Jujutsu workspace** — the agents detect `.jj/` and use `jj` instead) using TDD.

Blog post with the full rationale and walkthrough: **[How I cloned Matt Pocock into OpenCode Agents](https://mdias.info/posts/cloning-matt-pocock-opencode/)**.

> **2026-05 update.** Matt landed a major refactor of [`mattpocock/skills`](https://github.com/mattpocock/skills) between 2026-04-28 and 2026-05-07 — renamed several skills, deprecated others, introduced `CONTEXT.md` + `docs/adr/` as the domain-doc convention, and added `prototype`, `grill-with-docs`, `triage`, `to-prd`, `to-issues`, `setup-matt-pocock-skills`, and `handoff`. These agents were re-aligned with that refactor.
>
> **2026-06 update.** A second wave of upstream changes is now reflected here: `diagnose` was **renamed** to `diagnosing-bugs` and `write-a-skill` to `writing-great-skills`; `zoom-out` and `caveman` were **removed entirely**; and `implement`, `domain-modeling`, `codebase-design`, `resolving-merge-conflicts`, `ask-matt`, `decision-mapping`, `review`, `teach`, and `grilling` were **added**. `triage` also grew an external-PR triage surface. See [What's new](#whats-new) below.

## The agents

| File | Role |
| --- | --- |
| [`agents/pocock.md`](./agents/pocock.md) | **Orchestrator.** Loads Matt's skills on-demand through a phased workflow (`grill-with-docs` → `prototype` → `to-prd` → `to-issues` → dispatch), manages git worktrees / jj workspaces, and coordinates parallel workers. |
| [`agents/pocock-worker.md`](./agents/pocock-worker.md) | **Subagent.** Takes a single issue and a pre-created worktree/workspace, loads `tdd` (and `diagnosing-bugs` when bugs fight back), follows red-green-refactor, pushes a branch/bookmark, and opens a PR/MR. |

## Prerequisites

These agents **depend on** [Matt Pocock's skills](https://github.com/mattpocock/skills). They reference, among others:

- **Engineering**: `grill-with-docs`, `domain-modeling`, `to-prd`, `to-issues`, `triage`, `tdd`, `implement`, `diagnosing-bugs`, `prototype`, `codebase-design`, `improve-codebase-architecture`, `resolving-merge-conflicts`, `ask-matt`, `setup-matt-pocock-skills`
- **Productivity**: `grill-me`, `grilling`, `handoff`, `teach`, `writing-great-skills`
- **In-progress**: `decision-mapping`, `review`
- **Cloudflare-flavored** (loaded contextually when a project's signals match): `cloudflare`, `workers-best-practices`, `wrangler`, `durable-objects`, `agents-sdk`, `sandbox-sdk`, `cloudflare-email-service`, `playwright-skill`

Install Matt's skills first — they're the actual engineering discipline; these agents just orchestrate them.

The simplest path is the [`skills.sh`](https://skills.sh/mattpocock/skills) installer (recommended by Matt):

```bash
npx skills@latest add mattpocock/skills
```

Or do it manually by copying each skill into a flat `~/.config/opencode/skills/<skill-name>/` layout:

```bash
# Clone to a temp location
git clone --depth 1 https://github.com/mattpocock/skills /tmp/mp-skills

# Copy the skills you want into ~/.config/opencode/skills/ (flat structure expected by OpenCode)
mkdir -p ~/.config/opencode/skills
for cat in engineering productivity in-progress misc personal; do
  for skill in /tmp/mp-skills/skills/$cat/*/; do
    cp -R "$skill" ~/.config/opencode/skills/
  done
done
```

Note: Matt's repo nests skills under `skills/<category>/<name>/` but OpenCode expects them flat at `~/.config/opencode/skills/<name>/`. The script above flattens that for you.

## Installation

```bash
mkdir -p ~/.config/opencode/agents
curl -o ~/.config/opencode/agents/pocock.md https://raw.githubusercontent.com/mcdays94/pocock-agents/main/agents/pocock.md
curl -o ~/.config/opencode/agents/pocock-worker.md https://raw.githubusercontent.com/mcdays94/pocock-agents/main/agents/pocock-worker.md
```

Or clone this repo and symlink the files into `~/.config/opencode/agents/`.

## Per-repo setup

Many of Matt's engineering skills now read per-repo configuration that's seeded by `setup-matt-pocock-skills`. The first time you use Pocock in a new repo, the orchestrator will detect that `docs/agents/issue-tracker.md` is missing and run that skill to scaffold:

- **Issue tracker** — GitHub (via `gh`), GitLab (via `glab`), or local markdown under `.scratch/<feature>/`. Pocock will dispatch workers using whichever you choose.
- **Triage label vocabulary** — five canonical roles (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`) plus two categories (`bug`, `enhancement`), mapped to whatever label strings your tracker actually uses.
- **Domain doc layout** — single-context (`CONTEXT.md` + `docs/adr/` at the root) or multi-context (`CONTEXT-MAP.md` pointing to per-context `CONTEXT.md` files, typical for monorepos).

You don't need to run it pre-emptively — it's lazy. The orchestrator triggers it the first time a hard-dependency skill (`to-prd`, `to-issues`, `triage`) actually needs the config.

## Usage

Start an OpenCode session with the orchestrator:

```
opencode
> @pocock I want to build X
```

The default flow for a new code feature:

1. **`grill-with-docs`** interrogates the idea and writes domain terms to `CONTEXT.md` and load-bearing decisions to `docs/adr/` as you go.
2. **`prototype`** (optional) — when a logic or UI question is faster to answer with throwaway code than with prose.
3. **`to-prd`** synthesizes the conversation into a PRD and publishes it to your configured issue tracker. (It does **not** re-interview — that already happened in step 1.)
4. **`to-issues`** breaks the PRD into independently-grabbable, vertically-sliced issues with `ready-for-agent` labels.
5. **Dispatch** — the orchestrator creates one isolated checkout per ready issue (a `git worktree`, or a `jj workspace` for Jujutsu repos — detected via `.jj/`) and dispatches `pocock-worker` subagents in parallel, grouped into dependency waves. In a colocated jj repo the agents use `jj` exclusively, since git mutation commands would corrupt jj history.
6. Each worker loads `tdd` (and `diagnosing-bugs` if the bug fights back), red-green-refactors, pushes its branch, opens a PR/MR.

Other entry points (bug reports, performance regressions, refactors, architecture reviews) are mapped in the orchestrator's `Entry Points` table — see [`agents/pocock.md`](./agents/pocock.md). See the [blog post](https://mdias.info/posts/cloning-matt-pocock-opencode/) for the original full walkthrough; the post-2026-05 flow differs in a few places (`grill-with-docs` instead of `grill-me` + `ubiquitous-language`, `to-prd`/`to-issues` instead of `write-a-prd`/`prd-to-issues`, `diagnosing-bugs` for hard bugs, `triage` instead of `qa`/`triage-issue`/`github-triage`).

## What's new

If you set this up before May 2026, here's what changed in this update:

**Renamed skills** (1-for-1 replacements; old names are no longer loaded by the orchestrator):

| Old | New |
| --- | --- |
| `prd-to-issues` | `to-issues` (now accepts any plan/spec, not just PRDs) |
| `write-a-prd` | `to-prd` (no longer interviews — synthesizes existing context) |
| `github-triage` | `triage` (issue-tracker agnostic) |

**Deprecated** (folded into other skills):

| Old | Replacement |
| --- | --- |
| `triage-issue` | `triage` (one skill, three modes: incoming triage, reproduce, agent brief) |
| `qa` | `triage` |
| `design-an-interface` | `prototype` (empirical throwaway code, not theoretical sub-agent designs) |
| `ubiquitous-language` | `grill-with-docs` (writes `CONTEXT.md` inline as terms get sharpened) |
| `request-refactor-plan` | `grill-with-docs` → `to-prd` → `to-issues` (the regular pipeline) |

**New skills from the 2026-05 wave** referenced by the orchestrator:

- `diagnosing-bugs` — 6-phase debugging discipline (build feedback loop → reproduce → hypothesise → instrument → fix+regression test → cleanup). Genuinely the strongest practical addition; Phase 1 ("build a feedback loop") is the actual skill. (Named `diagnose` in the 2026-05 wave; renamed in 2026-06.)
- `prototype` — throwaway code to answer a design question, in two branches: terminal app for state/logic, multi-variation UI for visual design.
- `grill-with-docs` — grilling that writes `CONTEXT.md` and ADRs inline as decisions land.
- `handoff` — long-session handoff document for a fresh agent to pick up.
- `setup-matt-pocock-skills` — per-repo scaffolder for issue tracker, triage labels, domain doc layout.

**2026-06 wave** (reflected in this update):

- **Renamed:** `diagnose` → `diagnosing-bugs`, `write-a-skill` → `writing-great-skills`.
- **Removed entirely** (no replacement): `zoom-out` (use `grill-with-docs`'s codebase-exploration step to map unfamiliar code) and `caveman`.
- **Added** and now referenced by the agents:
  - `implement` — drive a finished PRD or set of issues to completion in one focused pass, leaning on `tdd` for the inner loop.
  - `domain-modeling` — owns `CONTEXT.md`/`CONTEXT-MAP.md` and ADR maintenance; `grill-with-docs` delegates glossary work to it.
  - `codebase-design` — the deep-module vocabulary (`Module`/`Interface`/`Depth`/`Seam`/`Leverage`/`Locality` + deletion test), split out of `improve-codebase-architecture`.
  - `review` — two-axis review (Standards + Spec) of a branch/PR/WIP, run in parallel sub-agents; now ships an always-on Fowler smell baseline on the Standards axis.
  - `resolving-merge-conflicts` — walk through an in-progress git merge/rebase conflict.
  - `ask-matt` — router that points you at the right user-invoked skill.
  - `decision-mapping` — turn a loose idea into a sequenced map of investigation tickets, resolved one at a time.
  - `teach` — teach the user a concept within the workspace.
  - `grilling` — the underlying grilling discipline shared by `grill-me`/`grill-with-docs`.
- `triage` gained an **external-PR** triage surface, on top of issues.

**Paradigm shifts**:

- `UBIQUITOUS_LANGUAGE.md` → `CONTEXT.md` (single-context) or `CONTEXT-MAP.md` + per-context `CONTEXT.md` (monorepos). The new format lives next to `docs/adr/` for Architecture Decision Records.
- Issue-tracker abstraction — the orchestrator is no longer GitHub-only. Workers dispatch using `gh`, `glab`, or local-markdown writes depending on `docs/agents/issue-tracker.md`.
- `to-prd` no longer interviews. Grilling is a separate, earlier phase.

## Customization

The agents are plain markdown. Open them, read them, and edit anything that doesn't fit your workflow. A few things you'll likely want to tune:

- **Context-triggered skills table** in `pocock.md` — the default triggers are Cloudflare-Workers-flavored (`wrangler.toml`, Durable Objects, `@xyflow/react`, Playwright). Swap in the signals that match your stack.
- **Model choice** — both agents default to `anthropic/claude-opus-4-7`. Change the `model:` frontmatter to whatever you have configured.
- **Permissions** — the worker denies `git push --force`, `git reset --hard`, and `git clean` by default, plus the jj equivalents (`jj op restore`, `jj op abandon`, `jj git push --force`). Loosen or tighten as needed.
- **Worker skill allow-list** — the worker can load `tdd`, `diagnosing-bugs`, `implement`, `review`, `resolving-merge-conflicts`, `triage`, `grill-with-docs`, `domain-modeling`, `prototype`, `codebase-design`, `to-issues`, `improve-codebase-architecture`, plus the contextual ones (`react-flow`, `playwright-skill`, Cloudflare suite, `portless`). Add or remove based on what you want workers to be able to invoke autonomously.

## Credits

All the hard work is in [Matt Pocock's skills](https://github.com/mattpocock/skills). These agents just compose them into an orchestrator/worker pattern with git worktree (or jj workspace) isolation.

## License

MIT
