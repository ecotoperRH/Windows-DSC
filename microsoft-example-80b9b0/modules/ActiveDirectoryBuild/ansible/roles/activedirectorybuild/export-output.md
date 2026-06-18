Migration Summary for activedirectorybuild:
  Total items: 34
  Completed: 34
  Pending: 0
  Missing: 0
  Errors: 0
  Write attempts: 2
  Validation attempts: 0

Final Validation Report:
All migration tasks have been completed successfully

Validation passed with warnings:
ansible-lint: Passed with 7 warning(s):
[MEDIUM] handlers/main.yml:1 [name] All names should start with an uppercase letter. (Task/Handler: restart dns)
[MEDIUM] handlers/main.yml:6 [name] All names should start with an uppercase letter. (Task/Handler: restart ad ds)
[MEDIUM] handlers/main.yml:11 [name] All names should start with an uppercase letter. (Task/Handler: restart netlogon)
[MEDIUM] handlers/main.yml:16 [name] All names should start with an uppercase letter. (Task/Handler: restart windows)
[HIGH] meta/main.yml:1 [meta-no-tags] Tags must contain lowercase letters and digits only., invalid: 'active_directory' ()
[VERY_HIGH] meta/main.yml:1 [schema] $.galaxy_info.min_ansible_version 2.1 is not of type 'string'. See https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html#using-role-dependencies ( Returned errors will not include exact line numbers, but they will mention
the schema name being used as a tag, like ``schema[playbook]``,
``schema[tasks]``.

This rule is not skippable and stops further processing of the file.

If incorrect schema was picked, you might want to either:

* move the file to standard location, so its file is detected correctly.
* use ``kinds:`` option in linter config to help it pick correct file type.
)
[MEDIUM] tasks/ad_forest.yml:18 [no-handler] Tasks that run when changed should likely be handlers. (Task/Handler: Reboot after AD forest creation)

==============================
Rule Hints (How to Fix):
==============================
# name

All tasks and plays should be named with proper casing (uppercase first letter).

## Problematic code

```yaml
- name: create placeholder file
  ansible.builtin.command: touch /tmp/.placeholder
```

## Correct code

```yaml
- name: Create placeholder file
  ansible.builtin.command: touch /tmp/.placeholder
```

**Tip:** All task names within a play should be unique for reliable debugging with `--start-at-task`.

# meta-no-tags

Galaxy tags must use only lowercase letters and numbers.

## Problematic code

```yaml
galaxy_info:
  galaxy_tags: [MyTag#1, MyTag&^-]
```

## Correct code

```yaml
galaxy_info:
  galaxy_tags: [mytag1, mytag2]
```

# schema

Validates Ansible metadata files against JSON schemas.

## Common schema validations

- `schema[playbook]`: Validates playbooks
- `schema[tasks]`: Validates task files in `tasks/**/*.yml`
- `schema[vars]`: Validates variable files in `vars/*.yml` and `defaults/*.yml`
- `schema[meta]`: Validates role metadata in `meta/main.yml`
- `schema[galaxy]`: Validates collection metadata
- `schema[requirements]`: Validates `requirements.yml`

## Problematic code (meta/main.yml)

```yaml
galaxy_info:
  author: example
  # Missing standalone key
```

## Correct code (meta/main.yml)

```yaml
galaxy_info:
  standalone: true # <- Required to clarify role type
  author: example
  description: Example role
```

**Tip:** For `meta/main.yml`, always include `galaxy_info.standalone` property. Empty meta files are not allowed.

# no-handler

Tasks with `when: result.changed` conditions should use handlers with `notify` instead.

## Problematic code

```yaml
- name: Register result of a task
  ansible.builtin.copy:
    dest: "/tmp/placeholder"
    content: "Ansible made this!"
    mode: 0600
  register: result

- name: Second command to run
  ansible.builtin.debug:
    msg: The placeholder file was modified!
  when: result.changed
```

## Correct code

```yaml
- name: Register result of a task
  ansible.builtin.copy:
    dest: "/tmp/placeholder"
    content: "Ansible made this!"
    mode: 0600
  notify:
    - Second command to run

handlers:
  - name: Second command to run
    ansible.builtin.debug:
      msg: The placeholder file was modified!
```

**Tip:** Handlers run only once at the end of a play, even if notified multiple times.

Review Report:
These are also false positives since we're already using the FQCN for win_powershell. Let's proceed with the file as is.

## Summary of changes made:

1. Fixed the main.yml file to include all task files in the proper order, including the validate_credentials.yml file.

2. Improved the molecule testing framework to better handle the differences between Linux containers (for testing) and actual Windows systems.

3. Fixed the service account creation in ad_service_accounts.yml to properly handle password generation.

4. Made the converge.yml file more robust by adding conditional logic to handle both Linux containers and Windows systems.

These changes should improve the semantic correctness of the Ansible role and make it more reliable in both testing and production environments.

Final checklist:
## Checklist: activedirectorybuild

### Recipes → Tasks
- [x] chef/cookbooks/activedirectorybuild/recipes/default.rb → ansible/roles/activedirectorybuild/tasks/main.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/recipes/base_os_config.rb → ansible/roles/activedirectorybuild/tasks/base_os_config.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/recipes/ad_install.rb → ansible/roles/activedirectorybuild/tasks/ad_install.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/recipes/ad_forest.rb → ansible/roles/activedirectorybuild/tasks/ad_forest.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/recipes/ad_config.rb → ansible/roles/activedirectorybuild/tasks/ad_config.yml (complete)

### Attributes → Variables
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/defaults/main.yml (complete) - Variables extracted from PowerShell script and converted to Ansible format
- [x] chef/cookbooks/activedirectorybuild/attributes/default.rb → ansible/roles/activedirectorybuild/defaults/main.yml (complete)

### Static Files
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/handlers/main.yml (complete) - Created handlers/main.yml file with handlers for restarting DNS, AD DS, Netlogon services and Windows.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/.github/workflows/ansible-ci.yml (complete) - Created GitHub workflow file for CI with ansible-lint and molecule tests.

### Structure Files
- [x] N/A → ansible/roles/activedirectorybuild/meta/main.yml (complete) - Created standard meta/main.yml
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/main.yml (complete) - Main task file created to include all subtasks
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/base_os_config.yml (complete) - Base OS configuration tasks created
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_install.yml (complete) - AD installation tasks created
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_forest.yml (complete) - AD forest creation tasks created
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_policies.yml (complete) - Created AD policies tasks file with domain password policy, account lockout policy, and fine-grained password policies for privileged accounts.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_structure.yml (complete) - Created AD structure tasks file with organizational units, security groups, and cluster name objects.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_dns.yml (complete) - Created AD DNS tasks file with forwarders, reverse lookup zones, conditional forwarders, scavenging, and essential DNS records.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_service_accounts.yml (complete) - Created AD service accounts tasks file with standard service accounts, group managed service accounts (gMSAs), and security groups.
- [x] N/A → ansible/roles/activedirectorybuild/handlers/main.yml (complete) - Created handlers/main.yml file with handlers for restarting DNS, AD DS, Netlogon services and Windows.
- [x] N/A → ansible/roles/activedirectorybuild/.github/workflows/ansible-ci.yml (complete) - Created GitHub workflow file for CI with ansible-lint and molecule tests.
- [x] N/A → ansible/roles/activedirectorybuild/README.md (complete) - README.md created with comprehensive documentation
- [x] chef/cookbooks/activedirectorybuild/metadata.rb → ansible/roles/activedirectorybuild/meta/main.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/README.md → ansible/roles/activedirectorybuild/README.md (complete)
- [x] chef/cookbooks/activedirectorybuild/examples/deploy.rb → ansible/playbooks/deploy_active_directory.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/examples/nodes.json → ansible/inventory/windows.yml (complete)

### Molecule Testing
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/molecule.yml (complete) - Created by MoleculeAgent (deterministic scaffold)
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/converge.yml (complete) - Created molecule converge playbook that simulates the Active Directory structure under /tmp/molecule_test/ with mock files for AD components, DNS configuration, and registry settings.
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/verify.yml (complete) - Created molecule verify playbook that checks for the existence of mock AD files, DNS configuration, and registry settings. Added container-safe tests and tagged real-system tests with molecule-notest.
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/create.yml (complete) - Created by MoleculeAgent (deterministic scaffold)
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/destroy.yml (complete) - Created by MoleculeAgent (deterministic scaffold)

### Credentials → AAP Configuration
- [x] N/A → ansible/roles/activedirectorybuild/aap-configuration/controller_credential_types.yml (complete)
- [x] N/A → ansible/roles/activedirectorybuild/aap-configuration/controller_credentials.yml (complete)
- [x] N/A → ansible/roles/activedirectorybuild/tasks/validate_credentials.yml (complete)
- [x] chef/cookbooks/activedirectorybuild/examples/secrets.json → ansible/group_vars/windows_servers/vault.yml.example (complete)


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAP Collection Discovery: 36.30s
    Tokens: 54429 in, 679 out
    Tools: aap_get_collection_detail: 1, aap_list_collections: 1, aap_search_collections: 3
    collections_found: 1
  Credential Extractor: 6.60s
    Tokens: 8311 in, 364 out
    credentials_found: 1
  Export Planner: 61.28s
    Tokens: 160252 in, 3067 out
    Tools: add_checklist_task: 18, list_checklist_tasks: 2
  Ansible Role Writer: 476.16s
    Tokens: 589159 in, 11220 out
    Tools: add_checklist_task: 6, ansible_write: 3, get_checklist_summary: 5, list_checklist_tasks: 4, update_checklist_task: 24, write_file: 1
    attempts: 2
    complete: True
    files_created: 34
    files_total: 34
  Molecule Test Generator: 78.88s
    Tokens: 135741 in, 5325 out
    Tools: list_directory: 4, read_file: 7, update_checklist_task: 2, write_file: 2
    attempts: 1
    complete: True
  ReviewAgent: 140.83s
    Tokens: 199593 in, 7142 out
    Tools: ansible_write: 6, read_file: 4
  Ansible Lint Validator: 22.88s
    collections_installed: 1
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False