# Project 1: On-Premises Active Directory Foundation

**Azure Cloud Lab Series — Part 1 of 4**

This project provisions a fully functional on-premises Active Directory environment hosted inside Microsoft Azure. All configuration is performed through the Azure Portal and Windows Server GUI — no CLI or scripting is used.

---

## Environment Summary

| Component | Value |
|---|---|
| Platform | Microsoft Azure |
| Domain Controller OS | Windows Server 2025 Datacenter: Azure Edition |
| Client OS | Windows 10 Enterprise 22H2 (Gen2) |
| Domain Name | corp.local |
| DC Static IP | 10.0.1.5 |
| Virtual Network | VNet-OnPrem |
| Subnet | Subnet-DC (10.0.1.0/24) |
| Resource Group | RG-Project1-OnPrem |
| Subscription | Subscription 1 |

---

## Architecture

```
Azure Subscription
└── Resource Group: RG-Project1-OnPrem
    ├── Virtual Network: VNet-OnPrem (10.0.0.0/16)
    │   └── Subnet: Subnet-DC (10.0.1.0/24)
    │
    ├── VM: DC1 (Windows Server 2025)
    │   ├── Private IP: 10.0.1.5 (Static)
    │   ├── Role: Domain Controller
    │   ├── Role: AD DS
    │   ├── Role: DNS Server
    │   └── Domain: corp.local
    │
    └── VM: CLIENT1 (Windows 10 Enterprise 22H2)
        ├── DNS: 10.0.1.5 (points to DC1)
        └── Domain Member: corp.local
```

**Active Directory OU Structure:**

```
corp.local
├── _CORP_USERS          - Standard domain user accounts
├── _CORP_COMPUTERS      - Workstations and member servers
├── _CORP_ADMINS         - Administrative accounts
├── _CORP_GROUPS         - Security and distribution groups
└── _SERVICE_ACCOUNTS    - Service and automation accounts
```

---

## Prerequisites

- Active Microsoft Azure subscription
- Access to the Azure Portal
- Basic understanding of networking concepts (IP addressing, DNS, subnets)

---

## Lab Steps

### Step 1 — Create the Resource Group

1. In the Azure Portal, search for **Resource groups** and select it
2. Click **+ Create**
3. Name: `RG-Project1-OnPrem`
4. Select your subscription and region
5. Click **Review + Create** → **Create**

![Step 1](Screenshot%201.png)
*Figure 1 — Resource group RG-Project1-OnPrem successfully created*

---

### Step 2 — Deploy the Virtual Network

1. Search **Virtual networks** → **+ Create**
2. Assign to `RG-Project1-OnPrem`, name: `VNet-OnPrem`
3. Under **IP Addresses**, set address space to `10.0.1.0/24`
4. Create a subnet named `Subnet-DC`
5. Click **Review + Create** → **Create**

![Step 2](Screenshot%202.png)
*Figure 2 — VNet-OnPrem deployment completed successfully*

---

### Step 3 — Deploy the Domain Controller VM (DC1)

1. Search **Virtual machines** → **+ Create** → **Azure virtual machine**
2. Resource group: `RG-Project1-OnPrem` | VM name: `DC1`
3. Image: **Windows Server 2025 Datacenter: Azure Edition — x64 Gen2**
4. Size: `Standard_B2s` or equivalent
5. Set a secure administrator username and password
6. Networking: VNet `VNet-OnPrem`, Subnet `Subnet-DC`, Public IP enabled, NSG allowing RDP (port 3389)
7. Click **Review + Create** → **Create**

![Step 3](Screenshot%203.png)
*Figure 3 — DC1 VM deployment completed — VM, NIC, NSG, and Public IP all provisioned successfully*

---

### Step 4 — Assign a Static Private IP to DC1

> **Note:** Azure reserves `.1–.4` within every subnet. DC1 will receive `10.0.1.5`, not `.4`. Update all references accordingly.

1. Navigate to **DC1** → **Networking** → click the NIC name (`dc1999`)
2. Select **IP configurations** from the left panel
3. Click **ipconfig1** → change from **Dynamic** to **Static**
4. Enter `10.0.1.5` → click **Save**
5. Update the VNet **Custom DNS** setting to `10.0.1.5`

![Step 4](Screenshot%204.png)
*Figure 4 — DC1 NIC showing static private IP 10.0.1.5 confirmed*

---

### Step 5 — RDP into DC1 & Open Server Manager

1. Navigate to **DC1** → **Connect** → **RDP**
2. Download the `.rdp` file and open it
3. Log in with the administrator credentials set during VM creation
4. Server Manager launches automatically — confirm 0 roles installed

![Step 5](Screenshot%205.png)
*Figure 5 — Server Manager Dashboard on DC1 — clean baseline, no roles installed*

---

### Step 6 — Install the AD DS Role

1. In Server Manager → **Manage** → **Add Roles and Features**
2. Select **Role-based installation**, destination server: **DC1**
3. Tick **Active Directory Domain Services**
4. Accept the additional features prompt → click through to **Install**
5. Confirm the red alert badge appears — promotion is pending

![Step 6](Screenshot%206.png)
*Figure 6 — AD DS role installed — alert badge indicates promotion required*

---

### Step 7 — Promote DC1 to Domain Controller

1. Click the yellow warning flag → **Promote this server to a domain controller**
2. Select **Add a new forest** | Root domain name: `corp.local`
3. Forest/Domain functional level: **Windows Server 2016+**
4. Ensure **DNS server** is checked
5. Set a **DSRM password**
6. Click **Next** through all screens → **Install**
7. DC1 reboots — after reboot confirm AD DS and DNS tiles show green

![Step 7](Screenshot%207.png)
*Figure 7 — Server Manager post-promotion — AD DS, DNS, and File and Storage Services all active*

---

### Step 8 — Verify DNS Forward Lookup Zone

1. Server Manager → **Tools** → **DNS**
2. Expand **DC1** → **Forward Lookup Zones** → **corp.local**
3. Confirm the following records exist:
   - SOA → `dc1.corp.local`
   - NS → `dc1.corp.local`
   - Host (A) → zone apex → `10.0.1.5`
   - Host (A) → `dc1` → `10.0.1.5` (static)

![Step 8](Screenshot%208.png)
*Figure 8 — DNS Manager — corp.local Forward Lookup Zone with all required records*

---

### Step 9 — Create Organisational Units (OUs)

1. Server Manager → **Tools** → **Active Directory Users and Computers**
2. Right-click **corp.local** → **New** → **Organizational Unit**
3. Create the following OUs:

| OU Name | Purpose |
|---|---|
| `_CORP_USERS` | Standard domain user accounts |
| `_CORP_COMPUTERS` | Workstations and member servers |
| `_CORP_ADMINS` | Administrative accounts |
| `_CORP_GROUPS` | Security and distribution groups |
| `_SERVICE_ACCOUNTS` | Service and automation accounts |

![Step 9](Screenshot%209.png)
*Figure 9 — Five custom OUs created under corp.local*

---

### Step 10 — Create Domain User Accounts

1. Right-click **_CORP_USERS** → **New** → **User**
2. Create the following test users:

| Full Name | Username |
|---|---|
| John Smith | jsmith |
| Jane Doe | jdoe |

3. Assign a strong password to each
4. Uncheck *User must change password at next logon*
5. Click **Finish**

![Step 10](Screenshot%2010.png)
*Figure 10 — John Smith and Jane Doe created in _CORP_USERS OU*

---

### Step 11 — Deploy CLIENT1 Windows 10 VM

1. Azure Portal → **Virtual machines** → **+ Create**
2. Resource group: `RG-Project1-OnPrem` | VM name: `CLIENT1`
3. Image: **Windows 10 Enterprise, version 22H2 — x64 Gen2**
4. Tick the eligible Windows 10/11 licence checkbox
5. Networking: VNet `VNet-OnPrem`, Subnet `Subnet-DC`
6. Click **Review + Create** → **Create**

> **Azure AD Auto-Join:** CLIENT1 may auto-join Entra ID during provisioning. Disconnect before joining the on-premises domain: Settings → Accounts → Access work or school → Disconnect.

![Step 11](Screenshot%2011.png)
*Figure 11 — CLIENT1 Windows 10 VM deployment completed successfully*

---

### Step 12 — Join CLIENT1 to corp.local

1. RDP into **CLIENT1** using the local administrator account
2. Right-click **This PC** → **Properties** → **Rename this PC (advanced)** → **Change...**
3. Under **Member of**, select **Domain** and enter `corp.local`
4. Click **OK** and enter domain admin credentials when prompted
5. Confirm the *Welcome to the corp.local domain* dialog
6. Restart CLIENT1

![Step 12](Screenshot%2012.png)
*Figure 12 — Welcome to the corp.local domain confirmation on CLIENT1*

---

### Step 13 — Verify Domain Login

1. On CLIENT1 login screen, select **Other user**
2. Log in as `corp\jsmith` using John Smith's domain password
3. Windows builds the domain profile — login confirmed

![Step 13](Screenshot%2013.png)
*Figure 13 — John Smith domain login successful on CLIENT1*

---

### Step 14 — Verify Domain Membership via whoami

1. Open **Command Prompt** on CLIENT1
2. Run:
   ```
   whoami
   ```
3. Expected output: `corp\jsmith`

![Step 14](Screenshot%2014.png)
*Figure 14 — whoami confirms corp\jsmith — domain-authenticated session verified*

---

### Step 15 — Verify Domain Membership via systeminfo

1. In Command Prompt, run:
   ```
   systeminfo | findstr /B /C:"Domain"
   ```
2. Expected output: `Domain:    corp.local`

![Step 15](Screenshot%2015.png)
*Figure 15 — systeminfo confirms Domain: corp.local*

---

### Step 16 — Verify CLIENT1 in AD Computers OU

1. RDP back into **DC1**
2. Open **Active Directory Users and Computers**
3. Navigate to **corp.local** → **_CORP_COMPUTERS**
4. Confirm **CLIENT1** appears with type **Computer**

![Step 16](Screenshot%2016.png)
*Figure 16 — CLIENT1 registered in _CORP_COMPUTERS OU — domain join fully verified*

---

## Cost Management

Always stop (deallocate) VMs when not in use — a deallocated VM incurs no compute charges.

| Action | Steps |
|---|---|
| Stop DC1 | Azure Portal → Virtual machines → DC1 → Stop |
| Stop CLIENT1 | Azure Portal → Virtual machines → CLIENT1 → Stop |
| Delete everything | Resource groups → RG-Project1-OnPrem → Delete resource group |

> Do not delete the resource group until Projects 2–4 are complete — subsequent labs build on this environment.

---

## Key Learnings

**Azure IP Reservation**
Azure reserves the first four addresses in every subnet. The first usable address is `.5`, not `.4`. Always verify the assigned IP in the portal.

**Entra ID Auto-Join**
Newly provisioned Azure VMs may automatically join Entra ID. Disconnect before attempting an on-premises domain join via Settings → Accounts → Access work or school.

**DNS is Critical for Domain Join**
The client VM's DNS must point to DC1's IP (`10.0.1.5`). Set this in the VNet's Custom DNS settings so it applies to all VMs on that network.

**Static IPs for Domain Controllers**
Domain controllers must always have a static private IP. A dynamic IP change breaks DNS resolution across the entire domain.

---

## Project Outcomes

| # | Outcome | Status |
|---|---|---|
| 1 | Resource group and VNet created in Azure Portal | Complete |
| 2 | DC1 deployed with static IP 10.0.1.5 | Complete |
| 3 | AD DS role installed via Server Manager | Complete |
| 4 | DC1 promoted to domain controller for corp.local | Complete |
| 5 | DNS forward lookup zone verified | Complete |
| 6 | Five custom OUs created | Complete |
| 7 | Domain users jsmith and jdoe created | Complete |
| 8 | CLIENT1 Windows 10 VM deployed on same VNet | Complete |
| 9 | CLIENT1 joined to corp.local domain | Complete |
| 10 | Domain login verified via login screen | Complete |
| 11 | Domain membership confirmed via whoami and systeminfo | Complete |
| 12 | CLIENT1 registered in _CORP_COMPUTERS OU | Complete |

---

## Lab Series Progress

| Project | Title | Status |
|---|---|---|
| Project 1 | On-Premises Active Directory Foundation | Complete |
| Project 2 | Hybrid Identity Integration | Next |
| Project 3 | SSO Implementation | Upcoming |
| Project 4 | Security Hardening | Upcoming |

---

*Prepared by Omokorede Oludairo*
