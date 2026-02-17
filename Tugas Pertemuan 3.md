[**(←) Back**](https://github.com/SyaifulKaromah/Tugas-Sistem-Terdistribusi/blob/main/README.md)
# **Tugas Pertemuan 3 - Persiapan Lingkungan Web Linux via SSH dan Pembuatan Domain via Webmin**

**Nama**    : M. Syaiful Karomah\
**NIM**     : 09011282328111\
**Kelas**   : SK6C

---
# **Bagian 1 - Persiapan Lingkungan Web Linux via SSH**
## 1. Aktifkan SSH Server di AlmaLinux
```
sudo systemctl start sshd
```
<img width="1152" height="445" alt="image" src="https://github.com/user-attachments/assets/ea575c6b-c0d3-40bc-8a0a-dacdca9d5be6" />

---

## 2. Jalankan SSH Server di Windows CMD dan Lakukan Update
```
ssh -p [port] [username]@[IP]
```
<img width="1115" height="989" alt="image" src="https://github.com/user-attachments/assets/9ceae0d9-af2c-4c34-a95f-5da524acf6aa" />

---

## 3. Install Paket Dasar
- Apache (httpd di Almalinux):
  ```
  sudo dnf install httpd -y
  ```

  <img width="1115" height="856" alt="image" src="https://github.com/user-attachments/assets/233929c3-94e8-4031-8cdb-8a2637dd6ec8" />

- MySQL (mariadb di Almalinux):
  ```
  sudo dnf install mariadb mariadb-server -y
  ```

  <img width="1115" height="951" alt="image" src="https://github.com/user-attachments/assets/a94b8de9-97ca-4cfa-aee8-53901e1476db" />

- PHP (Dukungan PHP untuk web):
  ```
  sudo dnf install php php-mysqlnd -y
  ```

  <img width="1115" height="837" alt="image" src="https://github.com/user-attachments/assets/0aaa2543-61d7-4f22-bc55-53cec8791d72" />


---

## 4. Start & Enable Service
- Start dan Enable
  ```
  sudo systemctl start httpd
  ```
  ```
  sudo systemctl enable httpd
  ```
  ```
  sudo systemctl start mariadb
  ```
  ```
  sudo systemctl enable mariadb
  ```

  <img width="1115" height="248" alt="image" src="https://github.com/user-attachments/assets/e83f8e2d-289c-49d0-81b9-8a9111d0a7f1" />

- Cek Status
  ```
  sudo systemctl status httpd
  ```
  ```
  sudo systemctl status mariadb
  ```

  <img width="1115" height="1055" alt="image" src="https://github.com/user-attachments/assets/f31a50bb-d780-4db9-837d-28afa948aaf9" />
  
---

## 5. Konfigurasi MariaDB (Secure Installation)
```
sudo mysql_secure_installation
```
- Tekan Enter kalau belum ada password root
- Set password root baru
- Hapus user anonim
- Disable remote root login
- Hapus database test
- Reload privileges

<img width="1115" height="1179" alt="image" src="https://github.com/user-attachments/assets/d071a14f-7baf-4d0c-ad6f-a3268f311a2c" />

---

## 6. Upload file web (via WinSCP)
- Login dengan protokol SFTP

  <img width="646" height="421" alt="image" src="https://github.com/user-attachments/assets/8bfd2fc5-8717-4d7a-8be9-699abf642fc9" />

- Upload folder template ke /home/capuchino/ dulu (karena /var/www/html/ butuh root). Contoh: ```komando```

  <img width="1077" height="689" alt="image" src="https://github.com/user-attachments/assets/1a5dc01b-3638-4579-b438-fc0c6824f781" />

---

## 7. Pindahkan ke Direktori Apache (Windows CMD via SSH)
```
sudo mv ~/komando /var/www/html/
```

<img width="1115" height="115" alt="image" src="https://github.com/user-attachments/assets/bcfeaf1c-e03b-438e-8da9-5675e44c40b8" />

---

## 8. Atur Permission & SELinux Context
```
sudo chown -R apacha:apache /var/www/html/komando
```
```
sudo chown -R 755 /var/www/html/komando
```
```
sudo chcon -Rt httpd_sys_content_t /var/www/html/komando
```
<img width="1115" height="134" alt="image" src="https://github.com/user-attachments/assets/d183dd57-809b-44ca-9ca2-db3d59d9df35" />

---

## 9. Buka Firewall untuk HTTP
```
sudo firewall-cmd --permanent --add-server=http
```
```
sudo firewall-cmd --reload
```

---

## 10. Restart Apache
```
sudo systemctl restart httpd
```

---

## 11. Buat Database web (bila perlu)

  ### 1. Login MariaDB
  ```
  mysql -u root -p
  ```
  
  <img width="1112" height="267" alt="image" src="https://github.com/user-attachments/assets/4ab4fd0a-6433-422d-80f0-87d0f4cbb248" />

  ### 2. Buat database baru
  ```
  CREATE DATABASE webdb;
  ```
  
  <img width="1115" height="153" alt="image" src="https://github.com/user-attachments/assets/2e7d293b-2018-4a9a-8dee-5d364761d76a" />

  lalu import db (jika ada)
  ```
  mysql -u root -p webdb < /home/capuchino/webku.sql
  ```

  <img width="1115" height="115" alt="image" src="https://github.com/user-attachments/assets/736dba85-6c4c-498f-bd2b-6b2bf1fe6eac" />
    
  → Hasilnya: semua tabel dari file webku.sql akan masuk ke database webdb.

  
  ### 3. Buat User dan Beri Hak Akses
  ```
  CREATE USER 'webuser'@'localhost' IDENTIFIED BY 'passworduser';
  ```
  ```
  GRANT ALL PRIVILEGES ON webdb.* TO 'webuser'@'localhost';
  ```
  ```
  FLUSH PRIVILEGES;
  ```

  <img width="1115" height="245" alt="image" src="https://github.com/user-attachments/assets/f1a61d72-cbd1-479e-a238-f256a7a8fc99" />


---
## 12. Akses di Browser
```
http://192.168.23.128/komando/landingpage.html
```

<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/8154cd1e-b19d-473a-ac71-78ae56109e68" />

# **Bagian 2 - Pembuatan Domain via Webmin**
## 1. Setup
```
curl -o webmin-setup-repo.sh https://raw.githubusercontent.com/webmin/webmin/master/webmin-setup-repo.sh
```
```
sudo sh webmin-setup-repo.sh
```

## 2. Install
```
sudo dnf install webmin -y
```
```
sudo dnf install bind -y
```

## 4. Setting Firewall
```
sudo firewall-cmd --permanent --add-port=10000/tcp
```
```
sudo firewall-cmd --reload
```

## 5. Aktifkan Layanan Webmin dan Bind
```
sudo systemctl start webmin
```
```
sudo systemctl start named
```
```
sudo systemctl enable named
```

## 6. Refresh Module
- Login Webmin (`http://IP-SERVER:10000`)
- Pergi ke **Webmin -> Webmin Configuration -> Refresh Modules**.
- Kalau bind sudah terinstal, menu **Servers -> BIND DNS Server** akan muncul ototmatis
  <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/0740c6e3-e8b2-4251-94b0-0265b740a7f2" />

## 7. Persiapan Pembuatan Domain
### 1. Konfigurasi Jaringan
  - Konfigurasi jaringan terlebih dahulu di Webmin (`Networking -> Network Configuration`)
    <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/b41c2407-bdc6-4806-8ea0-4aef1f1baa2a" />
  
  - lalu klik interface `ens160 -> add virtual interface`
    <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/e5ec065a-1dde-46af-b379-18faf3d908ac" />
  
  - Konfigurasi Virtual Interface nya, lalu klik `Create and Aply`
    <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/c8b1957e-bf04-4fb7-961f-5b0ade009f36" />
  
  - Cek apakah virtual interface telah di tambahkan
    <img width="1115" height="817" alt="image" src="https://github.com/user-attachments/assets/d5c45eaa-ac35-4502-b626-2e09c6c9fe6b" />

### 2. Konfigurasi BIND DNS Server
  - Masuk menu `Servers -> BIND DNS Server`
    <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/b5596c09-6ec2-4de7-a8bc-9d7387dc1f93" />

  - Kemudian `Create Master Zone`
    <img width="1560" height="580" alt="image" src="https://github.com/user-attachments/assets/7428299e-c28c-4747-a8fe-2696e0954e11" />

  - Lalu `Create` dan pilih IPv4 Address
    <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/e61f30f1-9f98-41da-9264-a68673daaeda" />

  - isi ip addressnya saja, lalu `Create`
    <img width="1563" height="332" alt="image" src="https://github.com/user-attachments/assets/8d2f61a7-1802-4120-bd35-ae0663f6a4e9" />

  - Buat lagi beberapa Address Record dengan name `www`
    <img width="1561" height="542" alt="image" src="https://github.com/user-attachments/assets/4379bfb8-aba2-460b-8769-6bb407d6701a" />

  - Ulangi untuk name=[`mail`, `ftp`, `ns1`, `ns1`]
    <img width="1541" height="189" alt="image" src="https://github.com/user-attachments/assets/43f42b8d-9b71-437c-b35d-661974138a1a" />

  - Lalu `Return to Record Types`, selanjutnya pilih `Name Server` dan isi dengan konfigurasi berikut
    <img width="1560" height="497" alt="image" src="https://github.com/user-attachments/assets/1259268f-0a45-43f4-9c02-805895676359" />

  - Sekarang lanjut membuat mail server (`Return to Record Types -> Mail Server`)
    <img width="1562" height="337" alt="image" src="https://github.com/user-attachments/assets/033e5aea-0071-4608-80c7-3e8b303ae3a0" />

  - Kemudian konfigurasi `Zone Default`
    <img width="1544" height="1093" alt="image" src="https://github.com/user-attachments/assets/89de8a86-3fee-4ff8-9884-4ce0fd4e47aa" />

      - Konfigurasi `Addresses and Topology`
    <img width="1559" height="751" alt="image" src="https://github.com/user-attachments/assets/5d091a07-e612-4cce-bf48-dc159ccf6d96" />

  - `Stop Bind` lalu `Start Bind` kembali (Restart)

  - berikan akses firwall ke dns
    ```
    sudo firewall-cmd --add-service=dns --permanent
    ```
    ```
    sudo firewall-cmd --reload
    ```
    
  - Seharusnya sudah bisa di cek:
    ```
    ping [domain]
    ```
    ```
    nslookup [domain]
    ```
    <img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/b80b472b-b1a8-40a6-9586-6be9510efb3f" />
    <p align="center"><i>Uji Coba di Linux</i></p>

    <img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/9383c7bd-af1c-4194-8786-691f16960c8a" />
    <p align="center"><i>Uji Coba di Windows</i></p>
    
### 3. Konfigurasi Apache Web Server
  - Klik `Server -> Apache Web Server -> Create virtual host`
  - konfigurasi seperti berikut:
    <img width="1565" height="520" alt="image" src="https://github.com/user-attachments/assets/61ef7f3d-b281-412a-b420-a92d45bb2bb8" />

  - Lalu create now

## 8. Akses Web Menggunakan Domain yang dibuat
- Buka browser linux, masuk ke web `komando-syf.net`
  <img width="1718" height="1082" alt="image" src="https://github.com/user-attachments/assets/bcb983c2-2776-43ad-a099-609822bb317f" />
  <p align="center"><i>Uji Coba di Linux</i></p>
  
- Buka browser windows, masuk kek web `komando-syf.net`
  <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/653235d8-d782-4536-81af-a19b77f7038c" />
  <p align="center"><i>Uji Coba di Windows(error)</i></p>
  
- Karena error `DNS_PROBE_FINISHED_NXDOMAIN`, kita perlu menyetel dns browser agar menggunakan `Use current service provider`. 
  <img width="874" height="291" alt="image" src="https://github.com/user-attachments/assets/86b3ba7c-f29d-4a97-8bfb-1a65877bdd8e" />
  <p align="center"><i>Tujuannya agar dia bisa mengakses dns yang sudah di buat sebelumnya, bukan menggunakan dns seperti cloudfire ataupun dns google.</i></p>

- Kemudian coba lagi:
  <img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/43763f41-6313-4ef1-b16b-b83c1943fefb" />
  <p align="center"><i>Uji Coba di Windows</i></p>
