[**(←) Back**](https://github.com/SyaifulKaromah/Tugas-Sistem-Terdistribusi/blob/main/README.md)

# **Tugas Pertemuan 2 - Konfigurasi SSH Server di AlmaLinux**

**Nama**    : M. Syaiful Karomah\
**NIM**     : 09011282328111\
**Kelas**   : SK6C

---
# **1. Pembuatan Script DIG**

  Script ini digunakan untuk menampilkan informasi DNS suatu domain secara terstruktur dengan fokus pada:
  
  * Status domain (**NOERROR / NXDOMAIN / SERVFAIL**) sebagai validasi domain
  * Alamat IP:
  
    * **IPv4 (A Record)**
    * **IPv6 (AAAA Record)**
  * Jumlah IP untuk analisis:
  
    * single server
    * multiple server (load balancing / CDN)


### **Langkah-langkah:**

### **1. Membuat file script**

  ```bash
  nano dig_full.sh
  ```

### **2. Mengisi script**

  ```bash
  #!/bin/bash
  # file: dig_full.sh
  # Usage: ./dig_full.sh domain.com
  
  # cek kalau argumen kosong
  if [ -z "$1" ]; then
      echo "Usage: $0 <domain>"
      exit 1  
  fi
  
  DOMAIN="$1"
  
  (
  # Ambil header (termasuk status domain)
  dig "$DOMAIN" ANY +tcp | sed '/ANSWER SECTION/,$d'
  
  # Section ANSWER (difokuskan pada IP)
  echo ";; ANSWER SECTION:"
  
  dig "$DOMAIN" A +noall +answer +tcp
  dig "$DOMAIN" AAAA +noall +answer +tcp
  
  # Footer (ringkasan query)
  echo ""
  dig "$DOMAIN" ANY +tcp | tail -n 4
  )
  ```

### **3. Memberikan izin eksekusi**

  ```bash
  chmod +x dig_full.sh
  ```

### **4. Menjalankan script**

  ```bash
  ./dig_full.sh [domain]
  ```

# **Penjelasan Singkat Script**

  * `dig ANY` → digunakan untuk mengambil informasi umum termasuk **status domain**
  * `sed '/ANSWER SECTION/,$d'` → mengambil bagian header saja (tanpa isi record)
  * `A Record` → menampilkan alamat **IPv4**
  * `AAAA Record` → menampilkan alamat **IPv6**
  * `tail -n 4` → menampilkan bagian akhir sebagai **ringkasan query**

# **Tujuan Penggunaan Script**

  Script ini dirancang untuk:
  
  * Membuktikan apakah domain **valid atau tidak**
  * Mengidentifikasi jumlah IP:
  
    * **Single IP → server tunggal**
    * **Multiple IP → load balancing / CDN**
  * Mempermudah analisis hasil DNS untuk kebutuhan laporan

---

# **Contoh Kasus:**

# **A. Domain Valid (NOERROR & Berhasil Resolve)**

## **A.1 Multiple IPv4**

### **Contoh 1: detik.com**

<img width="1025" height="507" alt="image" src="https://github.com/user-attachments/assets/2614b5d2-ee9b-4680-a39d-05247a04d7da" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Pemetaan IP:**

| No | IP Address      |
| -- | --------------- |
| 1  | 203.190.242.211 |
| 2  | 103.49.221.211  |

### **Analisis:**

* Terdapat lebih dari satu IPv4
* Menggunakan **multi-server**
* Indikasi **load balancing / CDN**

### **Contoh 2: kompas.com**

<img width="1026" height="546" alt="image" src="https://github.com/user-attachments/assets/d7428b1c-3be8-497b-8223-73e72a0e3e3c" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Pemetaan IP:**

| No | IP Address |
| -- | ---------- |
| 1  | 13.35.1.33 |
| 2  | 13.35.1.20 |
| 3  | 13.35.1.60 |
| 4  | 13.35.1.68 |

### **Analisis:**

* Menggunakan banyak IPv4
* Distribusi trafik ke beberapa server
* Infrastruktur terdistribusi

## **A.2 Multiple IPv4 & IPv6**

### **Contoh 1: google.com**

<img width="1018" height="660" alt="image" src="https://github.com/user-attachments/assets/d83f0c32-b666-42f1-b1f9-a1b0e03452c7" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Analisis:**

* Banyak IPv4 dan IPv6
* Infrastruktur global skala besar
* Menggunakan:

  * **Anycast**
  * **CDN**

### **Contoh 2: cloudflare.com**

<img width="1017" height="546" alt="image" src="https://github.com/user-attachments/assets/59c6c2ba-5174-4342-9c2b-3029b0812903" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Analisis:**

* Multi IPv4 & IPv6
* Fokus pada **CDN & security layer**

### **Contoh 3: example.com**

<img width="1025" height="544" alt="image" src="https://github.com/user-attachments/assets/b4c26efe-c44f-4eae-904c-aa8a82d5f2af" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Analisis:**

* Menggunakan IPv4 & IPv6
* Domain standar tetapi memakai **CDN modern**


## **A.3 Single IPv4 & IPv6**

### **Contoh: unsri.ac.id**

<img width="1017" height="510" alt="image" src="https://github.com/user-attachments/assets/22faebe5-4338-49a4-afb8-cb5fcad14e22" />

### **Validasi Domain:**

* Status: **NOERROR**

### **Pemetaan IP:**

| No | IP Address        |
| -- | ----------------- |
| 1  | 103.121.159.3     |
| 2  | 2001:df1:7000::a2 |

### **Analisis:**

* Hanya 1 IPv4 & 1 IPv6
* Infrastruktur sederhana
* Tidak menggunakan load balancing kompleks

## **A.4 Domain Valid Tanpa IP Address**

### **Contoh: ethereum.xyz**

<img width="1025" height="472" alt="image" src="https://github.com/user-attachments/assets/2751be90-1904-4e96-a551-ca35beaf58f6" />

### **Validasi Domain:**

* Status: **NOERROR**
* Artinya:

  * Domain **berhasil di-resolve oleh DNS**
  * Domain **terdaftar secara valid dalam sistem DNS global**

### **Pemetaan IP (Mapping Server):**

* Tidak terdapat A record (IPv4)
* Tidak terdapat AAAA record (IPv6)

### **Analisis Pemetaan:**

* Meskipun domain valid, tidak ditemukan alamat IP

* Hal ini menunjukkan:

  * Domain **tidak digunakan untuk layanan web (HTTP/HTTPS)**

* Kemungkinan penggunaan:

  * Hanya memiliki record lain seperti:

    * TXT (verifikasi domain)
    * MX (email)
  * Domain dalam kondisi **parkir / belum dikonfigurasi**
  * Digunakan untuk kebutuhan non-web

---

# **B. Validasi Tambahan DNS (SERVFAIL → NOERROR)**

## **Contoh 1: github.com**

<img width="1022" height="963" alt="image" src="https://github.com/user-attachments/assets/52487468-09d1-4e61-9670-261252c188ec" />

### **Validasi Domain:**

* Status awal: **SERVFAIL**
* Status setelah retry: **NOERROR**

### **Pemetaan IP (Mapping Server):**

| No | IP Address     | Keterangan |
| -- | -------------- | ---------- |
| 1  | 20.205.243.166 | Server 1   |

### **Analisis:**

* Percobaan pertama gagal (**SERVFAIL**)

* Setelah retry, domain berhasil menghasilkan IP

* Menunjukkan:

  * Gangguan sementara pada DNS
  * Resolver membutuhkan retry

## **Contoh 2: wikipedia.org**

<img width="1019" height="969" alt="image" src="https://github.com/user-attachments/assets/5a0194e8-c13a-43f5-9c5b-1c4d92b8a8f6" />

### **Validasi Domain:**

* Status awal: **SERVFAIL**
* Status setelah retry: **NOERROR**

### **Pemetaan IP (Mapping Server):**

| No | IP Address            | Keterangan      |
| -- | --------------------- | --------------- |
| 1  | 103.102.166.224       | Server 1 (IPv4) |
| 2  | 2001:df2:e500:ed1a::1 | Server 2 (IPv6) |

### **Analisis:**

* Awalnya tidak ada IP (**SERVFAIL**)

* Setelah retry:

  * Muncul **IPv4 dan IPv6**

* Menunjukkan:

  * Error DNS bersifat sementara
  * Sistem DNS tetap reliable setelah retry

---

# **C. Domain Tidak Ada (NXDOMAIN)**

## **Contoh: randomdomain12345.com**

<img width="1015" height="612" alt="image" src="https://github.com/user-attachments/assets/c7865cb8-94a3-4229-8338-ce2fbc0ff699" />

### **Validasi Domain:**

* Status: **NXDOMAIN**

### **Analisis:**

* Domain tidak terdaftar
* Tidak memiliki IP
* Tidak dapat di-resolve oleh DNS

