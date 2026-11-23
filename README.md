# Active-Directory-and-Linux-Server-Integration
Project: Integrating Ubuntu Server with Windows Active Directory

**Environment:** VirtualBox Home Lab  
**Operating Systems:** Windows Server and Ubuntu Server  
**Domain:** `companyx.local`

### Objective

Integrate an Ubuntu Server into an existing Windows Active Directory environment to provide centralized identity management, authentication, and group-based access control.

The goal was to allow Active Directory users to authenticate to Linux using their domain credentials while controlling server access and administrative privileges through AD security groups.

---

## 1. Lab Environment and Architecture

### Infrastructure

| System | Operating System | IP Address | Role |
|---|---|---|---|
| DC01 | Windows Server | 192.168.1.204 | Active Directory Domain Services and DNS |
| test-serv | Ubuntu Server | 192.168.1.210 | Domain-joined Linux server |
| pfSense | pfSense | 192.168.1.1 | Network gateway and routing |

**Domain:** `companyx.local`

The Ubuntu server uses two network interfaces:

- NAT interface (`enp0s3`) for internet connectivity.
- Internal interface (`enp0s8`) for communication with the Windows domain environment.

The internal interface uses the domain controller for DNS resolution.

### Architecture Diagram

```text
                 VirtualBox Lab
                       |
                 pfSense Gateway
                  192.168.1.1
                       |
                 Internal Network
                       |
           +-----------+-----------+
           |                       |
      Windows Server          Ubuntu Server
          DC01                  test-serv
     192.168.1.204           192.168.1.210
           |                       |
      Active Directory         SSSD / realmd
           |                       |
           +------ companyx.local-+
                       |
              Centralized Identity
               and Authentication
```

---

## 2. Software and Services

The following components were installed and configured on Ubuntu:

| Component | Purpose |
|---|---|
| realmd | Domain discovery and domain membership management |
| SSSD | Active Directory identity and authentication integration |
| adcli | Active Directory domain enrollment and machine account management |
| Kerberos | Domain authentication |
| LDAP | Directory information retrieval |
| libnss-sss | Allows Linux to retrieve identities through SSSD |
| libpam-sss | Integrates SSSD with Linux authentication |
| oddjob-mkhomedir | Supports automatic home directory creation |

Installation command:

```bash
sudo apt install realmd sssd sssd-ad sssd-tools adcli \
libnss-sss libpam-sss samba-common-bin oddjob oddjob-mkhomedir
```

---

## 3. Network and DNS Configuration

Configured the Ubuntu server with a static internal address:

- IP address: `192.168.1.210/24`
- DNS server: `192.168.1.204`
- DNS search domain: `companyx.local`
- Default gateway: `192.168.1.1` through the NAT interface

The NAT interface was configured to avoid overriding the DNS server provided by the internal AD network.

### DNS Verification

Verified domain resolution:

```bash
nslookup companyx.local
```

Verified Active Directory service discovery:

```bash
nslookup -type=SRV _ldap._tcp.companyx.local
```

```bash
nslookup -type=SRV _kerberos._tcp.companyx.local
```

Verified domain controller resolution:

```bash
nslookup dc01.companyx.local
```

These tests confirmed that Ubuntu could resolve the domain controller and discover the required Active Directory services.

### Hostname Configuration

Configured the Ubuntu server's fully qualified domain name:

`test-serv.companyx.local`

Verified hostname resolution:

```bash
hostname -f
```

```bash
getent hosts test-serv.companyx.local
```

The hostname was mapped to the internal server address in `/etc/hosts`.

---

## 4. Active Directory Domain Enrollment

Used realmd to discover the domain:

```bash
sudo realm discover --verbose companyx.local
```

Joined the Ubuntu server to Active Directory:

```bash
sudo realm join --verbose --user=Administrator companyx.local
```

Verified domain membership:

```bash
realm list
```

Verified the computer account and domain trust:

```bash
sudo adcli testjoin
```

**Result:** Successfully joined Ubuntu Server to the `companyx.local` Active Directory domain.

---

## 5. SSSD Identity Integration

Configured SSSD to communicate with Active Directory and retrieve domain user and group information.

The environment uses fully qualified account names:

```ini
use_fully_qualified_names = True
```

This means domain accounts are referenced using the `@companyx.local` suffix.

### Identity Verification

Tested AD user resolution:

```bash
id username@companyx.local
```

Tested directory account retrieval:

```bash
getent passwd username@companyx.local
```

Tested AD group resolution:

```bash
getent group Linux-Server-Access@companyx.local
```

Verified the SSSD service:

```bash
systemctl status sssd
```

**Result:** Ubuntu successfully retrieved Active Directory user identities, group memberships, UID and GID information through SSSD.

---

## 6. Group-Based Access Control

Created dedicated Active Directory security groups to manage Linux server access.

| AD Security Group | Purpose |
|---|---|
| Linux-Server-Access | Controls which domain users may log in |
| Linux-Server-Admins | Identifies users authorized for administrative privileges |

This separates standard server access from administrative permissions.

### Configuring Login Restrictions

Configured realmd to permit login only for members of the designated AD group:

```bash
sudo realm permit -g 'Linux-Server-Access@companyx.local'
```

Verified the access policy:

```bash
sudo realm list
```

Expected policy:

```text
login-policy: allow-permitted-logins
permitted-groups: Linux-Server-Access@companyx.local
```

### Access Testing

Performed two authentication tests:

| Test Account | Group Membership | Result |
|---|---|---|
| brian@companyx.local | Linux-Server-Access | Login successful |
| Standard test account | Not in Linux-Server-Access | Login denied |

**Result:** Confirmed that the Ubuntu server enforced AD group-based login restrictions.

---

## 7. Active Directory-Based Sudo Privileges

Configured administrative access through the `Linux-Server-Admins` security group.

Created a dedicated sudoers configuration file:

```bash
sudo visudo -f /etc/sudoers.d/linux-server-admins
```

Configured the group using its numeric GID:

```text
%#778401143 ALL=(ALL:ALL) ALL
```

The numeric GID was obtained through Active Directory group resolution.

Validated the sudoers configuration:

```bash
sudo visudo -c
```

Corrected file permissions to the standard sudoers permissions:

```bash
sudo chmod 440 /etc/sudoers.d/linux-server-admins
```

### Privilege Testing

Tested administrative privileges using the AD administrator account:

```bash
su - brian@companyx.local
```

```bash
sudo whoami
```

Expected output:

```text
root
```

Tested the standard AD user separately.

**Results:**

- AD administrator account successfully elevated to root.
- Standard AD user could log in but was denied sudo privileges.
- Local Ubuntu administrator account remained available for recovery and maintenance.

This demonstrated centralized administrative privilege assignment through Active Directory group membership.

---

## 8. Troubleshooting and Lessons Learned

Several configuration issues were encountered and resolved during implementation.

| Issue | Resolution |
|---|---|
| Domain discovery initially failed | Corrected DNS resolution to use the AD DNS server |
| NAT interface supplied incorrect DNS | Disabled DHCP-provided DNS on the NAT interface |
| Domain enrollment could not determine domain from hostname | Corrected local FQDN resolution |
| AD group lookup initially failed | Verified fully qualified group naming and refreshed SSSD cache |
| Unauthorized user authentication failed | Verified access restrictions against AD group membership |
| AD administrator could not use sudo | Troubleshot group membership, GID matching, sudoers syntax, and file permissions |
| Sudo privileges were not recognized immediately | Refreshed SSSD cache and established a new login session |

### Key Technical Lessons

- Active Directory DNS is essential for Linux domain discovery and authentication.
- A successful domain join does not automatically guarantee that user identity resolution is working.
- SSSD provides centralized identity and authentication integration.
- NSS allows Linux to retrieve AD user and group information.
- PAM integrates centralized authentication into Linux login processes.
- AD security groups can control Linux login permissions and administrative access.
- Sudoers configuration must be validated carefully.
- Numeric GID matching can be useful when troubleshooting domain group name matching.
- Authentication and authorization are separate processes and should be tested independently.

---

## 9. Skills Demonstrated

**Windows Server / Active Directory**
- Active Directory Users and Computers
- Security group administration
- User and group membership management
- Domain services and DNS

**Linux Administration**
- Ubuntu Server
- Linux networking and DNS
- Netplan
- System services and systemd
- SSSD and realmd
- Kerberos and LDAP integration
- NSS and PAM
- Linux user and group management
- Sudo and sudoers configuration
- Authentication troubleshooting

**Infrastructure and Security**
- Cross-platform identity management
- Centralized authentication
- Role-based access control (RBAC)
- Least-privilege access design
- Group-based administrative privileges
- Authentication and authorization testing

---

## 10. Project Outcome

Successfully integrated an Ubuntu Server into a Windows Active Directory environment.

Demonstrated centralized domain identity resolution, domain authentication, group-based login restrictions, and delegated administrative privileges using Active Directory security groups.

Validated the configuration through authorized and unauthorized login tests and administrative privilege testing.

**Project status:** Completed — initial integration and access-control objectives achieved.

**Environment:** Personal virtualization lab. This project demonstrates hands-on lab experience, not production server administration.

---

## 11. Potential Future Improvements

- Configure SSH access using AD credentials.
- Implement SSH key-based authentication.
- Configure centralized Linux authentication logging.
- Integrate Linux authentication events with Wazuh.
- Implement automated configuration backups.
- Configure Windows and Linux file-sharing interoperability.
- Document server recovery procedures.
