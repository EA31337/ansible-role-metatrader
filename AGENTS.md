# AGENTS.md

Persistent context for autonomous agents working on this Ansible role.

For project overview and install instructions, see [README.md](README.md).

## Setup & Environment Invariants

- Ansible role: `ea31337.metatrader`
- Supported OS: Debian/Ubuntu, NixOS (Nix), Windows
- Driver: Docker (Molecule)
- Python 3.10+ required; install via `pip install -r .devcontainer/requirements.txt`
- Collections: `community.docker`, `community.general`, `ansible.windows`
- Role dependencies: `ea31337.wine`, `ea31337.xvfb` (see `meta/main.yml`)
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

## Agent Directives (Contract Style)

- **NEVER** mock roles or modules, or use `exclude_paths` for `molecule/` and `tests/` as a workaround for issues.
- All issues **MUST** be resolved correctly at the root cause.
- Adhere strictly to project conventions and established standards.

## Project Invariants

- This project is an Ansible role for MetaTrader.
- It uses Molecule for testing and ansible-lint for linting.
- It depends on external roles such as `ea31337.wine` and `ea31337.xvfb`.

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
| `debian-latest` | `debian:latest` | WineHQ repo with `wine_release_codename: bookworm` |
| `nixos-latest` | `nixos/nix:latest` | Custom Dockerfile; privileged mode |
| `ubuntu-jammy` | `ubuntu:jammy` | WineHQ repo with `wine_release_codename: jammy` |
| `ubuntu-noble` | `ubuntu:noble` | WineHQ repo with `wine_release_codename: jammy` |
| `ubuntu-latest` | `ubuntu:latest` | WineHQ repo with `wine_release_codename: jammy` |

### Running Tests

```bash
# Install dependencies first
pip install -r .devcontainer/requirements.txt
ansible-galaxy role install -r requirements.yml --force
ansible-galaxy collection install -r requirements.yml -p collections

# Full test (all scenarios)
molecule test

# Single scenario
molecule test -s default

# Single platform in a scenario
molecule test -s default --platform-name debian-latest

# Step-by-step debugging (useful for troubleshooting)
molecule destroy -s default              # clean up any leftover state
molecule create -s default               # build images + start containers
molecule prepare -s default              # install Python, sudo, CA certs
molecule converge -s default             # run the role
molecule idempotence -s default          # verify idempotency (no changes)
molecule verify -s default               # run verification playbook
molecule destroy -s default              # clean up

# Syntax check only (fast validation)
molecule syntax -s default
molecule syntax -s mt4
molecule syntax -s mt5
```

### Step-by-step Testing With Timeout

For CI or automated environments, use timeouts:

```bash
# Test a single platform with timeout (10 minutes)
timeout 600 molecule test -s default --platform-name debian-latest

# If converge fails, debug interactively:
molecule create -s default --platform-name debian-latest
molecule converge -s default --platform-name debian-latest
# (inspect container state, then clean up)
molecule destroy -s default
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

- **Root cause**: `community.docker` collection not installed in the
  execution environment.
- **Fix**: Run `ansible-galaxy collection install -r requirements.yml`
  before `molecule test`. In CI, the `gofrolist/molecule-action` container
  must have the collection pre-installed or an install step must precede
  the test step.
- **CI context**: The Molecule workflow includes an
  `Install Ansible collections` step before the molecule action.
  `ansible.cfg` sets `collections_path` to include `./collections`.

### NixOS Docker build SSL failures

> `error: unable to download 'https://channels.nixos.org/nixpkgs-unstable':
> SSL peer certificate or SSH remote key was not OK (60)`

- **Root cause**: Sandboxed environments with MITM proxies or missing CA
  bundles break `nix-channel --update`.
- **Fix**: The custom `create.yml` detects host CA certificates in
  `/usr/local/share/ca-certificates/` and copies them into the Docker
  build context. The `Dockerfile.j2` injects these into the Nix cert
  bundle via `ssl-cert-file` in `nix.conf`.
- **Required hosts**: `channels.nixos.org`, `releases.nixos.org`,
  `cache.nixos.org`
- **Prevention**: All three Nix hosts must be in the firewall allowlist.

### NixOS: files in `/etc/ssl/certs/` vanish across Docker build layers

- **Root cause**: containerd/overlayfs bug causes files written to
  `/etc/ssl/certs/` in one Docker `RUN` layer to disappear in subsequent
  layers (NixOS image only).
- **Fix**: Store combined CA bundle in `/etc/nix/ca-bundle.crt` instead;
  `/etc/nix/` persists correctly across layers.
- **Prevention**: NEVER store persistent files under `/etc/ssl/certs/` in
  NixOS containers.

### NixOS containerd symlink error

> `path escapes from parent` during NixOS container creation.

- **Root cause**: containerd >= 2.2.0 / Go 1.24 rejects absolute symlinks
  in `/etc/passwd` and `/etc/group` that point into `/nix/store`.
- **Fix**: The `Dockerfile.j2` template converts these to relative symlinks
  via `realpath --relative-to`. See
  `molecule/resources/playbooks/Dockerfile.j2`.
- **Reference**: <https://github.com/containerd/containerd/issues/12683>

### molecule-docker broken conditionals deprecation

> `DEPRECATION WARNING: Conditional result (True) was derived from value
> of type 'str'`

- **Root cause**: `molecule-docker 2.1.0` create/destroy playbooks use
  `when: (lookup('env', 'HOME'))` which is a string, not boolean. This
  becomes an error in `ansible-core >= 2.23`.
- **Workaround**: All scenario configs set
  `allow_broken_conditionals: true` in
  `provisioner.config_options.defaults`. Custom `create.yml`/`destroy.yml`
  avoid this pattern.
- **Ref**:
  <https://github.com/ansible-community/molecule-plugins/issues>

### Wine APT cache update fails

> `Failed to update apt cache after 5 retries`

- **Root cause**: Firewall/network policy blocks `dl.winehq.org`, or
  `debian:latest` codename (e.g. `trixie`) is not in the WineHQ repo.
- **Fix**: Set `wine_release_codename: bookworm` for debian-latest in
  host_vars. Add `dl.winehq.org` to firewall allowlist.
- **CI context**: Works on standard GitHub Actions runners with internet
  access.

### MetaTrader setup download fails

> `curl: (6) Could not resolve host: download.mql5.com`

- **Root cause**: `download.mql5.com` blocked by network policy or DNS
  resolution failure inside Docker containers.
- **Fix**: Ensure `download.mql5.com` is accessible from inside Docker
  containers. The setup URL is configurable via `metatrader_setup_url`.
- **Fallback**: winetricks also tries `web.archive.org` as fallback; both
  hosts must be in the allowlist.

### Platform installer shows "Sorry, something went wrong"

> The setup bootstrapper downloads successfully but the actual
> installation fails with "Sorry, something went wrong: try again later!"

- **Root cause**: The `mt5setup.exe` bootstrapper is a small stub that
  downloads the platform CDN servers at runtime.
  If those servers (`www.mql5.com`, `cdn.mql5.com`, `trade.mql5.com`)
  are DNS-blocked, the installer cannot fetch platform files.
- **Fix**: Ensure **all** hosts are in the firewall allowlist
  (see [Required Hosts](#required-hosts) table below).
- **Diagnosis**: A "Proxy Server" dialog may also appear before the error
  if SSL interception is active. See
  [Debugging the MT5 installer](#debugging-the-mt5-installer) below.

### Debugging the MT5 installer

When the installer hangs or fails inside a container, use these steps:

```bash
# 1. Install xdotool in the container
docker exec CONTAINER apt-get install -y -q xdotool

# 2. List all visible X windows
docker exec -e DISPLAY=:0 CONTAINER \
  bash -c 'for wid in $(xdotool search --onlyvisible --name "." 2>/dev/null); do
    echo "Window $wid: $(xdotool getwindowname $wid 2>/dev/null)"
  done'

# 3. Check the active window title
docker exec -e DISPLAY=:0 CONTAINER \
  bash -c 'wid=$(xdotool getactivewindow 2>/dev/null) &&
           echo "Active window: $wid $(xdotool getwindowname "$wid" 2>/dev/null)"'

# 4. Close a blocking "Proxy Server" dialog
docker exec -e DISPLAY=:0 CONTAINER \
  xdotool search --name "Proxy Server" windowfocus key Escape

# 5. Take a screenshot of the X display
docker exec CONTAINER apt-get install -y -q imagemagick
docker exec -e DISPLAY=:0 CONTAINER import -window root /tmp/screen.png
docker cp CONTAINER:/tmp/screen.png ./screen.png

# 6. Inspect the generated AutoHotkey script
docker exec CONTAINER \
  bash -lc 'nl -ba /root/.wine/drive_c/windows/temp/_mt5_install/mt5_install.ahk | sed -n "1,160p"'

# 7. Check which hosts are reachable from the container via docker exec using curl.

# 8. Check if terminal file was installed.

# 9. Check running Wine/MT5 processes
docker exec CONTAINER ps aux | grep -E "mt5|terminal|wine" | grep -v defunct
```

How to analyze the output:

- If `xdotool search --onlyvisible --name "."` shows only `Default IME`
  plus `mt5_install.ahk`, the installer GUI likely did not open and
  AutoHotkey is probably showing an error instead.
- If the active/visible window is `Proxy Server`, dismiss it first and then
  re-check the visible windows list.
- If the screenshot shows an AutoHotkey syntax error instead of the MetaTrader
  installer window, inspect the generated `.ahk` file before investigating
  network access.
- In the 2026-04-24 `metatrader-on-ubuntu-latest` manual debug session, the
  screenshot matched a broken generated script:

  ```text
  32 ; Close a blocking Proxy
  33 Server dialog if it appears.
  34 if (WinExist(Proxy
  35 Server))
  ```

- That pattern means shell quoting broke the `w_ahk_do " ... "` block before
  it was written to the temporary AutoHotkey file, so the install was not
  actually stuck in the MT5 UI.
- If the screenshot instead shows the bootstrapper window with
  "Sorry, something went wrong", treat it as a connectivity issue
  and verify the required hosts listed below.

## Test Results Matrix

Results from testing on 2026-04-19 (re-test, all hosts
allowlisted):

| Step | debian-latest | nixos-latest | ubuntu-jammy | ubuntu-noble | ubuntu-latest |
| --- | :---: | :---: | :---: | :---: | :---: |
| create | ✅ | ✅ | ✅ | ✅ | ✅ |
| prepare | ✅ | ✅ | ✅ | ✅ | ✅ |
| converge | ❌ | ❌ | ❌ | ❌ | ❌ |
| — wine | ✅ | ✅ | ✅ | ✅ | ✅ |
| — xvfb | ✅ | ❌ | ✅ | ✅ | ✅ |
| — metatrader | ❌ | ⏭️ | ❌ | ❌ | ❌ |
| verify | ⏭️ | ⏭️ | ⏭️ | ⏭️ | ⏭️ |

### Failure Details

- **nixos-latest / xvfb**: The `ea31337.xvfb` role tries to include
  `OtherLinux.yml` which does not exist. NixOS reports `os_family` as
  `OtherLinux` and the xvfb role has no matching include file.
- **debian/ubuntu / metatrader verify**: Wine and MT5 setup download
  succeed (`mt5setup.exe` downloaded, `winetricks` verb completed with
  `rc=0`), but `terminal64.exe` / `metaeditor64.exe` are not found
  under `~/.wine/drive_c`. The MT5 stub installer runs but does not
  extract platform binaries — likely the runtime download from
  CDN servers (`cdn.mql5.com`, `trade.mql5.com`) is blocked
  or fails silently inside Wine.

## Common Tasks

### Before Each Commit

- Verify changes: `git diff --no-color`.
- NEVER use `git add .` without reviewing staged files.
- Run linters: `pre-commit run -a`.
- Run `molecule syntax` to catch playbook errors early.

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
