# AGENTS.md

Persistent context for autonomous agents working on this Ansible role.

For project overview and install instructions, see [README.md](README.md).
For project facts, key files and architecture mindmap, see [FACTS.mmd](docs/FACTS.mmd).
For execution flows and logic diagrams, see [FLOWS.mmd](docs/FLOWS.mmd).
For firewall configuration, see [.github/FIREWALL.md](.github/FIREWALL.md).

## Setup & Environment Invariants

- Ansible role: `ea31337.metatrader`
- Supported OS: Debian/Ubuntu, NixOS (Nix), Windows
- Driver: Docker (Molecule)
- Python 3.10+ required; install via `pip install -r .devcontainer/requirements.txt`
- Collections: See [FACTS.mmd](docs/FACTS.mmd)
- Role dependencies: See [FACTS.mmd](docs/FACTS.mmd)
- `community.docker` MUST be installed before Molecule can create/destroy
  containers.
- Install dependencies: `ansible-galaxy role install -r requirements.yml --force` and
  `ansible-galaxy collection install -r requirements.yml -p collections`

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
| `molecule/default/create.yml` | Custom Docker create (proxy CA injection) |
| `molecule/default/destroy.yml` | Custom Docker destroy playbook |
| `molecule/default/prepare.yml` | Container preparation (sudo, Python, certs) |
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
- MUST run `yamllint .` and `ansible-lint` before committing YAML changes.
- NEVER hardcode sensitive information; use variables.
- NEVER remove or modify unrelated tests.
- NEVER use `git add .` without verifying staged files.
- On variable changes, update both `defaults/main.yml` and `README.md`.
- MUST reference GitHub Actions by simple major version tags (e.g. `actions/checkout@v6`),
  not pinned patch versions (e.g. `@v6.1.0`), so minor/patch updates apply automatically.

## Agent Directives (Contract Style)

- **NEVER** mock roles or modules, or use `exclude_paths` for `molecule/` and `tests/` as a workaround for issues.
- All issues **MUST** be resolved correctly at the root cause.
- Adhere strictly to project conventions and established standards.

## Project Invariants

- This project is an Ansible role for MetaTrader.
- It uses Molecule for testing and ansible-lint for linting.
- It depends on external roles such as `ea31337.wine` and `ea31337.xvfb`.

## Docker Tests

The standalone Docker test playbooks in `tests/`, how to run them via `pipenv`, and
their troubleshooting matrix live in [tests/AGENTS.md](tests/AGENTS.md).

## Molecule Testing

Molecule scenarios, the platform matrix, how to run the tests, and Molecule-specific
troubleshooting live in [molecule/AGENTS.md](molecule/AGENTS.md).

## Testing & Verification Gates

- `yamllint .` - YAML lint (config: `.yamllint`)
- `ansible-lint` - Ansible best practices (config: `.ansible-lint`)
- `pre-commit run -a` - all pre-commit hooks

## Common Tasks

### Before Each Commit

- Verify changes: `git diff --no-color`.
- NEVER use `git add .` without reviewing staged files.
- Run linters: `pre-commit run -a`.
- Run `pipenv run molecule syntax` to catch playbook errors early.

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

### Updating Pre-commit Hooks

Run `pre-commit autoupdate`, then `pre-commit run -a`. Revert any hook that breaks and file an issue for it.

Known blockers (as of the 2026-09 update):

- `ansible-lint` v26.8.0 declares `language_version: python3.14`. Without a Python 3.14
  interpreter, either keep the ref pinned or override the hook with `language_version: python3`.
- `pre-commit-hooks` v6.0.0 removed `check-byte-order-marker`; replace it with
  `fix-byte-order-marker`.
- `markdownlint-cli` v0.49.1 needs node >= 22.20 (its dev dependency `ava@8`). If the hook pins
  `language_version: 22.14.0`, the env fails to install; pin markdownlint-cli or bump the pinned node.
- `ansible-lint` + `community.docker`: a stale, empty
  `.ansible/collections/ansible_collections/community/docker` directory shadows the real collection
  and causes `couldn't resolve module/action 'community.docker.docker_container'`. Remove it.
- `additional_dependencies` with a version range must use the block form
  (`- ansible-core>=2.16,<2.21`); the inline flow form splits on the comma into separate
  requirements, and the no-space form trips ansible-lint's `yaml[commas]` rule.

`pre-commit run -a` can also surface pre-existing failures (e.g. `yamlfix`/`black` reformatting,
`flake8` violations) unrelated to the ref bump; CI lints only changed files, so file these separately.

### Editing Files

- Max line length: 120 characters (enforced by `.yamllint` and
  `.markdownlint.yaml`).
- YAML indentation: 2 spaces.
- End all files with a newline.
- Keep lists and keys in lexicographical order when possible.

### Adding or Modifying Workflows

- Workflows live in `.github/workflows/`.
- Use `actionlint` to validate workflow syntax.
- Molecule workflow uses `gofrolist/molecule-action@v2` with per-platform
  matrix.
- `paths-ignore` excludes `**.md`, `**.cfg`, `.*`, `LICENSE`, `Pipfile*`
  from triggers.

## Firewall Issues

If network requests fail during molecule tests (e.g. `dl.winehq.org`,
`download.mql5.com`, `channels.nixos.org`, `galaxy.ansible.com`):

- Refer to <https://gh.io/copilot/firewall-config> for agent firewall
  setup.
- Do not work around blocked URLs; request allowlisting instead.

### Alpine bootstrap fails with TLS error

- **Root cause**: Alpine `apk update` fails with `TLS: unspecified error` when behind an SSL-intercepting proxy
  if the proxy CA is not in the build-time trust store.
- **Fix**: The custom `Dockerfile.j2` injects host CA certificates directly into `/etc/ssl/cert.pem`
  during the build phase so `apk` can fetch dependencies safely.
- **Prevention**: Verify `dl-cdn.alpinelinux.org` is reachable from inside the container.

### Required Hosts

| Host | Purpose |
| ---- | ------- |
| `cache.nixos.org` | Nix binary cache (pre-built packages) |
| `cdn.mql5.com` | CDN (MT5 platform files) |
| `channels.nixos.org` | Nix channel metadata (redirects to releases) |
| `codeload.github.com` | GitHub archive download (dependency) |
| `dl-cdn.alpinelinux.org` | Alpine Linux package repository |
| `dl.winehq.org` | WineHQ APT repository and GPG key |
| `download.mql5.com` | MetaTrader setup executable download |
| `galaxy.ansible.com` | Ansible Galaxy collections |
| `github.com` | AutoHotkey download (used by winetricks verb) |
| `mt5-trade.metaquotes.net` | Trade server (installer backend) |
| `raw.githubusercontent.com` | OpenSymbol font download (winetricks verb) |
| `releases.nixos.org` | Nix channel tarballs (redirect target) |
| `trade.mql5.com` | Trade server (installer registration) |
| `web.archive.org` | Winetricks fallback download mirror |
| `www.mql5.com` | Main website (installer backend) |

## References

- Project documentation: [README.md](README.md)
- Agent configuration:
  [.github/copilot-instructions.md](.github/copilot-instructions.md)
- Wine role: <https://github.com/EA31337/ansible-role-wine>
- Xvfb role: <https://github.com/EA31337/ansible-role-xvfb>
- Molecule docs: <https://docs.ansible.com/projects/molecule/>
- Ansible lint rules: <https://docs.ansible.com/projects/lint/rules/>
- Org baseline:
  <https://github.com/Cogni-AI-OU/.github/blob/main/AGENTS.md>
- Agents.md standard: <https://agents.md/>
