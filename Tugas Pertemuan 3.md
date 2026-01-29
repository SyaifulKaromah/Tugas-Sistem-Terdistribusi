# **Tugas Pertemuan 3**

**Nama**    : M. Syaiful Karomah\
**NIM**     : 09011282328111\
**Kelas**   : SK6C

---

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
