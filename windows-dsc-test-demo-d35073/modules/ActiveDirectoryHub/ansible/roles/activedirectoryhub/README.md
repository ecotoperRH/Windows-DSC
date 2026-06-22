# Active Directory Hub Role

This Ansible role configures a Windows server as an Active Directory Domain Controller.

## Requirements

- Windows Server 2016 or later
- Ansible 2.10 or later
- The following collections:
  - ansible.windows
  - microsoft.ad

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# Domain information
domain_name: "example.com"

# Directory paths
admin_path: "C:\\Admin"
api_folder_path: "C:\\API_Registration"

# Features to install
ad_features:
  - AD-Domain-Services
  - DNS
  - RSAT-AD-Tools
  - RSAT-DNS-Server

# Domain join credentials (should be overridden in vault)
domain_join:
  username: "Administrator"
  password: "ChangeMe123!"

# Domain controller safe mode credentials (should be overridden in vault)
domain_controller_join:
  password: "SafeMode123!"
```

## Dependencies

None.

## Example Playbook

```yaml
- hosts: domain_controllers
  roles:
    - role: activedirectoryhub
      vars:
        domain_name: "corp.example.com"
        domain_join:
          username: "admin@corp.example.com"
          password: "{{ vault_domain_admin_password }}"
        domain_controller_join:
          password: "{{ vault_safe_mode_password }}"
```

## License

MIT

## Author Information

This role was created by the Ansible Migration Team.