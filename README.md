# Systems Engineering & Active Directory Home Lab Guide

## 📌 Architectural Overview
This master documentation outlines the deployment, network segmentation, and configuration of an isolated Active Directory (AD) Enterprise Simulation Environment. The lab serves as a sandbox for enterprise administration, security auditing, and systems engineering validation.

---

## 🌐 Network Topology
The environment is strictly segmented into an isolated subnet to mimic an enterprise enclave while preventing conflict with production consumer routing.

*   **Subnet Gateway:** `10.0.10.1`
*   **Subnet Mask:** `255.255.255.0` (`/24`)
*   **Primary Domain Controller & DNS:** `10.0.10.10`
*   **DHCP Scope range:** `10.0.10.50` to `10.0.10.254`

### Network Interface Configuration (Static Example)
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.10.10 -PrefixLength 24 -DefaultGateway 10.0.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```
---
## 💻 Workstation Deployment (Windows 11 Client)
To provision Windows 11 client machines within a legacy or resource-constrained hypervisor environment without hardware restrictions preventing deployment, the registry must be modified during the Out-Of-Box Experience (OOBE).

### Hypervisor Deployment & TPM/RAM Check Bypass
1. Boot the Windows 11 installation media.
2. At the initial setup screen, invoke the command prompt using: `Shift + F10`
3. Launch the Registry Editor by typing: `regedit`
4. Navigate to: `HKEY_LOCAL_MACHINE\SYSTEM\Setup`
5. Create a new Key named: `LabConfig`
6. Inside `LabConfig`, provision the following `DWORD (32-bit)` values and set their data to `1`:
    *   `BypassTPMCheck` = `1`
    *   `BypassSecureBootCheck` = `1`
    *   `BypassRAMCheck` = `1`
7. Close the registry editor, type `exit` in the console, and proceed with the standard installation.

## Active Directory Infrastructure
The core Identity and Access Management (IAM) boundary is defined by a single-forest, single-domain architecture.

* Forest Functional Level: Windows Server 2022
* Domain Functional Level: Windows Server 2022
* Fully Qualified Domain Name (FQDN): mleejones.lab
* NetBIOS Name: MLEEJONES

### Automated Domain Deployment Script (PowerShell)
```powershell
# Install Active Directory Domain Services Role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Provision New Active Directory Forest
Install-ADDSForest `
    -CreateDnsDelegation:$false `
    -DatabasePath "C:\Windows\NTDS" `
    -DomainMode "WinThreshold" `
    -DomainName "mleejones.lab" `
    -DomainNetbiosName "MLEEJONES" `
    -ForestMode "WinThreshold" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -NoRebootOnCompletion:$false `
    -Force:\$true
```

---
## Phase 2.1: DHCP Deployment
Run the deployment script on the DC to provision the client lease pool:
* Scope range: 10.0.10.50 to 10.0.10.254
* Subnet Mask: 255.255.255.0 (/24)
* DNS Server: 10.0.10.10
* Gateway: 10.0.10.1

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName dc.mleejones.lab -IPAddress 10.0.10.10
Add-DhcpServerv4Scope -Name "Lab_Client_Scope" -StartRange 10.0.10.50 -EndRange 10.0.10.254 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 10.0.10.0 -DnsServer 10.0.10.10 -Router 10.0.10.1
```

---
## Phase 2.2: Organizational Unit (OU) Architecture
Established 'MLeeJones_Enterprise' container with segmented sub-OUs to separate privileges and prepare for targeted Group Policy Objects (GPOs):
* Admin_Accounts
* Standard_Users
* Workstations
* Servers

```powershell
\$DomainDN = "DC=mleejones,DC=lab"
New-ADOrganizationalUnit -Name "MLeeJones_Enterprise" -Path \$DomainDN
\(ParentOU = "OU=MLeeJones_Enterprise,\)DomainDN"
New-ADOrganizationalUnit -Name "Admin_Accounts" -Path \$ParentOU
New-ADOrganizationalUnit -Name "Standard_Users" -Path \$ParentOU
New-ADOrganizationalUnit -Name "Workstations" -Path \$ParentOU
New-ADOrganizationalUnit -Name "Servers" -Path \$ParentOU
```

---
## Phase 2.3: Workstation Domain Join
Successfully joined the Windows 11 client endpoint to the domain via the command line, targeting the specific computer container:
* Target OU: OU=Workstations,OU=MLeeJones_Enterprise,DC=mleejones,DC=lab

```powershell
Add-Computer -DomainName "mleejones.lab" -OUPath "OU=Workstations,OU=MLeeJones_Enterprise,DC=mleejones,DC=lab" -Restart
```

---
## Phase 2.4: Group Policy Object (GPO) Hardening
Baked baseline security controls directly into the OU structure prior to client deployment:
* **Account Lockout Policy:** Threshold set to 3 failures / 15-minute duration window domain-wide.
* **Sec_Workstation_Hardening GPO:** Enforced an interactive corporate legal notice banner across all endpoints in the Workstations OU.

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "mleejones.lab" -LockoutThreshold 3 -LockoutDuration 00:15:00 -LockoutObservationWindow 00:15:00

New-GPO -Name "Sec_Workstation_Hardening" -Comment "Security baselines for enterprise workstations"
\$RegistryPath = "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System"
Set-GPRegistryValue -Name "Sec_Workstation_Hardening" -Key \$RegistryPath -ValueName "LegalNoticeCaption" -Type String -Value "MLeeJones Enterprise Security Notice"
Set-GPRegistryValue -Name "Sec_Workstation_Hardening" -Key \$RegistryPath -ValueName "LegalNoticeText" -Type String -Value "Unauthorized access to this system is strictly prohibited. All activities are monitored and logged."
New-Gplink -Name "Sec_Workstation_Hardening" -Target "OU=Workstations,OU=MLeeJones_Enterprise,DC=mleejones,DC=lab" -LinkEnabled Yes
```

---
## Phase 2.5: Automated Identity Provisioning
Populated the directory with standard enterprise user identities using direct administrative commands to establish clear account attributes:
* Target OU: OU=Standard_Users
* Accounts created: asmith (Alice Smith), bjones (Bob Jones), cbrown (Charlie Brown)

```powershell
\$TargetOU = "OU=Standard_Users,OU=MLeeJones_Enterprise,DC=mleejones,DC=lab"
\$Password = ConvertTo-SecureString "P@ssword123!" -AsPlainText -Force

New-ADUser -Name "Alice Smith" -SamAccountName "asmith" -UserPrincipalName "asmith@mleejones.lab" -Path TargetOU -AccountPassword Password -ChangePasswordAtLogon false -Enabled true -Department "Engineering" -Title "Systems Engineer"
New-ADUser -Name "Bob Jones" -SamAccountName "bjones" -UserPrincipalName "bjones@mleejones.lab" -Path TargetOU -AccountPassword Password -ChangePasswordAtLogon false -Enabled true -Department "Operations" -Title "Operations Analyst"
New-ADUser -Name "Charlie Brown" -SamAccountName "cbrown" -UserPrincipalName "cbrown@mleejones.lab" -Path TargetOU -AccountPassword Password -ChangePasswordAtLogon false -Enabled true -Department "Security" -Title "SOC Analyst"
```

---
## Phase 2.6: Hypervisor Integration & Privilege Validation
Resolved Hyper-V Enhanced Session Mode RDP restrictions by executing localized security principal injections via the legacy CLI subsystem on the client machine to handle 64-bit redirection constraints. 

```cmd
net localgroup "Remote Desktop Users" "mleejones\asmith" /add
net localgroup "Remote Desktop Users" "mleejones\bjones" /add
net localgroup "Remote Desktop Users" "mleejones\cbrown" /add
```

Validated standard user boundaries utilizing deterministic UAC gating and directory permission isolation to confirm strict privilege separation.
---
## Phase 2.7: Fine-Grained Password Policies (FGPP) & Privilege Isolation

To enforce the Principle of Least Privilege (PoLP) and establish strict identity boundaries, a domain-wide "one-size-fits-all" security posture was rejected. High-privilege administrative accounts require a significantly more rigorous security envelope than standard business identities. 

A custom Password Settings Object (PSO) was programmatically provisioned and linked directly to the `Domain Admins` security principal to override default domain constraints without modifying the underlying forest schema.

### 🛠️ Programmatic PSO Deployment & Group Mapping

The policy enforces a strict 16-character minimum passpharase length and a compressed 60-day maximum lifetime window, backed by an aggressive 3-strike account lockout gating mechanism.

```powershell
# 1. Create the Fine-Grained Password Settings Object (PSO)
New-ADFineGrainedPasswordPolicy -Name "PSO_Strict_Admin_Policy" `
    -Precedence 10 `
    -ComplexityEnabled $true `
    -MinPasswordLength 16 `
    -MaxPasswordAge (New-TimeSpan -Days 60) `
    -LockoutThreshold 3 `
    -LockoutDuration (New-TimeSpan -Minutes 30) `
    -LockoutObservationWindow (New-TimeSpan -Minutes 30)

# 2. Assign the PSO to the Administrative Security Group
Add-ADFineGrainedPasswordPolicySubject -Identity "PSO_Strict_Admin_Policy" -Subjects "Domain Admins"
```

### 🔍 Deterministic Verification & Validation Loop

Active Directory utilizes a "silent baseline" logic when evaluating resultant password policies—returning null output if an identity is merely inheriting the base domain rules. To validate the inheritance override, a multi-stage testing pipeline was executed using a temporary privilege promotion loop.

```powershell
# Stage 1: Initial out-of-group evaluation (Expected: Null/No Output)
Get-ADUserResultantPasswordPolicy -Identity "asmith"

# Stage 2: Temporal Security Principal Escalation
Add-ADGroupMember -Identity "Domain Admins" -Members "asmith"

# Stage 3: Resultant Policy Evaluation (Expected: Explicit PSO Matrix Output)
Get-ADUserResultantPasswordPolicy -Identity "asmith"

# Stage 4: Strict De-escalation & Environment Cleanup (Enforcing Least Privilege)
Remove-ADGroupMember -Identity "Domain Admins" -Members "asmith" -Confirm:$false
```

### 🎯 Resultant Operational Metrics
- **Standard User Boundary:** Successfully validated that standard user identities default seamlessly to the base 12-character / 90-day structural policy.
- **Administrative Isolation:** Confirmed that any security principal injected into the administrative container automatically intercepts the stricter 16-character constraint with zero administrative GUI dependency.
- **Privilege Integrity:** Confirmed that executing the `Remove-ADGroupMember` pipeline instantly strips the resultant policy object and forces the target identity back to standard compliance limits.
---
## Phase 2.8: Group Policy Objects (GPO) Architecture & Inheritance Diagnostics

To enforce configuration standards across the enterprise simulation boundary, a modular Group Policy Object (GPO) infrastructure was deployed. Rather than utilizing clunky, non-scalable local modifications, structural system parameters were bound natively to the Organizational Unit (OU) typography using the industry-standard Functional-Scope Tiering methodology.

### 🏛️ Functional GPO Distribution Topology

- **User-Tier Object (`User_Desktop_Restrictions`):** Linked to the `Standard_Users` OU path. Enforces identity-level security profiles across the `HKEY_CURRENT_USER` (HKCU) registry hive, mandating a strict 15-minute (900 seconds) workstation idle screen-saver lockout.
- **Machine-Tier Object (`Sec_Workstation_Hardening`):** Linked to the `Workstations` OU path. Binds configuration compliance directly to the physical/virtual computer layer (`HKEY_LOCAL_MACHINE`), disabling local guest databases and restricting automated system printing driver allocations to local administrators.

### 🛠️ Scripted GPO Object Provisioning & Dynamic Path Alignment

To eliminate human entry errors and bypass replication timing race-conditions (HRESULT: `0x80005000` / `ADS_BAD_PATHNAME`), target container strings were harvested dynamically from the active directory database layer to forge explicit link paths.

```powershell
# 1. Provision User-Tier Compliance Object with Replication Sleep Buffer
New-GPO -Name "User_Desktop_Restrictions" -Comment "Enforces user workspace compliance and idle lockouts"
Start-Sleep -Seconds 2

# 2. Inject Workspace Security Controls into HKCU Hive
Set-GPRegistryValue -Name "User_Desktop_Restrictions" -Key "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop" -ValueName "ScreenSaveActive" -Type String -Value "1"
Set-GPRegistryValue -Name "User_Desktop_Restrictions" -Key "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop" -ValueName "ScreenSaverIsSecure" -Type String -Value "1"
Set-GPRegistryValue -Name "User_Desktop_Restrictions" -Key "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop" -ValueName "ScreenSaveTimeOut" -Type String -Value "900"

# 3. Dynamic Path Resolution & Interface Link Execution
$TrueUserOUPath = (Get-ADOrganizationalUnit -Filter "Name -eq 'Standard_Users'").DistinguishedName
New-GPLink -Name "User_Desktop_Restrictions" -Target $TrueUserOUPath -LinkEnabled Yes
```

### 🕵️‍♂️ Advanced Troubleshooting Case Study: Sabotage & Telemetry Diagnostics

To simulate a tier-2 service desk incident and analyze active directory failure states, a deliberate, two-stage infrastructure sabotage loop was engineered completely through the command-line interface.

#### Scenario 1: Upstream Inheritance Interception
An administrative injection was executed to block upstream inheritance on the target machine container using low-level database modifiers (`gpOptions = 1`). 

```powershell
# Execute parent container inheritance blockade via DC CLI
Set-ADObject -Identity (Get-ADOrganizationalUnit -Filter "Name -eq 'Workstations'").DistinguishedName -Replace @{gpOptions = 1}
```

- **Diagnostic Telemetry Evaluation:** Client-side diagnostics (`gpresult /scope computer /r`) exposed that the `Default Domain Policy` was successfully intercepted and moved to the filtered register (`Filtering: Not Applied (Unknown Reason)`).
- **Architectural Discovery:** The `Sec_Workstation_Hardening` object bypasses the blockade entirely and remains actively applied. This proves that container-level inheritance blocks cannot override policies linked *directly* to the endpoint's localized container.

#### Scenario 2: Explicit Structural Link Severance
To induce a true baseline outage, the direct link state was programmatically disabled at the directory tier.

```powershell
# Sever the direct linkage path via DC CLI
Set-GPLink -Name "Sec_Workstation_Hardening" -Target (Get-ADOrganizationalUnit -Filter "Name -eq 'Workstations'").DistinguishedName -LinkEnabled No
```

- **Diagnostic Telemetry Evaluation:** Forcing a policy refresh (`gpupdate /force`) under an authentic `runas` administrative security token successfully captured the target failure state, exposing the explicit diagnostic root cause:
  ```text
  The following GPOs were not applied because they were filtered out:
      Sec_Workstation_Hardening
          Filtering:  Disabled (Link)
  ```

### 🎯 Operational Verification Result
Environment successfully restored to a healthy production baseline. The testing cycle validates full technical competency in client-side telemetry analysis, multi-layered UAC privilege execution, and the underlying mathematical processing logic of Active Directory Group Policy inheritance.