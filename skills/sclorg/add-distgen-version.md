---
name: add-distgen-version
description: Add a new version to distgen-enabled sclorg container repositories
trigger: always
---

# Add New Version to Distgen-Enabled Container

This skill helps add a new version to any sclorg container repository that uses distgen for multi-distro support (e.g., nodejs, postgresql, mariadb, redis, etc.).

## Prerequisites

Before starting, gather the following information from the user:

1. **New version number** (e.g., "26" for Node.js 26)
2. **Previous version number** to use as a template (e.g., "24")
3. **Whether minimal variant is needed** (yes/no)
4. **REQUIRED: Fedora version to use** - **ASK THE USER** which Fedora version to target
   - Do NOT assume or copy from existing versions
   - Check current Fedora release: https://fedoraproject.org/
   - Explicitly ask: "Which Fedora version should I use for this new version? (e.g., fedora-45, fedora-46)"
   - Wait for user confirmation before proceeding
5. **Package-specific information**:
   - Package names for different distros (if they differ from previous version)
   - Build dependencies (if they changed)
   - Any version-specific requirements or constraints

## Distgen Structure Overview

Distgen-enabled repositories have this structure:

```
repository/
├── specs/
│   └── multispec.yml          # Version and distro definitions
├── manifest.yml               # File generation rules for standard images
├── manifest-minimal.yml       # File generation rules for minimal images (if applicable)
├── src/                       # Template files with Jinja2 syntax
│   ├── Dockerfile             # Main Dockerfile template
│   ├── Dockerfile.minimal     # Minimal Dockerfile template (if applicable)
│   ├── README.md             # README template
│   └── ...                   # Other template files
├── <version>/                # Generated directories for each version
│   ├── Dockerfile.rhel8
│   ├── Dockerfile.rhel9
│   ├── Dockerfile.c9s
│   └── ...
└── verify-distgen-integration.sh  # Verification script
```

## Step-by-Step Process

### 1. Read and Understand Current Configuration

First, read the existing configuration to understand the repository structure:

```bash
# Read the multispec configuration
Read specs/multispec.yml

# Read manifest files
Read manifest.yml
# If minimal images exist:
Read manifest-minimal.yml
```

### 2. Identify the Previous Version Configuration

Find the most recent version definition in `specs/multispec.yml` that will serve as a template. Look under the `specs.version` section for the previous version (e.g., "24") and note:

- Package names (`pkgs`)
- Build dependencies (`build_deps`)
- Minimal packages (`minimal_pkgs`) if applicable
- Base image references
- Special flags or features

### 3. Add New Fedora Distroinfo (if needed)

**IMPORTANT:** If the user-confirmed Fedora version is newer than any existing `fedoraXX` distroinfo section, you MUST create a new one.

For example, if the file has `fedora43` but the user confirmed Fedora 45, add a new `fedora45` section:

```yaml
specs:
  distroinfo:
    # ... existing distros ...
    
    fedora<USER_CONFIRMED_VERSION>:
      distros:
        - fedora-<USER_CONFIRMED_VERSION>-x86_64
      s2i_base: "{{ spec.fedora_base }}"
      s2i_minimal_base: "quay.io/fedora/fedora-minimal:<USER_CONFIRMED_VERSION>"
      fedora_minimal_base: "quay.io/fedora/fedora-minimal:<USER_CONFIRMED_VERSION>"
      full_image_name: "quay.io/fedora/$NAME-$NODEJS_VERSION:latest"
      org: "fedora"
      prod: "fedora"
      redhat_component: "${NAME}-${NODEJS_VERSION}-container"
      img_name: "fedora/$NAME-$NODEJS_VERSION"
      img_name_minimal: "fedora/$NAME-$NODEJS_VERSION-minimal"
      latest_fedora: "<USER_CONFIRMED_VERSION>"
      needs_versioned_links: true
      # Modern: Fedora uses versioned packages (nodejs$VERSION), not modules
```

### 4. Add New Version Definition

Add the new version to `specs/multispec.yml` in the `specs.version` section:

**For standard variant:**
```yaml
"<NEW_VERSION>":
  version: "<NEW_VERSION>"
  prev_version: "<PREV_VERSION>"
  short: "<NEW_VERSION>"
  prev_short: "<PREV_VERSION>"
  common_image_name: "{{ spec.org }}/<PACKAGE_NAME>-{{ spec.short }}-{{ spec.prod }}"
  c9s_image_name: "sclorg/<PACKAGE_NAME>-{{ spec.short }}-c9s"
  rhel_image_name: "rhel9/<PACKAGE_NAME>-{{ spec.short }}"
  latest_fedora: "f<USER_CONFIRMED_VERSION>"  # Use user-confirmed Fedora version
  fedora_base: "quay.io/fedora/s2i-core:<USER_CONFIRMED_VERSION>"  # Use user-confirmed Fedora version
  pkg_manager: "dnf"
  pkgs: >-
    {% if spec.prod in ['rhel10', 'c10s', 'fedora'] -%}
      # List versioned packages for modern distros
    {%- else -%}
      # List non-versioned packages for legacy distros
    {%- endif %}
  build_deps: >-
    {% if spec.prod == 'rhel8' -%}
      # RHEL8-specific build deps
    {%- elif spec.prod == 'fedora' -%}
      # Fedora-specific build deps
    {%- elif spec.prod in ['rhel9', 'c9s'] -%}
      # RHEL9/C9S-specific build deps
    {%- else -%}
      # Default build deps
    {%- endif %}
```

**For minimal variant (if applicable):**
```yaml
"<NEW_VERSION>-minimal":
  version: "<NEW_VERSION>"
  prev_version: "<PREV_VERSION>"
  short: "<NEW_VERSION>-minimal"
  prev_short: "<PREV_VERSION>-minimal"
  common_image_name: "{{ spec.org }}/<PACKAGE_NAME>-{{ spec.short }}-{{ spec.prod }}"
  c9s_image_name: "sclorg/<PACKAGE_NAME>-{{ spec.short }}-c9s"
  rhel_image_name: "rhel9/<PACKAGE_NAME>-{{ spec.short }}"
  latest_fedora: "f<USER_CONFIRMED_VERSION>"  # Use user-confirmed Fedora version
  fedora_minimal_base: "quay.io/fedora/fedora-minimal:<USER_CONFIRMED_VERSION>"  # Use user-confirmed Fedora version
  pkg_manager: "microdnf"
  minimal_pkgs: >-
    {% if spec.prod in ['rhel10', 'c10s', 'fedora'] -%}
      # Minimal packages for modern distros
    {%- else -%}
      # Minimal packages for legacy distros
    {%- endif %}
  # Copy pkgs and build_deps from standard variant
```

### 5. Update the Distribution Matrix

Add the new version to the `matrix.include` section at the end of `specs/multispec.yml`:

```yaml
matrix:
  include:
    # ... existing versions ...
    - version: "<NEW_VERSION>"
      distros:
        - rhel-8-x86_64      # If supported
        - rhel-9-x86_64
        - rhel-10-x86_64     # Modern RHEL
        - centos-stream-9-x86_64
        - centos-stream-10-x86_64  # Modern CentOS
        - fedora-<CURRENT_VERSION>-x86_64   # MANDATORY: Use the most recent Fedora version
    
    # If minimal variant exists:
    - version: "<NEW_VERSION>-minimal"
      distros:
        # Same distro list as standard variant
```

**CRITICAL: Fedora Version - MUST ASK USER**
- **DO NOT** copy the Fedora version from existing entries - they WILL be outdated
- **DO NOT** assume or guess the Fedora version
- **REQUIRED:** Ask the user explicitly: "Which Fedora version should I use?" 
- If user hasn't specified, check current release (https://fedoraproject.org or `cat /etc/fedora-release`) and ask: "Should I use Fedora XX?"
- Wait for user confirmation before proceeding
- Use the confirmed version for:
  - `fedora-XX-x86_64` in matrix entries
  - `fedoraXX` distroinfo section at the top of multispec.yml
  - Base image tags (`quay.io/fedora/s2i-core:XX` and `quay.io/fedora/fedora-minimal:XX`)

**Important:** Check which distros are supported for the new version:
- Modern versions (22+) typically support rhel10 and c10s
- Older versions may not support these modern distros
- Check upstream package availability for each distro

### 6. Generate Dockerfiles Using Distgen

After updating `specs/multispec.yml`, generate the Dockerfiles using the distgen tool:

```bash
# Install distgen if not available
python3 -m pip install distgen --user

# Generate files for all versions
dg --multispec specs/multispec.yml \
   --multispec-combinations version=<NEW_VERSION> \
   --distro fedora-43-x86_64 \
   --template src/Dockerfile \
   --output <NEW_VERSION>/Dockerfile.fedora

# Or use the common make target if available
make generate-all
# Or
make distgen
```

Typically, you should run distgen for each combination of (version, distro):
- For each distro in the matrix for your version
- For both standard and minimal variants (if applicable)

### 7. Verify Generated Files

Check that the generated files are correct:

```bash
# List generated files
ls <NEW_VERSION>/

# Check a sample Dockerfile
Read <NEW_VERSION>/Dockerfile.c9s

# Verify template variables were substituted correctly
grep "NODEJS_VERSION=<NEW_VERSION>" <NEW_VERSION>/Dockerfile.c9s
```

### 8. Run Verification Script

If a verification script exists, run it:

```bash
./verify-distgen-integration.sh
```

If the script doesn't exist or needs updating for the new version, update it to check:
- New version directories exist
- Dockerfiles contain correct version numbers
- Required distros are present
- Template variables were substituted

### 9. Update README Version Table

Regenerate the README.md version table to include the new version:

```bash
# Regenerate the version table (requires python and python-natsort)
make version-table

# If natsort is not installed:
pip3 install natsort --user
make version-table
```

This will automatically update the version table in README.md with entries for the new version across all supported distros.

### 10. Test Build

Test that the new version builds correctly:

```bash
# Test with one distro first
make build TARGET=c9s VERSIONS=<NEW_VERSION>

# Test all distros for the new version
make build VERSIONS=<NEW_VERSION>

# Test the minimal variant if applicable
make build VERSIONS=<NEW_VERSION>-minimal
```

### 11. Update CI Workflows

Update GitHub Actions workflows to enable testing and building for the new version:

```bash
# Read the CI workflow files
Read .github/workflows/container-tests.yml
Read .github/workflows/container-upstream-tests.yml
Read .github/workflows/build-and-push.yml
```

**Update container-tests.yml:**
Add the new version to the `versions` and `openshift-versions` arrays:
```yaml
versions: '[ "22", "22-minimal", "24", "24-minimal", "<NEW_VERSION>", "<NEW_VERSION>-minimal" ]'
openshift-versions: '[ "22", "22-minimal", "24", "24-minimal", "<NEW_VERSION>", "<NEW_VERSION>-minimal" ]'
```

**Update container-upstream-tests.yml:**
Add the new version to the test matrix:
```yaml
matrix:
  version: ["22", "22-minimal", "24", "24-minimal", "<NEW_VERSION>", "<NEW_VERSION>-minimal"]
  os_test: ["fedora", "rhel8", "rhel9", "rhel10", "c9s", "c10s"]
```

**Update build-and-push.yml:**
Add new matrix entries for builds (typically c10s and fedora for sclorg repos):
```yaml
# For sclorg namespace (c10s builds)
- dockerfile: "<NEW_VERSION>/Dockerfile.c10s"
  docker_context: <NEW_VERSION>
  registry_namespace: "sclorg"
  tag: "c10s"
  image_name: "<package>-<NEW_VERSION>-c10s"
  quayio_username: "QUAY_IMAGE_SCLORG_BUILDER_USERNAME"
  quayio_token: "QUAY_IMAGE_SCLORG_BUILDER_TOKEN"

- dockerfile: "<NEW_VERSION>-minimal/Dockerfile.c10s"
  docker_context: <NEW_VERSION>-minimal
  registry_namespace: "sclorg"
  tag: "c10s"
  image_name: "<package>-<NEW_VERSION>-minimal-c10s"
  quayio_username: "QUAY_IMAGE_SCLORG_BUILDER_USERNAME"
  quayio_token: "QUAY_IMAGE_SCLORG_BUILDER_TOKEN"

# For fedora namespace (fedora builds)
- dockerfile: "<NEW_VERSION>/Dockerfile.fedora"
  docker_context: <NEW_VERSION>
  registry_namespace: "fedora"
  tag: "fedora"
  quayio_username: "QUAY_IMAGE_FEDORA_BUILDER_USERNAME"
  quayio_token: "QUAY_IMAGE_FEDORA_BUILDER_TOKEN"
  image_name: "<package>-<NEW_VERSION>"

- dockerfile: "<NEW_VERSION>-minimal/Dockerfile.fedora"
  docker_context: <NEW_VERSION>-minimal
  registry_namespace: "fedora"
  tag: "fedora"
  quayio_username: "QUAY_IMAGE_FEDORA_BUILDER_USERNAME"
  quayio_token: "QUAY_IMAGE_FEDORA_BUILDER_TOKEN"
  image_name: "<package>-<NEW_VERSION>-minimal"
```

### 12. Commit Changes

Create commits for the changes:

```bash
# First commit: Core distgen changes
git add Makefile specs/multispec.yml verify-distgen-integration.sh README.md <NEW_VERSION>/ <NEW_VERSION>-minimal/
git commit -m "Add support for <PACKAGE> <NEW_VERSION> and <NEW_VERSION>-minimal

- Added version <NEW_VERSION> and <NEW_VERSION>-minimal definitions to specs/multispec.yml
- Added matrix entries for all supported distros (rhel8, rhel9, rhel10, c9s, c10s, fedora)
- Generated Dockerfiles for both standard and minimal variants
- Updated Makefile VERSIONS to include <NEW_VERSION> and <NEW_VERSION>-minimal
- Updated verification script to check version <NEW_VERSION>
- Updated README.md version table

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

# Second commit: CI workflow updates
git add .github/workflows/
git commit -m "Enable CI tests and builds for <PACKAGE> <NEW_VERSION>

- Updated container-tests.yml to test version <NEW_VERSION> and <NEW_VERSION>-minimal
- Updated container-upstream-tests.yml to include version <NEW_VERSION> in test matrix
- Added build-and-push.yml entries for <NEW_VERSION> and <NEW_VERSION>-minimal (c10s and fedora)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

### 13. Create TWO Separate Branches for PRs (Important!)

**GitHub Actions limitation**: Workflows triggered by `issue_comment` (PR comments) run from the **default branch** (master), NOT from the PR branch. This means the CI workflow updates in your version PR won't be used until they're merged to master first.

**Solution**: You need **TWO separate branches and TWO separate PRs**:

1. **Branch 1**: `enable-<NEW_VERSION>-tests` - Contains ONLY the CI workflow commit
2. **Branch 2**: `<NEW_VERSION>` or `add-<NEW_VERSION>` - Contains BOTH commits (implementation + workflows)

**After creating your two commits on the main implementation branch**, create the workflow-only branch:

```bash
# You're currently on your implementation branch (e.g., add-nodejs-26)
# with both commits already made

# 1. Switch to master and update it
git checkout master
git pull origin master

# 2. Create a new branch from master for workflow updates ONLY
git checkout -b enable-<NEW_VERSION>-tests

# 3. Cherry-pick ONLY the CI workflow commit
git cherry-pick <CI_WORKFLOW_COMMIT_HASH>  # The second commit you created (the workflows one)

# 4. Push the workflow-only branch
git push fork enable-<NEW_VERSION>-tests

# 5. Switch back to your implementation branch
git checkout <your-implementation-branch>

# 6. Push your implementation branch too
git push fork <your-implementation-branch>
```

**Create two PRs in this order:**

1. **First PR** (enable-<NEW_VERSION>-tests → master):
   - Title: "Enable CI tests for <PACKAGE> <NEW_VERSION>"
   - Description: "Updates CI workflows to enable testing for version <NEW_VERSION>. Must be merged before the main implementation PR can be tested."
   - **Get this merged to master FIRST**

2. **Second PR** (<NEW_VERSION> branch → master):
   - Title: "Add support for <PACKAGE> <NEW_VERSION>"
   - Description: Full description of the implementation
   - Note: "Depends on #XXX (workflow PR) being merged first for tests to work"

**Why this is necessary:**
- The main version PR contains both the implementation AND the workflow updates
- But GitHub Actions uses workflow files from **master**, not from the PR
- So version <NEW_VERSION> won't appear in the test matrix until the workflow updates are in master
- By merging the workflow PR first, subsequent test runs on the implementation PR will include version <NEW_VERSION>

**Workflow:**
1. Create both commits on your implementation branch
2. Create separate workflow-only branch via cherry-pick (as shown above)
3. Push BOTH branches
4. Create PR for workflow branch → **Get it merged to master first**
5. Create PR for implementation branch
6. After workflow PR is merged, re-trigger tests on implementation PR (add a comment like `[test]`)
7. Now version <NEW_VERSION> will be tested!
8. Once tests pass, merge the implementation PR

## Common Patterns by Repository Type

### Node.js-style repositories
- Versions include both standard and minimal variants
- Modern distros (rhel10, c10s, fedora) use versioned packages: `nodejs$NODEJS_VERSION`
- Legacy distros (rhel8, rhel9, c9s) use DNF modules: `dnf module enable nodejs:$VERSION`
- Special build tools for newer versions (gcc-toolset-13 for rhel8)

### PostgreSQL-style repositories
- Version numbers may be different format (e.g., "16", "15")
- May use SCL (Software Collections) for RHEL
- Database-specific configuration in templates

### General patterns
- Always copy structure from the previous version
- Check if package names changed upstream
- Verify base image versions are current
- Test on at least one distro before generating all

## Troubleshooting

### Template variables not substituted
- Check that the variable name matches what's defined in multispec.yml
- Verify Jinja2 syntax: `{{ spec.variable_name }}`
- Check conditional blocks: `{% if %}...{% endif %}`

### Distgen command fails
- Ensure distgen is installed: `python3 -m pip install distgen`
- Check YAML syntax in multispec.yml: `yamllint specs/multispec.yml`
- Verify template file exists: `ls src/Dockerfile*`

### Missing distros
- Check the matrix section includes all desired distros
- Verify distro names match exactly (e.g., `fedora-43-x86_64`)
- Check if distro is defined in the `distroinfo` section

### Generated files have wrong values
- Review the version definition in multispec.yml
- Check conditional logic in templates (`{% if spec.prod == 'fedora' %}`)
- Verify package names are correct for that distro version

## Example: Adding Node.js 26

```bash
# 0. FIRST: Ask user which Fedora version to use
# Example: "Which Fedora version should I use for Node.js 26? (e.g., fedora-45, fedora-46)"
# Wait for user response before proceeding!

# 1. Add fedora45 distroinfo section (if user confirmed Fedora 45)
Edit specs/multispec.yml  # Add fedora45 section with F45 base images

# 2. Edit specs/multispec.yml to add version "26" and "26-minimal"
Edit specs/multispec.yml  # Use user-confirmed Fedora version in base images

# 3. Add to matrix section
Edit specs/multispec.yml  # Add matrix entry with fedora-45-x86_64

# 4. Update Makefile
Edit Makefile  # Add 26 26-minimal to VERSIONS

# 5. Generate Dockerfiles
make generate

# 6. Verify
./verify-distgen-integration.sh

# 7. Update README version table
make version-table

# 8. Test build
make build TARGET=c9s VERSIONS=26

# 9. Update CI workflows
Edit .github/workflows/container-tests.yml
Edit .github/workflows/container-upstream-tests.yml
Edit .github/workflows/build-and-push.yml

# 10. Commit changes
git add Makefile specs/multispec.yml verify-distgen-integration.sh README.md 26/ 26-minimal/
git commit -m "Add support for Node.js 26 and 26-minimal

- Added version 26 and 26-minimal definitions to specs/multispec.yml
- Added matrix entries for all supported distros (rhel8, rhel9, rhel10, c9s, c10s, fedora)
- Generated Dockerfiles for both standard and minimal variants
- Updated Makefile VERSIONS to include 26 and 26-minimal
- Updated verification script to check version 26

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

git add .github/workflows/
git commit -m "Enable CI tests and builds for Node.js 26

- Updated container-tests.yml to test version 26 and 26-minimal
- Updated container-upstream-tests.yml to include version 26 in test matrix
- Added build-and-push.yml entries for 26 and 26-minimal (c10s and fedora)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

# 11. Create TWO separate branches for PRs

# First, note the commit hash of the CI changes (second commit)
CI_COMMIT=$(git log --oneline -1 | awk '{print $1}')

# Create workflow-only branch from master
git checkout master
git pull origin master
git checkout -b enable-nodejs-26-tests
git cherry-pick $CI_COMMIT  # Cherry-pick ONLY the CI workflow commit
git push fork enable-nodejs-26-tests

# Switch back to implementation branch and push it too
git checkout nodejs-26  # or whatever your implementation branch is called
git push fork nodejs-26

# 12. Create PRs in order:
# PR 1: enable-nodejs-26-tests → master (GET THIS MERGED FIRST!)
#   Title: "Enable CI tests for Node.js 26"
#   Description: "Updates CI workflows to enable testing for version 26. Must be merged before the main implementation PR can be tested."
#
# PR 2: nodejs-26 → master (create after PR 1 is merged)
#   Title: "Add support for Node.js 26 and 26-minimal"
#   Description: Full implementation details
#   Note: After PR 1 is merged, comment [test] to trigger tests with version 26 in the matrix
```

## Output

After completing this process, you should have:

1. ✅ **Confirmed Fedora version with user** - NEVER skip this step!
2. ✅ New `fedoraXX` distroinfo section added (if Fedora version is newer than existing ones)
3. ✅ Updated `specs/multispec.yml` with new version definition(s) using user-confirmed Fedora version
4. ✅ Updated `Makefile` with new version in VERSIONS
5. ✅ New version directory with Dockerfiles for all supported distros
6. ✅ New minimal version directory (if applicable)
7. ✅ Verification script passing (updated if needed)
8. ✅ Updated `README.md` version table via `make version-table`
9. ✅ Successfully tested build on at least one distro
10. ✅ Updated CI workflows (container-tests.yml, container-upstream-tests.yml, build-and-push.yml)
11. ✅ Two commits created (distgen changes + CI changes)
12. ✅ **TWO separate branches created:**
    - `enable-<VERSION>-tests` branch with ONLY the CI workflow commit
    - `<VERSION>` or `add-<VERSION>` branch with BOTH commits (implementation + workflows)
13. ✅ TWO separate PRs to be created:
    - PR for `enable-<VERSION>-tests` → master (to be merged FIRST)
    - PR for implementation branch → master (to be tested after workflow PR is merged)

## Notes

- **CRITICAL:** Always ask the user which Fedora version to use - NEVER copy from existing entries
- **CRITICAL:** If user confirms a newer Fedora version, create a new fedoraXX distroinfo section
- Always base the new version on the most recent previous version
- Test on one distro before generating all to catch template errors early  
- Check upstream package availability before adding distros to the matrix
- Modern distros (RHEL10+, C10S+, Fedora) typically use versioned packages
- Legacy distros (RHEL8/9, C9S) often use DNF modules or SCL
- Some repositories may have custom scripts for running distgen - check the Makefile
