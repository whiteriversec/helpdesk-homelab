# Helpdesk Homelab - Notes
## Entra ID & Intune

---

## Purpose

This homelab replicates the identity and device management environment found in modern Australian IT organisations and MSPs. The goal is to build hands-on familiarity with Microsoft Entra ID and Microsoft Intune - the two platforms that underpin the majority of helpdesk work in cloud-first and hybrid Microsoft environments.

Most entry-level helpdesk roles in Sydney require working knowledge of:
- User and group management in Entra ID
- MFA troubleshooting and reset
- Device compliance and enrolment via Intune
- Basic Conditional Access understanding
- Microsoft 365 licence assignment

This lab provides an isolated environment to build and drill all of these skills to improve familiarity and proficiency.

---

## Entra ID vs On-Premises Active Directory

**On-premises Active Directory (AD)**
Active Directory is Microsoft's traditional identity system, running on a Windows Server domain controller inside an organisation's own network. It uses protocols called LDAP (Lightweight Directory Access Protocol) and Kerberos for authentication. Everything happens on the organisation's own hardware. Users, computers, and policies are stored and managed locally.

In our on-prem lab, on-prem AD is represented by DC01 virtual machine running Windows Server 2022 with the corp.local domain. WRK01 is domain joined to corp.local and managed by Group Policy from DC01.

**Microsoft Entra ID**
Entra ID (formerly Azure Active Directory) is Microsoft's cloud-based identity service. It uses modern protocols like OAuth 2.0 and SAML for authentication. It is the identity backbone of Microsoft 365 - every Teams, SharePoint, Outlook, and Intune interaction is authenticated through Entra ID.

In this lab, Entra ID is represented by the 'igorshelpdesk.onmicrosoft.com' tenant. WRK02 is Entra ID joined and managed by Intune.

**Key differences:**

| | On-Premises AD | Entra ID |
|---|---|---|
| Location | Organisation's own servers | Microsoft's cloud |
| Protocols | LDAP, Kerberos | OAuth 2.0, SAML, OpenID Connect |
| Device management | Group Policy | Intune (MDM) |
| Authentication scope | Internal network resources | Cloud apps and services |
| Admin tool | Active Directory Users and Computers | Entra Admin Centre (entra.microsoft.com) |

**Hybrid environments**
Most large enterprises run both simultaneously. Entra Connect (formerly Azure AD Connect) synchronises users and groups from on-prem AD to Entra ID automatically. On-prem AD remains the source of truth - user creation happens on-prem first and syncs to the cloud. Entra ID cannot push new users back to on-prem AD.

In a hybrid environment, a helpdesk tech creates users in on-prem AD, not in Entra ID directly. The sync happens automatically within 30 minutes.

---

## Key Portals

| Portal | URL | Purpose |
|---|---|---|
| Entra Admin Centre | entra.microsoft.com | Primary identity management - users, groups, Conditional Access, MFA |
| Intune Admin Centre | intune.microsoft.com | Device management - compliance, configuration profiles, enrolment |
| Microsoft 365 Admin Centre | admin.microsoft.com | Licensing, billing, basic user and group management |
| Azure Portal | portal.azure.com | Full Azure platform - Entra ID is one section within it |
| My Account | myaccount.microsoft.com | End user self-service - password change, MFA setup, device management |
| Microsoft 365 Lighthouse | lighthouse.microsoft.com | MSP portal - single pane of glass across all managed client tenants |

Entra Admin Centre and Azure Portal both surface Entra ID - they access the same underlying service through different interfaces. Entra Admin Centre is scoped to identity only and is less cluttered. Azure Portal covers the entire Azure platform.

---

## Key Concepts

### Tenant
The tenant is the top-level container for an organisation's entire Microsoft cloud environment. One tenant holds all users, groups, devices, applications, licences, and policies. Every organisation has one tenant with a unique domain in the format `organisationname.onmicrosoft.com`. Custom domains (e.g. `company.com.au`) can be added and verified on top.

In an MSP context, each client is a separate tenant. The MSP manages multiple tenants via GDAP (Granular Delegated Admin Privileges) without needing a separate user account in each one.

### Users
Users are individual identity objects inside the tenant. Each has a UPN (User Principal Name) - their unique identifier in the format `username@tenantdomain.onmicrosoft.com`.

Users can be created via the portal (one at a time), PowerShell using the Microsoft Graph PowerShell SDK (bulk operations via CSV), or synced automatically from on-prem AD via Entra Connect in hybrid environments.

### Groups
Groups allow policies, licences, and permissions to be assigned at scale rather than to individual users. A user inherits whatever their group has. There are two primary types:

**Security Group** - used for policy assignment, licence assignment, and access control. This is what Intune compliance policies, Conditional Access policies, and permission assignments target.

**Microsoft 365 Group** - used for collaboration. Automatically creates a Teams channel, shared mailbox, SharePoint site, and shared calendar. Not used for policy assignment.

Groups have two membership modes:

**Assigned** - members are added manually. Simple and predictable. Requires manual maintenance as the org changes.

**Dynamic** - members are added automatically based on user attribute rules (e.g. all users where Department = "Finance"). Scales automatically but requires careful attribute management.

### Licences
Licences unlock specific features and services. Key licences relevant to this lab:

**Microsoft Intune Suite** - enables full Intune device management (MDM and MAM).

**Microsoft Entra ID P1** - enables Conditional Access, SSPR (Self-Service Password Reset), and automatic MDM enrolment.

**Microsoft Entra ID P2** - includes everything in P1 plus Identity Protection (risk-based Conditional Access), Privileged Identity Management (just-in-time admin access), and Access Reviews.

Licences are assigned per user. A user without an Intune licence cannot enrol a device. A user without Entra ID P1/P2 cannot be targeted by Conditional Access policies.

### Roles
Roles control what an administrator can do inside the tenant. Roles follow least privilege - each role grants only the permissions needed for that function.

| Role | What It Can Do |
|---|---|
| Global Administrator | Full control over everything in the tenant |
| Helpdesk Administrator | Reset passwords, manage MFA, view users |
| User Administrator | Create/delete users, manage group membership, assign licences |
| Intune Administrator | Full Intune management, no identity control |

In a real org a helpdesk L1 tech typically gets Helpdesk Administrator - enough to handle password resets and MFA issues without being able to modify security policies or create users.

### MDM and MAM
**MDM (Mobile Device Management)** - full device management. Intune controls the entire device - compliance policies, configuration profiles, app deployment, remote wipe. Used for corporate-owned devices.

**MAM (Mobile Application Management)** - app-only management without enrolling the full device. Intune manages specific work apps and their data without touching the rest of the device. Used for BYOD (Bring Your Own Device) scenarios where the employee's personal phone is used for work email and Teams.

### Conditional Access
Conditional Access is the policy engine that controls access to cloud resources based on signals - who is signing in, from where, on what device, with what risk level. It evaluates every sign-in and decides whether to allow, block, or require additional verification (MFA).

Requires Entra ID P1 or P2 licence.

### GDAP
GDAP (Granular Delegated Admin Privileges) is the system that controls what permissions an MSP technician has inside a customer's tenant. It replaced the older DAP (Delegated Admin Privileges) model which gave all MSP technicians Global Administrator access to all client tenants - a significant security risk.

With GDAP, the MSP assigns specific Entra roles per technician per client with defined expiry dates. A junior helpdesk tech might get Helpdesk Administrator on some clients but not others. A compromised technician account has limited blast radius rather than God-mode access across all clients.

### Compliance Policies vs Configuration Profiles (Intune)
These are the two pillars of Intune device management.

**Compliance policies** - the audit layer. Continuously checks whether a device meets defined requirements (BitLocker enabled, firewall on, Defender running, OS version current). Reports pass or fail. Feeds into Conditional Access - a noncompliant device can be blocked from accessing cloud resources.

**Configuration profiles** - the enforcement layer. Actively pushes settings to devices. Instead of checking if the firewall is on, a configuration profile turns it on and keeps it on. Covers device restrictions, endpoint protection, password policy, Wi-Fi, VPN, certificates, and more.