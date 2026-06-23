# Code Integrity Ansible Role

This repository contains an Ansible role for managing code integrity settings on Windows systems.

## Requirements

- Ansible 2.10+
- Python 3.8+
- For development: Molecule, Docker

## Installation

```bash
pip install -r requirements.txt
```

## Usage

Include the role in your playbook:

```yaml
- hosts: windows_servers
  roles:
    - role: code_integrity
      vars:
        enforce_mode: true
        ci_base_path: "C:\\Program Files\\CodeIntegrity"
```

## Role Variables

| Variable | Description | Default |
|----------|-------------|---------|
| enforce_mode | Whether to enforce code integrity policies | true |
| ci_base_path | Base path for code integrity files | "C:\\Program Files\\CodeIntegrity" |

## Testing

This role includes Molecule tests:

```bash
cd ansible/roles/code_integrity
molecule test
```

## CI/CD

This repository uses GitHub Actions for continuous integration. The workflow runs:
- Ansible Lint
- Molecule tests

## License

MIT