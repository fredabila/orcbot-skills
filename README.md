# Agent Skills Repository

This repository contains a collection of skills designed to elevate Gemini CLI agents. Each skill is self-contained and follows the `SKILL.md` specification.

## Directory Structure

```
.
├── README.md
└── skills/
    ├── [skill-name]/
    │   ├── SKILL.md          # Core instructions and metadata
    │   ├── scripts/          # Optional: Executable scripts
    │   ├── references/       # Optional: Domain-specific documentation
    │   └── assets/           # Optional: Templates and static files
```

## How to Use

To use these skills with Gemini CLI:

1. Clone this repository.
2. Install a skill using the Gemini CLI:
   ```bash
   gemini skills install skills/[skill-name]
   ```
3. Reload your skills in the interactive session:
   ```
   /skills reload
   ```

## Contributing

Each skill must include a `SKILL.md` file with the following frontmatter:

```yaml
---
name: skill-name
description: Clear, concise description of what the skill does and when to use it.
---
```

Refer to the `skills/template-skill` for a starting point.
