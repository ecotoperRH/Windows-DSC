---
source-path: one-time-config
---

# Migration Plan: Office365-TrustedSites.ps1

**TLDR**: This script configures Internet Explorer trusted sites for Office 365 by reading a list of domains from an XML file and adding them to the Windows registry. It sets both HTTP and HTTPS protocols as trusted (value 2) for each domain in the user's registry.

## Service Type and Configuration

**Service Type**: Other (Internet Explorer Configuration)

**Key Operations**:
- Reads trusted site domains from an XML file
- Creates registry keys for each domain in the Internet Explorer trusted sites zone
- Sets HTTP and HTTPS protocols as trusted for each domain
- Handles primary domains and subdomains separately in the registry structure

## File Structure

**Scripts:**
- one-time-config/special-use-case/Office365-TrustedSites.ps1

**Data Files:**
- one-time-config/special-use-case/Office365-TrustedSites.xml

## Module Explanation

The scripts perform operations in this order:

1. **Office365-TrustedSites.ps1** (`one-time-config/special-use-case/Office365-TrustedSites.ps1`):
   - Sets up variables including the script directory path and registry path
   - Loads the XML file containing trusted sites
   - Defines functions for creating registry keys and setting registry values
   - Loops through each site in the XML file
   - For each site, splits the domain into primary domain and subdomain
   - Creates registry keys for each domain and subdomain
   - Sets HTTP and HTTPS protocols as trusted (value 2) for each domain
   - Ansible equivalent: Use `ansible.windows.win_regedit` to manage registry keys and values

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Get-Content (XML) | ansible.builtin.copy | Copy XML file to target first |
| New-Item (Registry) | ansible.windows.win_regedit | Create registry keys |
| Set-ItemProperty | ansible.windows.win_regedit | Set registry values |
| Split-Path | ansible.builtin.set_fact | Handle path manipulation in Ansible |
| XML parsing | ansible.builtin.xml | Parse XML content |
| Loops (foreach, for) | ansible.builtin.loop | Loop through items |

## Dependencies

**PowerShell Module dependencies**: None  
**Windows Features**: None  
**External packages**: None  
**Service dependencies**: None (Internet Explorer settings only)

## Checks for the Migration

**Files to verify**: None (no files are created or modified)  
**Registry keys**:  
- HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\[domain]\[subdomain]
- Check for "http" and "https" values set to 2 for each domain

**Services to check**: None  
**Firewall rules**: None

## Pre-flight checks:
```powershell
# Check if registry keys exist for a sample domain
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\office365.com\*" -Name "http","https" -ErrorAction SilentlyContinue

# Count total number of trusted sites configured
(Get-ChildItem -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains" -Recurse | Where-Object { $_.Property -contains "http" }).Count
```