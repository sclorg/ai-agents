# AI Agents - Claude Code Skills

A collection of reusable Claude Code skills for automating development workflows.

## What's Inside

```
ai-agents/
└── skills/
    └── sclorg/
        └── add-distgen-version.md   # Automate adding versions to sclorg containers
```

## Quick Start - Local Usage

### Install a Skill in Your Project

```bash
# Clone this repository
git clone https://github.com/sclorg/ai-agents.git ~/ai-agents

# In your project, create a symlink
cd /path/to/your/project
mkdir -p .claude/skills
ln -s ~/ai-agents/skills/sclorg/add-distgen-version.md .claude/skills/

# Now use it with Claude Code
claude "Add Node.js 26 support"
```

### Alternative: Copy the Skill

```bash
# Just copy the skill file
cp ~/ai-agents/skills/sclorg/add-distgen-version.md /path/to/your/project/.claude/skills/
```

## Available Skills

### `sclorg/add-distgen-version`

**Purpose:** Add a new version to distgen-enabled sclorg container repositories

**Works with:** Node.js, PostgreSQL, MariaDB, Redis, and other sclorg containers using distgen

**Usage:** 
```bash
# In Claude Code, after symlinking the skill:
claude "Add version 26 to this repository"
# Or invoke directly:
claude "/add-distgen-version 26"
```

**What it does:**
1. Reads your existing `specs/multispec.yml` configuration
2. Adds new version definition (standard + minimal variants)
3. Updates the distribution matrix
4. Generates Dockerfiles for all distros (RHEL, CentOS, Fedora)
5. Updates Makefile and verification scripts
6. Runs integration tests

