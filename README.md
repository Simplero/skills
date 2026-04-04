# LLM Skills for Simplero

AI coding agent skills for building on [Simplero](https://simplero.com). Works with Claude Code, Codex, and other agents that support skills/plugins.

## Available Skills

| Skill | Description |
|-------|-------------|
| `simplero-page` | Build and manage landing pages on Simplero via the API |
| `import-course` | Import courses from any platform (Kajabi, Skool, Teachable, etc.) into Simplero |

## Installation

### Claude Code

Add the Simplero marketplace and install the plugin:

```bash
claude plugin marketplace add simplero/skills
claude plugin install simplero-skills
```

After installation, the skills are available as slash commands. Type `/simplero-page` in Claude Code to use it.

### Codex (OpenAI)

Clone the repo and point Codex at the skills directory:

```bash
git clone https://github.com/simplero/skills.git ~/.simplero-skills
```

Then add the skill content to your `AGENTS.md` or system prompt. Copy the contents of `skills/simplero-page/SKILL.md` into your agent configuration.

### Other AI Agents

Any agent that supports markdown-based skill files can use these. Each skill lives in its own directory under `skills/` with a `SKILL.md` file containing the instructions and metadata. Copy or reference the `SKILL.md` content in whatever way your agent supports custom instructions.

## Repository Structure

```
skills/
└── <skill-name>/
    └── SKILL.md          # Skill definition with frontmatter + instructions
.claude-plugin/
└── plugin.json           # Plugin metadata for Claude Code
```

Each `SKILL.md` file has YAML frontmatter with the skill name, description, and version, followed by the instructions the agent follows when the skill is invoked.

## Contributing

To add a new skill:

1. Create a new directory under `skills/` with your skill name (kebab-case)
2. Add a `SKILL.md` with frontmatter and instructions:
   ```markdown
   ---
   name: my-skill-name
   version: 0.1.0
   description: When to trigger this skill.
   user-invocable: true
   ---

   Your skill instructions here.
   ```
3. Open a pull request

## License

MIT
