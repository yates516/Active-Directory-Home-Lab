# Active Directory Home Lab

## Project Overview

Built and administered a virtualized Windows domain environment to practice foundational IT support, Windows Server administration, identity and access management, and troubleshooting. The lab uses a Windows Server domain controller and a domain-joined Windows 11 client to demonstrate user administration, DNS, Group Policy, and role-based file access.

**Domain:** `homelab.local`  
**Platform:** VMware Workstation Pro  
**Systems:** `DC01` (Windows Server, Active Directory Domain Services and DNS) and `CLIENT01` (Windows 11)

> This is an isolated educational lab using sample accounts and private network addresses, not a production enterprise environment.

## Objectives

* Deploy a domain controller and join a Windows client to the domain.
* Organize users and resources using Active Directory Organizational Units (OUs).
* Manage access using security groups rather than individual user permissions.
* Configure NTFS and SMB share permissions for departmental folders.
* Create a department-targeted Group Policy Object (GPO).
* Validate network connectivity, name resolution, and authorized versus denied access.

## Lab Architecture

|Component|Role|Configuration|
|-|-|-|
|VMware Workstation Pro|Virtualization|NAT virtual network|
|DC01|Domain controller and DNS server|`192.168.147.10` (static)|
|CLIENT01|Domain-joined workstation|Windows 11; DHCP-assigned address|
|Active Directory domain|Identity management|`homelab.local`|

The Windows client uses DC01 for DNS name resolution. The lab is hosted on a private VMware NAT network.

![Domain Connectivity](./screenshots/01-domain-connectivity.png)

*Validation:* `whoami` returned `homelab\\sjohnson`, `nslookup DC01.homelab.local` resolved to `192.168.147.10`, and ping returned four replies with no packet loss. The DNS lookup showed an initial timeout, but the requested name ultimately resolved.

## Active Directory Organization

Created a `Homelab Technologies` OU with sub-OUs for computers, groups, servers, and users. The Users OU contains departmental OUs for Executive, Finance, HR, IT, and Sales.

![Active Directory OU Structure](./screenshots/02-Active-Directory-OU-Structure.png)

## Security Groups and Role-Based Access

Created `IT\_Admins` and `Finance Users` security groups to manage access to department resources. Verified that the sample IT user Sarah Johnson belongs to `IT\_Admins`.

![Active Directory Security Groups](./screenshots/03-Active-Directory-Security-Groups.png)

![IT Admins Group Membership](./screenshots/04-IT-Admins-Group-Membership.png)

## Shared Folder Permissions

Configured departmental shared folders on DC01. For the IT folder, the `IT\_Admins` group has NTFS **Modify** permission; the SMB share grants **Everyone: Full Control**, leaving more granular restrictions to NTFS. The Finance folder grants the `Finance Users` group NTFS **Modify** permission.

When accessing files over SMB, both share and NTFS permissions apply. Effective access also depends on other applicable permissions and group memberships, so these screenshots demonstrate the configured group entries rather than proving exclusive access for every possible account.

![IT Shared NTFS Permissions](./screenshots/05-IT-Shared-NTFS-Permissions.png)

![IT Shared Share Permissions](./screenshots/06-IT-Shared-Share-Permissions.png)


## Group Policy

Created the **Finance - 10 Minute Screen Lock** GPO and linked it to the Finance OU. Configured these user settings under **Administrative Templates → Control Panel/Personalization**:

|Setting|Value|
|-|-|
|Enable screen saver|Enabled|
|Password protect the screen saver|Enabled|
|Screen saver timeout|600 seconds|

The configuration is intended to protect unattended sessions. The screenshots establish the GPO's link and settings; client-side application should be separately validated with a Finance user using `gpresult /r` or an equivalent test.

![Finance Group Policy Link](./screenshots/07-Finance-GPO-Link.png)

![Finance Group Policy Settings](./screenshots/08-Finance-GPO-Settings.png)

## Validation and Access Tests

* Confirmed a domain user session on CLIENT01 with `whoami`.
* Resolved the domain controller hostname with `nslookup`.
* Verified IP connectivity to DC01 using `ping`.
* Successfully opened `\\\\DC01\\IT Shared` as the sample IT user Sarah Johnson.
* Confirmed the same IT user received a permissions error when attempting to access `\\\\DC01\\Finance Shared`.

![Finance NTFS Permissions](./screenshots/09-Finance-NTFS-Permissions.png)

![Authorized IT Shared Folder Access](./screenshots/10-IT-Shared-Access-Test.png)

![Unauthorized Finance Access Denied](./screenshots/11-Unauthorized-Finance-Access-Denied.png)

## Troubleshooting & Lessons Learned

Throughout this project, I encountered technical challenges that helped me better understand how Active Directory, Windows permissions, and networking work together. I also completed simulated help desk troubleshooting scenarios to practice identifying problems, investigating their causes, and applying appropriate solutions.

### Troubleshooting NTFS Permission Inheritance

One of the most memorable challenges I encountered involved configuring NTFS permissions for departmental shared folders.

While testing folder access, I discovered that permissions inherited from the parent `C:\` directory were affecting the access restrictions I intended to implement.

To investigate the issue, I opened the folder's Advanced Security Settings and used the Effective Access tab to examine the permissions assigned to specific domain users.

After identifying the inherited permissions, I disabled inheritance, reviewed the permission entries, and adjusted the folder's access controls to better enforce departmental restrictions.

I then tested access using domain accounts to verify that users could access their authorized resources without receiving unnecessary permissions.

**What I learned:** This experience helped me understand how inherited permissions and security group membership affect NTFS access. It also reinforced the importance of verifying effective permissions rather than assuming a folder is configured correctly.

### Simulated Help Desk Ticket: DNS Resolution Failure

During one of my troubleshooting exercises, I investigated a scenario where CLIENT01 was experiencing issues communicating with the Active Directory domain.

The interesting part was that CLIENT01 could successfully reach the domain controller (DC01) using its IP address, but it could not resolve the server's hostname.

**Troubleshooting Process:**

1. Used `ping` to test connectivity between CLIENT01 and DC01.
2. Confirmed that the domain controller was reachable using its IP address (`192.168.147.10`).
3. Used `ipconfig /all` to examine CLIENT01's network configuration.
4. Identified that CLIENT01 was using an incorrect DNS server address.
5. Corrected the preferred DNS server to point to DC01 (`192.168.147.10`).
6. Retested hostname resolution and connectivity to verify that the issue was resolved.

**Root Cause:** CLIENT01 was not using the correct DNS server for the Active Directory environment.

**What I Learned:** This exercise helped me understand the difference between basic network connectivity and DNS name resolution. I learned that a computer can communicate with another device through its IP address while still being unable to resolve its hostname.

It also reinforced how important DNS is to Active Directory because domain-joined computers rely on DNS to locate domain controllers and other domain services.

### Simulated Help Desk Ticket: Group Policy Not Applying

During a simulated help desk troubleshooting exercise, I investigated why a Group Policy Object (GPO) was not applying to a user in the Finance department.

The GPO, named `Finance - 10 Minute Screen Lock`, was designed to enforce a 10-minute screen saver timeout and require a password when returning to the computer.

Although the policy was linked to the Finance organizational unit (OU), the intended user was not receiving the policy.

**Troubleshooting Process:**

1. Used `gpresult /r` to investigate which Group Policies were being applied to the affected user.
2. Identified the message `Filtering: Denied (Security)`, indicating that the policy was being blocked by security filtering.
3. Opened Group Policy Management on DC01 and located the Finance screen-lock GPO.
4. Examined the GPO's Scope tab and discovered that security filtering was configured for `IT_Admins` instead of `Finance Users`.
5. Corrected the security filtering to target the appropriate Finance security group.
6. Reviewed the use of `gpupdate /force` and `gpresult /r` to refresh policies and verify their application.

**Root Cause:** The GPO was incorrectly configured to target `IT_Admins`, preventing the intended Finance user from applying the policy.

**What I Learned:** This exercise helped me understand that linking a GPO to an organizational unit does not automatically guarantee that every user within that OU will receive the policy.

I also learned how to use `gpresult` to investigate Group Policy application issues and how security filtering affects which users and computers receive specific policies.

## Tools and Concepts

**Windows Server · Active Directory Domain Services · DNS · Windows 11 · VMware Workstation Pro · Group Policy Management · Active Directory Users and Computers · NTFS · SMB · RBAC · Least Privilege · `whoami` · `ipconfig` · `nslookup` · `ping` · `gpresult`**

## Future Improvements

* Verify Finance GPO application from a Finance user session and capture client-side evidence.
* Test access to Finance Shared using a Finance account.
* Review all applicable NTFS entries and effective access for additional user scenarios.
* Add a network diagram to illustrate the lab's network topology.

\---

*All names and addresses shown are from an isolated practice environment.*

