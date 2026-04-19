# Molecule Test Execution and Reporting

Your goal is to execute Molecule tests for the `ea31337.metatrader` Ansible
role, splitting them into smaller per-platform steps, and reporting the
results in a structured table.

## Context

Refer to [AGENTS.md](../../AGENTS.md) for setup, environment invariants,
and troubleshooting guidance.

## Prerequisites

Before running any tests, install all dependencies:

```bash
pip install -r .devcontainer/requirements.txt
ansible-galaxy role install -r requirements.yml --force
ansible-galaxy collection install -r requirements.yml -p collections --force
```

## Scenarios

| Scenario  | `metatrader_version` | Notes                              |
| --------- | -------------------- | ---------------------------------- |
| `default` | `5`                  | Default MT5 install                |
| `mt4`     | `4`                  | MT4 install with custom setup URL  |
| `mt5`     | `5`                  | Explicit MT5 install               |
| `mt5-win` | `5`                  | Windows container (disabled in CI) |

## Platforms (Linux Scenarios)

| Container        | Image              | Notes                                  |
| ---------------- | ------------------ | -------------------------------------- |
| `debian-latest`  | `debian:latest`    | WineHQ repo; codename: `bookworm`      |
| `nixos-latest`   | `nixos/nix:latest` | Custom Dockerfile; privileged mode     |
| `ubuntu-jammy`   | `ubuntu:jammy`     | WineHQ repo; codename: `jammy`         |
| `ubuntu-noble`   | `ubuntu:noble`     | WineHQ repo; codename: `jammy`         |
| `ubuntu-latest`  | `ubuntu:latest`    | WineHQ repo; codename: `jammy`         |

## Test Procedure

Run each Molecule step individually per platform to isolate failures.
Use the `default` scenario unless otherwise specified.

### Step 1 — Linting

```bash
yamllint .
ansible-lint
```

### Step 2 — Syntax Check

```bash
molecule syntax -s default
```

### Step 3 — Create All Platforms

```bash
molecule create -s default
```

### Step 4 — Prepare All Platforms

If create already triggered prepare, skip this step. Otherwise:

```bash
molecule prepare -s default
```

### Step 5 — Converge Per Platform

Run converge individually for each platform to isolate failures:

```bash
molecule converge -s default -- --limit debian-latest
molecule converge -s default -- --limit nixos-latest
molecule converge -s default -- --limit ubuntu-jammy
molecule converge -s default -- --limit ubuntu-noble
molecule converge -s default -- --limit ubuntu-latest
```

### Step 6 — Verify Per Platform (only if converge passed)

```bash
molecule verify -s default -- --limit <platform>
```

### Step 7 — Cleanup

```bash
molecule destroy -s default
```

## Reporting

After completing all steps, produce a results table using the following
template. Fill in each cell with one of: `✅` (passed), `❌` (failed),
or `⏭️` (skipped).

### Results Template

```markdown
## Test Results — YYYY-MM-DD

| Step         | debian-latest | nixos-latest | ubuntu-jammy | ubuntu-noble | ubuntu-latest |
| ------------ | ------------- | ------------ | ------------ | ------------ | ------------- |
| create       |               |              |              |              |               |
| prepare      |               |              |              |              |               |
| converge     |               |              |              |              |               |
| — wine       |               |              |              |              |               |
| — xvfb       |               |              |              |              |               |
| — metatrader |               |              |              |              |               |
| idempotence  |               |              |              |              |               |
| verify       |               |              |              |              |               |
```

### Failure Details

For each `❌`, include:

- **Platform**: container name
- **Step**: which molecule step failed
- **Task**: the Ansible task name that failed
- **Error**: brief error message or root cause
- **Suggestion**: potential fix or workaround

## Update Test Results Matrix

After completing tests, update the **Test Results Matrix** section in
[AGENTS.md](../../AGENTS.md) with the new results.
