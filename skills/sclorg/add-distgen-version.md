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
4. **Package-specific information**:
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

### 3. Add New Version Definition

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
  latest_fedora: "f45"  # Update as needed
  fedora_base: "quay.io/fedora/s2i-core:44"  # Update as needed
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
  latest_fedora: "f45"
  fedora_minimal_base: "quay.io/fedora/fedora-minimal:44"
  pkg_manager: "microdnf"
  minimal_pkgs: >-
    {% if spec.prod in ['rhel10', 'c10s', 'fedora'] -%}
      # Minimal packages for modern distros
    {%- else -%}
      # Minimal packages for legacy distros
    {%- endif %}
  # Copy pkgs and build_deps from standard variant
```

### 4. Update the Distribution Matrix

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
        - fedora-43-x86_64   # Update Fedora version as needed
    
    # If minimal variant exists:
    - version: "<NEW_VERSION>-minimal"
      distros:
        # Same distro list as standard variant
```

**Important:** Check which distros are supported for the new version:
- Modern versions (22+) typically support rhel10 and c10s
- Older versions may not support these modern distros
- Check upstream package availability for each distro

### 5. Generate Dockerfiles Using Distgen

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

### 6. Verify Generated Files

Check that the generated files are correct:

```bash
# List generated files
ls <NEW_VERSION>/

# Check a sample Dockerfile
Read <NEW_VERSION>/Dockerfile.c9s

# Verify template variables were substituted correctly
grep "NODEJS_VERSION=<NEW_VERSION>" <NEW_VERSION>/Dockerfile.c9s
```

### 7. Run Verification Script

If a verification script exists, run it:

```bash
./verify-distgen-integration.sh
```

If the script doesn't exist or needs updating for the new version, update it to check:
- New version directories exist
- Dockerfiles contain correct version numbers
- Required distros are present
- Template variables were substituted

### 8. Update README and Documentation

Update the main README.md to include the new version:

```bash
# The README is typically generated from src/README.md template
# Regenerate it using distgen, or manually update the version table
```

### 9. Test Build

Test that the new version builds correctly:

```bash
# Test with one distro first
make build TARGET=c9s VERSIONS=<NEW_VERSION>

# Test all distros for the new version
make build VERSIONS=<NEW_VERSION>

# Test the minimal variant if applicable
make build VERSIONS=<NEW_VERSION>-minimal
```

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
# 1. Edit specs/multispec.yml to add version "26" and "26-minimal"
Edit specs/multispec.yml

# 2. Add to matrix section
Edit specs/multispec.yml  # Add matrix entry

# 3. Generate Dockerfiles
make distgen

# 4. Verify
./verify-distgen-integration.sh

# 5. Test build
make build TARGET=c9s VERSIONS=26

# 6. Create PR
git add specs/multispec.yml 26/ 26-minimal/
git commit -m "Add support for Node.js 26 and 26-minimal"
```

## Output

After completing this process, you should have:

1. ✅ Updated `specs/multispec.yml` with new version definition(s)
2. ✅ New version directory with Dockerfiles for all supported distros
3. ✅ New minimal version directory (if applicable)
4. ✅ Verification script passing
5. ✅ Successfully tested build on at least one distro
6. ✅ Updated README and documentation

## Notes

- Always base the new version on the most recent previous version
- Test on one distro before generating all to catch template errors early  
- Check upstream package availability before adding distros to the matrix
- Modern distros (RHEL10+, C10S+, Fedora) typically use versioned packages
- Legacy distros (RHEL8/9, C9S) often use DNF modules or SCL
- Some repositories may have custom scripts for running distgen - check the Makefile
