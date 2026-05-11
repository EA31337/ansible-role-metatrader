# Ansible Role: MetaTrader

[![Build Status](https://github.com/EA31337/ansible-role-metatrader/workflows/Molecule/badge.svg)](https://github.com/EA31337/ansible-role-metatrader/actions)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-ea31337.metatrader-blue.svg)](https://galaxy.ansible.com/ea31337/metatrader/)

Role to install MetaTrader platform on Linux (via Wine) or Windows.

## Requirements

None.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
metatrader_setup_url: "https://download.mql5.com/cdn/web/metaquotes/mt5/mt5setup.exe"
metatrader_version: 5
```

## Dependencies

- [ea31337.wine](https://github.com/EA31337/ansible-role-wine)
- [ea31337.xvfb](https://github.com/EA31337/ansible-role-xvfb)

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: ea31337.metatrader
      vars:
        metatrader_version: 5
```

## License

GPL-3.0-or-later

## Author Information

This role was created in 2025 by [kenorb](https://github.com/kenorb).
