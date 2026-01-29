[**(←) Back**](https://github.com/SyaifulKaromah/Tugas-Sistem-Terdistribusi/blob/main/README.md)
# **Tugas Pertemuan 2**

**Nama**    : M. Syaiful Karomah\
**NIM**     : 09011282328111\
**Kelas**   : SK6C

---

## 1. Konfigurasi SSH Server di AlmaLinux

<img width="1150" height="1024" alt="Cuplikan layar 2026-01-25 110017" src="https://github.com/user-attachments/assets/6daa1298-c7a0-4473-b587-ec1526c27a8f" />

### 1.1 Update Sistem

```bash
sudo dnf update -y
```

### 1.2 Install OpenSSH Server

```bash
sudo dnf install -y openssh-server
```

### 1.3 Aktifkan dan Jalankan Service SSH

```bash
sudo systemctl enable --now sshd
```

### 1.4 Cek Status Service SSH

```bash
sudo systemctl status sshd
```

---

## 2. Mengganti Port SSH

### 2.1 Edit Konfigurasi SSH

```bash
sudo nano /etc/ssh/sshd_config
```

* Cari baris:

  ```
  #Port 22
  ```
  <img width="1151" height="1025" alt="Cuplikan layar 2026-01-25 110258" src="https://github.com/user-attachments/assets/1b6b1c69-c0d9-488f-b23b-a6fa4bda3e93" />

* Ubah menjadi port yang diinginkan, contoh:

  ```
  Port 2005
  ```
  <img width="1160" height="1023" alt="Cuplikan layar 2026-01-25 110326" src="https://github.com/user-attachments/assets/d950d5d6-bd3c-40ee-bbd9-fd84f63f80a1" />

---

## 3. Konfigurasi Firewall (Wajib)

<img width="1152" height="169" alt="1-Cuplikan layar 2026-01-25 112315" src="https://github.com/user-attachments/assets/6ff8a347-897f-4289-a648-6dc1c0681657" />

### 3.1 Buka Port SSH Baru

```bash
sudo firewall-cmd --permanent --add-port=2005/tcp
```

### 3.2 Reload Firewall

```bash
sudo firewall-cmd --reload
```

---

## 4. Konfigurasi SELinux (Wajib di AlmaLinux)

<img width="1152" height="174" alt="2-Cuplikan layar 2026-01-25 112315" src="https://github.com/user-attachments/assets/a78e1d62-0489-40f6-b579-f48830b6e727" />

### 4.1 Install Tools SELinux

```bash
sudo dnf install -y policycoreutils-python-utils
```

### 4.2 Tambahkan Port SSH ke SELinux

Jika port **belum pernah digunakan**:

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2005
```

Jika port **sudah pernah ditambahkan sebelumnya**:

```bash
sudo semanage port -m -t ssh_port_t -p tcp 2005
```

---

## 5. Restart Service SSH

<img width="1152" height="387" alt="3-Cuplikan layar 2026-01-25 112315" src="https://github.com/user-attachments/assets/f0678f31-b7c7-4e90-9851-420505801303" />

```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

---

## 6. Koneksi SSH dari Windows

### 6.1 Menggunakan Port Default (22)

```bash
ssh username@ip_address
```

<img width="1116" height="631" alt="Cuplikan layar 2026-01-25 110130" src="https://github.com/user-attachments/assets/e15df844-0918-4b61-b8e4-5beb5bde82ea" />

### 6.2 Menggunakan Port Custom

```bash
ssh -p 2005 username@ip_address
```

<img width="1111" height="628" alt="Cuplikan layar 2026-01-25 112419" src="https://github.com/user-attachments/assets/7a799b99-c9d4-4cc7-af0a-9c81dd8332b2" />


---

## 7. NMAP (Network Scanning)

<img width="1158" height="914" alt="Cuplikan layar 2026-01-25 115617" src="https://github.com/user-attachments/assets/f15ff9dc-f209-42ef-be24-bf053050ff85" />

<img width="1150" height="914" alt="Cuplikan layar 2026-01-25 115647" src="https://github.com/user-attachments/assets/a646ac78-e7d4-47aa-96fc-fe8b06e21631" />


### 7.1 Install NMAP

```bash
sudo dnf install -y nmap
```

### 7.2 Scan Host Aktif dalam Satu Network

```bash
nmap -Pn -sn <target>
```

**Contoh:**

```bash
nmap -Pn -sn 192.168.23.0/24
```

### 7.3 Interpretasi Hasil

```text
Nmap scan report for 192.168.23.1
Host is up.
...
Nmap scan report for 192.168.23.255
Host is up.
```

**Makna:**

* `Host is up` → perangkat aktif dan dapat dijangkau
* Tidak menampilkan port karena `-sn` hanya untuk host discovery

---
