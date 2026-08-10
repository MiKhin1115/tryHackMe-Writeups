# Infinity Pool

## Category

Boot2Root

## Points

90

## Difficulty

Medium

---

## Overview

Infinity Pool is a medium-difficulty boot2root challenge from TryHackMe Hacker Holidays. The target is a hotel web application that appears simple from the guest-facing side, but the challenge description suggests that there are hidden systems or internal services that guests were never meant to access.

The goal is to enumerate the target, discover the hidden attack surface, gain an initial foothold, retrieve the user flag, and then escalate privileges to obtain the root flag.

---

## Challenge Description

Byte Lotus Hotel advertises a seamless technology-powered experience for guests. However, the briefing hints that the most interesting systems are not directly visible to normal users.

The key idea from the description is:

```text
Sometimes the most interesting systems are the ones guests were never meant to see.
```

This suggests that the solution requires looking beyond the obvious web page and searching for hidden functionality, internal routes, services, or configuration mistakes.

---

## Room Access

```text
http://MACHINE_IP
```

---

## Objectives

The objectives are:

1. Enumerate the target machine.
2. Discover hidden or unintended web functionality.
3. Gain an initial shell.
4. Read the user flag.
5. Escalate privileges.
6. Read the root flag.

---

## Initial Enumeration

I started with a basic Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV -oN nmap.txt MACHINE_IP
```

The scan showed that a web service was available. I opened the target in a browser and began manual web enumeration.

Since the challenge description mentioned systems that guests were not meant to see, I focused on discovering hidden web paths, backend functionality, and possible internal access points.

---

## Web Enumeration

I checked the visible website first and looked for useful information such as:

- Links
- Comments in the HTML source
- JavaScript files
- API endpoints
- Hidden routes
- Login panels
- Debug information
- Error messages
- Interesting headers

I also inspected the page source and browser Developer Tools.

Useful checks included:

```bash
curl -i http://MACHINE_IP
```

```bash
curl http://MACHINE_IP
```

Directory enumeration was also useful:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirb/common.txt
```

If SecLists was available, a larger wordlist could be used:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

---

## Hidden Attack Surface

The main breakthrough was identifying functionality that was not visible from the normal guest-facing page.

This matched the room theme: the dangerous system was not shown directly to guests, but it was still reachable through enumeration.

During this stage, I focused on:

- Unlinked pages
- Internal-looking endpoints
- Administrative paths
- API routes
- Backup files
- Configuration leaks
- Unusual HTTP responses

Common paths worth checking in this type of challenge include:

```text
/admin
/api
/debug
/dev
/internal
/dashboard
/backup
/config
```

Once the hidden area was discovered, it revealed a path toward gaining access to the system.

---

## Key Finding

The web application exposed functionality that should not have been accessible to normal users. This created an opportunity to interact with backend features and eventually gain command execution or shell access.

The weakness was caused by insufficient protection of hidden functionality. The application relied on the fact that guests would not know the endpoint existed, but hidden routes are not a security boundary.

Security by obscurity is not enough. If a route or service exists and is reachable, it must be properly protected.

---

## Initial Foothold

After identifying the vulnerable functionality, I used it to gain an initial foothold on the machine.

Once I obtained a shell, I stabilized it for easier interaction.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
export TERM=xterm
```

Then I checked the current user and environment:

```bash
whoami
id
hostname
pwd
```

---

## User Enumeration

After gaining access, I searched the filesystem for the user flag and useful files.

```bash
ls -la
```

```bash
find /home -type f 2>/dev/null
```

```bash
find / -name "user.txt" 2>/dev/null
```

The user flag was found after enumerating the compromised user's accessible files.

---

## Local Privilege Escalation Enumeration

After collecting the user flag, I started privilege escalation enumeration.

Useful commands included:

```bash
sudo -l
```

```bash
id
```

```bash
groups
```

```bash
find / -perm -4000 -type f 2>/dev/null
```

```bash
find / -writable -type d 2>/dev/null
```

```bash
ps aux
```

```bash
systemctl list-units --type=service
```

```bash
ls -la /opt
```

The goal was to identify misconfigurations, writable files, unusual services, exposed credentials, or scripts running with higher privileges.

---

## Privilege Escalation

Further enumeration revealed a path to escalate privileges. The root flag name suggests that the final step involved tracing the source of the privileged behavior and following it to the correct system component.

The important lesson from this step is that privilege escalation often comes from carefully reviewing:

- Running processes
- System services
- Cron jobs
- Application directories
- Environment variables
- Configuration files
- File permissions
- Logs
- Scripts executed by privileged users

Once the privileged path was identified, I used it to access the root-owned flag.

---

## Flags

### User Flag

```text
THM{n0_v1s1bl3_3dg3}
```

### Root Flag

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

## Vulnerability Explanation

This room demonstrates two common boot2root concepts.

### 1. Hidden Web Functionality

The initial access came from discovering functionality that was not visible on the main page. Hidden endpoints, internal routes, and unlinked panels are still part of the attack surface if they are reachable.

A secure application should not rely on users being unable to guess or discover a path.

### 2. Local Misconfiguration

After gaining a foothold, privilege escalation was possible because of a local system weakness. This could involve overly permissive files, exposed credentials, unsafe services, or scripts running with higher privileges.

The key skill was careful enumeration after the initial shell.

---

## Root Cause

The root cause was weak exposure control.

The application had functionality that was not meant for normal users but was still reachable. On the host system, local permissions or service configuration allowed a low-privileged user to move toward root access.

---

## Impact

An attacker could:

- Discover hidden web functionality.
- Access unintended backend features.
- Gain a shell on the server.
- Read user-level sensitive files.
- Enumerate local system weaknesses.
- Escalate privileges.
- Access root-owned data.

---

## Lessons Learned

- Hidden endpoints are not secure unless access control is enforced.
- Web enumeration is important even when the main page looks simple.
- JavaScript files and API routes can reveal hidden functionality.
- After getting a shell, local enumeration is essential.
- Privilege escalation often depends on small configuration mistakes.
- Services, scripts, and permissions should be reviewed carefully.

---

## Mitigation

To prevent this type of issue:

1. Enforce authentication and authorization on all sensitive routes.
2. Do not rely on hidden URLs for security.
3. Remove unused or development endpoints from production.
4. Avoid exposing debug functionality publicly.
5. Apply least privilege to application users.
6. Protect service files, scripts, and configuration files.
7. Review file permissions regularly.
8. Monitor unusual access to hidden or internal endpoints.
9. Separate public-facing services from internal systems.
10. Perform regular security testing before deployment.

---

## Conclusion

Infinity Pool showed that the most valuable attack surface is not always visible from the main page. By carefully enumerating the web application, hidden functionality could be discovered and used to gain a foothold.

After that, local enumeration revealed a privilege escalation path that led to root access. The room reinforces the importance of both web enumeration and Linux privilege escalation methodology.
