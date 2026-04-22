# pocock-agents

A pair of [OpenCode](https://opencode.ai/) agents that turn [Matt Pocock](https://www.mattpocock.com/)'s skill-driven development workflow into something you can run end-to-end: grill an idea into a PRD, break it into GitHub issues, and dispatch parallel workers that each execute one issue on an isolated git worktree using TDD.

Blog post with the full rationale and walkthrough: **[How I cloned Matt Pocock into OpenCode Agents](https://mdias.info/posts/cloning-matt-pocock-opencode/)**.

## The agents

| File | Role |
| --- | --- |
| [`agents/pocock.md`](./agents/pocock.md) | **Orchestrator.** Loads Matt's skills on-demand through a phased workflow (`grill-me` → `write-a-prd` → `prd-to-issues` → dispatch), manages git worktrees, and coordinates parallel workers. |
| [`agents/pocock-worker.md`](./agents/pocock-worker.md) | **Subagent.** Takes a single GitHub issue and a pre-created worktree, loads the `tdd` skill, follows red-green-refactor, pushes a branch, and opens a PR. |

## Prerequisites

These agents **depend on** [Matt Pocock's skills](https://github.com/mattpocock/skills). They reference skills like `grill-me`, `write-a-prd`, `prd-to-issues`, `tdd`, `qa`, `triage-issue`, `design-an-interface`, `improve-codebase-architecture`, and several Cloudflare-specific ones (`cloudflare`, `workers-best-practices`, `durable-objects`, `agents-sdk`, `playwright-skill`, etc.).

Install Matt's skills first — they're the actual engineering discipline; these agents just orchestrate them.

```bash
# Clone Matt's skills into OpenCode's skill directory
git clone https://github.com/mattpocock/skills ~/.config/opencode/skills
```

(Or copy only the skills you want from [`mattpocock/skills`](https://github.com/mattpocock/skills) — see his repo for the full list.)

## Installation

```bash
mkdir -p ~/.config/opencode/agents
curl -o ~/.config/opencode/agents/pocock.md https://raw.githubusercontent.com/mcdays94/pocock-agents/main/agents/pocock.md
curl -o ~/.config/opencode/agents/pocock-worker.md https://raw.githubusercontent.com/mcdays94/pocock-agents/main/agents/pocock-worker.md
```

Or clone this repo and symlink the files into `~/.config/opencode/agents/`.

## Usage

Start an OpenCode session with the orchestrator:

```
opencode
> @pocock I want to build X
```

The agent will load `grill-me` and interrogate the idea. When the idea survives grilling it moves through design → PRD → issue breakdown → dispatch. Workers run in parallel on separate git worktrees and open a PR each. See the [blog post](https://mdias.info/posts/cloning-matt-pocock-opencode/) for the full walkthrough, including the dependency-wave pattern used for parallel dispatch.

## Customization

The agents are plain markdown. Open them, read them, and edit anything that doesn't fit your workflow. A few things you'll likely want to tune:

- **Context-triggered skills table** in `pocock.md` — the default triggers are Cloudflare-Workers-flavored (`wrangler.toml`, Durable Objects, `@xyflow/react`, Playwright). Swap in the signals that match your stack.
- **Model choice** — both agents default to `anthropic/claude-opus-4-7`. Change the `model:` frontmatter to whatever you have configured.
- **Permissions** — the worker denies `git push --force`, `git reset --hard`, and `git clean` by default. Loosen or tighten as needed.

## Credits

All the hard work is in [Matt Pocock's skills](https://github.com/mattpocock/skills). These agents just compose them into an orchestrator/worker pattern with git worktree isolation.

## License

MIT
