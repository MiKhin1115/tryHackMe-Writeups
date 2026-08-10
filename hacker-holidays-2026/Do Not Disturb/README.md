# Do Not Disturb

## Category

Boot2Root

## Points

90

## Difficulty

Medium

---

## Overview

Do Not Disturb is a medium-difficulty boot2root challenge from TryHackMe Hacker Holidays. The room focuses on a poolside platform that tracks guest sessions, cabanas, and sunbeds.

The briefing suggests that another attacker has already accessed the system before us. The goal is to follow their footprints, hijack a warm session, obtain the user flag, and then escalate privileges by abusing unsafe raw disk access.

---

## Challenge Description

The Byte Lotus platform tracks poolside activity, including cabanas, sunbeds, and active sessions. The challenge description hints that a session was left active and another person was able to sit down in it.

Important clues include:

```text
a session goes warm on a sunbed
a stranger sits down in it
a wallet signs a transaction its owner didn't authorise
a shell on the beach answers back
follow his footprints in
```

These clues point toward:

- Session hijacking
- Weak session ownership checks
- Existing attacker activity
- Web application abuse
- Local privilege escalation

---

## Room Access

```text
http://MACHINE_IP
```

---

## Objectives

The objectives are:

1. Gain access to the target through the web application.
2. Find the user flag.
3. Escalate privileges.
4. Find the root flag.

---

## Initial Enumeration

I started by scanning the target machine.

```bash
nmap -sC -sV -oN nmap.txt MACHINE_IP
```

The scan showed a web service running on the target. I opened the website in a browser and explored the poolside platform.

Since the challenge heavily mentioned sessions, I focused on:

- Cookies
- Session values
- Active guest sessions
- Cabana or sunbed reservations
- Warm or stale sessions
- User-specific pages
- Exposed session data
- Logs or traces from previous activity

---

## Web Application Analysis

The web application appeared to manage poolside resources such as sunbeds or cabanas. The challenge wording suggested that some sessions remained active after the original user left.

This means the application may not have been properly expiring sessions or verifying that a session belonged to the current user.

The main behavior to look for was whether a session could be reused, guessed, stolen, or accessed without proper authorization.

---

## Key Finding 1: Warm Session Hijacking

The application had weak session handling. A session that should have belonged to another user could be reused by someone else.

This is a session hijacking issue. The application failed to properly bind the session to the correct user and did not enforce strong ownership checks.

Possible causes include:

- Predictable session identifiers
- Exposed session tokens
- Session IDs stored in insecure places
- Stale sessions that remained valid
- Missing authorization checks on session-specific pages

---

## Exploitation Summary

The general exploitation process was:

1. Enumerate the web application.
2. Identify session-related functionality.
3. Find evidence of another active or warm session.
4. Reuse or hijack the session.
5. Access another user’s protected area.
6. Retrieve the user flag.

The important point is that the application trusted the session state without properly confirming that the current user was the legitimate owner.

---

## Getting Shell Access

After accessing the protected area, I continued looking for a way to gain shell access on the target.

Once command execution or a shell was obtained, I stabilized the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
export TERM=xterm
```

Then I checked the current user and environment.

```bash
whoami
id
hostname
pwd
```

---

## Local Enumeration

After gaining a shell, I performed local privilege escalation enumeration.

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
lsblk
```

```bash
mount
```

```bash
df -h
```

The local enumeration focused on permissions, groups, mounted filesystems, and block devices.

---

## Key Finding 2: Raw Disk Access

During enumeration, I found that the compromised user had access to raw disk devices or was part of a group that could read block devices.

This is a serious Linux privilege issue.

If a user can read raw disk devices, they may be able to bypass normal file permissions. Even if `/root/root.txt` is protected by filesystem permissions, the user may still be able to read the underlying disk data directly.

Common indicators of this issue include:

```bash
id
```

showing membership in a privileged group such as:

```text
disk
```

or readable block devices such as:

```text
/dev/sda
/dev/sda1
/dev/vda
/dev/vda1
```

---

## Why Raw Disk Access Is Dangerous

Linux file permissions protect files through the mounted filesystem. However, raw disk access allows a user to read the filesystem data at the block device level.

This can allow a low-privileged user to:

- Read root-owned files
- Extract sensitive configuration files
- Recover password hashes
- Access SSH keys
- Bypass normal permission checks

In many cases, membership in the `disk` group is effectively equivalent to root-level read access.

---

## Privilege Escalation Method

First, I identified the available block devices.

```bash
lsblk
```

Then I checked permissions on the devices.

```bash
ls -la /dev/sd* /dev/vd* 2>/dev/null
```

If the root filesystem partition was readable, it could be inspected using tools such as `debugfs`.

Example:

```bash
debugfs /dev/sda1
```

Inside `debugfs`, files can be accessed directly:

```text
cat /root/root.txt
```

If mounting was allowed, another approach would be to mount the partition read-only.

```bash
mkdir /tmp/disk
mount -o ro /dev/sda1 /tmp/disk
cat /tmp/disk/root/root.txt
```

The key issue was not needing normal root permissions because raw disk access bypassed the normal file permission checks.

---

## Flags

### User Flag

```text
THM{w4rm_s3ss10n_h1j4ck3d}
```

### Root Flag

```text
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

---

## Vulnerability Explanation

This room involved two main weaknesses.

### 1. Weak Session Management

The web application did not properly protect active sessions. A session belonging to one user could be reused by another user. This allowed unauthorized access to protected content.

### 2. Unsafe Raw Disk Access

The compromised user had excessive access to raw disk devices. This allowed reading sensitive files directly from the filesystem, bypassing normal Linux permissions.

---

## Root Cause

The root cause was a combination of poor web session security and weak local privilege separation.

The application failed to verify session ownership correctly. On the system side, the user had more access to disk devices than a normal low-privileged user should have.

---

## Impact

An attacker could:

- Hijack another guest’s active session.
- Access protected application data.
- Obtain a foothold on the system.
- Enumerate local permissions.
- Read raw disk contents.
- Bypass file permissions.
- Access root-owned files.

---

## Lessons Learned

- Sessions must be unpredictable and securely protected.
- Applications must verify session ownership on every request.
- Stale sessions should expire quickly.
- Sensitive actions should require re-authentication.
- Normal users should not have raw disk access.
- The `disk` group can be extremely dangerous.
- File permissions are not enough if the attacker can read the underlying block device.

---

## Mitigation

To prevent this issue:

1. Use secure, random session tokens.
2. Bind sessions to authenticated users.
3. Expire inactive sessions quickly.
4. Regenerate session IDs after login or privilege changes.
5. Enforce authorization checks on every protected request.
6. Do not expose session identifiers in URLs, logs, or client-side code.
7. Remove normal users from privileged groups such as `disk`.
8. Restrict access to block devices.
9. Monitor suspicious reads from raw disk devices.
10. Apply least privilege to all local user accounts.

---

## Conclusion

Do Not Disturb demonstrated how weak session handling can lead to unauthorized access through warm session hijacking. After obtaining access through the web application, local enumeration revealed excessive raw disk permissions.

By abusing raw disk access, it was possible to bypass normal filesystem protections and read root-owned data. The room highlights the importance of secure session management and strict local privilege separation.
