# AGENTS.md

Agent guidance for the `ansible-role-metatrader` Ansible role.

For project overview and install instructions, see [README.md](README.md).

## Setup & Environment Invariants

- Ansible role: `ea31337.metatrader`
- Supported OS: Alpine Linux (excluded from tests), Debian/Ubuntu, NixOS (Nix), Windows
- Driver: Docker (Molecule)
- Python 3.10+ required; install via `pip install -r .devcontainer/requirements.txt`
- Collections: `community.docker`, `community.general`, `ansible.windows`
- Role dependencies: `ea31337.wine`, `ea31337.xvfb` (see `meta/main.yml`)
- `community.docker` MUST be installed before Molecule can create/destroy containers.
- Install dependencies: `ansible-galaxy install -r requirements.yml`

## Key Files & Context Injection

| Path | Purpose |
| ---- | ------- |
| `defaults/main.yml` | Role defaults (`metatrader_setup_url`, `metatrader_version`) |
| `vars/main.yml` | Internal variables (`metatrader_become_method` per OS) |
| `tasks/main.yml` | Role entry point; installs MetaTrader via winetricks verb |
| `tasks/verify.yml` | Post-install verification (terminal.exe, metaeditor.exe) |
| `templates/mt4_install.verb.j2` | MT4 winetricks verb template |
| `templates/mt5_install.verb.j2` | MT5 winetricks verb template |
| `meta/main.yml` | Galaxy metadata + role dependencies (wine, xvfb) |
| `molecule/default/molecule.yml` | Default Molecule scenario config |
| `molecule/default/converge.yml` | Converge playbook (all scenarios) |
| `molecule/default/prepare.yml` | Container preparation (Python, sudo) |
| `molecule/default/verify.yml` | Verification playbook |
| `molecule/mt4/molecule.yml` | MT4 scenario config |
| `molecule/mt5/molecule.yml` | MT5 scenario config |
| `molecule/mt5-win/molecule.yml` | MT5 Windows scenario config |
| `molecule/resources/playbooks/Dockerfile.j2` | NixOS container Dockerfile template |
| `requirements.yml` | Ansible Galaxy collection + role dependencies |
| `ansible.cfg` | Ansible configuration (collections_path, callbacks) |
| `.github/workflows/molecule.yml` | CI: Molecule test matrix |
| `.github/workflows/check.yml` | CI: pre-commit / linting |
| `.github/workflows/test.yml` | CI: Docker container test |
| `.pre-commit-config.yaml` | Pre-commit hooks (yamllint, ansible-lint, etc.) |
| `.ansible-lint` | Ansible-lint configuration |
| `.yamllint` | YAML lint rules (max line length 120) |
| `.markdownlint.yaml` | Markdown lint rules (max line length 120) |

## Agent Directives

- MUST use FQCN for all modules (`ansible.builtin.*`, `community.general.*`).
- MUST keep YAML keys sorted alphabetically in config files when possible.
- MUST ensure idempotency in all Ansible tasks.
- MUST wrap lines at 120 characters (YAML and Markdown).
- MUST end files with a newline character.
- MUST use `true`/`false` for truthy values (not `yes`/`no`).
- NEVER hardcode sensitive information; use variables.
- NEVER remove or modify unrelated tests.
- On variable changes, update both `defaults/main.yml` and `README.md`.

## Molecule Scenarios

| Scenario | `metatrader_version` | Notes |
| -------- | -------------------- | ----- |
| `default` | `5` (from defaults) | Default MT5 install |
| `mt4` | `4` | MT4 install with custom setup URL |
| `mt5` | `5` | Explicit MT5 install |
| `mt5-win` | `5` | Windows container (disabled in CI) |

### Platforms (Linux scenarios)

| Container | Image | Notes |
| --------- | ----- | ----- |
| `debian-latest` | `debian:latest` | Uses apt; WineHQ repo |
| `nixos-latest` | `nixos/nix:latest` | Custom Dockerfile; requires `pre_build_image: false` |
| `ubuntu-jammy` | `ubuntu:jammy` | Uses apt; WineHQ repo |
| `ubuntu-noble` | `ubuntu:noble` | Uses apt; WineHQ repo |
| `ubuntu-latest` | `ubuntu:latest` | Uses apt; WineHQ repo |

### Running Tests

```bash
# Full test (all scenarios)
molecule test

# Single scenario
molecule test -s default

# Single platform in a scenario
molecule test -s default --platform-name debian-latest

# Individual steps
molecule create -s default
molecule converge -s default
molecule verify -s default
molecule destroy -s default

# Syntax check only
molecule syntax
```

## Testing & Verification Gates

- `molecule syntax` — YAML + playbook syntax validation
- `molecule converge` — full role execution on all containers
- `molecule idempotence` — re-run must produce zero changes
- `molecule verify` — asserts terminal.exe and metaeditor.exe exist
- `yamllint .` — YAML lint (config: `.yamllint`)
- `ansible-lint` — Ansible best practices (config: `.ansible-lint`)
- `pre-commit run -a` — all pre-commit hooks

## Troubleshooting Matrix

### `community.docker.docker_container` module not found

> Molecule destroy/create fails with:
> `ERROR! couldn't resolve module/action 'community.docker.docker_container'`

- **Root cause**: `community.docker` collection not installed in the execution
  environment.
- **Fix**: Run `ansible-galaxy collection install -r requirements.yml` before
  `molecule test`. In CI, the `gofrolist/molecule-action` container must have
  the collection pre-installed or an install step must precede the test step.
- **CI context**: The Molecule workflow includes an `Install Ansible collections`
  step before the molecule action. `ansible.cfg` sets `collections_path` to
  include `./collections`.

### NixOS Docker build SSL failures

> `error: unable to download 'https://channels.nixos.org/nixpkgs-unstable':
> SSL peer certificate or SSH remote key was not OK (60)`

- **Root cause**: Sandboxed environments with MITM proxies or missing CA bundles
  break `nix-channel --update`.
- **Fix (CI)**: Runs normally on GitHub Actions runners.
- **Fix (local)**: Ensure valid CA certificates. If behind proxy, set
  `NIX_SSL_CERT_FILE` or inject certs into the NixOS Docker image.

### NixOS containerd symlink error

> `path escapes from parent` during NixOS container creation.

- **Root cause**: containerd >= 2.2.0 / Go 1.24 rejects absolute symlinks
  in `/etc/passwd` and `/etc/group` that point into `/nix/store`.
- **Fix**: The `Dockerfile.j2` template converts these to relative symlinks
  via `realpath --relative-to`. See `molecule/resources/playbooks/Dockerfile.j2`.
- **Reference**: <https://github.com/containerd/containerd/issues/12683>

### molecule-docker broken conditionals deprecation

> `DEPRECATION WARNING: Conditional result (True) was derived from value
> of type 'str'`

- **Root cause**: `molecule-docker 2.1.0` create/destroy playbooks use
  `when: (lookup('env', 'HOME'))` which is a string, not boolean. This
  becomes an error in `ansible-core >= 2.23`.
- **Workaround**: All scenario configs set `allow_broken_conditionals: true`
  in `provisioner.config_options.defaults`.
- **Ref**: <https://github.com/ansible-community/molecule-plugins/issues/295>
- **Long-term fix**: Wait for upstream `molecule-docker` patch.

### Wine APT cache update fails

> `Failed to update apt cache after 5 retries`

- **Root cause**: Firewall/network policy blocks `dl.winehq.org` or the
  WineHQ repository is temporarily unavailable.
- **Fix**: Add `dl.winehq.org` to firewall allowlist. Retry the test.
- **CI context**: Works on standard GitHub Actions runners with internet access.

### MetaTrader setup download fails

> `curl: (7) Failed to connect to download.mql5.com`

- **Root cause**: `download.mql5.com` blocked by network policy.
- **Fix**: Ensure `download.mql5.com` is accessible. The setup URL is
  configurable via `metatrader_setup_url` variable.

## Common Tasks

### Before Each Commit

- Verify changes: `git diff --no-color`.
- NEVER use `git add .` without reviewing staged files.
- Run linters: `pre-commit run -a`.

### Linting and Validation

```bash
# All pre-commit checks
pre-commit run -a

# Individual checks
pre-commit run yamllint -a
pre-commit run ansible-lint -a
pre-commit run markdownlint -a
pre-commit run j2lint -a
pre-commit run actionlint -a
```

### Editing Files

- Max line length: 120 characters (enforced by `.yamllint` and `.markdownlint.yaml`).
- YAML indentation: 2 spaces.
- End all files with a newline.
- Keep lists and keys in lexicographical order when possible.

### Adding or Modifying Workflows

- Workflows live in `.github/workflows/`.
- Use `actionlint` to validate workflow syntax.
- Molecule workflow uses `gofrolist/molecule-action@v2` with per-platform matrix.
- `paths-ignore` excludes `**.md`, `**.cfg`, `.*`, `LICENSE`, `Pipfile*` from triggers.

## Firewall Issues

If network requests fail during molecule tests (e.g. `dl.winehq.org`,
`download.mql5.com`, `channels.nixos.org`, `galaxy.ansible.com`):

- Refer to <https://gh.io/copilot/firewall-config> for agent firewall setup.
- Do not work around blocked URLs; request allowlisting instead.

## References

- Project documentation: [README.md](README.md)
- Agent conventions: [.github/copilot-instructions.md](.github/copilot-instructions.md)
- Molecule docs: <https://docs.ansible.com/projects/molecule/>
- Ansible lint rules: <https://docs.ansible.com/projects/lint/rules/>
- Wine role: <https://github.com/EA31337/ansible-role-wine>
- Xvfb role: <https://github.com/EA31337/ansible-role-xvfb>
