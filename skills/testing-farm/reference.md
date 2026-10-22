# Testing Farm Onboarding Reference

## Testing Farm Overview

Testing Farm is a service maintained by Red Hat that provides testing infrastructure for running tests defined using TMT (Test Management Tool) with FMF (Flexible Metadata Format). It is available to Red Hat employees, Fedora/CentOS Stream contributors, and public projects co-maintained by Red Hat.

## API Instances

| Instance | API URL | Ranch | Who Can Use |
|----------|---------|-------|-------------|
| Public | `https://api.dev.testing-farm.io/v0.1` | public | Fedora/CentOS contributors, open-source projects |
| Red Hat Internal | `https://api.dev.testing-farm.io/v0.1` | redhat | Red Hat employees only |

## Onboarding & API Key

### Self-Service Token Creation

1. Visit https://docs.testing-farm.io and click "Sign in"
2. Authenticate via:
   - **Fedora SSO** (public ranch) — requires Fedora Account with CLA+1 status (contributor agreement + membership in `fedora-contributor` or `testing-farm` group)
   - **Red Hat SSO** (public + redhat ranches) — for Red Hat employees
3. Navigate to "Your Tokens" and create a new token
4. Each token is specific to a single ranch

### Service Account (for teams/CI)

Request an "Internal SSO (auth.redhat.com) Enabled Service Account" via IT IAM, then log in at testing-farm.io/tokens to generate a team token.

### Token Regeneration

Visit Testing Farm website → authenticate → "Your Tokens" → click "Regenerate" next to target token.

## TMT / FMF Basics

### Initialize TMT in a Repository

```bash
# Install tmt
pip install tmt

# Initialize fmf tree in repo root
tmt init

# Create a test plan
tmt plan create /plans/smoke

# Create a test
tmt test create /tests/smoke
```

### Minimal TMT Plan (`plans/smoke.fmf`)

```yaml
summary: Smoke tests
discover:
    how: fmf
execute:
    how: tmt
```

### TMT Plan with Shell Script (`plans/basic.fmf`)

```yaml
summary: Basic validation tests
discover:
    how: shell
    tests:
      - name: /smoke
        test: ./tests/smoke.sh
execute:
    how: tmt
```

### TMT Plan with Package Installation (`plans/full.fmf`)

```yaml
summary: Full test suite
discover:
    how: fmf
    filter: tier:1
prepare:
  - how: install
    package:
      - make
      - gcc
execute:
    how: tmt
```

### TMT Test Metadata (`tests/smoke/main.fmf`)

```yaml
summary: Smoke test for the application
test: ./test.sh
duration: 5m
require:
  - curl
```

### TMT Plan for Container Testing (`plans/container.fmf`)

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

## Testing Farm CLI

### Installation

```bash
pip install testing-farm
```

### Environment Variable

```bash
export TESTING_FARM_API_TOKEN="your-token-here"
```

### Basic Request

```bash
testing-farm request \
  --compose Fedora-latest \
  --git-url https://github.com/org/repo \
  --plan /plans/smoke
```

### Multi-Architecture Request

```bash
testing-farm request \
  --compose Fedora-Rawhide \
  --git-url https://github.com/org/repo \
  --plan /plans/smoke \
  --arch x86_64,aarch64
```

### List Available Composes

```bash
testing-farm composes
```

### List Requests

```bash
testing-farm list
```

### Cancel a Request

```bash
testing-farm cancel <request-id>
```

## GitHub Actions Integration

### Action: `sclorg/testing-farm-as-github-action@v4`

#### Required Permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  statuses: write
```

#### Required Secrets

- `TF_API_KEY` — Testing Farm API token (store in GitHub repo/org secrets)

#### Key Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `api_key` | Testing Farm API key | Required |
| `api_url` | API endpoint | `https://api.dev.testing-farm.io/v0.1` |
| `git_url` | Repo with test metadata | Current repo |
| `git_ref` | Git ref for tests | `github.ref` or `master` |
| `compose` | OS compose (e.g., `Fedora-latest`, `CentOS-Stream-9`) | `Fedora-latest` |
| `arch` | Target architecture | `x86_64` |
| `tmt_plan_regex` | Filter plans by regex | All plans |
| `tmt_plan_filter` | Advanced plan filtering | Empty |
| `tmt_context` | Context variables (`;` separated) | Empty |
| `tmt_path` | Metadata directory | `.` |
| `variables` | Env vars (`;` separated) | Empty |
| `secrets` | Env secrets (`;` separated) | Empty |
| `timeout` | Timeout in minutes | `480` |
| `update_pull_request_status` | Update PR checks | `false` |
| `pull_request_status_name` | Status check name | `Fedora` |
| `create_issue_comment` | Post summary comments | `false` |
| `create_github_summary` | Generate job summary | `true` |
| `copr` | COPR build identifier | Empty |
| `copr_artifacts` | COPR packages (`;` separated) | Empty |
| `environment_settings` | Custom env config (JSON) | Empty |
| `pipeline_settings` | Pipeline config (JSON) | Empty |
| `tmt_hardware` | Hardware requirements (JSON) | Empty |

#### Outputs

| Output | Description |
|--------|-------------|
| `request_id` | Testing Farm request ID |
| `request_url` | Request details URL |
| `test_log_url` | Test execution logs URL |

### Workflow: PR Comment Trigger

```yaml
name: Testing Farm - PR comment trigger
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
          tmt_plan_regex: "smoke"
          pull_request_status_name: "Testing Farm - Smoke"
          update_pull_request_status: "true"
```

### Workflow: Push to Main

```yaml
name: Testing Farm - push to main
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

### Workflow: PR Trigger

```yaml
name: Testing Farm - PR trigger
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
```

### Workflow: Matrix Build (Multiple Composes)

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
```

### Workflow: Container Provisioner (No Compose)

```yaml
name: Testing Farm - Container
on:
  push:
    branches: [main]

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: sclorg/testing-farm-as-github-action@v4
        with:
          api_key: ${{ secrets.TF_API_KEY }}
          compose: "null"
```

## GitLab CI Integration

### Using Testing Farm CLI Container Image

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

### GitLab CI with Merge Request Ref

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
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### GitLab CI with Multiple Composes (Parallel)

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

### GitLab Secret Setup

1. Navigate to your repository: **Settings → CI/CD → Variables → Add variable**
2. Set **Key** to `TESTING_FARM_API_TOKEN`
3. Set **Value** to your Testing Farm API token
4. Check **Mask variable** to prevent token leakage in logs
5. Optionally check **Protect variable** to limit to protected branches

### gitlab.cee.redhat.com (Red Hat Internal GitLab)

For Red Hat's internal GitLab instance:

1. Same CI/CD variable setup as public GitLab above
2. Use the **Red Hat ranch** API token (obtained via Red Hat SSO at testing-farm.io)
3. The Testing Farm CLI container image (`quay.io/testing-farm/cli:latest`) is accessible from internal runners
4. Red Hat ranch restriction: tests can only run from Red Hat-owned repositories or their forks
5. Internal compose names may differ (e.g., `RHEL-9.4.0-Nightly`, `RHEL-10.0-Nightly`)
6. For internal RHEL composes, the token must have `redhat` ranch access

## Common Composes

| Compose Name | Description |
|-------------|-------------|
| `Fedora-latest` | Latest stable Fedora |
| `Fedora-Rawhide` | Fedora development branch |
| `CentOS-Stream-9` | CentOS Stream 9 |
| `CentOS-Stream-10` | CentOS Stream 10 |
| `RHEL-9.x.0-Nightly` | RHEL 9.x nightly (Red Hat ranch only) |
| `RHEL-10.0-Nightly` | RHEL 10 nightly (Red Hat ranch only) |

Use `testing-farm composes` to list all available composes for your ranch.

## Troubleshooting

### Common Issues

1. **"Unauthorized" error**: Token is invalid or expired. Regenerate at testing-farm.io/tokens.
2. **"Ranch not found"**: Token is for a different ranch than the API URL targets.
3. **"No plans found"**: TMT metadata not initialized or `.fmf` directory missing.
4. **"Compose not available"**: Use `testing-farm composes` to list valid composes.
5. **Tests not discovered**: Check that `tmt run discover` works locally.
6. **GitLab variable not set**: Verify `TESTING_FARM_API_TOKEN` is configured in CI/CD settings.
7. **GitHub secret not set**: Verify `TF_API_KEY` is stored in repository or organization secrets.

### Verify TMT Locally

```bash
# List discovered tests
tmt run discover -v

# Dry-run a plan
tmt run --dry discover provision prepare execute

# Validate plans
tmt plan lint
tmt test lint
```

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
