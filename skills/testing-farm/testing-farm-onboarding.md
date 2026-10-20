---
name: testing-farm-onboarding
description: Onboard repositories to Testing Farm for test execution. Use this skill when users want to integrate Testing Farm with GitHub, GitLab, or gitlab.cee.redhat.com repositories. Covers API token setup, TMT/FMF test metadata creation, GitHub Actions workflows, GitLab CI pipelines, and Testing Farm CLI configuration.
---

# Testing Farm Onboarding Skill

This skill helps developers onboard their repositories to Testing Farm for automated test execution. It guides users through obtaining API access, initializing TMT test metadata, and configuring CI pipelines on GitHub, GitLab, or Red Hat's internal GitLab (gitlab.cee.redhat.com).

## How to Use This Skill

### Option 1: Symlink into your project (recommended)

```bash
# Clone the ai-agents repository (once)
git clone https://github.com/sclorg/ai-agents.git ~/ai-agents

# In your project repository, symlink the skill directory
cd /path/to/your/project
mkdir -p .claude/skills
ln -s ~/ai-agents/skills/testing-farm .claude/skills/testing-farm
```

### Option 2: Copy into your project

```bash
# Copy the entire skill directory
cp -r ~/ai-agents/skills/testing-farm /path/to/your/project/.claude/skills/
```

### Option 3: Install into your global Claude Code skills

```bash
# Symlink into your user-level skills directory
ln -s ~/ai-agents/skills/testing-farm ~/.claude/skills/testing-farm-onboarding
```

### Invoking the Skill

Once installed, use any of these approaches in Claude Code:

```bash
# Invoke the skill directly by name
/testing-farm-onboarding

# Or describe what you want — Claude Code will pick up the skill automatically
claude "Onboard this repo to Testing Farm"
claude "Add Testing Farm CI to this GitHub repository"
claude "Set up Testing Farm for our GitLab project"
claude "Integrate Testing Farm with gitlab.cee.redhat.com"
```

The skill will present interactive choices for platform, trigger type, and OS composes before generating any configuration.

## When to Use This Skill

Use this skill when the user:
- Asks to onboard their repository to Testing Farm
- Wants to run tests via Testing Farm on GitHub, GitLab, or gitlab.cee.redhat.com
- Needs to set up TMT/FMF test plans for Testing Farm
- Wants to configure the `testing-farm-as-github-action` GitHub Action
- Needs to integrate Testing Farm into GitLab CI pipelines
- Asks about Testing Farm API keys or tokens
- Wants to add Testing Farm to pull request / merge request workflows

## Prerequisites

Before using this skill, verify:
- User has a repository with tests (or is ready to create test plans)
- User understands which CI platform they target (GitHub, GitLab, or gitlab.cee.redhat.com)
- For internal Red Hat GitLab: user is a Red Hat employee with Red Hat SSO access

## Instructions

### Step 0: Platform Selection (Interactive)

**IMPORTANT:** Before doing anything else, use the `AskUserQuestion` tool to present the user with interactive choices. This determines which CI configuration sections to generate and which token type is needed.

Call `AskUserQuestion` with these questions (all in a single call):

**Question 1 — Platform (multiSelect: true):**
- header: "Platform"
- question: "Which platform(s) do you want to integrate Testing Farm with?"
- multiSelect: true
- options:
  - label: "GitHub", description: "GitHub Actions workflow using sclorg/testing-farm-as-github-action@v4"
  - label: "GitLab", description: "GitLab CI pipeline (.gitlab-ci.yml) using Testing Farm CLI container image"
  - label: "gitlab.cee.redhat.com", description: "Red Hat internal GitLab CI with Red Hat ranch token and RHEL composes"

**Question 2 — Trigger:**
- header: "Trigger"
- question: "When should Testing Farm tests run?"
- multiSelect: false
- options:
  - label: "PR / MR events", description: "Trigger on pull request (GitHub) or merge request (GitLab) open/sync"
  - label: "PR comment", description: "Trigger when a maintainer comments '[test]' on a PR (GitHub only)"
  - label: "Push to main", description: "Trigger on every push to the main/default branch"
  - label: "All of the above", description: "Configure all trigger types in one workflow"

**Question 3 — Compose:**
- header: "Compose"
- question: "Which OS compose(s) should tests run on?"
- multiSelect: false
- options:
  - label: "Fedora-latest", description: "Latest stable Fedora release (public ranch)"
  - label: "CentOS Stream", description: "CentOS-Stream-9 and/or CentOS-Stream-10 (public ranch)"
  - label: "RHEL", description: "RHEL nightly composes — requires Red Hat ranch token"
  - label: "Multiple / Matrix", description: "Test across multiple composes in parallel (matrix build)"

Wait for the user's selections. Then proceed based on their choices:

- **Only generate CI configuration (Step 4) for the platform(s) the user selected.** Do NOT output GitHub Actions config if only GitLab was selected, and vice versa.
- If the user selected **gitlab.cee.redhat.com**, note that a **Red Hat ranch** token is required and RHEL composes are available.
- If the user selected **RHEL** composes, verify they have (or will get) a Red Hat ranch token.
- If the user selected **Multiple / Matrix**, generate matrix build configurations for the chosen platform(s).
- Adapt the trigger configuration to match their selection (PR, comment, push, or all).

### Step 0b: Additional Assessment

After the interactive selection, ask the user these remaining questions as text and wait for responses:

```
A few more questions before we begin:

1. **Testing Farm API Token**: Have you already obtained a Testing Farm API token?
   - If NO: I'll guide you through the onboarding process.
   - Public ranch: Sign in via Fedora SSO at testing-farm.io (requires Fedora Account with CLA+1)
   - Red Hat ranch: Sign in via Red Hat SSO at testing-farm.io

2. **TMT Metadata**: Does your repository already have TMT/FMF test metadata (a `.fmf` directory)?
   - If NO: I'll help you create test plans and tests.

3. **Existing Setup**: Do you already have any Testing Farm configuration (even partial)?
```

Wait for user responses before proceeding.

### Step 1: Analyze the Repository

Scan the repository to understand the current setup:

1. **Detect existing TMT metadata:**
   ```bash
   ls -la .fmf/ 2>/dev/null
   find . -name "*.fmf" -not -path "./.git/*" 2>/dev/null | head -20
   ```

2. **Detect existing test scripts:**
   ```bash
   find . -name "test*.sh" -o -name "test*.py" -not -path "./.git/*" 2>/dev/null | head -20
   ls -la tests/ test/ 2>/dev/null
   ```

3. **Detect programming language and build system:**
   ```bash
   ls -la Makefile go.mod package.json requirements.txt Cargo.toml setup.py pyproject.toml *.spec 2>/dev/null
   ```

4. **Check for existing CI configuration:**
   ```bash
   ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
   ls .gitlab-ci.yml 2>/dev/null
   ```

5. **Check for existing Testing Farm configuration:**
   ```bash
   grep -r "testing-farm" .github/workflows/ --include="*.yml" --include="*.yaml" 2>/dev/null
   grep -r "testing-farm\|TESTING_FARM" .gitlab-ci.yml 2>/dev/null
   ```

Present findings:

```
I've analyzed your repository. Here's what I found:

**Language/Build:** [detected]
**Existing TMT metadata:** [None / Found - describe]
**Existing tests:** [list discovered test files]
**CI System:** [GitHub Actions / GitLab CI / None]
**Existing Testing Farm setup:** [None / Partial - describe]

Is this correct? Let me know if I missed anything.
```

Wait for user confirmation.

### Step 2: Initialize TMT Metadata (If Missing)

If the repository does not have TMT/FMF metadata, help the user create it.

1. **Initialize the FMF tree:**
   ```bash
   tmt init
   ```
   This creates a `.fmf/version` file at the repository root.

2. **Create a basic test plan** based on the user's test type:

   **For shell-based smoke tests (`plans/smoke.fmf`):**
   ```yaml
   summary: Smoke tests
   discover:
       how: fmf
   execute:
       how: tmt
   ```

   **For tests with package dependencies (`plans/tests.fmf`):**
   ```yaml
   summary: Test suite
   discover:
       how: fmf
       filter: tier:1
   prepare:
     - how: install
       package:
         - <required-packages>
   execute:
       how: tmt
   ```

   **For container-based tests (`plans/container.fmf`):**
   ```yaml
   summary: Container tests
   provision:
       how: container
       image: quay.io/centos/centos:stream9
   discover:
       how: fmf
   execute:
       how: tmt
   ```

3. **Create test definitions** — for each test, create a directory with `main.fmf` and a test script:

   **`tests/smoke/main.fmf`:**
   ```yaml
   summary: Smoke test
   test: ./test.sh
   duration: 5m
   ```

   **`tests/smoke/test.sh`:**
   ```bash
   #!/bin/bash
   set -euo pipefail
   # Add actual test commands here
   echo "Smoke test passed"
   ```

   Make test scripts executable:
   ```bash
   chmod +x tests/smoke/test.sh
   ```

4. **Verify TMT metadata locally:**
   ```bash
   tmt plan lint
   tmt test lint
   tmt run discover -v
   ```

   If `tmt` is not installed, suggest:
   ```bash
   pip install tmt
   ```

Present the created files and wait for user confirmation before proceeding.

### Step 3: Verify Tests Locally with Testing Farm CLI (Optional)

If the user has a Testing Farm API token, recommend a local test run:

```
Before configuring CI, I recommend verifying tests work via Testing Farm CLI:

1. Install the CLI:
   pip install testing-farm

2. Export your token:
   export TESTING_FARM_API_TOKEN="your-token-here"

3. Submit a test request:
   testing-farm request \
     --compose Fedora-latest \
     --git-url <your-repo-url> \
     --git-ref <your-branch> \
     --plan /plans/smoke

4. Check available composes:
   testing-farm composes

Have you verified the tests run successfully? (yes/no/skip)
```

Wait for response. If tests fail, help debug before proceeding to CI configuration.

### Step 4: Configure CI Pipeline

**IMPORTANT:** Only generate configuration for the platform(s) the user selected in Step 0. Skip sections for platforms they did NOT select.

#### Option A: GitHub Actions (only if user selected "GitHub" in Step 0)

Determine the trigger type from Step 0 and create the workflow accordingly.

**For PR trigger (most common):**

Create `.github/workflows/testing-farm.yml`:

```yaml
name: Testing Farm
on:
  pull_request_target:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  statuses: write

concurrency:
  group: testing-farm-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: sclorg/testing-farm-as-github-action@v4
        with:
          api_key: ${{ secrets.TF_API_KEY }}
          compose: Fedora-latest
          update_pull_request_status: "true"
          pull_request_status_name: "Testing Farm"
          create_github_summary: "true"
```

**For PR comment trigger (e.g., "[test]"):**

```yaml
name: Testing Farm
on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write
  statuses: write

concurrency:
  group: testing-farm-${{ github.event.issue.number }}
  cancel-in-progress: true

jobs:
  tests:
    runs-on: ubuntu-latest
    if: |
      github.event.issue.pull_request
      && contains(github.event.comment.body, '[test]')
      && contains(fromJson('["OWNER", "MEMBER"]'),
        github.event.comment.author_association)
    steps:
      - uses: sclorg/testing-farm-as-github-action@v4
        with:
          api_key: ${{ secrets.TF_API_KEY }}
          compose: Fedora-latest
          tmt_plan_regex: "smoke"
          update_pull_request_status: "true"
          pull_request_status_name: "Testing Farm - Smoke"
          create_github_summary: "true"
```

**For push to main:**

```yaml
name: Testing Farm - Main
on:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: testing-farm-main
  cancel-in-progress: true

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: sclorg/testing-farm-as-github-action@v4
        with:
          api_key: ${{ secrets.TF_API_KEY }}
          compose: Fedora-latest
          create_github_summary: "true"
```

**For matrix builds (multiple composes):**

```yaml
name: Testing Farm - Matrix
on:
  pull_request_target:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  statuses: write

jobs:
  tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        compose:
          - Fedora-latest
          - CentOS-Stream-9
          - CentOS-Stream-10
    steps:
      - uses: sclorg/testing-farm-as-github-action@v4
        with:
          api_key: ${{ secrets.TF_API_KEY }}
          compose: ${{ matrix.compose }}
          update_pull_request_status: "true"
          pull_request_status_name: "Testing Farm - ${{ matrix.compose }}"
          create_github_summary: "true"
```

**GitHub Secrets Setup:**

```
You need to add your Testing Farm API token as a GitHub secret:

1. Go to your repository → Settings → Secrets and variables → Actions
2. Click "New repository secret" (or "New organization secret" for org-wide)
3. Name: TF_API_KEY
4. Value: <your Testing Farm API token>
5. Click "Add secret"
```

#### Option B: GitLab CI (only if user selected "GitLab" in Step 0)

Add a Testing Farm job to `.gitlab-ci.yml`:

**Basic configuration:**

```yaml
stages:
  - test

testing-farm:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  script:
    - testing-farm request
      --compose Fedora-latest
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**With merge request ref (for accurate MR testing):**

```yaml
testing-farm-mr:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  script:
    - testing-farm request
      --git-ref $CI_MERGE_REQUEST_REF_PATH
      --git-url $CI_MERGE_REQUEST_PROJECT_URL
      --plan /plans/smoke
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Multiple composes (parallel jobs):**

```yaml
.testing-farm-base:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

testing-farm-fedora:
  extends: .testing-farm-base
  script:
    - testing-farm request
      --compose Fedora-latest
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke

testing-farm-centos:
  extends: .testing-farm-base
  script:
    - testing-farm request
      --compose CentOS-Stream-9
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke
```

**GitLab CI/CD Variable Setup:**

```
Add your Testing Farm API token as a GitLab CI/CD variable:

1. Navigate to your project → Settings → CI/CD → Variables
2. Click "Add variable"
3. Key: TESTING_FARM_API_TOKEN
4. Value: <your Testing Farm API token>
5. Check "Mask variable" (prevents token from appearing in logs)
6. Optionally check "Protect variable" (limits to protected branches)
7. Click "Add variable"
```

#### Option C: gitlab.cee.redhat.com (only if user selected "gitlab.cee.redhat.com" in Step 0)

Configuration is similar to public GitLab but with Red Hat-specific considerations:

**Important differences:**
- Use a **Red Hat ranch** API token (obtained via Red Hat SSO at testing-farm.io)
- Red Hat ranch can only execute tests from Red Hat-owned repositories or their forks
- Internal RHEL composes are available (e.g., `RHEL-9.4.0-Nightly`, `RHEL-10.0-Nightly`)
- The CLI container image `quay.io/testing-farm/cli:latest` is accessible from internal runners

**Configuration for `.gitlab-ci.yml`:**

```yaml
stages:
  - test

testing-farm:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  script:
    - testing-farm request
      --compose CentOS-Stream-9
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**For RHEL testing (Red Hat ranch only):**

```yaml
testing-farm-rhel:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  script:
    - testing-farm request
      --compose RHEL-9.4.0-Nightly
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**Multiple composes including RHEL:**

```yaml
.testing-farm-base:
  stage: test
  image: quay.io/testing-farm/cli:latest
  variables:
    TESTING_FARM_API_TOKEN: $TESTING_FARM_API_TOKEN
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

testing-farm-centos9:
  extends: .testing-farm-base
  script:
    - testing-farm request
      --compose CentOS-Stream-9
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke

testing-farm-centos10:
  extends: .testing-farm-base
  script:
    - testing-farm request
      --compose CentOS-Stream-10
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke

testing-farm-rhel9:
  extends: .testing-farm-base
  script:
    - testing-farm request
      --compose RHEL-9.4.0-Nightly
      --git-url $CI_PROJECT_URL
      --git-ref $CI_COMMIT_REF_NAME
      --plan /plans/smoke
```

**Secret setup for gitlab.cee.redhat.com:**

```
Add your Testing Farm Red Hat ranch API token:

1. Navigate to your project → Settings → CI/CD → Variables
2. Click "Add variable"
3. Key: TESTING_FARM_API_TOKEN
4. Value: <your Red Hat ranch Testing Farm API token>
5. Check "Mask variable"
6. Check "Protect variable" (recommended for internal repos)
7. Click "Add variable"

Note: Ensure your token is a Red Hat ranch token obtained via Red Hat SSO.
RHEL composes are only available with Red Hat ranch tokens.
```

### Step 5: Handle Partial Setup

If the repository already has some Testing Farm configuration:

**Scenario A: TMT metadata exists but no CI integration**
- Skip Step 2
- Verify metadata with `tmt plan lint` and `tmt test lint`
- Proceed to Step 4 for CI configuration

**Scenario B: CI integration exists but missing triggers**
- Review existing workflow/pipeline
- Add missing triggers (e.g., add push-to-main if only PR exists)
- Ensure concurrency groups are set

**Scenario C: GitHub Action using older version**
- Recommend upgrading to `sclorg/testing-farm-as-github-action@v4`
- Review changed inputs between versions

**Scenario D: Already fully configured**
- Ask what the user wants to change or improve
- Common enhancements: add composes, add PR status updates, add matrix builds

### Step 6: Verification Checklist

Present only the checklist items relevant to the platform(s) the user selected in Step 0.

```
Here's your verification checklist:

**Immediate:**
□ Review the changes to ensure they look correct
□ Verify TMT metadata: tmt plan lint && tmt test lint
□ Verify test discovery: tmt run discover -v

**After pushing:**
□ Open a PR / MR with the changes
□ Verify the Testing Farm job triggers
□ Check CI logs for successful test submission
□ Check the Testing Farm request URL in the logs for test results

**For GitHub (if selected):**
□ Verify PR status check appears (if update_pull_request_status enabled)
□ Verify GitHub Job Summary is generated
□ Check that TF_API_KEY secret is accessible to the workflow

**For GitLab / gitlab.cee.redhat.com (if selected):**
□ Verify TESTING_FARM_API_TOKEN variable is set in CI/CD settings
□ Verify the Testing Farm CLI container image is accessible
□ Check pipeline logs for "Request submitted" and request URL
```

## Best Practices

1. **Verify TMT metadata locally first** — run `tmt plan lint` and `tmt run discover` before CI
2. **Start with one compose** — add more after the basic setup works
3. **Use concurrency groups** — prevent parallel test runs for the same PR
4. **Set reasonable timeouts** — default is 480 minutes; reduce for fast tests
5. **Use PR status updates on GitHub** — gives immediate feedback on PRs
6. **Mask CI/CD variables** — always mask the API token in GitLab
7. **Use `pull_request_target` on GitHub** — needed for fork PRs to access secrets
8. **Propose, don't assume** — show proposed changes and wait for confirmation
9. **Keep existing CI logic** — don't break existing workflows/pipelines

## Reference

For detailed configuration options, CLI commands, workflow examples, TMT plan formats, and troubleshooting, see [reference.md](reference.md).

## Documentation Links

- Testing Farm Docs: https://docs.testing-farm.io
- Testing Farm Onboarding: https://docs.testing-farm.io/Testing%20Farm/0.1/onboarding.html
- Testing Farm Integrations: https://docs.testing-farm.io/Testing%20Farm/0.1/integrations.html
- Testing Farm CLI: https://docs.testing-farm.io/Testing%20Farm/0.1/cli.html
- Testing Farm API: https://testing-farm.gitlab.io/api/
- GitHub Action: https://github.com/sclorg/testing-farm-as-github-action
- TMT Documentation: https://tmt.readthedocs.io
- FMF Format: https://fmf.readthedocs.io
- Packit + Testing Farm: https://packit.dev/docs/configuration/upstream/tests
