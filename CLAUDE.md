# Mattermost Delivery Skills Marketplace

Shared Claude Code skills for Mattermost staff and partners managing customer delivery projects.

## Repository Structure

```
mattermost-delivery-skills/
├── CLAUDE.md                          # This file
├── skills/                            # Marketplace skills directory
│   ├── .claude-plugin/
│   │   └── marketplace.json           # Skill registry
│   └── rocketlane/                    # Skill directories (nested structure)
│       └── rocketlane/
│           ├── SKILL.md
│           └── references/
└── scripts/                           # Validation scripts
```

## Skill Directory Structure

Skills use the standard nested structure:

```
skills/
└── skill-name/
    └── skill-name/
        ├── SKILL.md
        ├── scripts/
        ├── references/
        └── assets/
```

## Adding a New Skill

1. Create nested directory: `mkdir -p skills/my-skill/my-skill/`
2. Create `SKILL.md` with YAML frontmatter
3. Register in `skills/.claude-plugin/marketplace.json`
4. Commit both skill and marketplace.json together

## Available Skills

### rocketlane
Manage Rocketlane projects and tasks via REST API. Create, update, list, and delete tasks with phase and visibility control. Designed for managing customer delivery projects.
