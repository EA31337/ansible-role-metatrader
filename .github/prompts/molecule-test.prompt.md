# Molecule Test Runner

Run all Molecule scenarios and report results as a table.

## Context

Refer to [AGENTS.md](../../AGENTS.md) for setup, environment invariants,
and troubleshooting guidance.

## Instructions

1. Install dependencies (if not already present):

   ```bash
   pip install -r .devcontainer/requirements.txt
   ansible-galaxy role install -r requirements.yml --force
   ansible-galaxy collection install -r requirements.yml -p collections
   ```

2. For each scenario below, run each Molecule step individually
   to isolate failures:

   ```bash
   molecule destroy -s <scenario>
   molecule create -s <scenario>
   molecule converge -s <scenario>
   molecule idempotence -s <scenario>
   molecule verify -s <scenario>
   molecule destroy -s <scenario>
   ```

3. Record every step outcome (✅ pass, ❌ fail, ⏭️ skipped).

4. Report results in a **Scenario × Platform** table (see template below).

## Scenarios

| Scenario  | `metatrader_version` | Notes                              |
| --------- | -------------------- | ---------------------------------- |
| `default` | `5`                  | Default MT5 install                |
| `mt4`     | `4`                  | MT4 install with custom setup URL  |
| `mt5`     | `5`                  | Explicit MT5 install               |
| `mt5-win` | `5`                  | Windows container (disabled in CI) |

## Platforms (shared by all Linux scenarios)

| Container       | Image              | Notes                              |
| --------------- | ------------------ | ---------------------------------- |
| `debian-latest` | `debian:latest`    | WineHQ repo; codename: `bookworm`  |
| `nixos-latest`  | `nixos/nix:latest` | Custom Dockerfile; privileged mode |
| `ubuntu-jammy`  | `ubuntu:jammy`     | WineHQ repo; codename: `jammy`     |
| `ubuntu-noble`  | `ubuntu:noble`     | WineHQ repo; codename: `jammy`     |
| `ubuntu-latest` | `ubuntu:latest`    | WineHQ repo; codename: `jammy`     |

## Results Template

Fill in each cell after running the tests.
Use ✅ for pass, ❌ for fail, ⏭️ for skipped.

### Step-Level Results (per scenario)

For each scenario, report per-platform step results:

| Platform        | create | prepare | converge | idempotence | verify |
| --------------- | :----: | :-----: | :------: | :---------: | :----: |
| `debian-latest` |        |         |          |             |        |
| `nixos-latest`  |        |         |          |             |        |
| `ubuntu-jammy`  |        |         |          |             |        |
| `ubuntu-noble`  |        |         |          |             |        |
| `ubuntu-latest` |        |         |          |             |        |

### Converge Sub-Step Results

For converge failures, break down by sub-step:

| Platform        | wine | xvfb | metatrader |
| --------------- | :--: | :--: | :--------: |
| `debian-latest` |      |      |            |
| `nixos-latest`  |      |      |            |
| `ubuntu-jammy`  |      |      |            |
| `ubuntu-noble`  |      |      |            |
| `ubuntu-latest` |      |      |            |

### Summary (all scenarios)

| Platform        | default | mt4 | mt5 | mt5-win |
| --------------- | :-----: | :-: | :-: | :-----: |
| `debian-latest` |         |     |     |         |
| `nixos-latest`  |         |     |     |         |
| `ubuntu-jammy`  |         |     |     |         |
| `ubuntu-noble`  |         |     |     |         |
| `ubuntu-latest` |         |     |     |         |

### Failure Details

For each ❌, include:

- **Platform**: container name
- **Step**: which molecule step failed
- **Task**: the Ansible task name that failed
- **Error**: brief error message or root cause
- **Suggestion**: potential fix or workaround

## Update Test Results Matrix

After completing tests, update the **Test Results Matrix** section in
[AGENTS.md](../../AGENTS.md) with the new results.

## Troubleshooting

- If NixOS fails with SSL errors, check `Dockerfile.j2` CA cert injection.
- If Wine GPG key download fails, verify `dl.winehq.org` is reachable.
- If pip fails inside molecule-action, ensure `create.yml`/`destroy.yml`
  use `ansible.builtin.command` instead of `ansible.builtin.pip`.
- If MetaTrader download fails, verify `download.mql5.com` is reachable.
- Refer to [AGENTS.md](../../AGENTS.md) for the full troubleshooting matrix.
