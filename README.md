# Agent Skills

A public collection of portable **agent skills** — reusable instruction sets that teach an AI coding agent how to perform a specific task well.

Every skill is a single self-contained [`SKILL.md`](https://agentskills.io) file: YAML frontmatter that tells the agent *when* to use the skill, followed by Markdown that tells it *how*. The format itself is plain text, so any agent that can discover and read `SKILL.md` files can load these.

Individual skills still have their own requirements — see the table below. The recommended installer also needs Node.js for `npx`.

## Skills

| Skill | Description | External requirements |
| --- | --- | --- |
| [`codex-imagegen`](skills/codex-imagegen/SKILL.md) | Generates and edits raster images by delegating to a locally authenticated Codex CLI. Produces real bitmaps instead of SVG/HTML placeholders, and picks the output location from the project's existing structure. | [Codex CLI](https://developers.openai.com/codex/cli) 0.150.1+ (logged in) |

## Installation

### Using the `skills` CLI (recommended)

The [`skills`](https://github.com/vercel-labs/skills) CLI detects which agents you have installed and links the skill into each one:

```bash
npx skills add mizumotok/agent-skills
```

Useful flags:

```bash
npx skills add mizumotok/agent-skills -l                # list available skills without installing
npx skills add mizumotok/agent-skills -s codex-imagegen # install a single skill
npx skills add mizumotok/agent-skills -g                # install globally instead of per project
npx skills update codex-imagegen                        # pull in later updates
```

### Try a skill without installing it

`use` prints a ready-to-paste prompt containing the skill, so you can try it in any agent — including ones with no skills directory at all:

```bash
npx skills use mizumotok/agent-skills@codex-imagegen
```

### Manual install

Copy the skill directory into wherever your agent discovers skills — pick the path for your agent from the table below. The directory name is the skill name, and `SKILL.md` must stay at its root.

```bash
git clone https://github.com/mizumotok/agent-skills.git
mkdir -p <YOUR_AGENT_SKILLS_DIR>
cp -r agent-skills/skills/codex-imagegen <YOUR_AGENT_SKILLS_DIR>/
```

Common locations:

| Agent | Per project | User-wide |
| --- | --- | --- |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `.agents/skills/` | `~/.codex/skills/` (also `~/.agents/skills/`) |
| Cursor | `.agents/skills/` | `~/.cursor/skills/` |
| GitHub Copilot | `.agents/skills/` | `~/.copilot/skills/` |
| Gemini CLI | `.agents/skills/` | `~/.gemini/skills/` |

`.agents/skills/` is the shared project-level convention across most agents. Commit it to share the skill with your team.

The `skills` CLI above knows the path for every agent it supports and is the easier route. If yours isn't listed, check its documentation for the skills directory — the file itself needs no changes.

### As a git submodule

To pin a specific revision in a team repository and update it deliberately:

```bash
git submodule add https://github.com/mizumotok/agent-skills.git vendor/agent-skills
mkdir -p .agents/skills
ln -s ../../vendor/agent-skills/skills/codex-imagegen .agents/skills/codex-imagegen
```

Commit both the submodule and the symlink. Anyone cloning the repository afterwards needs the submodule contents:

```bash
git clone --recurse-submodules <your-repo>   # or, in an existing clone:
git submodule update --init
```

<details>
<summary>Claude Code: install the whole repository as a plugin</summary>

The repository ships plugin manifests under `.claude-plugin/`, so Claude Code can install every skill at once and keep them updated.

In a session:

```
/plugin marketplace add mizumotok/agent-skills
/plugin install agent-skills@mizumotok
```

Or from the terminal:

```bash
claude plugin marketplace add mizumotok/agent-skills
claude plugin install agent-skills@mizumotok
claude plugin marketplace update mizumotok
```

Skills installed this way are namespaced as `/agent-skills:codex-imagegen`.

</details>

## Usage

A skill triggers in two ways:

- **Automatically** — the agent matches your request against the skill's `description` and loads it on its own.
  For example: "Create a hero image for the landing page, 16:9, no text."
- **Explicitly** — you name the skill directly. The syntax depends on the host:

| Host | Invocation |
| --- | --- |
| Claude Code | `/codex-imagegen A futuristic AI office with a Mac Studio on a black background. 16:9. No text.` |
| Claude Code (as a plugin) | `/agent-skills:codex-imagegen ...` |
| Codex | `$codex-imagegen ...` |

## License

[MIT](LICENSE)
