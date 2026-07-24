# Helpdesk Homelab - Setup
## Entra ID & Intune

---

## Lab Environment

### Network topology

| Device | IP | Role |
|---|---|---|
| Hyper-V Host | 192.168.10.1 | NAT gateway for CorpNet subnet |
| DC01 | 192.168.10.10 | Domain controller, DNS, on-prem AD for corp.local |
| WRK01 | 192.168.10.20 | Domain joined workstation for on-prem AD drills |
| WRK02 | 192.168.10.30 | Entra ID joined workstation for Intune drills (not domain joined to DC01) |

WRK02 uses 8.8.8.8 as DNS - it does not need to resolve corp.local and has no dependency on DC01 being online.

WRK01 requires DC01 to be running for domain authentication and DNS resolution.

### Tenant

| Item | Value |
|---|---|
| Tenant name | igorshelpdesk.onmicrosoft.com |


### Users

| Display Name | UPN | Group |
|---|---|---|
| Alex Helpdesk | alex.helpdesk@igorshelpdesk.onmicrosoft.com | IT Staff |
| Sarah Employee | sarah.employee@igorshelpdesk.onmicrosoft.com | Standard Users |
| Mike Manager | mike.manager@igorshelpdesk.onmicrosoft.com | Standard Users |

### Groups

| Group | Type | Members |
|---|---|---|
| IT Staff | Security, Assigned | Alex Helpdesk |
| Standard Users | Security, Assigned | Sarah Employee, Mike Manager |

---

## Setup Methodology

### Phase 1 - Tenant Provisioning

The Microsoft 365 Developer Program is now restricted to Visual Studio Enterprise/Professional subscribers. The workaround is to stack free trials manually inside a single tenant. This provided some issues at first but we managed to get it working.

1. Create a free Azure account at `azure.microsoft.com/en-au/free`
2. Navigate to `admin.microsoft.com` and activate the **Microsoft Intune Suite Trial** - this provisions the `igorshelpdesk.onmicrosoft.com` tenant with 250 licences and a 90-day trial
3. Inside the `igorshelpdesk` tenant, activate the **Microsoft Entra ID P2 Trial** via Billing → Purchase services
4. Wait 10-15 minutes for licence propagation before proceeding

Intune and Entra ID must be in the same tenant. Intune automatically creates an Entra ID tenant when provisioned. Do not attempt to use a separately created Entra ID tenant with a separately provisioned Intune instance - they will not link.

### Phase 2 - User Creation

Navigate to `entra.microsoft.com` → Users → + New user → Create new user.

Create all three users before creating groups. For each user:
- Set display name and UPN
- Set a temporary password
- Assign licences immediately on creation (Intune Suite + Entra ID P2)

Users created:
- Alex Helpdesk (`alex.helpdesk@igorshelpdesk.onmicrosoft.com`)
- Sarah Employee (`sarah.employee@igorshelpdesk.onmicrosoft.com`)
- Mike Manager (`mike.manager@igorshelpdesk.onmicrosoft.com`)

### Phase 3 - Group Creation and Member Assignment

Navigate to `entra.microsoft.com` → Groups → + New group.

For each group:
- Group type: Security
- Membership type: Assigned
- Add members immediately during group creation - do not create empty groups

Groups created:
- IT Staff → Alex Helpdesk
- Standard Users → Sarah Employee, Mike Manager

Always verify member count after saving. An empty group silently fails - any policy targeting an empty group simply does not apply with no error message. This was the root cause of Intune enrolment failures during initial lab setup.

### Phase 4 - Intune Configuration

**NOTE:** Configuration and compliance needs revisiting. We set this up without looking too much into it for the sake of building the homelab and getting drills up and running ASAP.

**Automatic MDM Enrolment**

Navigate to `entra.microsoft.com` → Mobility (MDM and WIP) → Microsoft Intune.

- MDM user scope: **All** - use All for lab environments. Setting to Some adds a group targeting dependency that silently fails if group membership is empty.
- WIP user scope: **None** - MAM/WIP takes precedence over MDM if enabled. Any value other than None will block MDM enrolment silently.
- Verify MDM URLs are populated. If blank, click **Restore default MDM URLs** and save.

**Compliance Policy**

Navigate to `intune.microsoft.com` → Devices → Compliance → + Create policy → Windows 10 and later.

Name: `Standard Users Compliance Policy`
Assigned to: Standard Users group

Key settings:

NOTE: This section needs revisiting as we used a guide to set this up without deep understanding for the sake of speedily setting up the homelab.

Device Health:
- BitLocker: Require
- Secure Boot: Not configured (Hyper-V Generation 2 VMs cannot support Secure Boot)
- Code Integrity: Require

System Security:
- Minimum password length: 8
- Maximum minutes of inactivity before password required: 15
- Password expiration (days): 90
- Firewall: Require
- TPM: Require
- Antivirus: Require
- Antispyware: Require
- Defender Antimalware: Require
- Security intelligence up to date: Require
- Real-time protection: Require
- Defender for Endpoint machine risk score: Clear

Actions for noncompliance:
- Mark device noncompliant: Immediately
- Retire device: 30 days

**Configuration Profile 1 - Device Restrictions**

Navigate to `intune.microsoft.com` → Devices → Configuration → + Create → New policy → Windows 10 and later → Templates → Device restrictions.

Name: `Standard Users Device Restrictions`
Assigned to: Standard Users group

Key settings:
- Screen capture: Block
- Manual unenrollment: Block
- Microsoft account: Block
- Settings app: Block
- Minimum password length: 8
- Sign-in failures before device wipe: 10
- SmartScreen for Microsoft Edge: Require
- Malicious site access: Block
- Unverified file download: Block
- Defender real-time monitoring: Enable
- Behaviour monitoring: Enable
- Scan all downloads: Enable

**Configuration Profile 2 - Endpoint Protection**

Navigate to `intune.microsoft.com` → Devices → Configuration → + Create → New policy → Windows 10 and later → Templates → Endpoint protection.

Name: `Standard Users Endpoint Protection`
Assigned to: Standard Users group

Key settings:

Windows Firewall:
- Firewall: Enable on Domain, Private, and Public profiles
- Stealth mode: Block on all profiles
- Inbound connections: Block on all profiles
- Outbound connections: Allow on all profiles

Windows Encryption (BitLocker):
- Warning for other disk encryption: Block
- BitLocker fixed drive recovery: Enable
- Save BitLocker recovery information to Entra ID: Enable
- Store recovery information before enabling BitLocker: Require

Microsoft Defender Exploit Guard - Attack Surface Reduction rules (all set to Block):
- Flag credential stealing from LSASS (Local Security Authority Subsystem Service)
- Office apps injecting into other processes
- Office apps creating executable content
- Office apps launching child processes
- Win32 imports from Office macro code
- Obfuscated JS/VBS/PS/macro code
- JS/VBS executing payload downloaded from internet
- Process creation from PSExec and WMI commands
- Untrusted and unsigned processes from USB
- Executables that don't meet prevalence/age/trusted list criteria
- Execution of executable content dropped from email
- Advanced ransomware protection

Controlled Folder Access:
- Folder protection: Block disk modification

Microsoft Defender Credential Guard:
- Credential Guard: Enable with UEFI lock

Microsoft Defender Security Center:
- Tamper protection: Enable

Local Device Security Options:
- Block Microsoft accounts: Block
- Display last signed-in user: Disabled
- Anonymous enumeration of SAM accounts: Block
- LAN Manager authentication level: Send NTLMv2 response only, refuse LM and NTLM
- UAC elevation for standard users: Automatically deny elevation requests
- UAC run all administrators in admin approval mode: Enable
- Local Administrator account: Disabled
- Guest account: Disabled

Xbox Services: All disabled

### Phase 5 - WRK02 Setup and Enrolment

WRK02 must NOT be domain joined to corp.local. It is a pure Entra ID joined device managed entirely by Intune.

**Hyper-V VM settings:**
- Generation 2
- 4GB RAM fixed allocation (disable dynamic memory)
- 2+ virtual processors
- CorpNet virtual switch
- TPM enabled under Security settings
- Secure Boot disabled

**Windows 11 Enterprise installation:**
- Use `oobe\bypassnro` at the Microsoft account sign-in screen if needed to create a local account during setup
- Rename PC to WRK02 before joining Entra ID

**Network settings (set via Settings → Network → Ethernet → Edit → Manual):**
- IP: 192.168.10.30
- Subnet mask: 255.255.255.0
- Gateway: 192.168.10.1
- DNS: 8.8.8.8

**Create local admin fallback account before enrolment:**
```powershell
net user localadmin Password123! /add
net localgroup administrators localadmin /add
```

**Take a Hyper-V checkpoint before enrolment** - revert to this if enrolment fails rather than retrying on a corrupted state.

**Entra ID join:**
Settings → Accounts → Access work or school → Connect → Join this device to Microsoft Entra ID → sign in as `sarah.employee@igorshelpdesk.onmicrosoft.com`

**Verify enrolment:**

Run `dsregcmd /status` and confirm:

| Field | Expected Value |
|---|---|
| AzureAdJoined | YES |
| DomainJoined | NO |
| MdmUrl | populated (not blank) |
| IsUserAzureAD | YES |
| AzureAdPrt | YES |

Then check `intune.microsoft.com` → Devices → All devices - WRK02 should appear within 5-10 minutes of a successful Entra ID join with MDM URLs populated.

