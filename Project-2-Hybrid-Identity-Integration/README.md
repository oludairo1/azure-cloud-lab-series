# Project 2 — Hybrid Identity Integration

**Azure Lab Series · Omokorede Oludairo**

> Connecting an on-premises Active Directory forest to Microsoft Entra ID using Entra Connect Sync, Password Hash Synchronisation, and Seamless SSO — fully documented with live screenshots.

---

## Environment Summary

| Field | Detail |
|---|---|
| **Custom Domain** | cloudidlab.net (registered via GoDaddy) |
| **On-Prem Forest** | corp.local — Windows Server 2025 Datacenter: Azure Edition |
| **Sync Tool** | Microsoft Entra Connect Sync (downloaded from Entra Admin Center) |
| **Auth Method** | Password Hash Synchronisation (PHS) + Seamless SSO |
| **Entra Licence** | Entra ID P2 — Connect Health monitoring enabled |
| **Tenant** | Mocktest909.onmicrosoft.com |

---

## Architecture Overview

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│   ON-PREMISES (Azure VM)        │         │   MICROSOFT ENTRA ID (Cloud)    │
│                                 │         │                                 │
│  DC1 — Windows Server 2025      │  ─────► │  Tenant: Mocktest909            │
│  corp.local AD Forest           │  sync   │  Custom Domain: cloudidlab.net  │
│  Users: jsmith, jdoe, itadmin   │         │  3 synced users confirmed       │
│                                 │         │  PHS + Seamless SSO enabled     │
│  Entra Connect Sync (MSOL_)     │         │  Connect Health: 0 errors       │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

| Component | Technology | Notes |
|---|---|---|
| Domain Controller | Windows Server 2025 (Azure Edition) | DC1 — corp.local forest |
| Sync Engine | Microsoft Entra Connect Sync | Installed on DC1 |
| Cloud Directory | Microsoft Entra ID (P2) | Tenant: Mocktest909.onmicrosoft.com |
| Custom Domain | cloudidlab.net (GoDaddy) | Verified & set as primary |
| Auth Method | Password Hash Sync + Seamless SSO | Enabled |
| Health Monitor | Entra Connect Health (P2) | Zero sync errors confirmed |

> ⚠️ **Real-World Note:** The Microsoft Download Centre no longer hosts Entra Connect Sync. As of 2026 it must be downloaded from: **Entra Admin Center → Microsoft Entra Connect → Get started → Download Connect Sync Agent.**

> ⚠️ **Real-World Note:** Windows Server 2025 Datacenter: Azure Edition was used as 2022 was unavailable at provisioning time. All steps are identical.

---

## Phases

- [Phase 1 — Register & Verify Custom Domain in Entra ID](#phase-1--register--verify-custom-domain-in-entra-id)
- [Phase 2 — Update On-Prem AD UPN Suffix & User Accounts](#phase-2--update-on-prem-ad-upn-suffix--user-accounts)
- [Phase 3 — Download & Install Microsoft Entra Connect Sync](#phase-3--download--install-microsoft-entra-connect-sync)
- [Phase 4 — Validate Synchronisation](#phase-4--validate-synchronisation)
- [Real-World Observations & Lessons Learned](#real-world-observations--lessons-learned)
- [Project Completion Checklist](#project-completion-checklist)

---

## Phase 1 — Register & Verify Custom Domain in Entra ID

Entra ID tenants use a default `.onmicrosoft.com` domain. A custom, routable domain must be registered and verified before UPN suffix matching will work during sync. The domain `cloudidlab.net` was registered via GoDaddy.

### Step 1 — Navigate to Entra ID Custom Domain Names

**Entra Admin Center → left menu → Domain names.**
Existing tenant domains visible (`Mocktest909.onmicrosoft.com` listed as primary).

![Phase 1 — Step 1: Navigate to Entra ID Custom Domain Names](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%201.png)

---

### Step 2 — Add cloudidlab.net as Custom Domain

Click **+ Add custom domain** → enter `cloudidlab.net` in the side panel → click **Add domain**.

![Phase 1 — Step 2: Add cloudidlab.net as Custom Domain](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%202.png)

---

### Step 3 — Retrieve TXT Verification Record

Entra ID displays the TXT record values: Alias: `@`, Destination: [verification string], TTL: `3600`.

![Phase 1 — Step 3: Retrieve TXT Verification Record](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%203.png)

---

### Step 4 — Add TXT Record at GoDaddy

**GoDaddy DNS Management → new TXT record →** Type: `TXT`, Name: `@`, Value: [verification string] → **Save**.

![Phase 1 — Step 4: Add TXT Record at GoDaddy](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%204.png)

---

### Step 5 — Domain Verified — Notification Confirmed

Green notification: *'Successfully verified domain name cloudidlab.net for use within Mock test'.*

![Phase 1 — Step 5: Domain Verified — Notification Confirmed](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%205.png)

---

### Step 6 — Domain Properties — Verified Status

`cloudidlab.net` shows: Type: Custom, Status: **Verified**, Primary domain: No (before promotion).

![Phase 1 — Step 6: Domain Properties — Verified Status](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%206.png)

---

### Step 7 — Make Primary — Confirmation Dialog

*'Do you want to make cloudidlab.net your primary domain?'* → **Yes**.

![Phase 1 — Step 7: Make Primary — Confirmation Dialog](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%207.png)

---

### Step 8 — Primary Domain Set — Confirmed

Green notification: *'Domain name cloudidlab.net has been made primary of Mock test'*. Primary domain: **Yes**.

![Phase 1 — Step 8: Primary Domain Set — Confirmed](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%201-Step%208.png)

---

## Phase 2 — Update On-Prem AD UPN Suffix & User Accounts

The internal `corp.local` suffix is non-routable and unrecognised by Entra ID. Two actions are required: add the verified suffix to AD, then update each user account.

### Step 1 — Open Active Directory Domains and Trusts on DC1

**Server Manager → Tools → Active Directory Domains and Trusts** (or `Win+R` → `domain.msc`). `corp.local` forest visible.

![Phase 2 — Step 1: Open Active Directory Domains and Trusts on DC1](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%202-Step%201.png)

---

### Step 2 — Add cloudidlab.net as Alternative UPN Suffix

**Right-click root node → Properties → Alternative UPN Suffixes** → type `cloudidlab.net` → **Add**. Confirmed in list (highlighted blue).

![Phase 2 — Step 2: Add cloudidlab.net as Alternative UPN Suffix](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%202-Step%202.png)

---

### Step 3 — Verify Updated UPN Suffixes in PowerShell

`Get-ADUser -Filter * | Select-Object Name, UserPrincipalName` confirms: John Smith and Jane Doe updated to `@cloudidlab.net`. IT Admin intentionally left on `@corp.local` to test sync behaviour.

![Phase 2 — Step 3: Verify Updated UPN Suffixes in PowerShell](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%202-Step%203.png)

> 🧪 **Deliberate Test:** IT Admin was intentionally left with `@corp.local` UPN. Result: Entra ID synced the account but substituted the UPN to `itadmin@Mocktest909.onmicrosoft.com` — a real-world behaviour fully documented in Phase 4.

---

## Phase 3 — Download & Install Microsoft Entra Connect Sync

The **Customize** path was chosen (not Express Settings) due to the non-routable `corp.local` domain requiring manual UPN suffix configuration.

### Step 1 — Download Connect Sync Agent from Entra Admin Center

**entra.microsoft.com → Microsoft Entra Connect → Get started → Connect Sync section → Download Connect Sync Agent.**

![Phase 3 — Step 1: Download Connect Sync Agent from Entra Admin Center](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%201.png)

---

### Step 2 — AzureADConnect.msi Downloaded Successfully

`AzureADConnect.msi` (148,332 KB / ~145 MB) downloaded to the Downloads folder on DC1.

![Phase 3 — Step 2: AzureADConnect.msi Downloaded Successfully](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%202.png)

---

### Step 3 — Express Settings — corp.local Non-Routable Warning

Orange warning: *'corp.local is not a routable domain. It is recommended to use custom settings.'* Decision: click **Customize**.

![Phase 3 — Step 3: Express Settings — corp.local Non-Routable Warning](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%203.png)

---

### Step 4 — Required Components — All Defaults

No existing sync service found. All optional checkboxes left unticked. Fresh clean install. Click **Install**.

![Phase 3 — Step 4: Required Components — All Defaults](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%204.png)

---

### Step 5 — User Sign-In — Password Hash Sync + Seamless SSO

**Password Hash Synchronisation** selected. **Enable single sign-on** ticked. Correct auth method for this topology.

![Phase 3 — Step 5: User Sign-In — Password Hash Sync + Seamless SSO](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%205.png)

---

### Step 6 — Connect to Microsoft Entra ID

Enter Global Admin credentials: `OmokoredeOludairo@Mocktest909.onmicrosoft.com` (`.onmicrosoft.com` used as root admin account).

![Phase 3 — Step 6: Connect to Microsoft Entra ID](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%206.png)

---

### Step 7 — Connect Directories — corp.local Detected

Active Directory type auto-detected. Forest: `corp.local` in dropdown. Click **Add Directory**.

![Phase 3 — Step 7: Connect Directories — corp.local Detected](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%207.png)

---

### Step 8 — AD Forest Account — Create New AD Account

**Create new AD account** selected. Enterprise Admin: `CORP\azureadmin`. Entra Connect creates a dedicated `MSOL_` sync service account.

![Phase 3 — Step 8: AD Forest Account — Create New AD Account](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%208.png)

---

### Step 9 — corp.local Connected — Green Tick Confirmed

Configured Directories: `corp.local (Active Directory)` with green tick. Next button activated.

![Phase 3 — Step 9: corp.local Connected — Green Tick Confirmed](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%209.png)

---

### Step 10 — Entra Sign-In Config — UPN Suffix Mapping

UPN table: `corp.local` = Not Added, `cloudidlab.net` = Verified. *'Continue without matching all UPN suffixes'* ticked to acknowledge `corp.local` warning.

> **Why this is acceptable:** Accounts with a `corp.local` UPN will receive a fallback `onmicrosoft.com` UPN in Entra ID — as demonstrated by the IT Admin account in Phase 4.

![Phase 3 — Step 10: Entra Sign-In Config — UPN Suffix Mapping](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%2010.png)

---

### Step 11 — Uniquely Identifying Users

Users represented only once across all directories. Let Azure manage the source anchor. Defaults correct for single forest.

![Phase 3 — Step 11: Uniquely Identifying Users](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%2011.png)

---

### Step 12 — Optional Features — Password Hash Sync Only

Only **Password Hash Synchronisation** ticked. All other features left off.

![Phase 3 — Step 12: Optional Features — Password Hash Sync Only](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%2012.png)

---

### Step 13 — Ready to Configure — Final Summary

Summary: Sync services, SSO, Source Anchor, Mocktest909 AAD Connector, corp.local Connector, PHS, Export Deletion Threshold (500). Start sync ticked. Staging mode off. Click **Install**.

![Phase 3 — Step 13: Ready to Configure — Final Summary](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%203-Step%2013.png)

---

## Phase 4 — Validate Synchronisation

Validation performed from both the cloud portal (Entra ID) and the on-premises sync engine (Synchronisation Service Manager on DC1). Entra Connect Health (P2 feature) checked as the final verification layer.

### Step 1 — Entra ID Users — 3 Synced Users Confirmed

Filter: On-premises sync enabled == Yes → **3 users found.**

| Display Name | User Principal Name | On-Premises Sync |
|---|---|---|
| John Smith | jsmith@cloudidlab.net | Yes |
| Jane Doe | jdoe@cloudidlab.net | Yes |
| IT Admin | itadmin@Mocktest909.onmicrosoft.com | Yes (UPN auto-substituted) |

![Phase 4 — Step 1: Entra ID Users — 3 Synced Users Confirmed](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%204-Step%201.png)

---

### Step 2 — John Smith Profile — Full Hybrid Identity Data

| Attribute | Value |
|---|---|
| On-premises sync enabled | Yes |
| Last sync | Jun 5 2026 10:07 PM |
| Distinguished Name | CN=John Smith,OU=_CORP_USERS |
| SAM Account Name | jsmith |
| On-premises UPN | jsmith@cloudidlab.net |
| Domain | corp.local |
| Immutable ID | Confirmed |

![Phase 4 — Step 2: John Smith Profile — Full Hybrid Identity Data](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%204-Step%202.png)

---

### Step 3 — Sync Service Manager — 6 Successful Operations

| Connector | Operation | Status |
|---|---|---|
| corp.local | Export | ✅ Success |
| Mocktest909 | Export | ✅ Success |
| corp.local | Full Synchronisation | ✅ Success |
| Mocktest909 | Full Synchronisation | ✅ Success |
| Mocktest909 | Full Import | ✅ completed-no-objects (expected) |
| corp.local | Full Import | ✅ Success |

> **Note:** `completed-no-objects` on the Mocktest909 Full Import is not an error. On a fresh install with no cloud-only users, there is nothing to import from Entra ID back to the sync engine.

![Phase 4 — Step 3: Sync Service Manager — 6 Successful Operations](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%204-Step%203.png)

---

### Step 4 — Entra Connect Dashboard — Sync Enabled, PHS, SSO

- Sync status: **Enabled**
- Last sync: Less than 1 hour ago
- Password Hash Sync: **Enabled**
- Seamless SSO: **Enabled** (1 domain)
- Federation: Disabled

![Phase 4 — Step 4: Entra Connect Dashboard — Sync Enabled, PHS, SSO](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%204-Step%204.png)

---

### Step 5 — Connect Health — Zero Sync Errors (P2 Feature)

| Error Category | Count |
|---|---|
| Duplicate Attribute | 0 |
| Data Mismatch | 0 |
| Data Validation Failure | 0 |
| Large Attribute | 0 |
| Federated Domain | 0 |
| Existing Admin Role | 0 |
| Other | 0 |

![Phase 4 — Step 5: Connect Health — Zero Sync Errors (P2 Feature)](https://raw.githubusercontent.com/oludairo1/azure-cloud-lab-series/main/Project-2-Hybrid-Identity-Integration/screenshots/Phase%204-Step%205.png)

---

## Real-World Observations & Lessons Learned

### 1 — Download Location Changed
Microsoft moved Entra Connect off the Download Centre. The new location is: **Entra Admin Center → Microsoft Entra Connect → Get started → Download Connect Sync Agent.** This reflects Microsoft's ongoing consolidation of identity tooling into the Entra portal.

### 2 — Customize vs Express Settings
Express Settings would have auto-selected the non-routable `corp.local` UPN suffix, causing sync failures. **Customize** allows explicit UPN suffix selection. This is the correct approach whenever the internal AD domain is non-routable (`.local`, `.internal`, `.lan`).

### 3 — IT Admin UPN Substitution
IT Admin was intentionally left on `@corp.local`. Entra ID did not reject the sync — it substituted the UPN to `itadmin@Mocktest909.onmicrosoft.com` automatically. This is how Entra ID handles non-routable UPN suffixes in production.

### 4 — MSOL_ Service Account
The wizard created a dedicated `MSOL_` service account in AD for ongoing sync operations. This account must not be deleted or disabled — doing so will break sync and require reconfiguration of Entra Connect to restore it.

### 5 — completed-no-objects Status
The Full Import from `Mocktest909.onmicrosoft.com` showed `completed-no-objects`. This is not an error — on a fresh install with no cloud-only users, there is nothing to import from Entra ID back to the sync engine.

### 6 — Seamless SSO Auto-Enabled
Seamless SSO was enabled by the wizard and confirmed in the Entra Connect dashboard (1 domain). This provides the foundation for **Project 3: SSO Implementation**.

---

## Project Completion Checklist

| Phase | Task | Status |
|---|---|---|
| Phase 1 | cloudidlab.net registered at GoDaddy | ✅ |
| Phase 1 | TXT record added to GoDaddy DNS | ✅ |
| Phase 1 | Domain verified in Entra ID | ✅ |
| Phase 1 | Domain set as primary in Entra ID | ✅ |
| Phase 2 | cloudidlab.net added as UPN suffix in AD Domains and Trusts | ✅ |
| Phase 2 | John Smith and Jane Doe UPNs updated to @cloudidlab.net | ✅ |
| Phase 2 | IT Admin left on @corp.local — sync behaviour documented | ✅ |
| Phase 3 | Entra Connect downloaded from Entra Admin Center | ✅ |
| Phase 3 | Customize path selected — corp.local warning acknowledged | ✅ |
| Phase 3 | Password Hash Sync + Seamless SSO configured | ✅ |
| Phase 3 | corp.local forest connected with green tick | ✅ |
| Phase 3 | Installation completed — no errors | ✅ |
| Phase 4 | 3 users visible in Entra ID with on-premises sync: Yes | ✅ |
| Phase 4 | John Smith profile shows full hybrid identity attributes | ✅ |
| Phase 4 | Sync Service Manager — 6 successful operations | ✅ |
| Phase 4 | Entra Connect dashboard — sync, PHS, SSO all enabled | ✅ |
| Phase 4 | Connect Health (P2) — zero errors across all 7 categories | ✅ |
| All | All screenshots captured and labelled | ✅ |

---

## Cost Management — Closing Resources

Once this project is complete, stop or deallocate all VMs to avoid unnecessary charges:

1. **Azure Portal → Virtual Machines**
2. Select **DC1** → click **Stop** → confirm
3. Select **CLIENT1** → click **Stop** → confirm

> ⚠️ Do **not** delete the resource group yet — DC1's on-premises AD and Entra Connect Sync are required for **Projects 3 and 4**. Delete everything only after the full lab series is complete.

---

## Repository Structure

```
Project-2-Hybrid-Identity-Integration/
│
├── README.md                    ← This file
├── screenshots/
│   ├── Phase1-Step1.png
│   ├── Phase1-Step2.png
│   ├── Phase1-Step3.png
│   ├── Phase1-Step4.png
│   ├── Phase1-Step5.png
│   ├── Phase1-Step6.png
│   ├── Phase1-Step7.png
│   ├── Phase1-Step8.png
│   ├── Phase2-Step1.png
│   ├── Phase2-Step2.png
│   ├── Phase2-Step3.png
│   ├── Phase3-Step1.png
│   ├── Phase3-Step2.png
│   ├── Phase3-Step3.png
│   ├── Phase3-Step4.png
│   ├── Phase3-Step5.png
│   ├── Phase3-Step6.png
│   ├── Phase3-Step7.png
│   ├── Phase3-Step8.png
│   ├── Phase3-Step9.png
│   ├── Phase3-Step10.png
│   ├── Phase3-Step11.png
│   ├── Phase3-Step12.png
│   ├── Phase3-Step13.png
│   ├── Phase4-Step1.png
│   ├── Phase4-Step2.png
│   ├── Phase4-Step3.png
│   ├── Phase4-Step4.png
│   └── Phase4-Step5.png
└── Project2_Hybrid_Identity_Integration.pdf   ← Full formatted PDF
```

---

*Azure Lab Series — Omokorede Oludairo | cloudidlab.net | Entra ID P2*
