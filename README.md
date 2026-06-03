## 🔬 Active Directory Lab 1
*Windows Server 2025 · Azure · Identity & Access Management*

[![Certification](https://img.shields.io/badge/Cert%20Alignment-CompTIA%20Network%2B%20%7C%20Security%2B%20%7C%20Azure%20Administrator-blue)](https://www.comptia.org)
[![Cost](https://img.shields.io/badge/Estimated%20Cost-%240-brightgreen)](https://azure.microsoft.com/free)
[![Time](https://img.shields.io/badge/Time%20to%20Complete-3--5%20hours-orange)]()
[![Platform](https://img.shields.io/badge/Platform-Azure%20%7C%20VirtualBox-informational)]()

| Field | Value |
|---|---|
| Certification alignment | CompTIA Network+ · Security+ · Azure Administrator |
| Free tools | Windows Server 2025 Evaluation (180 days) · Azure Free Account |
| Time to complete | 3–5 hours across multiple sessions |
| Estimated cost | $0 — fully covered by free tiers and evaluation licenses |
| Career relevance | IT Support · Sysadmin · Cloud Engineer · Security Analyst |

---

## 🫆 The Business Problem This Lab Solves 

Every enterprise running Windows infrastructure depends on Active Directory (AD) as the foundation of its identity and access management (IAM) strategy — controlling authentication, authorization, and access governance across the entire environment.

Active Directory is the identity backbone of on-premises and hybrid cloud architectures. It enforces role-based access control (RBAC), manages privileged access, and applies security policies across users, endpoints, and resources — defining who authenticates to which systems, which security groups control access to file shares and applications, and which Group Policy Objects (GPOs) govern compliance configurations across the domain.

From a zero trust security perspective, AD is where identity verification begins. When a user is provisioned, group membership automatically grants least-privilege access to email, shared drives, and business-critical applications. When a user is offboarded, a single account disable revokes all access simultaneously — reducing the attack surface and eliminating orphaned credentials that threat actors exploit.

This is not legacy technology. Modern hybrid cloud environments synchronize on-premises Active Directory with Microsoft Entra ID (Azure AD) via Azure AD Connect, extending identity management into the cloud. The same core competencies — user lifecycle management, group policy administration, OU structure design, and privileged identity governance — underpin enterprise Azure, Microsoft 365, and cloud security deployments.

Hands-on Active Directory experience is directly applicable to roles in cloud infrastructure, IAM engineering, SOC analysis, and endpoint security — and remains the most targeted system in ransomware and Active Directory attacks such as Pass-the-Hash, Kerberoasting, and DCSync.

> 
| Role | How this lab applies |
|---|---|
| IT Support / Help Desk | Password resets, account unlocks, group membership changes — the top three ticket types in any enterprise |
| Sysadmin | Designing OU structure, deploying GPOs, managing domain-joined machines at scale |
| Cloud Engineer | Entra ID (cloud AD) uses the same concepts: users, groups, roles, conditional access. On-prem AD knowledge transfers directly |
| Security Analyst | AD is the most targeted system in ransomware attacks. Understanding how it works is the foundation of defending it |

---

## 🧱 What This Lab Builds

| Component | Details |
|---|---|
| **Domain** | `lab.local` — one forest, one domain |
| **Domain Controller** | Windows Server 2025 Datacenter — Standard_B2s (Azure) |
| **Organisational Units** | IT, Finance, HR, Sales, Computers |
| **Security Groups** | IT_Admins, Finance_Users, HR_Users, Sales_Users |
| **User Accounts** | alice.chen (IT), bob.patel (Finance), carol.jones (HR), david.smith (Sales) |
| **Group Policy Object** | IT Security Policy — password length, complexity, screen lock, USB block |
| **Help Desk Tasks** | Password reset, account unlock, offboarding, inactive account audit |

---

## What You Will Need

Before starting this lab you need a ** virtual machines** in Azure. 

| VM | Role | Purpose |
|---|---|---|
| adVM | Domain Controller | Runs Active Directory, DNS, and Group Policy |

> **Important:** Create the VM before starting the lab steps. 

---
## 📐 Architecture
<img width="1369" height="1149" alt="image" src="https://github.com/user-attachments/assets/50e9a534-16e2-4216-ae5c-00e47ec38dfb" />

---

## Step 1 — Provision the Virtual Machine  💻


Sign in to [portal.azure.com](https://portal.azure.com) → **Virtual machine** → **Create**.



| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter — Gen2 |
| Size | Standard_B2s (2 vCPU, 4 GB RAM) |
| Authentication | Password (set a strong one — used for RDP) |
| Public inbound ports | Allow RDP (3389) |
| OS disk | Standard SSD |

Click **Review + Create** → **Create** and wait for deployment to complete.

---

### Enable clipboard between your machine and the VM ###

1. Open the Remote Desktop application on your local machine.
2. Enter the VM's public IP address.
3. Click **Show Options** → **Local Resources** tab.
4. Ensure **Clipboard** is checked under *Local devices and resources*.
5. Click **Connect**.

> Download the RDP file from the Azure portal (**Connect → Download RDP File**) and open it with the native Remote Desktop app rather than the browser-based console. The browser console has very limited clipboard support.

---

### Step 2 — Install Active Directory Domain Services

RDP into adVM the VM using it's public IP address. Server Manager opens automatically on login.

> **What is a Domain Controller?**
> A Domain Controller (DC) is a server that runs Active Directory. It is the brain of the entire identity system. When a user logs in anywhere on the domain their credentials are checked against the Domain Controller. There is usually more than one in an enterprise for redundancy but we are building one here.

In Server Manager: **Manage → Add Roles and Features → Next → Server Roles → check Active Directory Domain Services → Add Features → Install**

Wait for installation to complete — takes 2–3 minutes. When complete click **Close** — do not restart yet.

Or run this in PowerShell on the DC:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```
<img width="1847" height="867" alt="adds installed" src="https://github.com/user-attachments/assets/8c65e9a2-2d00-4ee3-83bc-a71a33064ea9" />

```powershell
# PowerShell alternative
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

**Install Group Policy Management Console (GPMC) now** — Step 5 requires the Group Policy Management Console. Without it, the GPO tools will not appear in Server Manager. Install it now, so it's configured when you need it.


```powershell
Install-WindowsFeature -Name GPMC
```
<img width="1872" height="940" alt="pmc-installed" src="https://github.com/user-attachments/assets/7929a66e-ac9f-4bcc-865c-65ead0e045f0" />

> Once GPMC is installed **Group Policy Management** will appear in the Tools menu in Server Manager. This is completely separate from Active Directory Users and Computers — do not look for GPOs inside ADUC.

---

### Step 3 — Promote the Server to a Domain Controller

Installing the AD DS role does not create a domain. Promotion creates your forest, domain, and makes this server the authoritative DNS and identity server.
> **What is a Forest and Domain?**
> A Forest is the top-level container of your entire Active Directory structure. A Domain is a boundary inside the forest with a name — ours is `lab.local`. Most small-to-medium organizations have one domain inside one forest.
> 
1. In Server Manager, click the **yellow warning flag** (top right).
2. Click **Promote this server to a domain controller**.
3. Select **Add a new forest**.
4. Set Root domain name to: `lab.local`
5. Click **Next** — set a DSRM password write it down
6. Click through DNS Options and NetBIOS pages - accept the defaults
7. Click **Install** — the server restarts automatically when complete.

Or promote via Powershell:

```powershell
# PowerShell alternative
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName 'lab.local' `
  -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```

> After the restart RDP back into adVM. You are now logged into a Domain Controller.

---

### Step 4 — Build the Organisational Structure

Open **Active Directory Users and Computers (ADUC)** from **Tools** in Server Manager.
**It should look like this** ⤵

<img width="942" height="656" alt="adou screenshot" src="https://github.com/user-attachments/assets/fe296260-0557-48cc-a9f3-0e116c987214" />

### Create Organisational Units
> **What is an Organizational Unit (OU)?**
> An OU is a folder inside Active Directory. You use OUs to organize users and computers by department. The real power is that you can link a Group Policy to an OU — every user or computer inside automatically gets the policy applied.

Right-click your domain (`lab.local`) → **New → Organizational Unit**. Create one OU per department plus one for computers.



```powershell
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
```

### Create Security Groups

Right-click each OU → **New → Group**. Set Group scope to **Global**, Group type to **Security**.

```powershell
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

### Create User Accounts and Group Memberships

> ⚠️ **Run the entire block at once** — not line by line. The `$password` variable must be defined before the `New-ADUser` commands execute. Select all, then paste the full block into PowerShell and press Enter.

```powershell
# Run this entire block together

# Step 1 — define the password variable first
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

# Step 2 — create users
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

# Step 3 — assign group memberships
Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

---

## Step 5 — Configure Group Policy

Open **Group Policy Management** from the Tools menu in Server Manager on the **DC**.

> **What is a Group Policy Object (GPO)?**
> A GPO is a collection of settings applied automatically to every user or computer inside an OU. Password complexity requirements, screen lock timers, USB restrictions — all enforced from a single GPO across every machine without touching each one individually.

1. Expand **Forest: lab.local → Domains → lab.local**.
2. Right-click the **IT** OU → **Create a GPO in this domain and link it here**.
3. Name it: `IT Security Policy`
4. Right-click the new GPO → **Edit**.
5. Apply the following settings:

| Policy Path | Setting | Value |
|---|---|---|
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Minimum password length | 12 |
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Password must meet complexity requirements | Enabled |
| Computer Config → Windows Settings → Security → Local Policies → Security Options | Interactive logon: Machine inactivity limit | 900 seconds |
| Computer Config → Administrative Templates → System → Removable Storage Access | All removable storage classes: Deny all access | Enabled |

<img width="932" height="657" alt="gpo open" src="https://github.com/user-attachments/assets/e4047487-9a53-43d0-a41e-c511efc8acc6" />
<img width="982" height="722" alt="gpo password enforce" src="https://github.com/user-attachments/assets/b12f23a6-2189-4723-ab25-a86a16062552" />
<img width="976" height="715" alt="gpo lock" src="https://github.com/user-attachments/assets/e5c8c220-413c-4cef-9cfa-b275c05a2c04" />
<img width="1297" height="716" alt="gpo deny storage device" src="https://github.com/user-attachments/assets/e40e0715-3922-4e4e-9d08-cee516d0b41e" />

---

### Step 6 — Common Help Desk Tasks

Run these PowerShell commands on **DC**.

**Reset a password**
```powershell
Set-ADAccountPassword -Identity "bob.patel" -Reset `
  -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true
```

**Unlock a locked account**
```powershell
Unlock-ADAccount -Identity "carol.jones"
```

**Disable an account**
```powershell
# Disable — preserves account history and group memberships for audit
Disable-ADAccount -Identity "david.smith"

# Find all currently disabled accounts
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
```

**Audit inactive accounts**
```powershell
# Accounts not logged in for 90+ days
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, LastLogonDate

# Check a specific user's group memberships
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

---

## Verification Checklist

Run these commands after completing the lab to confirm everything is working correctly.

| Check | Command | Expected Result |
|---|---|---|
| Domain controller is running | `Get-ADDomainController` | Returns DC info including forest `lab.local` |
| All OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs |
| Users exist and are enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Returns 4 test accounts |
| Group membership is correct | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO is linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows `IT Security Policy` as linked |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PowerShell prompts for `Name:` when creating users | You ran `New-ADUser` before defining `$password`. Run the **entire block from Step 4** at once — the variable must be defined first. |
| Cannot copy and paste into the VM | Open the RDP client → **Show Options → Local Resources** → check **Clipboard**. Reconnect. Alternatively, download the RDP file from the Azure portal and open it with the native Remote Desktop app. |
| Promotion fails — DNS conflict | Set the NIC's preferred DNS to `127.0.0.1` before promoting, or use the VM's static IP. |
| Cannot RDP after domain join | Log in as `LAB\Administrator`, not just `Administrator`. |
| GPO not applying | Run `gpupdate /force` on the target machine. Then run `gpresult /r` to confirm which policies are applied. |
| User cannot log in after creation | Confirm the account is **Enabled** and `ChangePasswordAtLogon` is not set if you want the first login to work without a prompt. |
| AD Users and Computers not showing | Run `dsa.msc` from the Run dialog. Or install RSAT: `Add-WindowsFeature RSAT-ADDS` |

---

## Key Concepts

<details>
<summary><strong>What is a Domain Controller?</strong></summary>

A Domain Controller (DC) is the brain of the entire identity system. It runs Active Directory, handles authentication for every domain-joined machine, and is the single source of truth for all user accounts, group memberships, and policies. When a user logs in anywhere on the domain, their credentials are verified against the DC.

</details>

<details>
<summary><strong>What is a Forest and Domain?</strong></summary>

A **Forest** is the top-level container of your entire Active Directory structure — think of it as the organisation itself. A **Domain** is a boundary inside the forest with a DNS-style name (`lab.local`). Everything inside a domain is managed together. Most SMBs have one domain in one forest. Large enterprises may have multiple.

</details>

<details>
<summary><strong>What is an Organisational Unit (OU)?</strong></summary>

An OU is a folder inside Active Directory. You use OUs to organise users, computers, and groups by department or function. The real power of an OU is that you can link a Group Policy to it — every user or computer inside that OU automatically receives that policy.

</details>

<details>
<summary><strong>What is a Security Group?</strong></summary>

A Security Group is a container of user accounts. Instead of granting access to a resource (file share, application, etc.) to individual users one at a time, you grant access to the group. This is role-based access control: add a user to the group and they instantly inherit all group access. Remove them and all access is revoked simultaneously.

</details>

<details>
<summary><strong>What is a Group Policy Object (GPO)?</strong></summary>

A GPO is a collection of settings that Windows enforces automatically on every user or computer inside an OU. You create one GPO, link it to an OU, and every machine in that OU gets those rules on next login or after `gpupdate /force`. Password policies, screen lock timers, USB restrictions — all enforced centrally without touching individual machines.

</details>

---

## Why This Matters for Cloud Roles

Active Directory is not legacy technology. Hybrid environments sync on-premises AD identities to **Microsoft Entra ID** (formerly Azure AD). The core concepts — users, groups, roles, conditional access — are identical. Every enterprise cloud environment you'll work in assumes this knowledge.

Active Directory is also the **most targeted system in ransomware attacks**. Understanding how it is built is the foundation of understanding how it is attacked and defended.

---

## Resources

- [Microsoft Docs — Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [Microsoft Docs — Group Policy Overview](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top)
- [Azure Free Account](https://azure.microsoft.com/free)
- [Windows Server 2025 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)

---

> **Portfolio tip:** Document everything you build. Screenshot each completed step. When an interviewer asks about Active Directory experience — this lab with screenshots is your proof.
