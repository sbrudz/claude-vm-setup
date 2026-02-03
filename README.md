# Claude VM Setup

[Agent Skills](https://agentskills.io) for bootstrapping a Linux VM for development with Claude Code.

For code quality, architecture, and UX design skills, see [dev-ethos](https://github.com/sbrudz/dev-ethos).

## Available skills

| Skill | Description |
|-------|-------------|
| [bootstrapping-vm](skills/bootstrapping-vm/SKILL.md) | Bootstraps a Linux VM for development with Claude Code by installing superpowers and configuring GitHub credentials. |
| [installing-superpowers](skills/installing-superpowers/SKILL.md) | Checks if the obra/superpowers plugin collection is installed in Claude Code and installs it if missing. |
| [configuring-github-credentials](skills/configuring-github-credentials/SKILL.md) | Configures a Linux VM with git, GitHub CLI, SSH keys, and GitHub authentication. Distro-aware package installation. |

## Installation

### Claude Code (plugin marketplace)

```
/plugin marketplace add sbrudz/claude-vm-setup
/plugin install claude-vm-setup@claude-vm-setup
```

### Install a single skill

Use the `/install-skill` command in Claude Code and provide the GitHub URL to the skill directory, e.g.:

```
/install-skill https://github.com/sbrudz/claude-vm-setup/tree/main/skills/bootstrapping-vm
```

### Other agents

Any agent that supports the [Agent Skills specification](https://agentskills.io/specification) can use these skills. Point the agent at the `SKILL.md` file in the relevant skill directory.

## License

MIT
