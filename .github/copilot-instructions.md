# Copilot Instructions for ansible-role-wine

You are expected to be an expert in:

- Ansible
- Python
- Jinja2
- Molecule
- Linux (Alpine, Debian/Ubuntu, Nix)
- YAML

## Code Standards

- Follow PEP 8 for Python.
- Include docstrings and type hints where applicable
- Maintain consistent YAML indentation
- Optimize for readability first, performance second
- Prefer modular, DRY approaches and list comprehensions when appropriate
- Use environment variables for configuration, never hardcode sensitive info
- Write clean, documented, error-handling code with appropriate logging

## General Approach

- Be accurate, thorough and terse
- Cite sources at the end, not inline
- Provide immediate answers with clear explanations
- Skip repetitive code in responses; use brief snippets showing only changes
- Suggest alternative solutions beyond conventional approaches
- Treat the user as an expert

## Ansible Guidelines

- Ensure idempotency in all tasks
- Follow standard role structure: tasks/, handlers/, templates/, defaults/, meta/
- Use ansible-lint and write Molecule tests for verification
- Use descriptive task names and include helpful comments

## YAML Guidelines

Rules:

- yaml[truthy]: Truthy value should be one of [false, true]

## Project Specifics

This role installs and configures wine package with distribution-specific approaches:

- **Alpine Linux**: Uses Apk package manager
- **Debian/Ubuntu**: Uses Apt package manager
- **Nix**: Uses nix-env in lightweight Nix environments

Notes:

- Project utilizes Codespaces with config file at .devcontainer/devcontainer.json
  and requirements at .devcontainer/requirements.txt
- GitHub Actions are used to validate the code by running
  pre-commit checks (see .pre-commit-config.yaml file) and Molecule (molecule/).
- Service management uses supervisord across platforms.
- Formatting rules are defined in .yamllint (YAML) and .markdownlint.yaml (Markdown) files.

### Key Variables

Variables are defined in defaults/main.yml and vars/main.yml files.

Notes:

- On variable changes, update main.yml files and README.md accordingly.

### Testing Approach

- Use Molecule with Docker driver
