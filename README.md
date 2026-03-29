# domain-modeling

A Claude Code skill for defining data models and core business processes.

## What it does

Guides you through 6 steps to clarify:
1. What entities the system stores
2. How entities relate to each other
3. State machines and lifecycle transitions
4. Sub-entities, derived fields, and business rules
5. Core business processes with rollback logic
6. A complete model summary document

## Installation

```bash
# Add this plugin repository to Claude Code
/plugin marketplace add your-username/domain-modeling-plugin

# Install the skill
/plugin install domain-modeling
```

Or manually:
```bash
mkdir -p ~/.claude/skills/domain-modeling
cp -r skills/domain-modeling/* ~/.claude/skills/domain-modeling/
```

## Usage

```
/domain-modeling
```

Works for any project regardless of language or framework.

## Trigger

- Slash command: `/domain-modeling`

## Tags

- modeling
- data-model
- architecture
- process
