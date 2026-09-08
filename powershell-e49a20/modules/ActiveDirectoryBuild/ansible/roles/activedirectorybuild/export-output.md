## Migration Summary for activedirectorybuild

- **Total items:** 28
- **Completed:** 28
- **Pending:** 0
- **Missing:** 0
- **Errors:** 0
- **Write attempts:** 1
- **Validation attempts:** 0

### Final Validation Report

All migration tasks have been completed successfully

All validations passed

### Review Report

---

## Review Summary

### Findings

| # | Category | Severity | File | Issue | Status |
|---|----------|----------|------|-------|--------|
| 1 | **Invalid Assertions** | 🔴 High | `tasks/validate_credentials.yml` | `assert` checked `username is defined` and `password is defined` four times each — the raw injected vars, not the four actual role credential variables (`activedirectorybuild_default_dc_password`, `activedirectorybuild_dc_join_password`, `activedirectorybuild_domain_join_password`, `activedirectorybuild_temp_password`). A missing credential would silently pass validation. | ✅ Fixed |
| 2 | **Idempotency Failure** | 🔴 High | `tasks/ad_forest.yml` | `win_service` tasks for ADWS and NTDS had no `retries`/`delay`/`until`. After `microsoft.ad.domain` reboots the DC, these services take 30–120 seconds to start — the tasks would fail immediately on the first run. | ✅ Fixed |
| 3 | **Idempotency Failure** | 🟡 Medium | `tasks/base_os_hardening.yml` | `Set-TimeZone` and `Set-ExecutionPolicy` used `changed_when: true` unconditionally — always reported changed even when the timezone/policy was already correct. | ✅ Fixed |
| 4 | **Idempotency Failure** | 🟡 Medium | `tasks/ad_password_policy.yml` | `Set-ADDefaultDomainPasswordPolicy` used `changed_when: true` unconditionally — always reported changed on every run regardless of current policy state. | ✅ Fixed |
| 5 | **Idempotency Failure** | 🟡 Medium | `tasks/ad_dns.yml` | `Set-DnsServerForwarder` ran unconditionally with `changed_when: true` — no guard to check whether the forwarders were already configured. | ✅ Fixed |
| 6 | **Molecule Correctness** | 🟡 Medium | `molecule/default/converge.yml` | `gather_facts: true` set but no Ansible facts were referenced anywhere in the play (wasteful, adds ~2s per run). All 9 `ansible.builtin.copy` tasks had `backup: true`, causing `.bak` file accumulation in `/tmp/molecule_test/` on every re-run, which would cause `stat` assertions in verify.yml to see unexpected files. | ✅ Fixed |

### Changes Made

| File | Change |
|------|--------|
| `tasks/validate_credentials.yml` | Replaced 8 duplicate `username`/`password` assertions with 8 correct assertions covering all 4 role credential variables (`activedirectorybuild_*_password`) plus non-empty length checks |
| `tasks/ad_forest.yml` | Added `retries: 12`, `delay: 15`, `register:`, and `until: not failed` to both `win_service` tasks (ADWS, NTDS) — allows up to 3 minutes for services to start after DC promotion reboot |
| `tasks/base_os_hardening.yml` | Added `Get-TimeZone` and `Get-ExecutionPolicy` pre-check tasks (`changed_when: false`); gated `Set-TimeZone` and `Set-ExecutionPolicy` behind `when:` conditions comparing current vs desired state |
| `tasks/ad_password_policy.yml` | Added `Get-ADDefaultDomainPasswordPolicy \| ConvertTo-Json` pre-check task (`changed_when: false`); gated `Set-ADDefaultDomainPasswordPolicy` behind a `when:` condition comparing all 9 policy fields |
| `tasks/ad_dns.yml` | Added `Get-DnsServerForwarder` pre-check task (`changed_when: false`); gated `Set-DnsServerForwarder` behind `when:` checking both forwarder IPs are present |
| `molecule/default/converge.yml` | Changed `gather_facts: true` → `gather_facts: false`; removed `backup: true` from all 9 `ansible.builtin.copy` tasks |

### No Issues Found

- **Category 1 – Missing Prerequisites:** All Windows registry, file, and AD object tasks operate on paths/objects that either already exist on a Windows DC or are created by prior tasks in the correct order.
- **Category 2 – Missing Package Dependencies:** All Windows features (AD DS, DNS, RSAT, BitLocker) are installed in `install_features.yml` before any AD/DNS configuration tasks run.
- **Category 4 – Ordering Issues:** `main.yml` orchestrates tasks in the correct sequence: credentials → OS hardening → feature install → forest promotion → password policy → sites → KDS key → groups → gMSAs → OUs → computers → DNS → users.
- **Category 5 – Invalid Module Parameters:** No `variables:` misuse or unsupported module parameters found. All `microsoft.ad.*` and `ansible.windows.*` module parameters are valid.
- **Category 6 – Missing Argument Specs:** `meta/argument_specs.yml` exists and covers all 24 variables defined in `defaults/main.yml` plus the 4 required credential variables.
- **Category 7 – Molecule `become: true`:** Not present in converge.yml or verify.yml.
- **Category 7 – Molecule `include_role`:** Not present in converge.yml.
- **Category 7 – Molecule paths outside `/tmp/molecule_test/`:** All paths correctly use the `/tmp/molecule_test/` prefix.
- **Category 7 – Molecule `prepare.yml`:** Does not exist.
- **Category 7 – Molecule `molecule-notest` tags:** All Windows-only checks in verify.yml Play 4 (`service_facts`, `win_shell` AD queries) are correctly tagged `molecule-notest`.

### Final Checklist

## Checklist: activedirectorybuild

### Recipes → Tasks
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/main.yml (complete) - Main orchestration file including validate_credentials first, then all sub-task files in order
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/base_os_hardening.yml (complete) - Fixed: added pre-check tasks (Get-TimeZone, Get-ExecutionPolicy) with changed_when: false, then gated Set-TimeZone and Set-ExecutionPolicy behind when: conditions comparing current vs desired state.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/install_features.yml (complete) - Installs all AD DS, DNS, RSAT, BitLocker features as a list with sub-features
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_forest.yml (complete) - Fixed: added retries: 12 / delay: 15 / until: not failed to both ADWS and NTDS win_service tasks to survive the post-reboot startup window (up to 3 minutes).
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_password_policy.yml (complete) - Fixed: added a read task (Get-ADDefaultDomainPasswordPolicy | ConvertTo-Json) with changed_when: false, then gated Set-ADDefaultDomainPasswordPolicy behind a when: condition comparing all 9 policy fields.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_sites.yml (complete) - Creates/renames AD replication sites: cool-name-SE1 (renamed from default), cool-name-LAS, cool-name-ORG
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_kds_key.yml (complete) - Creates KDS Root Key with backdated effective time using Add-KdsRootKey -Force
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_groups.yml (complete) - Creates all 11 AD security groups (Universal/Security scope) using microsoft.ad.group
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_gmsas.yml (complete) - Creates gmsaSVC-SQL and gmsaSVC-SCVMM gMSA accounts using microsoft.ad.service_account
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_ous.yml (complete) - Creates full OU hierarchy in dependency order using microsoft.ad.ou
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_computers.yml (complete) - Pre-stages 6 CNOs (COREADMIN, CORESQL, CORESQLDR, COREPRIMARY, CORESECONDARY, CORESOFS) as disabled computer accounts
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_dns.yml (complete) - Fixed: added Get-DnsServerForwarder pre-check task with changed_when: false, then gated Set-DnsServerForwarder behind when: checking if both forwarder IPs are already present in current config.
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/ad_users.yml (complete) - Creates AD-Domain-Join, DC-Domain-Join service accounts and AdminOne/Two/Three admin users
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/tasks/validate_credentials.yml (complete) - Fixed: replaced 4x duplicate username/password checks with the 4 actual role credential variables (activedirectorybuild_default_dc_password, activedirectorybuild_dc_join_password, activedirectorybuild_domain_join_password, activedirectorybuild_temp_password) plus non-empty length checks.

### Structure Files
- [x] N/A → ansible/roles/activedirectorybuild/meta/main.yml (complete) - Created standard meta/main.yml
- [x] N/A → ansible/roles/activedirectorybuild/meta/argument_specs.yml (complete) - Full argument specs for all role variables including credential variables injected by AAP
- [x] N/A → ansible/roles/activedirectorybuild/handlers/main.yml (complete) - Handlers for reboot after domain promotion and feature install
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/defaults/main.yml (complete) - Converted all DSC variables and configuration to Ansible defaults
- [x] ActiveDirectoryBuild.ps1 → ansible/roles/activedirectorybuild/vars/vault.yml (complete) - Credential variables mapped from Azure Automation to AAP credential type injected variables

### Dependencies (requirements.yml)
- [x] collection:ansible.windows → ansible/roles/activedirectorybuild/requirements.yml (complete) - Requirements for ansible.windows and microsoft.ad collections

### Molecule Testing
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/molecule.yml (complete) - Created by MoleculeAgent (deterministic scaffold)
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/converge.yml (complete) - Fixed: changed gather_facts: true to gather_facts: false (no facts used in play). Removed backup: true from all 9 ansible.builtin.copy tasks to prevent .bak file accumulation on re-runs.
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/verify.yml (complete) - Split into 4 plays: (1) base OS/filesystem markers, (2) AD groups/gMSAs/OUs, (3) CNOs/DNS/users, (4) Windows-only AD DS/service checks tagged molecule-notest. Uses stat+assert+slurp pattern throughout. All loops use prefixed loop_vars and bracket notation.
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/create.yml (complete) - Created by MoleculeAgent (deterministic scaffold)
- [x] N/A → ansible/roles/activedirectorybuild/molecule/default/destroy.yml (complete) - Created by MoleculeAgent (deterministic scaffold)

### Credentials → AAP Configuration
- [x] N/A → ansible/roles/activedirectorybuild/aap-configuration/controller_credential_types.yml (complete)
- [x] N/A → ansible/roles/activedirectorybuild/aap-configuration/controller_credentials.yml (complete)
- [x] N/A → ansible/roles/activedirectorybuild/tasks/validate_credentials.yml (complete)


### Telemetry

```
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAP Collection Discovery: 34.49s
    Tokens: 139671 in, 1511 out
    Tools: aap_get_collection_detail: 3, aap_list_collections: 2, aap_search_collections: 9
    collections_found: 0
  Credential Extractor: 14.04s
    Tokens: 21944 in, 1086 out
    credentials_found: 4
  Export Planner: 125.41s
    Tokens: 808544 in, 5332 out
    Tools: add_checklist_task: 24, list_checklist_tasks: 2, list_directory: 3, read_file: 1
  Ansible Role Writer: 430.73s
    Tokens: 3153754 in, 21324 out
    Tools: ansible_doc_lookup: 13, ansible_lint: 3, ansible_write: 20, file_search: 1, list_checklist_tasks: 2, list_directory: 4, read_file: 6, update_checklist_task: 18
    attempts: 1
    complete: True
    files_created: 22
    files_total: 27
  Molecule Test Generator: 98.14s
    Tokens: 266391 in, 9278 out
    Tools: list_checklist_tasks: 1, list_directory: 1, read_file: 12, update_checklist_task: 2, write_file: 2
    attempts: 1
    complete: True
  ReviewAgent: 160.72s
    Tokens: 429326 in, 12982 out
    Tools: add_checklist_task: 7, ansible_write: 5, get_checklist_summary: 1, list_checklist_tasks: 1, list_directory: 4, read_file: 27, update_checklist_task: 6, write_file: 1
  Ansible Lint Validator: 45.38s
    collections_installed: 2
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False
```