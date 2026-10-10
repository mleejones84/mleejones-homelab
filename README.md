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

## 🏛️ Active Directory Infrastructure
The core Identity and Access Management (IAM) boundary is defined by a single-forest, single-domain architecture.

*   **Forest Functional Level:** Windows Server 2022
*   **Domain Functional Level:** Windows Server 2022
*   **Fully Qualified Domain Name (FQDN):** `mleejones.lab`
*   **NetBIOS Name:** `MLEEJONES`

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

---

## 📋 Next Phases & Task Backlog
- [ ] **Phase 2.1:** Configure DHCP Server scopes and DNS forwarders on `10.0.10.10`.
- [ ] **Phase 2.2:** Establish standard Organizational Unit (OU) structures (`Tier0_Admins`, `Workstations`, `Users`).
- [ ] **Phase 2.3:** Join the Windows 11 client machine to the `mleejones.lab` domain environment.
- [ ] **Phase 2.4:** Build out Group Policy Objects (GPOs) for basic workstation hardening.