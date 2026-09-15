# Molecule Testing

## Molecule Scenarios

| Scenario | `metatrader_version` | Notes |
| -------- | -------------------- | ----- |
| `default` | `5` (from defaults) | Default MT5 install |
| `mt4` | `4` | MT4 install with custom setup URL |
| `mt5` | `5` | Explicit MT5 install |
| `mt5-win` | `5` | Windows container (disabled in CI) |

### Platforms (Linux scenarios)

Platform names follow the `<role>-<scenario>-<platform>` convention, so each
scenario gets its own containers. For the `default` scenario:

| Container | Image | Notes |
| --------- | ----- | ----- |
| `metatrader-default-ubuntu-noble` | `ubuntu:noble` | WineHQ repo with `wine_release_codename: noble` |
| `metatrader-default-ubuntu-latest` | `ubuntu:latest` | WineHQ repo with `wine_release_codename: noble` |

The other scenarios use the same suffixes with their own scenario segment, e.g.
`metatrader-mt4-ubuntu-noble`, `metatrader-mt5-ubuntu-noble`. The role and
scenario prefixes keep the Docker containers (named after the platform) unique
across roles and scenarios, so generic names such as `ubuntu-noble` do not
collide with concurrent Molecule runs of other roles or scenarios.

### Running Tests

```bash
# Install dependencies first
pip install -r .devcontainer/requirements.txt
ansible-galaxy role install -r requirements.yml --force
ansible-galaxy collection install -r requirements.yml -p collections

# Full test (all scenarios)
pipenv run molecule test

# Single scenario
pipenv run molecule test -s default

# Single platform in a scenario
pipenv run molecule test -s default --platform-name metatrader-default-ubuntu-noble

# Step-by-step debugging (useful for troubleshooting)
pipenv run molecule destroy -s default              # clean up any leftover state
pipenv run molecule create -s default               # build images + start containers
pipenv run molecule prepare -s default              # install Python, sudo, CA certs
pipenv run molecule converge -s default             # run the role
pipenv run molecule idempotence -s default          # verify idempotency (no changes)
pipenv run molecule verify -s default               # run verification playbook
pipenv run molecule destroy -s default              # clean up

# Syntax check only (fast validation)
pipenv run molecule syntax -s default
pipenv run molecule syntax -s mt4
pipenv run molecule syntax -s mt5
```

Molecule and Ansible are installed via the project `Pipfile`, so run every command through `pipenv`
(they are not on `PATH`).

### Sandboxed / firewalled environments

Sandboxed or firewalled environments may block outbound NAT on the default Docker bridge, resolve
DNS to unroutable IPv6 addresses, or already run an X server on `:0`. Opt in to the following
environment variables as needed:

| Variable | Purpose | Example |
| -------- | ------- | ------- |
| `MOLECULE_DOCKER_FORCE_IPV4` | Prefer IPv4 for DNS resolution in containers. | `true` |
| `MOLECULE_DOCKER_NETWORK` | Docker network for containers and image builds. | `host` |
| `MOLECULE_XVFB_DISPLAY_BASE` | Unique X display per host when sharing the host network. | `90` |

Example invocation:

```bash
MOLECULE_DOCKER_FORCE_IPV4=true \
MOLECULE_DOCKER_NETWORK=host \
MOLECULE_XVFB_DISPLAY_BASE=90 \
pipenv run molecule test -s default
```

### Step-by-step Testing With Timeout

For CI or automated environments, use timeouts:

```bash
# Test a single platform with timeout (15 minutes)
timeout 900 pipenv run molecule test -s default --platform-name metatrader-default-ubuntu-noble

# If converge fails, debug interactively:
pipenv run molecule create -s default --platform-name metatrader-default-ubuntu-noble
pipenv run molecule converge -s default --platform-name metatrader-default-ubuntu-noble
# (inspect container state, then clean up)
pipenv run molecule destroy -s default
```

## Molecule Gates

- `pipenv run molecule syntax` - YAML + playbook syntax validation
- `pipenv run molecule converge` - full role execution on all containers
- `pipenv run molecule idempotence` - re-run must produce zero changes
- `pipenv run molecule verify` - asserts terminal.exe and metaeditor.exe exist

## Troubleshooting Matrix

### Molecule prepare fails with DNS resolution errors

> `Temporary failure resolving 'deb.debian.org'` (or `azure.archive.ubuntu.com`), followed by
> `E: Unable to locate package python3` during the `prepare` step.

- **Root cause**: `docker0` has lost the `bridge` network's configured gateway address, so containers
  on the default bridge have no working gateway and cannot resolve DNS or reach the network.
- **Check**: `ip -4 addr show docker0` vs
  `docker network inspect bridge --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`.
  If the gateway address is missing from `docker0`, this is the cause.
- **Fix (durable)**: `sudo systemctl restart docker` recreates `docker0` with the configured gateway.
  This restarts the daemon and stops any running containers.
- **Fix (non-disruptive, not persistent)**: `sudo ip addr add 172.17.0.1/16 dev docker0`
  (use the gateway reported by the check above).
- **Workaround (no sudo)**: run the tests on the host network, which bypasses the broken bridge.

```bash
MOLECULE_DOCKER_NETWORK=host \
MOLECULE_DOCKER_FORCE_IPV4=true \
MOLECULE_XVFB_DISPLAY_BASE=90 \
pipenv run molecule test
```

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

### GitHub Actions Molecule report step fails with summary size limit

- **Root cause**: GitHub job summaries are capped at 1 MiB, but full Molecule HTML-to-Markdown conversions can exceed it.
- **Fix**: Upload full Molecule HTML reports as workflow artifacts and append only a concise filtered summary
  (e.g., Play Recap, errors, and warnings) to `$GITHUB_STEP_SUMMARY`.

### Molecule report `EACCES: permission denied`

- **Root cause**: The report file generated by `gofrolist/molecule-action` is owned by root with restricted permissions
  because it is created inside a Docker container.
- **Fix**: Run `sudo chown "$USER":"$USER"` on the report file before attempting to read it (for summary) or upload it.

### Wine APT cache update fails

> `Failed to update apt cache after 5 retries`

- **Root cause**: Firewall/network policy blocks `dl.winehq.org`, or
  `debian:latest` codename (e.g. `trixie`) or Ubuntu 26.04 (`resolute`)
  is not in the WineHQ repo.
- **Fix**: Set `wine_release_codename: bookworm` for debian-latest or `noble` for
  `metatrader-<scenario>-ubuntu-latest` in host_vars. Add `dl.winehq.org` to
  firewall allowlist.
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
  If those servers (`www.mql5.com`, `cdn.mql5.com`, `trade.mql5.com`,
  `mt5-trade.metaquotes.net`)
  are DNS-blocked, the installer cannot fetch platform files.
- **Fix**: Ensure **all** hosts are in the firewall allowlist
  (see [Required Hosts](../AGENTS.md#required-hosts) table below).
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
- In the 2026-04-24 `metatrader-on-ubuntu-noble` manual debug session, the
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
- If `winetricks` exits after ~10 minutes with
  `warning: Note: command load_mt4_install/load_mt5_install returned status 1.`
  while `mt5setup.exe` is still running, verify DNS resolution for installer
  backend hosts from inside the container:

  ```bash
  docker exec CONTAINER bash -lc '
    for h in download.mql5.com www.mql5.com cdn.mql5.com trade.mql5.com mt5-trade.metaquotes.net; do
      printf "%s: " "$h"
      timeout 10s getent hosts "$h" >/dev/null && echo DNS_OK || echo DNS_FAIL
    done
  '
  ```

  DNS failures for `cdn.mql5.com` or `mt5-trade.metaquotes.net` can leave
  the installer window open indefinitely and cause the AutoHotkey timeout.

## Test Results Matrix

Results from testing on 2026-04-24 (step-by-step Molecule re-test,
all Linux scenarios):

### `default`

| Step | metatrader-default-ubuntu-noble |
| --- | :---: |
| destroy | ✅ |
| create | ✅ |
| prepare | ✅ |
| converge | ✅ |
| - wine | ✅ |
| - xvfb | ✅ |
| - metatrader | ✅ |
| idempotence | ✅ |
| verify | ✅ |
| destroy (final) | ✅ |

### `mt4`

| Step | metatrader-mt4-ubuntu-noble |
| --- | :---: |
| destroy | ✅ |
| create | ✅ |
| prepare | ✅ |
| converge | ✅ |
| - wine | ✅ |
| - xvfb | ✅ |
| - metatrader | ✅ |
| idempotence | ✅ |
| verify | ✅ |
| destroy (final) | ✅ |

### `mt5`

| Step | metatrader-mt5-ubuntu-noble |
| --- | :---: |
| destroy | ✅ |
| create | ✅ |
| prepare | ✅ |
| converge | ✅ |
| - wine | ✅ |
| - xvfb | ✅ |
| - metatrader | ✅ |
| idempotence | ✅ |
| verify | ✅ |
| destroy (final) | ✅ |

### Improvements applied

- **Robustness**: AHK scripts now exit with code 1 on timeout, preventing false positives if the installer fails.
- **Connectivity**: AHK script now handles "Proxy Server" dialog while waiting for the main window.
- **Startup/Shutdown**: Improved terminal startup and shutdown sequence with longer wait times and multiple process checks.
- **Reliability**: Corrected `Send` command syntax and added `WinActivate` for more reliable key delivery.
- **Compatibility**: The `w_ahk_do` override now joins arguments with a space to prevent splitting into multiple lines.
