# Active Directory Basics Lab

A hands-on lab built in Microsoft Azure simulating a real enterprise IT environment. This covers everything from spinning up a Windows Server VM to managing users, groups, domain joining machines, and pushing Group Policy. All tasks reflect what you'd actually be doing in a Tier 1 helpdesk or sysadmin role.

---

## Environment

| | |
|---|---|
| **Platform** | Microsoft Azure |
| **Domain Controller** | Windows Server 2022 Datacenter |
| **Client Machine** | Windows Server 2022 (simulating Win 10/11) |
| **VM Size** | Standard E2S v3 — 2 vCPUs, 16GB RAM |
| **Remote Access** | Azure Bastion |
| **Domain** | lab.local |

---

## Lab Structure

```
lab.local
├── Branch One
│   ├── Users
│   ├── Computers
│   └── Groups
└── Branch Two
    ├── Users
    ├── Computers
    └── Groups
```

---

## Part 1 — Deploying the Server

Started by creating a Windows Server 2022 VM in Azure. Used Azure Bastion for remote access instead of RDP so no firewall rules were needed. Set the private IP to static since a domain controller's IP should never change — if it does, DNS breaks and nothing can find the domain.

![screenshot](screenshots/azure-vm-creation.png)

![screenshot](screenshots/static-ip-config.png)

Once Bastion finished deploying (takes about 5-10 minutes), connected to the server through the browser and installed the following roles via Server Manager → Add Roles and Features:

- Active Directory Domain Services (AD DS)
- DHCP Server
- DNS Server
- Print and Document Services
- Web Server (IIS)
- Group Policy Management

Rebooted after installation.

![screenshot](screenshots/server-manager-roles.png)

---

## Part 2 — Promoting to Domain Controller

After the reboot, opened Server Manager and clicked the notification flag for AD DS post-deployment configuration. Selected "Add a new forest" and set the root domain name to `lab.local`. After setting the DSRM password and clicking install, the server rebooted and came back as a fully functioning domain controller.

![screenshot](screenshots/promote-to-dc.png)

![screenshot](screenshots/new-forest-setup.png)

---

## Part 3 — Building the AD Structure

Opened Active Directory Users and Computers (ADUC). Everything in AD is an object — users, computers, and groups all fall under this. Organizational Units (OUs) are the folders that organize everything.

Created the following structure under lab.local:

- Branch One → Users, Computers, Groups
- Branch Two → Users, Computers, Groups

The reason for keeping a separate Computers OU is for hybrid Azure AD joining — computers need to be in a syncing OU so they can be joined to both on-prem AD and the cloud.

![screenshot](screenshots/aduc-ou-structure.png)

---

## Part 4 — Managing Users and Groups

Created the following users under Branch One → Users:

- Jake Is Cool (`jake.iscool@lab.local`)
- Tom Brady (`tom.brady@lab.local`)
- Mike Smith — created by **copying** Tom Brady, which automatically inherited his group memberships

![screenshot](screenshots/create-user.png)

![screenshot](screenshots/copy-user.png)

Created a security group called `IT Workers` under Branch One → Groups and added Tom Brady and Mike Smith as members.

![screenshot](screenshots/it-workers-group.png)

**Security Groups vs Distribution Lists**

Security groups control access to resources and GPOs. Distribution lists are for email — when you send to the list, everyone in it gets a copy. Two very different purposes.

---

## Part 5 — Common Helpdesk Tasks in AD

These are the kinds of tickets you'll be handling constantly at Tier 1.

**Password Reset**
Right-click user → Reset Password. Always verify the caller's identity first. Enable "User must change password at next logon" so they set their own password that you don't know.

![screenshot](screenshots/password-reset.png)

**Unlocking an Account**
Double-click user → Account tab → Unlock Account. Always check that the account is actually locked before touching it — sometimes users say they're locked out when the real issue is something else.

![screenshot](screenshots/unlock-account.png)

**Disabling an Account**
Right-click user → Disable Account. Disabled accounts show a downward arrow icon in ADUC. Be careful enabling these — a disabled account is often disabled on purpose, usually because someone was terminated.

![screenshot](screenshots/disabled-account.png)

**Moving Users Between OUs**
Dragged Tom Brady from Branch One → Branch Two. This matters because the OU a user is in determines which Group Policies apply to them. Wrong OU means wrong policies — wrong printers, wrong drive mappings, etc.

![screenshot](screenshots/move-user.png)

**Editing User Properties**
Added description, phone number, manager, job title, and department to Mike Smith. This data syncs with Microsoft Teams for org charts and email signatures.

![screenshot](screenshots/user-properties.png)

**Attribute Editor**
Used the Attribute Editor to add proxy addresses (email aliases) for Mike Smith. Capital SMTP = primary address, lowercase smtp = alias. Common task at T1/T2/T3 for email issues.

![screenshot](screenshots/attribute-editor.png)

**Searching for Users**
Used the Find Objects icon in ADUC to search the entire directory. Useful when you have thousands of users and don't know which OU someone is in. The Object tab shows the exact path.

![screenshot](screenshots/find-objects.png)

---

## Part 6 — Domain Joining a Client PC

Spun up a second VM called `lab-PC` in the same resource group and subnet as the DC (10.0.0.0/24). Before domain joining, set the DNS server on the client's NIC to point to the DC at `10.0.0.4`. Without this the client can't resolve `lab.local` and the domain join fails.

![screenshot](screenshots/azure-nic-dns.png)

Went to Settings → System → About → Advanced System Settings → Computer Name → Change → selected Domain and typed `lab.local`. Got the "Welcome to the lab.local domain" message and restarted.

![screenshot](screenshots/domain-join-success.png)

**DNS Troubleshooting**

Hit this error on the first attempt: *"Active Directory Domain Controller for domain lab.local could not be contacted."* Ran `ipconfig /all` and confirmed DNS was pointing to Azure's DNS instead of the DC. Fixed by manually setting DNS to `10.0.0.4` in the NIC properties.

```cmd
ncpa.cpl       # Opens network adapter settings directly
ipconfig /all  # Check current DNS server
```

![screenshot](screenshots/ipconfig-dns-fix.png)

After the domain join, refreshed ADUC and `lab-PC` appeared automatically in the default Computers OU. Moved it to Branch One → Computers.

![screenshot](screenshots/lab-pc-in-aduc.png)

---

## Part 7 — Local Users and Groups

There's an important difference between local accounts and domain accounts. Domain accounts authenticate against the DC — if the PC is offline or not on VPN, domain login may not work. Local accounts always work regardless of network.

```cmd
net user                        # List all local users on the machine
net localgroup administrators   # List members of local admins group
```

![screenshot](screenshots/net-user-output.png)

**Temp Admin Workflow**

Used when a user can't reach the DC (working from home, not on VPN) and you need admin access to their machine via RMM:

```cmd
# Create
net user tempadmin TempPass123! /add
net localgroup administrators tempadmin /add

# Delete after use — always
net localgroup administrators tempadmin /delete
net user tempadmin /delete
```

> Always delete temp admin accounts immediately after use. Leaving them is a security risk and will get flagged in compliance audits.

![screenshot](screenshots/temp-admin.png)

---

## Part 8 — AD Recycle Bin

Enabled the AD Recycle Bin through Active Directory Administrative Center → right-click domain → Enable Recycle Bin. This allows recovery of accidentally deleted objects without needing to restore from backup.

Deleted `lab-PC` from AD to test it, then recovered it through the Deleted Objects container → right-click → Restore. The computer reappeared in its original OU with the domain trust relationship fully intact.

![screenshot](screenshots/recycle-bin-deleted.png)

![screenshot](screenshots/recycle-bin-restored.png)

**Computer Object Attributes**

Computers have an Attribute Editor just like users. In environments with LAPS (Local Administrator Password Solution), the attribute `ms-mcs-admpwd` stores a rotating local admin password. This is the secure way to manage local admin accounts across all machines without using the same password everywhere.

![screenshot](screenshots/computer-attributes.png)

---

## Part 9 — Group Policy Objects (GPO)

Group Policy is how you centrally control settings across all users and computers in the domain without touching each machine individually. You create GPOs, configure settings inside them, then link them to an OU or the domain so they start applying.

**Two types of GPO settings:**
- **Computer Configuration** — applies to the machine regardless of who logs in
- **User Configuration** — applies to the logged-in user regardless of which machine they're on

Opened Group Policy Management Console on the DC.

![screenshot](screenshots/gpmc.png)

---

**GPO 1 — Disable Domain Firewall (Computer Config)**

Created a new GPO called `Disable Domain Firewall`. Edited it at:

Computer Configuration → Policies → Windows Settings → Security Settings → Windows Defender Firewall → Windows Defender Firewall Properties → Domain Profile → Firewall State: Off

Linked it at Branch One OU level. Set security filtering to `lab-client` only for testing.

![screenshot](screenshots/firewall-gpo-settings.png)

Ran `gpupdate /force` on the client and confirmed with `gpresult`. Domain firewall was off.

![screenshot](screenshots/firewall-off-client.png)

---

**GPO 2 — Hide Control Panel (User Config)**

Created a new GPO called `Hide Control Panel`. Edited it at:

User Configuration → Policies → Administrative Templates → Control Panel → Prohibit access to Control Panel and PC Settings → Enabled

Linked at the domain level so it applies regardless of which OU the user is in. Security filtered for `tom.brady` and `lab-client`.

![screenshot](screenshots/hide-control-panel-gpo.png)

Logged into the client as Tom Brady, ran `gpupdate /force`, and confirmed Control Panel was blocked.

![screenshot](screenshots/control-panel-blocked.png)

---

**Key GPO Commands**

```cmd
gpupdate /force                  # Force client to pull latest policies from DC
gpresult /r /scope:computer      # Show GPOs applied to the computer
gpresult /r /user:tom.brady      # Show GPOs applied to a specific user
```

![screenshot](screenshots/gpupdate-output.png)

![screenshot](screenshots/gpresult-output.png)

---

**GPO Troubleshooting**

The Hide Control Panel GPO wasn't applying to Tom Brady initially. After troubleshooting, found that User Config GPOs require both the **user** and the **computer** to be in security filtering — not just the user. Adding `lab-client` to security filtering alongside `tom.brady` fixed it.

For a broader fix you can add `Domain Computers` to security filtering so all machines can read the GPO while only the specified users get it applied.

---

**GPO Workflow**

```
1. Create GPO in Group Policy Objects folder
2. Edit settings (Computer Config or User Config)
3. Set security filtering to test user/computer only
4. Link GPO to OU or domain
5. Run gpupdate /force on client
6. Verify with gpresult
7. Roll out to Authenticated Users once confirmed working
```

---

## Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| DC could not be contacted during domain join | DNS pointing to Azure DNS instead of DC | Set DNS manually to 10.0.0.4 in NIC settings |
| GPO not applying to Tom Brady | Computer object not in security filtering for User Config GPO | Added lab-client to security filtering alongside tom.brady |
| Bastion not connecting | VM was stopped | Started the VM before connecting |

---

## What I Learned

- How to deploy and configure Windows Server in Azure
- How to use Azure Bastion for secure remote access
- How to promote a server to a Domain Controller
- What OUs are and how they organize AD objects
- How to perform common Tier 1 AD tasks — password resets, unlocks, disabling accounts, moving users
- The difference between Security Groups and Distribution Lists
- How DNS must point to the DC for domain joining to work
- The difference between local and domain credentials
- How to safely create and delete temp admin accounts
- How to enable and use the AD Recycle Bin
- What LAPS is and why it matters
- How to create, link, and troubleshoot Group Policy Objects
- Why User Config GPOs need both the user and computer in security filtering

---

## Author
Haider Shah

## Tools Used
- Microsoft Azure
- Windows Server 2022
- Active Directory Users and Computers (ADUC)
- Active Directory Administrative Center
- Group Policy Management Console (GPMC)
- PowerShell / Command Prompt
