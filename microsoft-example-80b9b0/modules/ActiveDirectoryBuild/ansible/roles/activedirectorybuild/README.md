# Active Directory Build Role

This Ansible role automates the deployment and configuration of Active Directory on Windows Server.

## Requirements

- Windows Server 2016 or later
- Ansible 2.10 or later
- The following collections:
  - ansible.windows
  - microsoft.ad

## Role Variables

See `defaults/main.yml` for all variables and their default values.

### Base OS Configuration

```yaml
timezone: "UTC"
execution_policy: "RemoteSigned"
uac_setting: "AlwaysNotify"
admin_path: "C:\\Admin"
api_folder_path: "C:\\API"
```

### Active Directory Configuration

```yaml
# Windows features to install for Active Directory
windows_features:
  - AD-Domain-Services
  - DNS
  - RSAT-AD-Tools
  - RSAT-DNS-Server

# Active Directory forest configuration
domain_name: "example.com"
netbios_name: "EXAMPLE"
safe_mode_password: "P@ssw0rd123!"  # This should be overridden in vault
database_path: "C:\\Windows\\NTDS"
sysvol_path: "C:\\Windows\\SYSVOL"
log_path: "C:\\Windows\\NTDS"
domain_mode: "WinThreshold"  # Windows Server 2016
forest_mode: "WinThreshold"  # Windows Server 2016
```

### Organizational Units, Groups, and Users

The role allows you to define organizational units, groups, and users to be created in Active Directory:

```yaml
# Organizational Units structure
organizational_units:
  - name: "Users"
    path: "DC=example,DC=com"
  - name: "Groups"
    path: "DC=example,DC=com"

# AD Groups
ad_groups:
  - name: "IT_Admins"
    path: "OU=Groups,DC=example,DC=com"
    scope: "Global"
    category: "Security"

# AD Users
ad_users:
  - name: "admin"
    path: "OU=Users,DC=example,DC=com"
    password: "P@ssw0rd123!"  # This should be overridden in vault
    groups:
      - "IT_Admins"
```

## Example Playbook

```yaml
---
- name: Deploy Active Directory
  hosts: windows_servers
  roles:
    - role: activedirectorybuild
      vars:
        domain_name: "corp.example.com"
        netbios_name: "CORP"
```

## Security Considerations

- It is strongly recommended to store sensitive information like passwords in an Ansible Vault.
- Use a strong password for the safe_mode_password and user passwords.
- Consider implementing a more secure password policy for production environments.

## License

MIT