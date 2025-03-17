# Ansible Role Code Review Guidelines

Your goal is to review Ansible role code
to ensure it meets high-quality standards and follows best practices.

## Ensure Code Quality

- Follow PEP 8 for Python.
- Maintain consistent YAML indentation.
- Optimize for readability first, performance second.
- Ensure idempotency in all Ansible tasks.
- Use environment variables for configuration, never hardcode sensitive info.
- Write clean, documented, error-handling code with appropriate logging.

## Check for Linting and Formatting

- Ensure YAML files adhere to [.yamllint](../../.yamllint) rules.
- Ensure Markdown files adhere to [.markdownlint.yaml](../../.markdownlint.yaml) rules.
- Ensure Ansible playbooks and roles pass `ansible-lint`.

## Review Ansible Role Structure

- Follow standard role structure:

  - [tasks/](../../tasks/)
  - [handlers/](../../handlers/)
  - [templates/](../../templates/)
  - [defaults/](../../defaults/)
  - [meta/](../../meta/)

- Use descriptive task names and include helpful comments.
- Ensure indentation is correct, especially for arguments used for Ansible modules.

## Verify Variables and Defaults

- Ensure variables are defined in
  [defaults/main.yml](../../defaults/main.yml) and [vars/main.yml](../../vars/main.yml).
- Update [README.md](../../README.md) with any changes to variables.
- Ensure variables are used consistently across tasks and playbooks.

## Check for Idempotency

- Ensure all tasks are idempotent and do not cause unnecessary changes on repeated runs.

## Review Molecule Tests

- Ensure Molecule scenarios are defined and cover all supported platforms.
- Verify that Molecule tests are comprehensive and validate the role's functionality.
- Check [molecule/](../../molecule/) directory for test configurations.

## Ensure Proper Error Handling

- Write error-handling code with appropriate logging.
- Ensure tasks fail gracefully and provide meaningful error messages.

## Check for Dependencies

- Ensure all dependencies are listed in
  [.devcontainer/requirements.txt](../../.devcontainer/requirements.txt) and [requirements.yml](../../requirements.yml).
- Verify that the role does not have any missing dependencies.

## Review GitHub Actions

- Ensure GitHub Actions workflows are correctly configured to run pre-commit checks and Molecule tests.
- Verify that the workflows cover all necessary validation steps.
- Check [.github/workflows/](../) directory for workflow configurations.

## Documentation

- Ensure the [README.md](../../README.md) file is up-to-date
  and provides clear instructions for installation, usage, and variables.
- Include any additional documentation as needed for clarity.
