# AGENTS.md

## Agent Directives (Contract Style)

- **NEVER** mock roles or modules, or use `exclude_paths` for `molecule/` and `tests/` as a workaround for issues.
- All issues **MUST** be resolved correctly at the root cause.
- Adhere strictly to project conventions and established standards.

## Project Invariants

- This project is an Ansible role for MetaTrader.
- It uses Molecule for testing and ansible-lint for linting.
- It depends on external roles such as `ea31337.wine` and `ea31337.xvfb`.
