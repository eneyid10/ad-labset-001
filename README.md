# Lab 01 — Active Directory
### Windows Server 2025 · Azure Free Account · Identity & Access Management

![Platform](https://img.shields.io/badge/Platform-Windows%20Server%202025-0078D4?style=flat-square&logo=windows&logoColor=white)
![Cloud](https://img.shields.io/badge/Cloud-Microsoft%20Azure-0089D6?style=flat-square&logo=microsoft-azure&logoColor=white)
![Cost](https://img.shields.io/badge/Cost-%240%20Free%20Tier-22c55e?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-3–5%20Hours-f59e0b?style=flat-square)
![Certs](https://img.shields.io/badge/Aligned-Network%2B%20%7C%20Security%2B%20%7C%20Azure%20Admin-6366f1?style=flat-square)

---
# [Watch Me Do This Lab Here]([https://www.example.com](https://www.loom.com/share/458749e7e0fa4dca988b09084f350943))

## Overview

This lab documents the end-to-end build of an Active Directory environment on Windows Server 2025, hosted on an Azure free-tier VM. It covers domain controller promotion, organisational unit design, role-based access control via security groups, Group Policy enforcement, and common help desk operations — all skills that transfer directly to enterprise Windows and hybrid cloud environments.

Active Directory is the identity backbone of the majority of enterprise organisations. Understanding how to build and operate it is foundational knowledge for IT Support, Sysadmin, Cloud Engineer, and Security Analyst roles alike. Hybrid environments sync on-premises AD to Microsoft Entra ID (formerly Azure AD), meaning this knowledge applies equally to cloud roles.

---

## Architecture

<img width="679" height="423" alt="a51c326870b749c49f29a0e66829af957e81246c00694163ba3bb6a070129044" src="https://github.com/user-attachments/assets/a38f30d9-f9a0-434d-ae5c-5cf626ff3b26" />

## Lab Details

| Field | Value |
|---|---|
| **OS** | Windows Server 2025 Datacenter — Gen2 |
| **VM Size** | Standard_B2s (2 vCPU, 4 GB RAM) |
| **Hosting** | Microsoft Azure Free Tier |
| **Domain** | `lab.local` |
| **Forest** | `lab.local` |
| **NetBIOS** | `LAB` |
| **Estimated Cost** | $0 — free tier credits + 180-day eval licence |
| **Time to Complete** | 3–5 hours across multiple sessions |
| **Cert Alignment** | CompTIA Network+, Security+, Azure Administrator |

---

## What This Lab Covers

| Skill | Real-World Application |
|---|---|
| Promote Windows Server to Domain Controller | First step in every enterprise Windows environment |
| Create Organisational Units (OUs) | Structure directory by department for targeted policy application |
| Create users, groups, and group memberships | Foundation of role-based access control at any scale |
| Configure Group Policy Objects (GPOs) | Centrally enforce security settings across all managed machines |
| Join a machine to the domain | Connect workstations as managed, policy-enforced resources |
| Configure role-based access with security groups | Least privilege applied practically |
| Reset passwords and manage account lifecycle | Top help desk task — done correctly from day one |

---

## Prerequisites

- Azure free account ([azure.microsoft.com/free](https://azure.microsoft.com/free)) **or** VirtualBox with 8 GB RAM on host
- Remote Desktop client (native app — not browser-based console)
- Basic familiarity with Windows Server Manager

---

## Infrastructure Setup

### Azure VM Configuration

| Setting | Value | Notes |
|---|---|---|
| Region | East US | Lowest cost, best free-tier availability |
| Image | Windows Server 2025 Datacenter Gen2 | Includes 180-day evaluation licence |
| Size | Standard_B2s | Minimum comfortable size for AD DS |
| Authentication | Password | Used for RDP access |
| Inbound port | RDP 3389 | Required for remote connection |
| OS Disk | Standard SSD | Included in free tier |

> **Cost tip:** Stop the VM (not delete) between sessions. A B2s runs at ~$0.05/hr. Stopping pauses compute billing — your $200 free credit lasts significantly longer.

### Enable Clipboard Over RDP

Open Remote Desktop → **Show Options** → **Local Resources** tab → check **Clipboard**.

Alternatively, download the RDP file from the Azure portal (Connect → Download RDP File) and open it with the native Remote Desktop app. The browser-based console has very limited clipboard support and is not recommended for lab work.

---

## Steps Performed

### Step 1 — Install Active Directory Domain Services

Installed the AD DS role and Group Policy Management Console via Server Manager or PowerShell.

```powershell
# Install AD DS role and management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Install Group Policy Management Console (required for Step 5)
Install-WindowsFeature -Name GPMC
```

> Install GPMC at this stage — it is a separate feature from AD DS and is required before configuring Group Policy in Step 5.

---

### Step 2 — Promote the Server to Domain Controller

Promoted the server to create the `lab.local` forest and domain. The server automatically restarts and becomes the authoritative DNS and identity server for the domain.

```powershell
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName 'lab.local' `
  -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword `
    (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```

> Record the DSRM password securely — it is only needed for disaster recovery, but without it you are locked out of recovery options entirely.

---

### Step 3 — Build Organisational Structure

Created five OUs, four department security groups, four user accounts, and assigned each user to their department group.

```powershell
# Create OUs
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"

# Create security groups
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

> **Run the user creation block all at once.** The `$password` variable must be defined before `New-ADUser` commands execute. Running line by line causes PowerShell to prompt for a Name parameter and the script fails.

```powershell
# Create users and assign group membership — run entire block together
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "bob.patel" -GivenName "Bob" -Surname "Patel" `
  -SamAccountName "bob.patel" -UserPrincipalName "bob.patel@lab.local" `
  -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "carol.jones" -GivenName "Carol" -Surname "Jones" `
  -SamAccountName "carol.jones" -UserPrincipalName "carol.jones@lab.local" `
  -Path "OU=HR,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "david.smith" -GivenName "David" -Surname "Smith" `
  -SamAccountName "david.smith" -UserPrincipalName "david.smith@lab.local" `
  -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $password -Enabled $true

Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

---

### Step 4 — Configure Group Policy

Created and linked a GPO named **IT Security Policy** to the IT OU enforcing four security controls.

| Policy Path | Setting | Value |
|---|---|---|
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Minimum password length | 12 |
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Password must meet complexity requirements | Enabled |
| Computer Config → Windows Settings → Security → Local Policies → Security Options | Interactive logon: Machine inactivity limit | 900 seconds |
| Computer Config → Administrative Templates → System → Removable Storage Access | All removable storage classes: Deny all access | Enabled |

GPO tested by joining a second VM to `lab.local`, moving its computer account into the IT OU, running `gpupdate /force`, and logging in as `alice.chen` to verify policy enforcement.

---

### Step 5 — Help Desk Operations

Practiced the four most common AD help desk tasks against the test accounts.

```powershell
# Reset password — force change on next login
Set-ADAccountPassword -Identity "bob.patel" -Reset `
  -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true

# Unlock a locked account
Unlock-ADAccount -Identity "carol.jones"

# Disable account on offboarding (preserves audit history — do not delete)
Disable-ADAccount -Identity "david.smith"
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName

# Audit — inactive accounts + group membership report
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, LastLogonDate

Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

---

## Verification

Run each command to confirm the environment is built correctly.

| Check | Command | Expected Result |
|---|---|---|
| Domain controller running | `Get-ADDomainController` | Returns DC info including forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs |
| Users enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Lists 4 test accounts |
| Group membership correct | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows IT Security Policy as linked |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PowerShell prompts for `Name:` when creating users | The `$password` variable was not defined before `New-ADUser` ran. Copy the full user creation block and run it all at once. |
| Cannot copy/paste into the VM | RDP client → Show Options → Local Resources → check Clipboard. Or download the RDP file from the Azure portal and open it with the native Remote Desktop app instead of the browser console. |
| Promotion fails — DNS conflict | Set the NIC's preferred DNS to `127.0.0.1` before promoting, or use the VM's static IP. |
| Cannot RDP after domain join | Log in as `LAB\Administrator` (domain admin), not just `Administrator`. |
| GPO not applying | Run `gpupdate /force` on the target machine, then `gpresult /r` to confirm applied policies. |
| User cannot log in after creation | Confirm the account is Enabled and `ChangePasswordAtLogon` is not blocking first login. |
| AD Users and Computers not showing | Run `dsa.msc` from the Run dialog, or run `Add-WindowsFeature RSAT-ADDS`. |

---

## Key Concepts

**Domain Controller** — The server that runs Active Directory. All authentication in the domain is processed here. Every account, group, and policy lives on the DC.

**Forest / Domain** — The forest is the top-level container for the entire AD structure. A domain lives inside the forest with a DNS-style name (`lab.local`). Most organisations have one domain per forest.

**Organisational Unit (OU)** — A folder inside AD used to organise users, computers, and groups by department or function. The primary mechanism for scoping Group Policy.

**Security Group** — A container for user accounts used to grant access to resources. Assign access to the group once; manage who has access by managing group membership. This is role-based access control in practice.

**Group Policy Object (GPO)** — A collection of settings enforced automatically across every user or computer in a linked OU. No agent required — Windows enforces the policy natively on login and at `gpupdate` intervals.

---

## Tools Used

| Tool | Purpose | Cost |
|---|---|---|
| Windows Server 2025 Datacenter | Domain Controller OS | Free — 180-day evaluation licence |
| Microsoft Azure | VM hosting | Free — $200 credit, B2s covered by free tier |
| Active Directory Domain Services | Identity and access management | Included with Windows Server |
| Group Policy Management Console | GPO creation and management | Included with Windows Server |
| PowerShell / ADUC | Administration and automation | Included with Windows Server |

---

