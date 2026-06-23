# Code Integrity Role

This role manages Windows Code Integrity policies to control which applications can run on Windows systems.

## Requirements

- Windows Server 2019 or newer
- PowerShell 5.1 or newer

## Role Variables

| Variable | Description | Default |
|----------|-------------|---------|
| enforce_mode | Whether to apply policy in enforce mode (true) or audit mode (false) | false |
| ci_base_path | Base path for storing Code Integrity policies | C:\ConfigMgr\CodeIntegrity |

## Dependencies

None.

## Example Playbook

```yaml
- hosts: windows_servers
  roles:
    - role: code_integrity
      enforce_mode: false  # Start in audit mode
```

## License

MIT

## Author Information

Ansible Team