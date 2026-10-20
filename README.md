# AI Agents - Claude Code Skills

A collection of reusable Claude Code skills for automating development workflows.

## What's Inside

```
ai-agents/
└── skills/
    ├── sclorg/
    │   └── add-distgen-version.md        # Automate adding versions to sclorg containers
    └── testing-farm/
        ├── testing-farm-onboarding.md    # Onboard repos to Testing Farm (GitHub, GitLab, gitlab.cee.redhat.com)
        └── reference.md                  # Testing Farm configuration reference
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

### `testing-farm/testing-farm-onboarding`

**Purpose:** Onboard repositories to Testing Farm for automated test execution

**Works with:** GitHub, GitLab, and gitlab.cee.redhat.com (Red Hat internal GitLab)

**Install:**
```bash
# Symlink the entire skill directory into your project
cd /path/to/your/project
mkdir -p .claude/skills
ln -s ~/ai-agents/skills/testing-farm .claude/skills/testing-farm

# Or copy it
cp -r ~/ai-agents/skills/testing-farm /path/to/your/project/.claude/skills/

# Or install globally for all projects
ln -s ~/ai-agents/skills/testing-farm ~/.claude/skills/testing-farm-onboarding
```

**Usage:**
```bash
# Invoke directly by name:
/testing-farm-onboarding

# Or describe what you want — Claude Code picks up the skill automatically:
claude "Onboard this repo to Testing Farm"
claude "Add Testing Farm CI to this GitHub repository"
claude "Set up Testing Farm for our GitLab project"
claude "Integrate Testing Farm with gitlab.cee.redhat.com"
```

**What it does:**
1. Presents interactive platform/trigger/compose selection
2. Analyzes the repository for existing TMT metadata and CI config
3. Initializes TMT/FMF test plans and test definitions if missing
4. Configures GitHub Actions (`sclorg/testing-farm-as-github-action@v4`) or GitLab CI pipelines
5. Guides secret/token setup for each platform
6. Supports matrix builds across multiple OS composes (Fedora, CentOS Stream, RHEL)

