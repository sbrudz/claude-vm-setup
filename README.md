# Claude VM Setup

[Agent Skills](https://agentskills.io) for bootstrapping a Linux VM for development with Claude Code.

For code quality, architecture, and UX design skills, see [agent-skills](https://github.com/sbrudz/agent-skills).

## Quick Start

Paste this prompt into a fresh Claude Code session to bootstrap the VM:

```
Bootstrap this VM for development.

Step 1: Install plugin collections by running these commands:

claude plugin marketplace add obra/superpowers-marketplace
claude plugin install superpowers@superpowers-marketplace
claude plugin marketplace add sbrudz/claude-vm-setup
claude plugin install claude-vm-setup@claude-vm-setup
claude plugin marketplace add sbrudz/agent-skills
claude plugin install agent-skills@sbrudz-skills

Step 2: Configure GitHub credentials. Fetch and follow the skill instructions at:
https://raw.githubusercontent.com/sbrudz/claude-vm-setup/main/skills/configuring-github-credentials/SKILL.md

Also fetch the package manager reference at:
https://raw.githubusercontent.com/sbrudz/claude-vm-setup/main/skills/configuring-github-credentials/references/package-managers.md

Step 3: After everything is complete, tell me to restart Claude Code so all plugins take effect.
```

This installs three plugin collections ([superpowers](https://github.com/obra/superpowers), [claude-vm-setup](https://github.com/sbrudz/claude-vm-setup), [agent-skills](https://github.com/sbrudz/agent-skills)), then configures git, GitHub CLI, and SSH keys. The process is interactive — Claude will prompt you for your GitHub username and guide you through authentication.

## Available skills

| Skill | Description |
|-------|-------------|
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
/install-skill https://github.com/sbrudz/claude-vm-setup/tree/main/skills/configuring-github-credentials
```

### Other agents

Any agent that supports the [Agent Skills specification](https://agentskills.io/specification) can use these skills. Point the agent at the `SKILL.md` file in the relevant skill directory.

## License

MIT
