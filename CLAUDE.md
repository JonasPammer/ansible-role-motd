# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible role (`jonaspammer.motd`) for configuring login banners (MOTD - Message of the Day) on Linux machines. It supports two distinct implementation modes:

1. **Static MOTD**: Traditional approach with a single `/etc/motd` text file
2. **Dynamic MOTD (update-motd)**: Modern approach using Ubuntu's update-motd framework where MOTD is assembled from executable scripts

The role is published to Ansible Galaxy and follows strict quality standards with comprehensive testing across multiple distributions and Ansible versions.

## Development Commands

### Testing

```bash
# Run all tests (linting + molecule tests across all Ansible versions)
tox

# Test with specific distribution
MOLECULE_DISTRO=ubuntu2204 tox

# Test with specific Ansible version
tox -e py3-ansible-9

# Run pre-commit hooks only
tox -e pre-commit

# Keep container running after test for debugging
MOLECULE_DESTROY=never MOLECULE_DISTRO=debian12 tox

# Show installed package versions (for debugging)
CI=true tox
```

### Linting

```bash
# Run all pre-commit hooks
pre-commit run --all-files

# Install pre-commit hooks for automatic checking
pre-commit install

# Lint YAML files
yamllint .

# Lint Ansible files
ansible-lint
```

### Molecule Debugging

When a molecule test fails and you used `MOLECULE_DESTROY=never`:

```bash
# Find the container
docker ps

# Enter the container
docker exec -it <container-id> /bin/bash

# Inspect test artifacts (created by molecule/resources/debug.yml)
cat /var/tmp/vars.yml
cat /var/tmp/environment.yml

# Clean up afterwards
docker stop <container-id>
docker container rm <container-id>
# or
docker container prune
```

## Architecture

### Two-Mode Design Pattern

The role's core architecture is built around the `motd_type` variable which determines the execution path:

- `motd_type: static` → executes `tasks/setup-static.yml`
- `motd_type: update-motd` → executes `tasks/setup-update-motd.yml`

The type is auto-selected based on distribution (defined in `defaults/main.yml` using a fallback chain pattern):
- Ubuntu defaults to `update-motd`
- Other distributions default to `static` (currently limited by TODO in setup-update-motd.yml:8)

### Task Execution Flow

1. **tasks/main.yml** - Entry point
   - Runs variable assertions (`tasks/assert.yml`)
   - Stats the static MOTD file at `/etc/motd`
   - Includes appropriate setup task file based on `motd_type`

2. **tasks/setup-static.yml**
   - Generates static `/etc/motd` from template
   - Backs up and removes dynamic MOTD directory if it exists

3. **tasks/setup-update-motd.yml** (Currently Ubuntu-only)
   - Backs up and removes static `/etc/motd` if it exists
   - Installs required packages (boxes, neofetch, screenfetch)
   - Removes unwanted scripts from update-motd directory
   - Generates each script template via `tasks/add-dynamic-script.yml`

4. **tasks/add-dynamic-script.yml** (Loop helper)
   - Generates script from template to temp directory
   - Runs shellcheck validation (--severity=error)
   - Only copies to final directory if shellcheck passes
   - Uses `ignore_errors: true` to prevent one bad script from failing entire role

### Directory Structure

```
ansible-role-motd/
├── defaults/main.yml         # User-configurable variables with OS-specific defaults
├── vars/main.yml             # Internal variables (motd_dynamic_scripts_directory)
├── tasks/
│   ├── main.yml             # Entry point
│   ├── assert.yml           # Variable validation (currently empty)
│   ├── setup-static.yml     # Static MOTD implementation
│   ├── setup-update-motd.yml # Dynamic MOTD implementation
│   └── add-dynamic-script.yml # Loop include for script generation
├── templates/
│   ├── issue.net.jinja2     # Static banner template
│   ├── 00-legal.jinja2      # Dynamic script: legal banner (uses boxes)
│   └── 10-sysinfo.jinja2    # Dynamic script: system info (neofetch/screenfetch)
├── handlers/main.yml         # No handlers currently defined
├── molecule/                 # Molecule test scenarios
│   ├── default/
│   │   ├── molecule.yml     # Test configuration
│   │   ├── converge.yml     # Apply the role
│   │   └── verify.yml       # Verification tasks (currently minimal)
│   └── resources/
│       ├── prepare.yml      # Install dependencies (bootstrap, core_dependencies, shellcheck)
│       └── debug.yml        # Debug output collection
└── meta/main.yml            # Galaxy metadata
```

## Key Implementation Patterns

### OS-Specific Variable Selection

The role uses a sophisticated fallback chain pattern for OS-specific variables in `defaults/main.yml`:

```yaml
motd_type: "{{
  _motd_type[ansible_distribution ~ '_' ~ ansible_distribution_major_version]|default(
  _motd_type[ansible_os_family ~ '_' ~ ansible_distribution_major_version])|default(
  _motd_type[ansible_distribution])|default(
  _motd_type[ansible_os_family])|default(
  _motd_type['default']) }}"
```

This checks in order:
1. `Distribution_MajorVersion` (e.g., `Ubuntu_20`)
2. `OSFamily_MajorVersion` (e.g., `Debian_20`)
3. `Distribution` (e.g., `Ubuntu`)
4. `OSFamily` (e.g., `Debian`)
5. `default`

### Template Validation with ShellCheck

Dynamic MOTD scripts are bash scripts, so the role validates them with shellcheck before deployment. This prevents syntax errors from breaking login:

- Templates are generated to a temp directory first
- Shellcheck validates each script with `--severity=error`
- Only scripts that pass validation are copied to the final directory
- Failed validations log warnings but don't fail the entire role

### Backup and Preservation Logic

Both modes carefully handle existing MOTD configurations:

- `motd_static_motd_backup` / `motd_dynamic_scripts_backup` control backup behavior
- Backups are created before removal when enabled
- The role ensures only its managed content exists (removes other scripts/files)

## Supported Platforms

Actively tested distributions (see `.github/workflows/ci.yml` and `meta/main.yml`):
- Ubuntu 20.04 LTS (focal), 22.04 LTS (jammy)
- Debian 11 (bullseye), 12 (bookworm)
- Rocky Linux 8, 9
- Fedora 39

Tested Ansible versions (Ansible core in parentheses):
- Ansible 6 (core 2.13)
- Ansible 7 (core 2.14)
- Ansible 8 (core 2.15)
- Ansible 9 (core 2.16)

**Important limitation**: Dynamic MOTD currently only works on Ubuntu (see TODO at `tasks/setup-update-motd.yml:4-10`). Implementation needed for other distributions.

## Template System

### Built-in Templates

The role includes three templates in `templates/`:

1. **issue.net.jinja2**: Legal banner template
   - Uses `motd_legal_location_name` variable if defined
   - Shared by both static mode and dynamic mode's 00-legal script

2. **00-legal.jinja2**: Dynamic MOTD script
   - Includes issue.net.jinja2 content via heredoc
   - Uses `boxes -d parchment` if available for ASCII art border

3. **10-sysinfo.jinja2**: System information script
   - Runs neofetch, falls back to screenfetch, then uname -a

### Custom Templates

Users can provide their own templates. The role searches for templates in the playbook's `templates/` directory first, then falls back to the role's bundled templates. To avoid conflicts, custom templates should use different names than the bundled ones.

## Testing Infrastructure

### Molecule Configuration

- **Driver**: Docker using geerlingguy's systemd-enabled images
- **Provisioner**: Ansible with verbose output and YAML callbacks
- **Verifier**: Ansible (currently minimal verification in verify.yml)
- **Prepare**: Installs dependencies via requirements.yml roles

### Tox Integration

The `tox.ini` defines test matrix:
- Tests run across 4 Ansible versions (6, 7, 8, 9)
- Environment variable `MOLECULE_DISTRO` selects distribution
- `TOX_SKIP_ENV` can skip specific test environments
- Pre-commit environment for linting

### CI/CD Workflows

- **ci.yml**: Runs on PRs, pushes to master, and weekly schedule
  - Lint job: yamllint with GitHub annotations
  - Molecule job: Matrix of 7 distributions × 4 Ansible versions
  - Uploads debug artifacts from test runs
  - Manual dispatch supports specific distro/Ansible version selection

- **release-to-galaxy.yml**: Triggered when tags are pushed (versions must not start with 'v')

### Pre-commit Hooks

Extensive pre-commit configuration ensures code quality:
- Conventional Commits enforcement (commitlint)
- YAML linting (yamllint, prettier)
- Python formatting (black, reorder-python-imports)
- Python linting (flake8, mypy)
- Secret detection (detect-secrets)

## Project Conventions

### Version Management

- Versions are defined as Git tags (without 'v' prefix)
- Tags trigger automatic import to Ansible Galaxy
- Follows Semantic Versioning

### Commit Messages

- Uses Conventional Commits specification
- Required for core contributors with push access
- Casual contributors don't need to follow (PRs are squash merged)

### CookieCutter Template

This project is generated from [cookiecutter-ansible-role](https://github.com/JonasPammer/cookiecutter-ansible-role) and should be kept in sync using cruft. Before making changes, check if they should be applied to the template instead.

### Code Quality

- ansible-lint runs with `skip_list: [name]` - task naming is optional per project guidelines
- All Python code must pass black, flake8, and mypy
- All YAML must pass yamllint and prettier
- All bash scripts in templates must pass shellcheck

## Common Development Scenarios

### Adding a New Dynamic MOTD Script

1. Create `templates/XX-scriptname.jinja2` with proper shebang
2. Add script name to `motd_dynamic_scripts_templates` in `defaults/main.yml`
3. Add any required packages to `_motd_dynamic_scripts_system_packages`
4. Test with shellcheck locally: `shellcheck templates/XX-scriptname.jinja2 --severity=error`
5. Run molecule tests to verify

### Adding Support for New Distribution

1. Add platform to `meta/main.yml`
2. Update `defaults/main.yml` with OS-specific defaults if needed
3. If adding update-motd support for non-Ubuntu, implement TODO in `setup-update-motd.yml`
4. Add distribution to CI matrix in `.github/workflows/ci.yml`
5. Test with: `MOLECULE_DISTRO=newdistro tox`

### Modifying Variable Defaults

When changing defaults in `defaults/main.yml`, also update the corresponding documentation in `README.adoc` to keep them in sync (see comment at top of defaults/main.yml).
