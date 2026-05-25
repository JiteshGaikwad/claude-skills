# Claude Code Skills

A collection of custom [Claude Code](https://claude.ai/claude-code) skills for specialized domains.

## Available Skills

| Skill | Description | Files |
|-------|-------------|-------|
| [amazon-connect](amazon-connect/) | Amazon Connect contact center — flows, routing, Contact Lens, metrics, Customer Profiles, Cases, Q Connect, AI agents, Lambda integration, streaming, EventBridge, and AWS SDK v3 API reference | 59 |

## Installation

### Option 1: Personal skills (all projects)

```bash
# Clone the entire repo into your personal skills directory
git clone https://github.com/<your-username>/claude-skills.git ~/.claude/skills/claude-skills
```

### Option 2: Project-level (single project)

```bash
# Clone into your project's .claude/skills directory
git clone https://github.com/<your-username>/claude-skills.git .claude/skills/claude-skills
```

### Option 3: Single skill only

```bash
# Copy just the skill you need
cp -r amazon-connect/ ~/.claude/skills/amazon-connect
```

## Usage

Skills auto-trigger based on conversation context, or invoke manually:

```
/amazon-connect
```

## Structure

Each skill is a self-contained directory with a `SKILL.md` entry point:

```
claude-skills/
├── README.md
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

Create a new directory at the repo root with a `SKILL.md`:

```
my-new-skill/
└── SKILL.md
```

See [Claude Code skill docs](https://docs.anthropic.com/en/docs/claude-code/skills) for the SKILL.md format.

## License

MIT
