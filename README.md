# Gren Skill

An [Agent Skills](https://agentskills.io/specification)-format skill that gives coding agents working knowledge of the [Gren](https://gren-lang.org) programming language: syntax, types, `Array`/`Dict`/`Set`, pipeline style, `Maybe`/`Result` error handling, the `Cmd`/`Sub`/`Task` effect model, ports, and how to read dependency source from `gren_packages/`.

The skill uses progressive disclosure: a compact `SKILL.md` is loaded when a task touches Gren code, and the agent lazily reads the focused reference docs under [`docs/`](./docs) only when a question needs deeper detail.

## Layout

```text
gren-skill/
├── SKILL.md            # frontmatter + core reference
└── docs/               # focused reference docs, loaded on demand
    ├── 01-values-and-functions.md
    ├── 02-type-signatures.md
    ├── 03-decisions.md
    ├── 04-data-structures.md
    ├── 05-pipeline-style.md
    ├── 06-errors.md
    ├── 07-commands-and-subs.md
    ├── 08-task-ports.md
    ├── 09-differences-from-elm.md
    └── 10-finding-source.md
```

The repository root **is** the skill directory. Since agents discover skills as `<skills-root>/<name>/SKILL.md`, install it under a directory named `gren`.

## Install

The portable location — understood by Pi, OMP, OpenCode, and other Agent Skills implementations — is `.agents/skills/` (project) or `~/.agents/skills/` (user). Cloning directly there is the one-liner:

```bash
git clone git@github.com:dkoontz/gren-skill.git ~/.agents/skills/gren
```

If you keep this repo checked out elsewhere (e.g. you want to hack on the skill), symlink instead so edits are picked up in place:

```bash
ln -s ~/dev/gren-skill ~/.agents/skills/gren
```

### Pi

Pi implements the Agent Skills specification and scans `~/.agents/skills/` and `.agents/skills/` recursively, in addition to its own user/project skills directories (see [Pi skills docs](https://pi.dev/docs/latest/skills)).

```bash
# user-level
git clone git@github.com:dkoontz/gren-skill.git ~/.agents/skills/gren
# or project-level, from the repo root
git clone git@github.com:dkoontz/gren-skill.git .agents/skills/gren
```

At startup Pi advertises the skill's name and description; it reads the full `SKILL.md` only when a task matches. Force it explicitly with:

```text
/skill:gren help me fix this update function
```

Run `/reload` after editing the skill mid-session.

### OMP

OMP discovers skills from its native locations plus the Claude, Codex, Agents, and OpenCode layouts. Project-level:

```bash
git clone git@github.com:dkoontz/gren-skill.git .omp/skills/gren
# equivalently, via the shared convention:
git clone git@github.com:dkoontz/gren-skill.git .agents/skills/gren
```

User-level: clone into `~/.omp/agent/skills/gren`. The skill then appears in the system prompt's skill list; OMP (and the model) loads it with `read skill://gren`, and the bundled docs resolve as `skill://gren/docs/04-data-structures.md`. Force it interactively with `/skill:gren [args]`.

### OpenCode

OpenCode loads `SKILL.md` files on demand via its native `skill` tool (see [OpenCode skills docs](https://opencode.ai/docs/skills/)). It searches `.opencode/skills/`, `~/.config/opencode/skills/`, and the Claude- and Agent-compatible `.claude/skills/` / `.agents/skills/` roots (project and global).

```bash
# project-level
git clone git@github.com:dkoontz/gren-skill.git .opencode/skills/gren
# or global
git clone git@github.com:dkoontz/gren-skill.git ~/.config/opencode/skills/gren
```

OpenCode requires the frontmatter `name` to match the directory name — `gren` does. To also expose it as a `/gren` slash command, add this to the frontmatter of your installed copy:

```yaml
metadata:
  opencode/slash: "true"
```

Access can be gated per-agent with `permission.skill` patterns in `opencode.json`.

### Other agents

Any tool implementing the Agent Skills layout works by cloning into its expected root:

| Agent      | Project                    | User                          |
| ---------- | -------------------------- | ----------------------------- |
| Pi         | `.agents/skills/gren`      | `~/.agents/skills/gren`       |
| OMP        | `.omp/skills/gren`         | `~/.omp/agent/skills/gren`    |
| OpenCode   | `.opencode/skills/gren`    | `~/.config/opencode/skills/gren` |
| Claude Code| `.claude/skills/gren`      | `~/.claude/skills/gren`       |
| Codex      | `.agents/skills/gren`      | `~/.codex/skills/gren`        |

## Verify it loaded

From any of the above agents, ask something Gren-specific — "what does `when ... is` do in this `.gren` file?" — and check that the agent answers in Gren syntax (`when … is`, `Array`, `{}` unit) rather than Elm syntax. Most agents also list discovered skills in their startup diagnostics or support forcing the skill directly (`/skill:gren`, `skill({ name: "gren" })`, `read skill://gren`).

## License

GPL-3.0 — see [LICENSE.md](./LICENSE.md).
