# Windows Server 2022 Active Directory Lab

A hands-on Active Directory homelab built in VirtualBox for practicing AD administration — OU design, user/group management, NTFS & share permissions, and Group Policy — as part of CompTIA Network+ study and IT job search prep (targeting entry-level Network Technician / IT Support roles).

## Environment

| Component | Detail |
|---|---|
| Host OS | Linux Mint (8GB RAM) |
| Hypervisor | Oracle VirtualBox |
| Guest OS | Windows Server 2022 (Desktop Experience) |
| Allocated RAM | 2GB (dropped from 4GB after a host freeze — see Lessons Learned) |
| Network Mode | Bridged adapter |
| Domain Name | `lab.local` |
| NetBIOS Name | `LAB` |
| DC Hostname | `WIN-FBN4HA8NO8A` |
| DC Static IP | `192.168.0.80` / `255.255.255.0` / GW `192.168.0.1` |
| DNS | `127.0.0.1` (self — the DC also runs the DNS role) |

## 1. VM & Network Setup

- Rebuilt the VM from scratch after losing the admin password on a prior build.
- Diagnosed a host freeze caused by over-allocating RAM (4GB on an 8GB host); dropped to 2GB and confirmed stability (idle CPU/memory behavior) before proceeding.
- Configured a **static IP** before promoting to a DC — AD DS depends heavily on stable DNS, and a DHCP-leased DC risks breaking domain resolution if its address changes.
- Chose an IP (`192.168.0.80`) confirmed to sit outside the router's DHCP range to avoid future collisions.

## 2. AD DS Role Install & Promotion

- Installed the **Active Directory Domain Services** role via Server Manager (Add Roles and Features) — stages the AD binaries but doesn't create a domain yet.
- Promoted the server via **"Add a new forest"**:
  - Root domain: `lab.local`
  - Forest/Domain functional level: Windows Server 2016
  - DNS Server role and Global Catalog installed alongside
  - Set a DSRM (Directory Services Restore Mode) password — separate from the Domain Admin password, used only for AD database recovery boots
- Confirmed the DC now serves as the authoritative DNS server for `lab.local`, a private namespace public resolvers (e.g. 8.8.8.8) have no knowledge of — AD's SRV records (`_ldap._tcp`, `_kerberos._tcp`, etc.) only exist in this self-hosted zone.

## 3. OU Structure

Built a three-department OU hierarchy under `lab.local`, each with sub-OUs to separate object types so GPOs and delegation can target users vs. computers precisely:

```
lab.local
├── Sales
│   ├── Users
│   ├── Computers
│   └── Groups
├── IT
│   ├── Users
│   ├── Computers
│   └── Groups
└── HR
    ├── Users
    ├── Computers
    └── Groups
```

**Key concept:** OUs organize a *single* domain — departments are not separate domains. A domain only splits further in rare cases (mergers, legal isolation requirements, geo-replication needs).

## 4. Users, Groups, and Shared Resources

Same pattern repeated for all three departments:

| Department | Users Created | Security Group | Shared Folder | Drive Letter |
|---|---|---|---|---|
| Sales | John Smith (`jsmith`), Jane Doe (`jdoe`), Mike Johnson (`mjohnson`) | `Sales-Staff` (Global, Security) | `C:\Sales-Files` | `S:` |
| IT | Sarah Chen (`schen`), Tom Reilly (`treilly`) | `IT-Staff` (Global, Security) | `C:\IT-Files` | `I:` |
| HR | Emily Watson (`ewatson`), David Park (`dpark`) | `HR-Staff` (Global, Security) | `C:\HR-Files` | `H:` |

For each shared folder, two independent permission layers were configured:

- **Share Permissions** (Sharing → Advanced Sharing → Permissions) — governs access only over the network path (`\\server\share`). Default "Everyone" removed; the department's security group added with **Change**.
- **NTFS Permissions** (Security tab) — governs access at the filesystem level, always enforced (network or local). Department's security group added with **Modify**.

**Key concept:** when both layers apply (network access to a share), Windows enforces the **more restrictive** of the two. A common real-world "can't save to the shared drive" ticket traces back to a mismatch between these two layers — reproduced and diagnosed live in this lab.

## 5. Group Policy

### a) Local Logon Right (lab-only shortcut)

By default, regular domain users **cannot log on locally to a Domain Controller** ("Allow log on locally" is restricted to privileged groups via the Default Domain Controllers Policy) — intentional hardening, since a DC holds the entire domain's identity database. In a real environment, users log into their own workstations, not the DC.

Since this lab has only one VM, `Sales-Staff`, `IT-Staff`, and `HR-Staff` were temporarily granted this right (`gpmc.msc` → Default Domain Controllers Policy → Computer Configuration → Security Settings → Local Policies → User Rights Assignment) purely to test login and permissions end-to-end on the DC itself.

### b) GPO-Based Drive Mapping

Built one GPO per department (`Sales-Drive-Mapping`, `IT-Drive-Mapping`, `HR-Drive-Mapping`), each linked to its respective OU:

- **Location:** `User Configuration → Preferences → Windows Settings → Drive Maps`
- **Action:** Create
- **Path:** `\\WIN-FBN4HA8NO8A\<Department>-Files`
- **Reconnect:** enabled

**Key concept — Computer vs. User Configuration:** rule of thumb used throughout — *"does this setting need to follow the person, or follow the machine?"* Drive mappings follow the **user**, so they live in User Configuration; a user should get their mapped drive on whichever machine they log into.

**Key concept — GPMC vs. Group Policy Management Editor:**
- **GPMC** (`gpmc.msc`) — creates GPOs, links them to OUs, and scopes them via Security Filtering. Answers *"where/who does this policy reach."*
- **Group Policy Management Editor** (right-click a GPO → Edit) — configures the policy's actual settings. Answers *"what does this policy do."* Has no awareness of linking or filtering — that logic lives entirely in GPMC.

## 6. Real Debugging (the most valuable part)

Two GPO drive mappings initially failed silently — the GPO showed as **Applied** in `gpresult /r`, but no drive letter appeared for the test user. Root cause both times: a single mistyped character in the UNC path, typed manually into the GPO (no host↔VM clipboard sharing configured):

- `WIN0FBN4HA8NO8A` instead of `WIN-FBN4HA8NO8A` (zero instead of hyphen)
- A capital `O` mistyped as a zero elsewhere in the hostname

**Diagnosis process:**
1. Confirmed the GPO was actually delivered (`gpresult /r` → checked "Applied Group Policy Objects")
2. Since delivery was confirmed, isolated the fault to the GPO's own configuration, not linking/filtering
3. Compared the typed path character-by-character against the real hostname (`hostname` command) rather than trusting it visually

This mirrors a real-world troubleshooting pattern: a GPO confirmed delivered but producing no result almost always points to a data-entry error inside the policy's own settings, not a permissions or linking problem.

## Key Takeaways / Concepts Demonstrated

- Domain vs. Forest vs. OU vs. Container — structural hierarchy of AD
- OUs organize objects and scope GPOs/delegation; they are **not** security principals themselves (rights/permissions are granted to users/groups, never directly to an OU)
- Group membership and OU placement are independent systems — moving a user's OU doesn't affect group-based resource access, and vice versa
- Share permissions vs. NTFS permissions are two separate, simultaneously-enforced gates
- GPO scope = **where it's linked** (GPMC) + **who it's filtered to** (Security Filtering), completely separate from **what it configures** (the Editor)
- Computer Configuration vs. User Configuration = machine-scoped vs. person-scoped settings
- DNS is not optional infrastructure for AD — it's how domain-joined machines locate authentication services (SRV records)
- In a real environment, file/print resources typically live on a **member server**, not the DC, to keep identity infrastructure isolated from application/data workloads

## Lab Limitations (Shortcuts vs. Real-World Practice)

- All shared folders live on the DC's own `C:` drive — in production this would be a dedicated file server (a domain-joined member server, not a DC)
- Regular users were granted local logon rights on the DC purely to test permissions on a single-VM setup — this would be a security finding in any real audit
- Single-domain, single-forest, single-DC lab — no replication, multi-site topology, or trust relationships

## Next Steps

- [ ] Fine-grained password policy GPO
- [ ] Login banner GPO
- [ ] Delegation of control (scoped admin rights per-OU without full Domain Admin)
- [ ] Optional: second VM as a dedicated file server, to move shares off the DC properly
