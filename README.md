# Claude Skills

[![skills.sh](https://skills.sh/b/JiteshGaikwad/claude-skills)](https://skills.sh/JiteshGaikwad/claude-skills)

> Custom skills for [Claude Code](https://claude.ai/claude-code). For information about the Agent Skills standard, see [agentskills.io](http://agentskills.io).

Skills are folders of instructions and resources that Claude loads dynamically to improve performance on specialized tasks. Learn more:
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [Creating custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)

## Available Skills

| Skill | Description | Files |
|-------|-------------|-------|
| [amazon-connect](skills/amazon-connect/) | Amazon Connect contact center — flows, routing, Contact Lens, metrics, Customer Profiles, Cases, Q Connect, AI agents, Lambda integration, streaming, EventBridge, and AWS SDK v3 API reference | 116 |

## Installation

### All skills

```bash
git clone https://github.com/JiteshGaikwad/claude-skills.git ~/.claude/skills/claude-skills
```

### Single skill only

```bash
# Copy just the skill you need
cp -r skills/amazon-connect/ ~/.claude/skills/amazon-connect
```

### Project-level (single project)

```bash
git clone https://github.com/JiteshGaikwad/claude-skills.git .claude/skills/claude-skills
```

## Usage

Skills auto-trigger based on conversation context, or invoke manually:

```
/amazon-connect
```

## Repository Structure

```
claude-skills/
├── README.md
└── skills/
    ├── amazon-connect/
    │   ├── SKILL.md              # Entry point + routing index
    │   ├── core/                 # Instance, telephony, security
    │   ├── flows/                # Flow designer, blocks, Lambda
    │   ├── channels/             # Voice, chat, email, tasks
    │   ├── ai/                   # AI agents, Q Connect, Lex
    │   ├── analytics/            # Contact Lens, metrics, dashboards
    │   ├── streaming/            # Kinesis, EventBridge, agent events
    │   ├── agent-experience/     # Workspace, CCP, developer guide
    │   ├── data/                 # Customer Profiles, Cases
    │   ├── testing/              # Flow simulation
    │   ├── api/                  # All 9 service APIs + SDK patterns
    │   └── recent-changes.md     # Latest features
    └── <future-skill>/
        └── SKILL.md
```

## Adding New Skills

Create a new directory under `skills/` with a `SKILL.md`:

```
skills/my-new-skill/
└── SKILL.md
```

Frontmatter template:

```yaml
---
name: my-skill-name
description: A clear description of what this skill does and when to use it
---

# Skill Title

## Instructions
[Core behavior and approach]

## Examples
- Example usage 1
- Example usage 2

## Guidelines
- Guideline 1
- Guideline 2
```

## Note

These skills contain structured reference documentation derived from publicly available sources. No proprietary code or data is included.
