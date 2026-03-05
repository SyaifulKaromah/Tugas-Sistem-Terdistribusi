[**(←) Back**](https://github.com/SyaifulKaromah/Tugas-Sistem-Terdistribusi/blob/main/README.md)

# **PROJECT UTS — Remote Server Management: CRUD Data JSON/XML, Socket Programming & SSH Security (Client-Server Linux)**

**Nama**    : M. Syaiful Karomah\
**NIM**     : 09011282328111\
**Kelas**   : SK6C

---

## Informasi Sistem

| Parameter            | Nilai                    |
|----------------------|--------------------------|
| OS                   | AlmaLinux 9 (RHEL-based) |
| SSH Username         | `capuchino`              |
| SSH Password         | `besokLibur`             |
| IP Address Server    | `192.168.10.10`          |
| IP Address Client    | `192.168.10.20`          |
| IP Address Client (Socket [x]) | `192.168.10.99`          |
| Port SSH             | `2005`                   |
| Port Socket Server   | `5000`                   |
| Format Data Exchange | JSON dan XML             |
| Library Client       | Paramiko + Socket        |

### Topologi Jaringan

```
┌──────────────────────┐          ┌──────────────────────┐
│   CLIENT NODE        │          │   SERVER NODE        │
│   192.168.10.20      │          │   192.168.10.10      │
│                      │─SSH:2005►│                      │
│  client.py           │          │  server.py           │
│  (Paramiko + Socket) │─TCP:5000►│  (Socket)            │
│                      │          │                      │
│  JSON / XML Parser   │          │  data.json, data.xml │
└──────────────────────┘          └──────────────────────┘
```

---

# A. Konfigurasi Server

> Dilakukan **langsung di mesin server** (akses fisik / console).  
> Tujuan: mengaktifkan SSH agar server bisa diremote dari client.

## A.1 Install SSH Server

```bash
sudo dnf install openssh-server -y
```

## A.2 Aktifkan SSH Service

```bash
sudo systemctl enable sshd
sudo systemctl start sshd
```

## A.3 Edit Konfigurasi SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Ubah bagian berikut:

```
Port 2005
PermitRootLogin no
PasswordAuthentication yes
AllowUsers capuchino
```

Simpan, lalu restart:

```bash
sudo systemctl restart sshd
```

## A.4 Buka Port di Firewall

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.20" port port="2005" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.99" port port="2005" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.20" port port="5000" protocol="tcp" accept'
sudo firewall-cmd --reload
```

## A.5 Verifikasi

```bash
sudo systemctl status sshd
sudo firewall-cmd --list-all

```

> Server sekarang siap diremote dari client melalui port 2005.

---

# B. Konfigurasi Server via SSH dari Client

> Dilakukan **dari mesin client**, setelah masuk SSH ke server.  
> Semua perintah di bawah dijalankan di dalam sesi SSH.

## B.1 Login SSH ke Server

```bash
ssh capuchino@192.168.10.10 -p 2005
# Password: besokLibur
```
<img width="1153" height="691" alt="image" src="https://github.com/user-attachments/assets/36b93cbc-2e97-4e57-879a-891b97b5969f" />

## B.2 Buat Folder dan File Data

```bash
mkdir -p ~/server
cd ~/server
```

Buat `data.json`:

```bash
nano data.json
```

```json
{
  "mahasiswa": {
    "nama": "M. Syaiful Karomah",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}
```

Buat `data.xml`:

```bash
nano data.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <mahasiswa>
    <nama>M. Syaiful Karomah</nama>
    <nim>09011282328111</nim>
    <kelas>SK6C</kelas>
    <matakuliah>Sistem Terdistribusi</matakuliah>
  </mahasiswa>
  <server>
    <ip>192.168.10.10</ip>
    <port_ssh>2005</port_ssh>
    <port_socket>5000</port_socket>
  </server>
</root>
```

## B.3 Buat server.py

```bash
nano server.py
```

```python
import socket
import os
import json
import xml.etree.ElementTree as ET
from datetime import datetime

HOST = '0.0.0.0'
PORT = 5000
ALLOWED_IP = '192.168.10.20'  # Filter: hanya terima koneksi dari IP client


def xml_to_dict(element):
    d = {}
    for child in element:
        if len(child) == 0:
            d[child.tag] = child.text
        else:
            d[child.tag] = xml_to_dict(child)
    return d


def handle_client(conn, addr):
    print(f"[{datetime.now()}] Koneksi dari {addr[0]}")

    # Filter IP — tolak selain client
    if addr[0] != ALLOWED_IP:
        print(f"[BLOCKED] {addr[0]}")
        conn.sendall(json.dumps({"status": "error", "message": "IP tidak diizinkan"}).encode())
        conn.close()
        return

    try:
        raw = conn.recv(4096).decode()
        req = json.loads(raw)
        action = req.get("action", "")
        print(f"[REQUEST] action={action}")

        if action == "list_files":
            files = os.listdir(".")
            conn.sendall(json.dumps({"status": "ok", "files": files}).encode())

        elif action == "get_file":
            filename = req.get("filename", "")
            if os.path.exists(filename):
                with open(filename, "rb") as f:
                    content = f.read()
                header = json.dumps({"status": "ok", "filename": filename, "size": len(content)})
                conn.sendall((header + "\n").encode())
                conn.sendall(content)
            else:
                conn.sendall(json.dumps({"status": "error", "message": f"{filename} tidak ditemukan"}).encode())

        elif action == "get_json_data":
            with open("data.json") as f:
                data = json.load(f)
            conn.sendall(json.dumps({"status": "ok", "format": "json", "data": data}).encode())

        elif action == "get_xml_data":
            tree = ET.parse("data.xml")
            data = xml_to_dict(tree.getroot())
            conn.sendall(json.dumps({"status": "ok", "format": "xml_parsed", "data": data}).encode())

        else:
            conn.sendall(json.dumps({"status": "error", "message": "action tidak dikenal"}).encode())

    except Exception as e:
        conn.sendall(json.dumps({"status": "error", "message": str(e)}).encode())
    finally:
        conn.close()


server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind((HOST, PORT))
server.listen(5)
print(f"[SERVER] Listening di port {PORT}, hanya menerima dari {ALLOWED_IP}")

while True:
    conn, addr = server.accept()
    handle_client(conn, addr)
```

## B.4 Jalankan Server

```bash
python3 server.py
```
<img width="1150" height="295" alt="image" src="https://github.com/user-attachments/assets/4478d44a-9eaa-4502-9838-8388a2ccd5cf" />

Output yang muncul:

```
[SERVER] Listening di port 5000, hanya menerima dari 192.168.10.20
```

> Biarkan terminal ini tetap berjalan. Lanjut ke bagian C di mesin client.

---

# C. Konfigurasi PC Client

> Dilakukan **di mesin client** (192.168.10.20).  
> Buka terminal baru — jangan tutup sesi SSH ke server dari bagian B.

## C.1 Install Paramiko

```bash
pip3 install paramiko
```

## C.2 Buat Folder Client

```bash
mkdir -p ~/client/hasil
cd ~/client
```

## C.3 Buat client.py

```bash
nano client.py
```

```python
import paramiko
import socket
import json
import xml.etree.ElementTree as ET

# ====== KONFIGURASI ======
SSH_HOST    = "192.168.10.10"
SSH_PORT    = 2005
SSH_USER    = "capuchino"
SSH_PASS    = "besokLibur"
SOCKET_PORT = 5000


# ====== PARAMIKO: SSH ======
def connect_ssh():
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(SSH_HOST, port=SSH_PORT, username=SSH_USER, password=SSH_PASS)
    print("[SSH] Terhubung ke server")
    return ssh

def remote_exec(ssh, command):
    _, stdout, stderr = ssh.exec_command(command)
    return stdout.read().decode().strip(), stderr.read().decode().strip()


# ====== SOCKET: DATA EXCHANGE ======
def socket_request(action, filename=""):
    req = json.dumps({"action": action, "filename": filename})
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((SSH_HOST, SOCKET_PORT))
    sock.sendall(req.encode())
    chunks = []
    while True:
        chunk = sock.recv(4096)
        if not chunk:
            break
        chunks.append(chunk)
    sock.close()
    return b"".join(chunks)


# ====== HELPER ======
def get_remote_files(ssh, ext=None):
    out, _ = remote_exec(ssh, "ls ~/server/")
    files = out.split()
    if ext:
        files = [f for f in files if f.endswith(ext)]
    return files

def pilih_file(ssh, ext=None, label="file"):
    files = get_remote_files(ssh, ext)
    if not files:
        print(f"[!] Tidak ada {label} di server.")
        return None
    print(f"\nDaftar {label} di server:")
    for i, f in enumerate(files, 1):
        print(f"  {i}. {f}")
    try:
        idx = int(input("Pilih nomor: ").strip()) - 1
        if 0 <= idx < len(files):
            return files[idx]
        print("[!] Nomor tidak valid.")
        return None
    except ValueError:
        print("[!] Input tidak valid.")
        return None

def xml_to_dict(element):
    d = {}
    for child in element:
        if len(child) == 0:
            d[child.tag] = child.text or ""
        else:
            d[child.tag] = xml_to_dict(child)
    return d

def dict_to_xml(parent, data):
    for k, v in data.items():
        el = ET.SubElement(parent, k)
        if isinstance(v, dict):
            dict_to_xml(el, v)
        else:
            el.text = str(v) if v else ""

def flatten_keys(d, prefix=""):
    keys = []
    for k, v in d.items():
        full_key = f"{prefix}.{k}" if prefix else k
        if isinstance(v, dict):
            keys.extend(flatten_keys(v, full_key))
        else:
            keys.append((full_key, v))
    return keys

def simpan_ke_server(ssh, filename, data, is_json):
    if is_json:
        content = json.dumps(data, indent=2, ensure_ascii=False)
    else:
        root_new = ET.Element("root")
        dict_to_xml(root_new, data)
        ET.indent(root_new)
        xml_str = ET.tostring(root_new, encoding="unicode")
        content = '<?xml version="1.0" encoding="UTF-8"?>\n' + xml_str
    escaped = content.replace("'", "'\\''")
    remote_exec(ssh, f"echo '{escaped}' > ~/server/{filename}")
    print(f"[SIMPAN] {filename} berhasil disimpan di server.")
    out, _ = remote_exec(ssh, f"cat ~/server/{filename}")
    print(f"[VERIFIKASI] Isi terbaru {filename}:\n{out}")


# ====== MENU 1: LIST FILE ======
def list_remote_files():
    resp = json.loads(socket_request("list_files"))
    if resp["status"] == "ok":
        print("\n[FILES] Daftar file di server:")
        for f in resp["files"]:
            print(f"  - {f}")


# ====== MENU 2: DOWNLOAD FILE ======
def get_file(ssh):
    filename = pilih_file(ssh, label="semua file")
    if not filename:
        return
    raw = socket_request("get_file", filename)
    header_bytes, _, content = raw.partition(b"\n")
    meta = json.loads(header_bytes)
    if meta["status"] == "ok":
        path = f"hasil/hasil_{filename}"
        with open(path, "wb") as f:
            f.write(content)
        print(f"[FILE] Disimpan: {path}")
    else:
        print(f"[ERROR] {meta['message']}")


# ====== MENU 3: AMBIL DATA JSON via SOCKET ======
def get_json_data():
    resp = json.loads(socket_request("get_json_data"))
    if resp["status"] == "ok":
        data = resp["data"]
        print("[JSON] Data diterima:")
        print(json.dumps(data, indent=2, ensure_ascii=False))
        with open("hasil/data_olahan.json", "w") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        print("[JSON] Disimpan: hasil/data_olahan.json")


# ====== MENU 4: AMBIL DATA XML via SOCKET ======
def get_xml_data():
    resp = json.loads(socket_request("get_xml_data"))
    if resp["status"] == "ok":
        data = resp["data"]
        print("[XML] Data diterima (parsed dari server):")
        print(json.dumps(data, indent=2, ensure_ascii=False))
        root = ET.Element("root")
        dict_to_xml(root, data)
        ET.indent(root)
        ET.ElementTree(root).write("hasil/data_rekonstruksi.xml", encoding="unicode", xml_declaration=True)
        print("[XML] Disimpan: hasil/data_rekonstruksi.xml")


# ====== MENU 5: READ & PARSE JSON/XML via PARAMIKO ======
def paramiko_read(ssh):
    files = get_remote_files(ssh, ".json") + get_remote_files(ssh, ".xml")
    if not files:
        print("[!] Tidak ada file JSON/XML di server.")
        return
    print("\nDaftar file JSON/XML di server:")
    for i, f in enumerate(files, 1):
        print(f"  {i}. {f}")
    try:
        idx = int(input("Pilih nomor: ").strip()) - 1
        if not (0 <= idx < len(files)):
            print("[!] Nomor tidak valid.")
            return
    except ValueError:
        print("[!] Input tidak valid.")
        return

    filename = files[idx]
    out, err = remote_exec(ssh, f"cat ~/server/{filename}")
    if err:
        print(f"[ERROR] {err}")
        return

    print(f"\n[PARAMIKO] Isi file {filename}:")
    if filename.endswith(".json"):
        data = json.loads(out)
        print(json.dumps(data, indent=2, ensure_ascii=False))
    elif filename.endswith(".xml"):
        root = ET.fromstring(out)
        data = xml_to_dict(root)
        print(json.dumps(data, indent=2, ensure_ascii=False))


# ====== MENU 6: EDIT / TAMBAH / HAPUS FIELD via PARAMIKO ======
def paramiko_edit(ssh):
    files = get_remote_files(ssh, ".json") + get_remote_files(ssh, ".xml")
    if not files:
        print("[!] Tidak ada file JSON/XML di server.")
        return
    print("\nDaftar file JSON/XML di server:")
    for i, f in enumerate(files, 1):
        print(f"  {i}. {f}")
    try:
        idx = int(input("Pilih nomor: ").strip()) - 1
        if not (0 <= idx < len(files)):
            print("[!] Nomor tidak valid.")
            return
    except ValueError:
        print("[!] Input tidak valid.")
        return

    filename = files[idx]
    out, err = remote_exec(ssh, f"cat ~/server/{filename}")
    if err:
        print(f"[ERROR] {err}")
        return

    is_json = filename.endswith(".json")
    if is_json:
        data = json.loads(out)
    else:
        data = xml_to_dict(ET.fromstring(out))

    print(f"\n[EDIT] File: {filename}")
    print("  1. Edit nilai field")
    print("  2. Tambah field baru")
    print("  3. Hapus field")
    aksi = input("Aksi: ").strip()

    if aksi == "1":
        flat = flatten_keys(data)
        print("\nField yang tersedia:")
        for i, (k, v) in enumerate(flat, 1):
            print(f"  {i}. {k} = {v}")
        try:
            fidx = int(input("Pilih nomor field: ").strip()) - 1
            if not (0 <= fidx < len(flat)):
                print("[!] Nomor tidak valid.")
                return
        except ValueError:
            print("[!] Input tidak valid.")
            return
        field_path = flat[fidx][0]
        nilai_baru = input(f"Nilai baru untuk '{field_path}': ").strip()
        keys = field_path.split(".")
        d = data
        for k in keys[:-1]:
            d = d[k]
        d[keys[-1]] = nilai_baru
        print(f"[EDIT] {field_path} → {nilai_baru}")

    elif aksi == "2":
        groups = [k for k, v in data.items() if isinstance(v, dict)]
        groups.append("(root)")
        print("\nTambah field ke grup:")
        for i, g in enumerate(groups, 1):
            print(f"  {i}. {g}")
        try:
            gidx = int(input("Pilih nomor: ").strip()) - 1
            if not (0 <= gidx < len(groups)):
                print("[!] Nomor tidak valid.")
                return
        except ValueError:
            print("[!] Input tidak valid.")
            return
        nama_field = input("Nama field baru: ").strip()
        nilai_field = input("Nilai field baru: ").strip()
        if groups[gidx] == "(root)":
            data[nama_field] = nilai_field
        else:
            data[groups[gidx]][nama_field] = nilai_field
        print(f"[TAMBAH] '{nama_field}' = '{nilai_field}' berhasil ditambahkan.")

    elif aksi == "3":
        flat = flatten_keys(data)
        print("\nField yang tersedia:")
        for i, (k, v) in enumerate(flat, 1):
            print(f"  {i}. {k} = {v}")
        try:
            fidx = int(input("Pilih nomor field yang ingin dihapus: ").strip()) - 1
            if not (0 <= fidx < len(flat)):
                print("[!] Nomor tidak valid.")
                return
        except ValueError:
            print("[!] Input tidak valid.")
            return
        field_path = flat[fidx][0]
        konfirmasi = input(f"[!] Yakin hapus field '{field_path}'? (y/n): ").strip().lower()
        if konfirmasi != "y":
            print("[BATAL]")
            return
        keys = field_path.split(".")
        d = data
        for k in keys[:-1]:
            d = d[k]
        del d[keys[-1]]
        print(f"[HAPUS] Field '{field_path}' berhasil dihapus.")
    else:
        print("[!] Aksi tidak valid.")
        return

    simpan_ke_server(ssh, filename, data, is_json)


# ====== MENU 7: BUAT FILE BARU JSON/XML via PARAMIKO ======
def paramiko_create(ssh):
    print("\nPilih format file:")
    print("  1. JSON")
    print("  2. XML")
    fmt = input("Format: ").strip()
    if fmt not in ("1", "2"):
        print("[!] Format tidak valid.")
        return

    nama_file = input("Nama file (tanpa ekstensi): ").strip()
    if not nama_file:
        print("[!] Nama file tidak boleh kosong.")
        return

    ext = ".json" if fmt == "1" else ".xml"
    filename = nama_file + ext

    existing = get_remote_files(ssh)
    if filename in existing:
        print(f"[!] File {filename} sudah ada di server.")
        return

    data = {}
    pakai_grup = input("Pakai grup/section? (y/n): ").strip().lower()

    if pakai_grup == "y":
        while True:
            nama_grup = input("\nNama grup (kosongkan jika selesai): ").strip()
            if not nama_grup:
                break
            data[nama_grup] = {}
            try:
                jumlah = int(input(f"Berapa field di grup '{nama_grup}'? ").strip())
            except ValueError:
                print("[!] Input tidak valid.")
                continue
            for i in range(jumlah):
                k = input(f"  Nama field {i+1}: ").strip()
                v = input(f"  Nilai field {i+1}: ").strip()
                data[nama_grup][k] = v
    else:
        try:
            jumlah = int(input("Berapa field? ").strip())
        except ValueError:
            print("[!] Input tidak valid.")
            return
        for i in range(jumlah):
            k = input(f"  Nama field {i+1}: ").strip()
            v = input(f"  Nilai field {i+1}: ").strip()
            data[k] = v

    simpan_ke_server(ssh, filename, data, fmt == "1")


# ====== MENU 8: HAPUS FILE di SERVER via PARAMIKO ======
def paramiko_delete(ssh):
    filename = pilih_file(ssh, label="semua file")
    if not filename:
        return
    konfirmasi = input(f"[!] Yakin hapus '{filename}'? (y/n): ").strip().lower()
    if konfirmasi == "y":
        remote_exec(ssh, f"rm ~/server/{filename}")
        print(f"[HAPUS] File {filename} berhasil dihapus dari server.")
    else:
        print("[BATAL] Penghapusan dibatalkan.")


# ====== MENU 9: MONITOR SERVER via PARAMIKO ======
def paramiko_monitor(ssh):
    print("\n========================================")
    print("  MONITOR SERVER")
    print("========================================")
    out, _ = remote_exec(ssh, "hostname")
    print(f"Hostname   : {out}")
    out, _ = remote_exec(ssh, "hostname -I | awk '{print $1}'")
    print(f"IP Address : {out}")
    out, _ = remote_exec(ssh, "uptime -p")
    print(f"Uptime     : {out}")
    out, _ = remote_exec(ssh, "top -bn1 | grep 'Cpu(s)' | awk '{print $2+$4\"%\"}'")
    print(f"CPU Usage  : {out}")
    out, _ = remote_exec(ssh, "free -h | awk '/^Mem:/ {print $3\"/\"$2}'")
    print(f"RAM Usage  : {out}")
    out, _ = remote_exec(ssh, "df -h / | awk 'NR==2 {print $3\"/\"$2\" (\"$5\" used)\"}'")
    print(f"Disk Usage : {out}")
    out, _ = remote_exec(ssh, "ls ~/server/ | wc -l")
    print(f"Total File : {out} file di server")
    print("========================================")


# ====== MAIN ======
def main():
    ssh = connect_ssh()

    while True:
        print("\n========================================")
        print("  MENU CLIENT")
        print("========================================")
        print("1. List file di server          (Socket)")
        print("2. Download file dari server    (Socket)")
        print("3. Ambil data JSON dari server  (Socket)")
        print("4. Ambil data XML dari server   (Socket)")
        print("5. Read & parse JSON/XML        (Paramiko)")
        print("6. Edit / tambah / hapus field  (Paramiko)")
        print("7. Buat file JSON/XML baru      (Paramiko)")
        print("8. Hapus file di server         (Paramiko)")
        print("9. Monitor server               (Paramiko)")
        print("0. Keluar")
        print("========================================")
        choice = input("Pilihan: ").strip()

        if choice == "1":
            list_remote_files()
        elif choice == "2":
            get_file(ssh)
        elif choice == "3":
            get_json_data()
        elif choice == "4":
            get_xml_data()
        elif choice == "5":
            paramiko_read(ssh)
        elif choice == "6":
            paramiko_edit(ssh)
        elif choice == "7":
            paramiko_create(ssh)
        elif choice == "8":
            paramiko_delete(ssh)
        elif choice == "9":
            paramiko_monitor(ssh)
        elif choice == "0":
            break
        else:
            print("[!] Pilihan tidak valid.")

    ssh.close()
    print("[SSH] Koneksi ditutup")


if __name__ == "__main__":
    main()
```
<img width="1158" height="637" alt="image" src="https://github.com/user-attachments/assets/40a43f0f-c9e4-448c-94c7-ccd9f13e34d6" />

---

# D. Demonstrasi Projek

> Pastikan bagian A, B, dan C sudah selesai dikonfigurasi sebelum memulai demonstrasi.

---

## D.1 Jalankan Server (Terminal 1 — via SSH)

Login SSH ke server dari mesin client:

```bash
ssh capuchino@192.168.10.10 -p 2005
```

Masuk ke folder server dan jalankan:

```bash
cd ~/server
python3 server.py
```

Output:

```
[SERVER] Listening di port 5000, hanya menerima dari 192.168.10.20
```
<img width="1150" height="294" alt="image" src="https://github.com/user-attachments/assets/84a88635-ec5b-4f4c-a3cd-e8c3bde84ee8" />

> Biarkan terminal ini tetap berjalan.

---

## D.2 Jalankan Client (Terminal 2 — di mesin client)

Buka terminal baru di mesin client:

```bash
cd ~/client
python3 client.py
```

Output awal:

```
[SSH] Terhubung ke server

========================================
  MENU CLIENT
========================================
1. List file di server          (Socket)
2. Download file dari server    (Socket)
3. Ambil data JSON dari server  (Socket)
4. Ambil data XML dari server   (Socket)
5. Read & parse JSON/XML        (Paramiko)
6. Edit / tambah / hapus field  (Paramiko)
7. Buat file JSON/XML baru      (Paramiko)
8. Hapus file di server         (Paramiko)
9. Monitor server               (Paramiko)
0. Keluar
========================================
```
<img width="1148" height="475" alt="image" src="https://github.com/user-attachments/assets/a2f8e366-292e-4bf4-8120-b0165aa8f634" />

---

## D.3 Demonstrasi Tiap Fitur

### Menu 1 — List File di Server (Socket)

```
Pilihan: 1

[FILES] Daftar file di server:
  - data.json
  - data.xml
  - server.py
```
<img width="1148" height="540" alt="image" src="https://github.com/user-attachments/assets/b56693ac-2b04-4f69-97e2-144bc80dd41c" />

---

### Menu 2 — Download File dari Server (Socket)

```
Pilihan: 2

Daftar semua file di server:
  1. data.json
  2. data.xml
  3. server.py
Pilih nomor: 1

[FILE] Disimpan: hasil/hasil_data.json
```
<img width="1149" height="585" alt="image" src="https://github.com/user-attachments/assets/0b78b7cc-e154-4338-b0eb-2a8739c5fde0" />

---

### Menu 3 — Ambil Data JSON (Socket)

```
Pilihan: 3

[JSON] Data diterima:
{
  "mahasiswa": {
    "nama": "M. Syaiful Karomah",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}
[JSON] Disimpan: hasil/data_olahan.json
```
<img width="1150" height="741" alt="image" src="https://github.com/user-attachments/assets/9772b779-1929-448b-ac4b-004cb0b513d2" />

---

### Menu 4 — Ambil Data XML (Socket)

```
Pilihan: 4

[XML] Data diterima (parsed dari server):
{
  "mahasiswa": {
    "nama": "M. Syaiful Karomah",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}
[XML] Disimpan: hasil/data_rekonstruksi.xml
```
<img width="1148" height="737" alt="image" src="https://github.com/user-attachments/assets/0cb31d04-5dc2-41b6-90f4-4af04754e6bc" />

---

### Menu 5 — Read & Parse JSON/XML (Paramiko)

```
Pilihan: 5

Daftar file JSON/XML di server:
  1. data.json
  2. data.xml
Pilih nomor: 1

[PARAMIKO] Isi file data.json:
{
  "mahasiswa": {
    "nama": "M. Syaiful Karomah",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}
```
<img width="1147" height="826" alt="image" src="https://github.com/user-attachments/assets/2a4740b7-84f8-4f42-8479-591dd25db3d7" />

---

### Menu 6 — Edit / Tambah / Hapus Field (Paramiko)

**Edit nilai field:**

```
Pilihan: 6

Daftar file JSON/XML di server:
  1. data.json
  2. data.xml
Pilih nomor: 1

[EDIT] File: data.json
  1. Edit nilai field
  2. Tambah field baru
  3. Hapus field
Aksi: 1

Field yang tersedia:
  1. mahasiswa.nama = M. Syaiful Karomah
  2. mahasiswa.nim = 09011282328111
  3. mahasiswa.kelas = SK6C
  4. mahasiswa.matakuliah = Sistem Terdistribusi
  5. server.ip = 192.168.10.10
  6. server.port_ssh = 2005
  7. server.port_socket = 5000
Pilih nomor field: 1
Nilai baru untuk 'mahasiswa.nama': Syaiful
[EDIT] mahasiswa.nama → Syaiful
[SIMPAN] data.json berhasil disimpan di server.
[VERIFIKASI] Isi terbaru data.json:
{
  "mahasiswa": {
    "nama": "Syaiful",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}

```
<img width="1145" height="293" alt="image" src="https://github.com/user-attachments/assets/738fde8d-3b08-4aaa-a577-72431812df06" />

<img width="1149" height="883" alt="image" src="https://github.com/user-attachments/assets/e86b7489-d710-4523-ba6d-a1d06dad857d" />

**Tambah field baru:**

```
[EDIT] File: data.json
  1. Edit nilai field
  2. Tambah field baru
  3. Hapus field
Aksi: 2

Tambah field ke grup:
  1. mahasiswa
  2. server
  3. (root)
Pilih nomor: 1
Nama field baru: usia
Nilai field baru: 21
[TAMBAH] 'usia' = '21' berhasil ditambahkan.
[SIMPAN] data.json berhasil disimpan di server.
[VERIFIKASI] Isi terbaru data.json:
{
  "mahasiswa": {
    "nama": "Syaiful",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi",
    "usia": "21"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}

```
<img width="1154" height="988" alt="image" src="https://github.com/user-attachments/assets/e16b244c-372c-41b9-bcf1-966dbbac33d9" />

**Hapus field:**

```
[EDIT] File: data.json
  1. Edit nilai field
  2. Tambah field baru
  3. Hapus field
Aksi: 3

Field yang tersedia:
  1. mahasiswa.nama = Syaiful
  2. mahasiswa.nim = 09011282328111
  3. mahasiswa.kelas = SK6C
  4. mahasiswa.matakuliah = Sistem Terdistribusi
  5. mahasiswa.usia = 21
  6. server.ip = 192.168.10.10
  7. server.port_ssh = 2005
  8. server.port_socket = 5000
Pilih nomor field yang ingin dihapus: 5
[!] Yakin hapus field 'mahasiswa.usia'? (y/n): y
[HAPUS] Field 'mahasiswa.usia' berhasil dihapus.
[SIMPAN] data.json berhasil disimpan di server.
[VERIFIKASI] Isi terbaru data.json:
{
  "mahasiswa": {
    "nama": "Syaiful",
    "nim": "09011282328111",
    "kelas": "SK6C",
    "matakuliah": "Sistem Terdistribusi"
  },
  "server": {
    "ip": "192.168.10.10",
    "port_ssh": 2005,
    "port_socket": 5000
  }
}

```
<img width="1153" height="1041" alt="image" src="https://github.com/user-attachments/assets/fff6aa64-6737-4ca8-9621-8d4be5a5735d" />

---

### Menu 7 — Buat File JSON/XML Baru (Paramiko)

**Buat file JSON:**

```
Pilihan: 7

Pilih format file:
  1. JSON
  2. XML
Format: 1
Nama file (tanpa ekstensi): data_baru
Pakai grup/section? (y/n): y

Nama grup (kosongkan jika selesai): dosen
Berapa field di grup 'dosen'? 2
  Nama field 1: nama
  Nilai field 1: Adi Hermansyah
  Nama field 2: pengampu mata kuliah        
  Nilai field 2: Sistem Terdistribusi

Nama grup (kosongkan jika selesai): 
[SIMPAN] data_baru.json berhasil disimpan di server.
[VERIFIKASI] Isi terbaru data_baru.json:
{
  "dosen": {
    "nama": "Adi Hermansyah",
    "pengampu mata kuliah": "Sistem Terdistribusi"
  }
}

```
<img width="1146" height="910" alt="image" src="https://github.com/user-attachments/assets/23088894-aa2a-4483-b402-f8be4b0f61d4" />

**Buat file XML:**

```
Pilihan: 7

Pilih format file:
  1. JSON
  2. XML
Format: 2
Nama file (tanpa ekstensi): data_baru
Pakai grup/section? (y/n): y

Nama grup (kosongkan jika selesai): dosen
Berapa field di grup 'dosen'? 2   
  Nama field 1: nama
  Nilai field 1: Adi Hermansyah
  Nama field 2: pengampu mata kuliah
  Nilai field 2: Sistem Terdistribusi

Nama grup (kosongkan jika selesai): 
[SIMPAN] data_baru.xml berhasil disimpan di server.
[VERIFIKASI] Isi terbaru data_baru.xml:
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <dosen>
    <nama>Adi Hermansyah</nama>
    <pengampu mata kuliah>Sistem Terdistribusi</pengampu mata kuliah>
  </dosen>
</root>

```
<img width="1146" height="926" alt="image" src="https://github.com/user-attachments/assets/45516a4d-08ac-4f8f-930d-e58a46e0ffd5" />

---

### Menu 8 — Hapus File di Server (Paramiko)

```
Pilihan: 8

Daftar semua file di server:
  1. data_baru.json
  2. data_baru.xml
  3. data.json
  4. data.xml
  5. server.py
Pilih nomor: 1
[!] Yakin hapus 'data_baru.json'? (y/n): y
[HAPUS] File data_baru.json berhasil dihapus dari server.

========================================
  MENU CLIENT
========================================
1. List file di server          (Socket)
...
8. Hapus file di server         (Paramiko)
========================================
Pilihan: 8       

Daftar semua file di server:
  1. data_baru.xml
  2. data.json
  3. data.xml
  4. server.py
Pilih nomor: 1
[!] Yakin hapus 'data_baru.xml'? (y/n): y
[HAPUS] File data_baru.xml berhasil dihapus dari server.

```
<img width="1148" height="846" alt="image" src="https://github.com/user-attachments/assets/f280ce4e-29f1-46ab-bf2a-a2099f26c2c6" />

---

### Menu 9 — Monitor Server (Paramiko)

```
Pilihan: 9

========================================
  MONITOR SERVER
========================================
Hostname   : localhost.localdomain
IP Address : 192.168.23.100
Uptime     : up 1 hour, 52 minutes
CPU Usage  : 2%
RAM Usage  : 1,8Gi/7,5Gi
Disk Usage : 7,0G/17G (42% used)
Total File : 3 file di server
========================================

```
<img width="1153" height="678" alt="image" src="https://github.com/user-attachments/assets/561db097-59c8-448f-b73d-4b8287961289" />

---

## D.4 Verifikasi Hasil di Client dan Server Log

Cek file hasil di client:

```bash
ls ~/client/hasil/
```

Output:

```
data.json  data_olahan.json  data_rekonstruksi.xml  hasil_data.json  paramiko_read.json
```
<img width="1150" height="294" alt="image" src="https://github.com/user-attachments/assets/2cf1087b-a206-4991-b09e-361904379748" />

Log yang muncul di server:

```
[SERVER] Listening di port 5000, hanya menerima dari 192.168.10.20
[2026-03-01 15:07:47.148517] Koneksi dari 192.168.10.20
[REQUEST] action=list_files
[2026-03-01 15:08:58.843310] Koneksi dari 192.168.10.20
[REQUEST] action=get_file
[2026-03-01 15:09:24.846735] Koneksi dari 192.168.10.20
[REQUEST] action=get_json_data
[2026-03-01 15:10:14.040571] Koneksi dari 192.168.10.20
[REQUEST] action=get_xml_data
[2026-03-01 15:15:00.151827] Koneksi dari 192.168.10.20
[REQUEST] action=get_json_data

```
<img width="1151" height="473" alt="image" src="https://github.com/user-attachments/assets/7df5d9c5-8c40-43a6-9c2f-96fc108e5430" />

---

## D.5 Verifikasi SSH Filter (Uji Keamanan)

Ada **2 layer filter** yang diuji secara terpisah.

---

### Layer 1 — Firewall (Port SSH 2005)

Tambah IP sementara di client:

```bash
sudo ip addr add 192.168.10.99/24 dev ens160
sudo ip addr del 192.168.10.20/24 dev ens160
```

Jalankan `client.py` dari client — SSH masuk tapi socket diblokir `server.py`:

```
python3 client.py

[SSH] Terhubung ke server
Pilihan: 1

Traceback (most recent call last):
  ...
ConnectionResetError: [Errno 104] Connection reset by peer
```
<img width="1149" height="755" alt="image" src="https://github.com/user-attachments/assets/518b0ddd-07d7-4ddf-86c2-c19ba13b31b6" />

Log di server:

```
[BLOCKED] 192.168.10.99
```
<img width="1151" height="485" alt="image" src="https://github.com/user-attachments/assets/81bbce4e-a447-46fd-aaca-043d3e050cd3" />

Hapus setelah demo:

```bash
# Di client
sudo ip addr del 192.168.10.99/24 dev ens160
sudo ip addr add 192.168.10.20/24 dev ens160
```
<img width="1150" height="481" alt="image" src="https://github.com/user-attachments/assets/0e1e57fb-0a1f-42ee-8770-eaa5da34890b" />

---

### Layer 2 — AllowUsers (User Filter SSH)

Ganti sementara `SSH_USER` di `client.py`:

```python
SSH_USER = "root"  # user tidak ada di AllowUsers
```
<img width="1150" height="474" alt="image" src="https://github.com/user-attachments/assets/e26b0085-9add-4a49-9381-79a7183fc02b" />

Jalankan:

```bash
python3 client.py
```

Output:

```
paramiko.ssh_exception.AuthenticationException: Authentication failed.
```
<img width="1149" height="823" alt="image" src="https://github.com/user-attachments/assets/7155b98e-4798-4977-9ee7-c085582264cc" />

Kembalikan setelah demo:

```python
SSH_USER = "capuchino"
```

# Alur Sistem

Sistem ini terdiri dari dua node yang saling berkomunikasi menggunakan dua jalur berbeda: **SSH via Paramiko** untuk manajemen dan pengolahan data, dan **Socket TCP** untuk pertukaran data secara langsung.

---

## Gambaran Umum

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT NODE                              │
│                      192.168.10.20                              │
│                                                                 │
│   client.py                                                     │
│   ├── Paramiko (SSH :2005) ──────────────────────────────────┐  │
│   │   ├── connect_ssh()     → Koneksi SSH programatik        │  │
│   │   ├── remote_exec()     → Eksekusi perintah di server    │  │
│   │   └── simpan_ke_server()→ Write file JSON/XML ke server  │  │
│   │                                                          │  │
│   └── Socket (TCP :5000) ────────────────────────────────┐   │  │
│       ├── list_files        → Minta daftar file          │   │  │
│       ├── get_file          → Download file              │   │  │
│       ├── get_json_data     → Ambil & parse data JSON    │   │  │
│       └── get_xml_data      → Ambil & parse data XML     │   │  │
└─────────────────────────────────────────────────────────────────┘
             │ SSH :2005 (Paramiko)          │ TCP :5000 (Socket)
             ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        SERVER NODE                              │
│                      192.168.10.10                              │
│                                                                 │
│   sshd (:2005)          server.py (:5000)                       │
│   ├── AllowUsers        ├── ALLOWED_IP filter                   │
│   ├── firewalld         ├── handle list_files                   │
│   └── PermitRootLogin   ├── handle get_file                     │
│                         ├── handle get_json_data                │
│   ~/server/             └── handle get_xml_data                 │
│   ├── data.json                                                 │
│   ├── data.xml                                                  │
│   └── server.py                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Alur per Jalur

### Jalur 1 — SSH via Paramiko (Port 2005)

Digunakan untuk: **Read, Write, Create, Delete file dan Monitor server**

```
client.py                    firewalld              sshd                  ~/server/
    │                            │                    │                       │
    │── connect_ssh() ──────────►│                    │                       │
    │   (src: 192.168.10.20)     │                    │                       │
    │                        cek rich rule            │                       │
    │                        IP .20 → LOLOS           │                       │
    │                            │──────────────────►│                       │
    │                            │               cek AllowUsers              │
    │                            │               capuchino → LOLOS           │
    │◄── SSH session established ─────────────────────│                       │
    │                            │                    │                       │
    │── remote_exec("cat data.json") ────────────────►│──────────────────────►│
    │◄── isi data.json ──────────────────────────────►│◄──────────────────────│
    │   parse JSON/XML di client │                    │                       │
    │                            │                    │                       │
    │── remote_exec("echo '...' > data.json") ───────►│──────────────────────►│
    │◄── file tersimpan di server ───────────────────►│◄──────────────────────│
```

---

### Jalur 2 — Socket TCP (Port 5000)

Digunakan untuk: **List file, Download file, Ambil data JSON/XML**

```
client.py                    server.py (port 5000)        ~/server/
    │                               │                          │
    │── socket.connect(:5000) ─────►│                          │
    │                           cek ALLOWED_IP                 │
    │                           IP .20 → LOLOS                 │
    │                               │                          │
    │── {"action": "get_json_data"} ►│                          │
    │                               │── open("data.json") ────►│
    │                               │◄── isi file ─────────────│
    │                           parse JSON                     │
    │◄── {"status":"ok","data":{...}}│                          │
    │   simpan ke hasil/            │                          │
```

---

## Alur Lengkap per Menu

| Menu | Jalur | Alur |
|------|-------|------|
| 1. List file | Socket | client → socket → server.py list dir → response JSON |
| 2. Download file | Socket | client → socket → server.py baca file → kirim bytes |
| 3. Ambil JSON | Socket | client → socket → server.py baca data.json → parse → response |
| 4. Ambil XML | Socket | client → socket → server.py baca data.xml → parse → response |
| 5. Read JSON/XML | Paramiko | client → SSH → remote_exec cat file → parse di client |
| 6. Edit/Tambah/Hapus field | Paramiko | client → SSH → cat file → edit di client → echo write ke server |
| 7. Buat file baru | Paramiko | client → input data → format JSON/XML → SSH → echo write ke server |
| 8. Hapus file | Paramiko | client → SSH → remote_exec rm file di server |
| 9. Monitor server | Paramiko | client → SSH → remote_exec hostname/uptime/free/df → tampil di client |

---

## Alur Keamanan

### Filter Layer 1 — firewalld (Port SSH 2005)

```
IP .20 ──► firewalld ──► rich rule: LOLOS ──► sshd ──► session SSH
IP .99 ──► firewalld ──► rich rule: LOLOS ──► sshd ──► session SSH (untuk demo)
IP lain ─► firewalld ──► tidak ada rule ──► DROP ──► No route to host
```

> IP `.99` sengaja diberi rich rule di port 2005 agar bisa masuk SSH, namun tetap diblokir oleh `server.py` di level socket (port 5000).

### Filter Layer 2 — server.py (Port Socket 5000)

```
IP .20 ──► server.py ──► ALLOWED_IP match ──► proses request ──► response
IP .99 ──► server.py ──► ALLOWED_IP tidak match ──► [BLOCKED] ──► ConnectionResetError di client
IP lain ─► tidak pernah sampai ke server.py (SSH ditolak firewalld sejak awal) ──► No route to host
```

### Filter Layer 3 — sshd AllowUsers (Port SSH 2005)

```
user: capuchino ──► AllowUsers match ──► autentikasi berhasil ──► session SSH
user: root      ──► AllowUsers tidak match ──► Authentication failed
user: lain      ──► AllowUsers tidak match ──► Authentication failed
```


# OPSIONAL - Percobaan dengan Tampilan GUI
## Window-based GUI
- Kode program client_gui.py di PC Client
  ```python
  import tkinter as tk
  from tkinter import ttk, messagebox
  import paramiko, socket, json, xml.etree.ElementTree as ET
  import threading, time, re, math
  from collections import deque
  
  # ══════════════════════════════════════════════
  #  KONFIGURASI
  # ══════════════════════════════════════════════
  SSH_HOST    = "192.168.10.10"
  SSH_PORT    = 2005
  SSH_USER    = "capuchino"
  SSH_PASS    = "besokLibur"
  SOCKET_PORT = 5000
  HISTORY_LEN = 30
  
  # ══════════════════════════════════════════════
  #  TEMA — Catppuccin Mocha
  # ══════════════════════════════════════════════
  C = {
      "base":     "#1e1e2e",
      "mantle":   "#181825",
      "crust":    "#11111b",
      "surface0": "#313244",
      "surface1": "#45475a",
      "surface2": "#585b70",
      "overlay0": "#6c7086",
      "overlay1": "#7f849c",
      "text":     "#cdd6f4",
      "subtext":  "#a6adc8",
      "blue":     "#89b4fa",
      "green":    "#a6e3a1",
      "red":      "#f38ba8",
      "yellow":   "#f9e2af",
      "peach":    "#fab387",
      "mauve":    "#cba6f7",
      "teal":     "#94e2d5",
      "sky":      "#89dceb",
      "sapphire": "#74c7ec",
      "lavender": "#b4befe",
  }
  
  # ══════════════════════════════════════════════
  #  NETWORK
  # ══════════════════════════════════════════════
  def connect_ssh():
      ssh = paramiko.SSHClient()
      ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
      ssh.connect(SSH_HOST, port=SSH_PORT, username=SSH_USER, password=SSH_PASS)
      return ssh
  
  def remote_exec(ssh, cmd):
      _, out, err = ssh.exec_command(cmd)
      return out.read().decode().strip(), err.read().decode().strip()
  
  def socket_request(action, filename=""):
      req = json.dumps({"action": action, "filename": filename})
      s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
      s.connect((SSH_HOST, SOCKET_PORT))
      s.sendall(req.encode())
      buf = []
      while True:
          chunk = s.recv(4096)
          if not chunk: break
          buf.append(chunk)
      s.close()
      return b"".join(buf)
  
  # ══════════════════════════════════════════════
  #  HELPERS
  # ══════════════════════════════════════════════
  def xml_to_dict(el):
      d = {}
      for child in el:
          d[child.tag] = xml_to_dict(child) if len(child) else (child.text or "")
      return d
  
  def dict_to_xml(parent, data):
      for k, v in data.items():
          el = ET.SubElement(parent, k)
          if isinstance(v, dict): dict_to_xml(el, v)
          else: el.text = str(v) if v else ""
  
  def write_to_server(ssh, filename, content_str):
      escaped = content_str.replace("'", "'\\''")
      remote_exec(ssh, f"echo '{escaped}' > ~/server/{filename}")
  
  def parse_mem(s):
      s = s.strip()
      try:
          num = float(re.sub(r'[^0-9.]', '', s))
          if 'Gi' in s or 'G' in s: return num * 1024
          if 'Mi' in s or 'M' in s: return num
          if 'Ki' in s or 'K' in s: return num / 1024
          return num
      except: return 0
  
  def hex_to_rgb(h):
      h = h.lstrip('#')
      return tuple(int(h[i:i+2], 16) for i in (0, 2, 4))
  
  def rgb_to_hex(r, g, b):
      return f"#{int(r):02x}{int(g):02x}{int(b):02x}"
  
  def lerp_color(c1, c2, t):
      r1,g1,b1 = hex_to_rgb(c1)
      r2,g2,b2 = hex_to_rgb(c2)
      return rgb_to_hex(r1+(r2-r1)*t, g1+(g2-g1)*t, b1+(b2-b1)*t)
  
  # ══════════════════════════════════════════════
  #  SYNTAX HIGHLIGHT
  # ══════════════════════════════════════════════
  HL = {
      "json_key":   C["blue"],
      "json_str":   C["green"],
      "json_num":   C["peach"],
      "json_bool":  C["red"],
      "json_brace": C["mauve"],
      "xml_tag":    C["blue"],
      "xml_attr":   C["peach"],
      "xml_val":    C["green"],
      "xml_prolog": C["overlay0"],
      "py_kw":      C["mauve"],
      "py_builtin": C["sky"],
      "py_str":     C["green"],
      "py_comment": C["overlay0"],
      "py_num":     C["peach"],
      "py_deco":    C["red"],
      "py_func":    C["blue"],
  }
  
  def highlight(widget, content, ft):
      widget.config(state="normal")
      widget.delete("1.0", "end")
      widget.insert("1.0", content)
      for tag in widget.tag_names():
          widget.tag_remove(tag, "1.0", "end")
      for tag, color in HL.items():
          widget.tag_config(tag, foreground=color)
  
      def apply_tag(pattern, tag, flags=0):
          for m in re.finditer(pattern, content, flags):
              widget.tag_add(tag, f"1.0+{m.start()}c", f"1.0+{m.end()}c")
  
      if ft == "json":
          apply_tag(r'[{}\[\]]', "json_brace")
          apply_tag(r'\b(true|false|null)\b', "json_bool")
          apply_tag(r'\b(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)\b', "json_num")
          for m in re.finditer(r':\s*("(?:[^"\\]|\\.)*")', content):
              widget.tag_add("json_str", f"1.0+{m.start(1)}c", f"1.0+{m.end(1)}c")
          for m in re.finditer(r'("(?:[^"\\]|\\.)*")\s*:', content):
              widget.tag_add("json_key", f"1.0+{m.start(1)}c", f"1.0+{m.end(1)}c")
      elif ft == "xml":
          apply_tag(r'<\?[^?]*\?>', "xml_prolog")
          apply_tag(r'</?[a-zA-Z][a-zA-Z0-9_:.-]*', "xml_tag")
          apply_tag(r'/?>', "xml_tag")
          for m in re.finditer(r'([a-zA-Z_][a-zA-Z0-9_:.-]*)=(\"[^\"]*\")', content):
              widget.tag_add("xml_attr", f"1.0+{m.start(1)}c", f"1.0+{m.end(1)}c")
              widget.tag_add("xml_val",  f"1.0+{m.start(2)}c", f"1.0+{m.end(2)}c")
      elif ft == "py":
          apply_tag(r'#[^\n]*', "py_comment")
          apply_tag(r'"""[^"]*"""', "py_str")
          apply_tag(r"'''[^']*'''", "py_str")
          apply_tag(r'"[^"\n]*"', "py_str")
          apply_tag(r"'[^'\n]*'", "py_str")
          apply_tag(r'@[a-zA-Z_][a-zA-Z0-9_.]*', "py_deco")
          apply_tag(r'\b(import|from|def|class|return|if|elif|else|for|while|try|except|finally|with|as|in|not|and|or|is|lambda|pass|break|continue|raise|yield|global|nonlocal|del|assert|True|False|None)\b', "py_kw")
          apply_tag(r'\b(print|len|range|open|str|int|float|list|dict|tuple|set|type|isinstance|enumerate|zip|map|filter|sorted|sum|min|max|abs|round|input|super)\b', "py_builtin")
          apply_tag(r'\b[0-9]+(?:\.[0-9]+)?\b', "py_num")
          for m in re.finditer(r'\bdef\s+([a-zA-Z_][a-zA-Z0-9_]*)', content):
              widget.tag_add("py_func", f"1.0+{m.start(1)}c", f"1.0+{m.end(1)}c")
  
      widget.config(state="disabled")
  
  
  # ══════════════════════════════════════════════
  #  ANIMATED DOT — identik conn-dot HTML
  #  3 state: "wait"(kuning) / "ok"(hijau+ripple) / "fail"(merah)
  #  Ripple: 3 cincin mengembang fade-out seperti CSS @keyframes ripple
  # ══════════════════════════════════════════════
  class AnimatedConnDot(tk.Canvas):
      """
      Titik koneksi animasi identik HTML .dot-pulse + @keyframes ripple.
  
      HTML CSS:
          .dot-pulse { width:7px; height:7px; border-radius:50%; background:currentColor }
          .conn-dot.ok .dot-pulse::after {
              content:''; position:absolute; inset:-3px; border-radius:50%;
              background: var(--green); opacity:0.25;
              animation: ripple 2s infinite;
          }
          @keyframes ripple {
              0%   { transform: scale(1);   opacity: 0.25; }
              100% { transform: scale(2.5); opacity: 0;    }
          }
  
      Penerjemahan ke Canvas:
      - Satu titik solid radius DOT_R di tengah
      - Satu filled circle (bukan outline) yang mengembang scale 1→2.5
        dengan opacity 0.25→0, periode 2 detik, linear
      - Background canvas cocok dengan parent agar blend terlihat benar
      """
      DOT_R  = 4       # px, radius titik utama (setara 7px diameter)
      SIZE   = 24      # px, ukuran canvas
      FPS    = 50      # frame per detik
      PERIOD = 2000    # ms, identik 2s CSS
  
      def __init__(self, parent, bg=None, **kw):
          try:
              _bg = bg or parent.cget("bg")
          except Exception:
              _bg = C["surface0"]
          self._bg = _bg
          super().__init__(parent, width=self.SIZE, height=self.SIZE,
                           bg=_bg, highlightthickness=0, bd=0, **kw)
          self._state    = "wait"
          self._t        = 0.0   # 0.0–1.0, posisi dalam satu cycle
          self._after_id = None
          self._draw_static()
  
      def set_state(self, state):
          """'wait' | 'ok' | 'fail'"""
          if state == self._state:
              return
          self._state = state
          if self._after_id:
              try: self.after_cancel(self._after_id)
              except: pass
              self._after_id = None
          self._t = 0.0
          if state == "ok":
              self._tick()
          else:
              self._draw_static()
  
      # ── internal ─────────────────────────────────────────
      def _dot_color(self):
          return {"ok": C["green"], "fail": C["red"], "wait": C["yellow"]}.get(
              self._state, C["overlay0"])
  
      def _draw_static(self):
          self.delete("all")
          cx = cy = self.SIZE // 2
          r  = self.DOT_R
          self.create_oval(cx-r, cy-r, cx+r, cy+r,
                           fill=self._dot_color(), outline="")
  
      def _tick(self):
          ms = 1000 // self.FPS
          self._t = (self._t + ms / self.PERIOD) % 1.0
          self._draw_ripple()
          self._after_id = self.after(ms, self._tick)
  
      def _draw_ripple(self):
          self.delete("all")
          cx = cy = self.SIZE // 2
  
          # ── Ripple circle (filled, mengembang) ──
          # CSS: scale 1→2.5 dari radius inset -3px (DOT_R + 3 = 7)
          # Jadi radius awal = DOT_R+3, akhir = (DOT_R+3)*2.5
          r0   = self.DOT_R + 3             # radius awal ripple
          r_max = r0 * 2.5                  # radius akhir (scale 2.5)
          t    = self._t                    # 0→1, linear identik CSS
          r_now = r0 + (r_max - r0) * t    # interpolasi linear
          alpha = 0.25 * (1.0 - t)         # 0.25→0, linear
  
          # Blend warna hijau × alpha ke background
          gr, gg, gb   = hex_to_rgb(C["green"])
          bgr, bgg, bgb = hex_to_rgb(self._bg)
          rr = int(bgr + (gr - bgr) * alpha)
          rg = int(bgg + (gg - bgg) * alpha)
          rb = int(bgb + (gb - bgb) * alpha)
          ripple_color = rgb_to_hex(rr, rg, rb)
  
          # Gambar filled circle ripple (identik filled opacity circle di CSS)
          self.create_oval(cx - r_now, cy - r_now,
                           cx + r_now, cy + r_now,
                           fill=ripple_color, outline="")
  
          # ── Titik solid di atas ──
          r = self.DOT_R
          self.create_oval(cx-r, cy-r, cx+r, cy+r,
                           fill=C["green"], outline="")
  
  
  class AnimatedModal:
      """
      Modal dialog dengan spring-bounce entry animation.
      SOLUSI kosong: semua widget dibangun SEBELUM animasi dimulai,
      lalu window ditampilkan bertahap via -alpha.
      Tidak pakai overrideredirect agar title bar dan WM_DELETE_WINDOW berfungsi.
      """
      SPRING_STEPS = 18
      SPRING_MS    = 12   # total ~216ms
  
      def __init__(self, root, title, width=520, height=510):
          self.root    = root
          self._width  = width
          self._height = height
  
          # ── Posisi center ──
          root.update_idletasks()
          rx = root.winfo_rootx()
          ry = root.winfo_rooty()
          rw = root.winfo_width()
          rh = root.winfo_height()
          self._final_x = rx + (rw - width)  // 2
          self._final_y = ry + (rh - height) // 2
  
          # ── Buat Toplevel ──
          self.win = tk.Toplevel(root)
          self.win.title(title)
          self.win.configure(bg=C["base"])
          self.win.resizable(False, False)
          self.win.geometry(
              f"{width}x{height}+{self._final_x}+{self._final_y}")
  
          # Sembunyikan dulu — konten diisi dulu baru animasi
          self.win.attributes("-alpha", 0.0)
          self.win.protocol("WM_DELETE_WINDOW", self.close)
  
          # PENTING: transient agar muncul di atas root, grab SETELAH konten diisi
          self.win.transient(root)
  
      def start(self):
          """Panggil SETELAH semua widget sudah di-pack ke self.win."""
          self.win.update_idletasks()   # paksa render konten dulu
          self.win.grab_set()
          self.win.focus_force()
          self._animate_in()
  
      def _animate_in(self):
          steps = self.SPRING_STEPS
  
          def spring(t):
              # Aproksimasi cubic-bezier(0.34, 1.4, 0.64, 1) — spring overshoot
              if t >= 1: return 1.0
              s = t * t * (3 - 2*t)                        # smoothstep base
              return s + math.sin(t * math.pi) * (1-t) * 0.3
  
          start_y = self._final_y + 12  # mulai 12px di bawah
  
          def step(i=0):
              if not self.win.winfo_exists(): return
              t   = i / steps
              t_s = spring(t)
              alpha = min(max(t_s, 0.0), 1.0)
              cur_y = int(start_y + (self._final_y - start_y) * min(t_s, 1.0))
              try:
                  self.win.geometry(
                      f"{self._width}x{self._height}"
                      f"+{self._final_x}+{cur_y}")
                  self.win.attributes("-alpha", alpha)
              except: return
              if i < steps:
                  self.win.after(self.SPRING_MS, lambda: step(i+1))
              else:
                  self.win.geometry(
                      f"{self._width}x{self._height}"
                      f"+{self._final_x}+{self._final_y}")
                  self.win.attributes("-alpha", 1.0)
          step()
  
      def close(self):
          FADE = 6
          MS   = 12
          try: self.win.grab_release()
          except: pass
  
          def fade(i=0):
              if not self.win.winfo_exists(): return
              t = i / FADE
              try: self.win.attributes("-alpha", max(0.0, 1.0 - t*t*(3-2*t)))
              except: pass
              if i < FADE:
                  self.win.after(MS, lambda: fade(i+1))
              else:
                  try: self.win.destroy()
                  except: pass
          fade()
  
      # ── public ─────────────────────────────────
      def set_state(self, state):
          """state: 'wait' | 'ok' | 'fail'"""
          self._state = state
          if state == "ok":
              self._ripple_t = 0.0
              self._start_anim()
          else:
              self._stop_anim()
              self._draw_static()
  
      # ── private ────────────────────────────────
      def _color_for_state(self):
          return {"ok": C["green"], "fail": C["red"], "wait": C["yellow"]}.get(self._state, C["overlay0"])
  
      def _start_anim(self):
          if self._anim_id: return
          self._tick()
  
      def _stop_anim(self):
          if self._anim_id:
              try: self.after_cancel(self._anim_id)
              except: pass
              self._anim_id = None
  
      def _draw_static(self):
          self.delete("all")
          cx = cy = self.SIZE // 2
          color = self._color_for_state()
          # titik solid
          r = self.DOT_R
          self.create_oval(cx-r, cy-r, cx+r, cy+r,
                           fill=color, outline="")
  
      def _tick(self):
          if self._state != "ok":
              self._stop_anim()
              self._draw_static()
              return
  
          interval_ms = 1000 // self.FPS
          self._ripple_t = (self._ripple_t + interval_ms / self.PERIOD) % 1.0
          self._draw_ripple()
          self._anim_id = self.after(interval_ms, self._tick)
  
      def _draw_ripple(self):
          self.delete("all")
          cx = cy = self.SIZE // 2
          color = C["green"]
  
          # ── Ripple rings (2 rings, offset 0.5) identik HTML ::after
          #    CSS: scale 1→2.5, opacity 0.25→0 selama 2s
          for offset in (0.0, 0.5):
              t = (self._ripple_t + offset) % 1.0
              scale   = 1.0 + t * 1.5          # 1.0 → 2.5
              opacity = 1.0 - t                # 1.0 → 0.0 (untuk blend factor)
  
              if opacity <= 0.02: continue
  
              # blend warna hijau ke bg parent sesuai opacity
              r_c, g_c, b_c   = hex_to_rgb(color)
              bg_r, bg_g, bg_b = hex_to_rgb(self._parent_bg)
              # opacity 0.25 max di HTML → blend factor max = 0.25
              blend = opacity * 0.25
              ring_hex = rgb_to_hex(
                  bg_r + (r_c - bg_r) * blend,
                  bg_g + (g_c - bg_g) * blend,
                  bg_b + (b_c - bg_b) * blend,
              )
  
              rad = self.DOT_R * scale
              # outline cincin
              self.create_oval(cx - rad, cy - rad, cx + rad, cy + rad,
                               outline=ring_hex, width=2, fill="")
  
          # ── Titik solid utama ──
          r = self.DOT_R
          self.create_oval(cx - r, cy - r, cx + r, cy + r,
                           fill=color, outline="")
  
  
  # ══════════════════════════════════════════════
  #  MINI CHART — canvas identik HTML drawMiniChart()
  #  Grid 25/50/75, gradient fill, smooth line, dot terakhir
  # ══════════════════════════════════════════════
  class MiniChart(tk.Canvas):
      def __init__(self, parent, color, height=70, **kw):
          super().__init__(parent, bg=C["surface0"], height=height,
                           highlightthickness=0, **kw)
          self.color = color
          self.data  = deque([0.0] * HISTORY_LEN, maxlen=HISTORY_LEN)
          self.bind("<Configure>", lambda e: self._draw())
  
      def push(self, val):
          self.data.append(max(0.0, min(100.0, float(val))))
          self._draw()
  
      @staticmethod
      def _dim(hex_color, factor=0.28):
          """
          Identik drawMiniChart() HTML:
          rgba(r,g,b,0.28) → blend ke bg surface0 (#313244)
          """
          r, g, b = hex_to_rgb(hex_color)
          bg_r, bg_g, bg_b = hex_to_rgb(C["surface0"])
          r2 = int(r * factor + bg_r * (1 - factor))
          g2 = int(g * factor + bg_g * (1 - factor))
          b2 = int(b * factor + bg_b * (1 - factor))
          return f"#{r2:02x}{g2:02x}{b2:02x}"
  
      def _draw(self):
          self.delete("all")
          w = self.winfo_width()
          h = self.winfo_height()
          if w < 2 or h < 2: return
  
          pts = list(self.data)
          n   = len(pts)
          if n < 2: return
          step = w / (n - 1)
  
          # ── Grid lines 25/50/75 — identik HTML (dashed 2,4) ──
          for pct in (25, 50, 75):
              y = h - (pct / 100) * h
              # simulasi dash dengan segmen pendek
              x = 0
              dash_on, dash_off = 2, 4
              while x < w:
                  x2 = min(x + dash_on, w)
                  self.create_line(x, y, x2, y,
                                   fill=C["surface1"], width=1)
                  x += dash_on + dash_off
  
          # ── Filled area (polygon) — identik gradient CSS ──
          # HTML: rgba(r,g,b,0.28) atas → rgba(r,g,b,0.02) bawah
          # Tkinter tidak support gradient polygon langsung →
          # simulasi dengan beberapa strip horizontal makin pudar
          fill_color_top = self._dim(self.color, 0.28)
          fill_color_bot = self._dim(self.color, 0.04)
  
          # Buat polygon coords
          poly_pts = [0, h]
          for i, v in enumerate(pts):
              x = i * step
              y = h - (v / 100) * (h - 4)
              poly_pts += [x, y]
          poly_pts += [(n-1)*step, h]
  
          # Gambar gradient area dengan multiple strips
          STRIPS = 12
          # Hitung y min dari data untuk clip
          data_ys = [h - (v/100)*(h-4) for v in pts]
          y_min = min(data_ys) if data_ys else 0
  
          for i in range(STRIPS):
              t_top = i / STRIPS
              t_bot = (i+1) / STRIPS
              y_top = y_min + (h - y_min) * t_top
              y_bot = y_min + (h - y_min) * t_bot
  
              # alpha interpolasi
              alpha_top = 0.28 * (1 - t_top)
              alpha_bot = 0.28 * (1 - t_bot)
              c_mix = lerp_color(fill_color_top, fill_color_bot, t_top)
  
              # clip polygon ke strip horizontal
              # buat polygon yang di-clip antara y_top dan y_bot
              strip_pts = []
              # bottom edge
              strip_pts += [0, y_bot, (n-1)*step, y_bot]
              # reverse top edge dari data
              for i2, v in reversed(list(enumerate(pts))):
                  x = i2 * step
                  y = h - (v/100)*(h-4)
                  y_clipped = max(y, y_top)
                  strip_pts += [x, y_clipped]
  
              if len(strip_pts) >= 6:
                  self.create_polygon(strip_pts, fill=c_mix, outline="", smooth=False)
  
          # ── Solid fill polygon (alpha rendah tapi flat) ──
          fill_flat = self._dim(self.color, 0.18)
          self.create_polygon(poly_pts, fill=fill_flat, outline="")
  
          # ── Smooth line — identik HTML (lineJoin round, width 2) ──
          line_pts = []
          for i, v in enumerate(pts):
              x = i * step
              y = h - (v / 100) * (h - 4)
              line_pts += [x, y]
          if len(line_pts) >= 4:
              self.create_line(line_pts, fill=self.color,
                               width=2, smooth=True, joinstyle="round",
                               capstyle="round")
  
          # ── Dot di titik terakhir — identik HTML arc r=3 ──
          lx = (n-1) * step
          ly = h - (pts[-1] / 100) * (h - 4)
          self.create_oval(lx-3, ly-3, lx+3, ly+3,
                           fill=self.color, outline="")
  
  
  # ══════════════════════════════════════════════
  #  ANIMATED PROGRESS BAR — identik HTML bar-fill
  #  CSS transition: width 0.6s cubic-bezier(0.4,0,0.2,1)
  #  dan background 0.4s (color threshold)
  # ══════════════════════════════════════════════
  class AnimatedBar(tk.Canvas):
      """
      Progress bar dengan:
      - Smooth width animation (easing cubic-bezier approx)
      - Color transition: green → peach → red (threshold 50/80%)
      - Height 6px identik HTML .bar-bg
      """
      EASE_STEPS = 20    # jumlah step animasi
      EASE_MS    = 30    # ms per step → ~600ms total (identik 0.6s CSS)
  
      def __init__(self, parent, base_color, **kw):
          super().__init__(parent, height=6, bg=C["surface1"],
                           highlightthickness=0, bd=0, **kw)
          self.base_color  = base_color
          self._target_pct = 0.0
          self._cur_pct    = 0.0
          self._cur_color  = base_color
          self._anim_id    = None
          self.bind("<Configure>", lambda e: self._redraw())
  
      def set_value(self, pct, override_color=None):
          """pct: 0–100 float. override_color: warna threshold dari luar."""
          self._target_pct  = max(0.0, min(100.0, float(pct)))
          self._target_color = override_color or self._color_for(pct)
          self._animate()
  
      def _color_for(self, pct):
          if pct > 80: return C["red"]
          if pct > 50: return C["peach"]
          return self.base_color
  
      def _animate(self):
          if self._anim_id:
              try: self.after_cancel(self._anim_id)
              except: pass
  
          steps_done   = [0]
          start_pct    = self._cur_pct
          start_color  = self._cur_color
          end_pct      = self._target_pct
          end_color    = self._target_color
  
          def step():
              steps_done[0] += 1
              t = steps_done[0] / self.EASE_STEPS
              # cubic-bezier(0.4,0,0.2,1) approx dengan smoothstep
              t = t * t * (3 - 2 * t)  # smoothstep
              if t > 1: t = 1.0
  
              self._cur_pct   = start_pct + (end_pct - start_pct) * t
              self._cur_color = lerp_color(start_color, end_color, t)
              self._redraw()
  
              if steps_done[0] < self.EASE_STEPS:
                  self._anim_id = self.after(self.EASE_MS, step)
              else:
                  self._cur_pct   = end_pct
                  self._cur_color = end_color
                  self._anim_id   = None
                  self._redraw()
  
          self._anim_id = self.after(0, step)
  
      def _redraw(self):
          self.delete("all")
          w = self.winfo_width()
          h = self.winfo_height()
          if w < 2 or h < 2: return
          filled = int(w * self._cur_pct / 100)
          if filled > 0:
              # rounded corners — buat polygon dengan ujung bulat
              r  = h // 2
              x1, y1, x2, y2 = 0, 0, filled, h
              if filled >= h:
                  # kiri bulat
                  self.create_arc(x1, y1, x1+h, y2, start=90, extent=180,
                                  fill=self._cur_color, outline="")
                  # tengah
                  if filled > h:
                      self.create_rectangle(x1+r, y1, x2-r, y2,
                                            fill=self._cur_color, outline="")
                  # kanan bulat
                  self.create_arc(x2-h, y1, x2, y2, start=270, extent=180,
                                  fill=self._cur_color, outline="")
              else:
                  self.create_rectangle(x1, y1, x2, y2,
                                        fill=self._cur_color, outline="")
  
  
  # ══════════════════════════════════════════════
  #  TOAST — identik HTML toast (slide dari kanan, auto-dismiss)
  #  CSS: @keyframes tIn { from { translateX(20px) scale(0.95); opacity:0 } }
  # ══════════════════════════════════════════════
  class Toast:
      ANIM_STEPS = 8
      ANIM_MS    = 18    # ~150ms slide-in
      OUT_MS     = 14    # ~110ms slide-out
  
      def __init__(self, root):
          self.root  = root
          self._wins = []
  
      def show(self, msg, kind="info", duration=2500):
          color = {"info": C["blue"], "ok": C["green"],
                   "error": C["red"],  "warn": C["yellow"],
                   "err":  C["red"]}.get(kind, C["blue"])
          fg = C["crust"]
  
          win = tk.Toplevel(self.root)
          win.overrideredirect(True)
          win.attributes("-topmost", True)
          win.configure(bg=color)
          win.attributes("-alpha", 0.0)
  
          lbl = tk.Label(win, text=f"  {msg}  ", bg=color, fg=fg,
                         font=("Segoe UI", 10, "bold"), pady=8, padx=4)
          lbl.pack()
          win.update_idletasks()
  
          sw = self.root.winfo_screenwidth()
          sh = self.root.winfo_screenheight()
          w  = win.winfo_reqwidth()
          h  = win.winfo_reqheight()
  
          # hitung posisi stack (beberapa toast aktif)
          stack_offset = sum(
              (t.winfo_reqheight() + 7)
              for t in self._wins if t.winfo_exists()
          )
          final_x = sw - w - 24
          final_y = sh - h - 60 - stack_offset
  
          self._wins.append(win)
  
          # ── Slide-in dari kanan (translateX +20px → 0) + fade ──
          def animate_in(step=0):
              if not win.winfo_exists(): return
              t = step / self.ANIM_STEPS
              t_ease = t * t * (3 - 2*t)  # smoothstep
  
              # x: final_x+20 → final_x
              cur_x = int(final_x + 20 * (1 - t_ease))
              # scale: 0.95→1.0 (simulasi via opacity ramp)
              alpha = t_ease
              win.geometry(f"{w}x{h}+{cur_x}+{final_y}")
              win.attributes("-alpha", min(alpha, 1.0))
              if step < self.ANIM_STEPS:
                  self.root.after(self.ANIM_MS, lambda: animate_in(step+1))
  
          animate_in()
  
          # ── Auto dismiss ──
          self.root.after(duration, lambda: self._dismiss(win, final_x, h, final_y))
  
      def _dismiss(self, win, final_x, h, final_y):
          if not win.winfo_exists(): return
  
          def animate_out(step=0):
              if not win.winfo_exists():
                  self._cleanup(win); return
              t = step / self.ANIM_STEPS
              t_ease = t * t * (3 - 2*t)
  
              cur_x = int(final_x + 20 * t_ease)
              alpha = 1.0 - t_ease
              try:
                  win.geometry(f"{win.winfo_reqwidth()}x{h}+{cur_x}+{final_y}")
                  win.attributes("-alpha", max(alpha, 0.0))
              except: pass
  
              if step < self.ANIM_STEPS:
                  self.root.after(self.OUT_MS, lambda: animate_out(step+1))
              else:
                  self._cleanup(win)
  
          animate_out()
  
      def _cleanup(self, win):
          try:
              win.destroy()
              if win in self._wins: self._wins.remove(win)
          except: pass
  
  
  # ══════════════════════════════════════════════
  #  LOG PANEL
  # ══════════════════════════════════════════════
  class LogPanel(tk.Frame):
      def __init__(self, parent, **kw):
          super().__init__(parent, bg=C["crust"], **kw)
          hdr = tk.Frame(self, bg=C["surface0"], height=26)
          hdr.pack(fill="x")
          hdr.pack_propagate(False)
          tk.Label(hdr, text="📋  Log Aktivitas", font=("Segoe UI", 9, "bold"),
                   bg=C["surface0"], fg=C["subtext"]).pack(side="left", padx=10)
          tk.Button(hdr, text="✕ Hapus", font=("Segoe UI", 8),
                    bg=C["surface0"], fg=C["overlay0"], relief="flat",
                    cursor="hand2", command=self._clear).pack(side="right", padx=6)
          self._box = tk.Text(self, bg=C["crust"], fg=C["green"],
                              font=("Consolas", 9), relief="flat",
                              state="disabled", height=5, wrap="word",
                              highlightthickness=0)
          self._box.pack(fill="both", expand=True, padx=4, pady=2)
  
      def log(self, msg, color=None):
          ts = time.strftime("%H:%M:%S")
          self._box.config(state="normal")
          tag = f"c{id(msg)}{time.time()}"
          self._box.tag_config(tag, foreground=color or C["green"])
          self._box.insert("end", f"[{ts}]  {msg}\n", tag)
          self._box.see("end")
          self._box.config(state="disabled")
  
      def _clear(self):
          self._box.config(state="normal")
          self._box.delete("1.0", "end")
          self._box.config(state="disabled")
  
  
  # ══════════════════════════════════════════════
  #  ROUNDED BUTTON — identik HTML .btn hover
  #  Animasi: translateY(-1px) + brightness(1.1)
  # ══════════════════════════════════════════════
  class RoundedButton(tk.Frame):
      """
      Tombol rounded dengan:
      - Hover: warna lebih terang + naik 1px (translateY -1px)
      - Click: balik ke posisi normal
      Pakai tk.Frame + tk.Label agar ukuran terukur dengan benar sebelum pack.
      """
      def __init__(self, parent, text, command=None,
                   bg=C["surface1"], fg=C["text"],
                   font=("Segoe UI", 9, "bold"),
                   padx=10, pady=5, radius=5, **kw):
          # Frame luar sebagai spacer vertikal (ruang hover naik)
          try:
              parent_bg = parent.cget("bg")
          except Exception:
              parent_bg = C["mantle"]
  
          super().__init__(parent, bg=parent_bg,
                           padx=0, pady=0, bd=0,
                           highlightthickness=0, **kw)
  
          self._text    = text
          self._font    = font
          self._bg      = bg
          self._fg      = fg
          self._command = command
          self._hover   = False
          self._press   = False
          self._parent_bg = parent_bg
  
          # Canvas di dalam frame — ukurannya adaptif
          self._canvas = tk.Canvas(self, bg=parent_bg,
                                   highlightthickness=0, bd=0,
                                   cursor="hand2")
          self._canvas.pack(fill="both", expand=True)
  
          # Teks ukuran — pakai tk.Label sementara untuk mengukur
          tmp = tk.Label(self, text=text, font=font,
                         padx=padx, pady=pady)
          tmp.update_idletasks()
          self._btn_w = max(tmp.winfo_reqwidth(), 80)
          self._btn_h = tmp.winfo_reqheight()
          tmp.destroy()
  
          self._canvas.config(width=self._btn_w, height=self._btn_h + 3)
  
          self._canvas.bind("<Enter>",           self._on_enter)
          self._canvas.bind("<Leave>",           self._on_leave)
          self._canvas.bind("<ButtonPress-1>",   self._on_press)
          self._canvas.bind("<ButtonRelease-1>", self._on_release)
          self._canvas.bind("<Configure>",       lambda e: self._draw())
          self._draw()
  
      def config_btn(self, **kw):
          if "text" in kw: self._text = kw["text"]
          if "bg"   in kw: self._bg   = kw["bg"]
          if "fg"   in kw: self._fg   = kw["fg"]
          self._draw()
  
      def _on_enter(self, e):
          self._hover = True
          self._draw()
  
      def _on_leave(self, e):
          self._hover = False
          self._press = False
          self._draw()
  
      def _on_press(self, e):
          self._press = True
          self._draw()
  
      def _on_release(self, e):
          was_press = self._press
          self._press = False
          self._draw()
          if was_press and self._command:
              self._canvas.after(10, self._command)
  
      def _draw(self):
          c = self._canvas
          c.delete("all")
          w = c.winfo_width()
          h = c.winfo_height()
          if w < 4 or h < 4:
              return
  
          r = 5
          # hover → naik 1px (gambar di y=0 bukan y=1)
          dy = 0 if (self._hover and not self._press) else 1
          y1, y2 = dy, h - 2 + dy
  
          # brightness identik HTML filter:brightness(1.1)
          if self._hover and not self._press:
              color = self._brighten(self._bg, 1.12)
          else:
              color = self._bg
  
          self._rounded_rect(c, 1, y1, w - 1, y2, r, color)
          cx = w // 2
          cy = (y1 + y2) // 2
          c.create_text(cx, cy, text=self._text,
                        fill=self._fg, font=self._font, anchor="center")
  
      @staticmethod
      def _rounded_rect(c, x1, y1, x2, y2, r, color):
          c.create_arc(x1,       y1,       x1+2*r, y1+2*r, start=90,  extent=90,  fill=color, outline="")
          c.create_arc(x2-2*r,   y1,       x2,     y1+2*r, start=0,   extent=90,  fill=color, outline="")
          c.create_arc(x1,       y2-2*r,   x1+2*r, y2,     start=180, extent=90,  fill=color, outline="")
          c.create_arc(x2-2*r,   y2-2*r,   x2,     y2,     start=270, extent=90,  fill=color, outline="")
          c.create_rectangle(x1+r, y1,   x2-r, y2, fill=color, outline="")
          c.create_rectangle(x1,   y1+r, x2,   y2-r, fill=color, outline="")
  
      @staticmethod
      def _brighten(hex_color, factor):
          r, g, b = hex_to_rgb(hex_color)
          return rgb_to_hex(min(255, int(r*factor)),
                            min(255, int(g*factor)),
                            min(255, int(b*factor)))
  
  
  
  
  
  # ══════════════════════════════════════════════
  #  NAV BUTTON — identik HTML .nav-btn
  #  Active state: bar biru kiri (::before 3px)
  #  Hover: background surface0
  # ══════════════════════════════════════════════
  class NavButton(tk.Canvas):
      """
      Tombol navigasi sidebar dengan:
      - Active indicator bar (3px biru, kiri) — identik HTML ::before
      - Hover background transition
      """
      HEIGHT = 36
      RADIUS = 5
  
      def __init__(self, parent, icon, label, command=None, **kw):
          super().__init__(parent, height=self.HEIGHT, bg=C["mantle"],
                           highlightthickness=0, bd=0,
                           cursor="hand2", **kw)
          self._icon    = icon
          self._label   = label
          self._command = command
          self._active  = False
          self._hover   = False
  
          self.bind("<Enter>",           self._on_enter)
          self.bind("<Leave>",           self._on_leave)
          self.bind("<ButtonRelease-1>", self._on_click)
          self.bind("<Configure>",       lambda e: self._draw())
  
      def set_active(self, active):
          self._active = active
          self._draw()
  
      def _on_enter(self, e):
          self._hover = True
          self._draw()
  
      def _on_leave(self, e):
          self._hover = False
          self._draw()
  
      def _on_click(self, e):
          if self._command: self._command()
  
      def _draw(self):
          self.delete("all")
          w = self.winfo_width()
          h = self.winfo_height()
          if w < 4: return
  
          # background
          if self._active or self._hover:
              bg = C["surface0"]
          else:
              bg = C["mantle"]
  
          # rounded rect background (margin 6px kiri-kanan, 1px atas-bawah)
          mx = 6
          self._rounded_rect(mx, 1, w-mx, h-1, self.RADIUS, bg)
  
          # ── Active indicator: bar biru 3px di kiri ── identik ::before
          if self._active:
              bar_y1 = h * 0.25
              bar_y2 = h * 0.75
              # rounded ujung kanan saja
              self.create_rectangle(mx, bar_y1, mx+3, bar_y2,
                                    fill=C["blue"], outline="")
              # ujung kanan bulat
              self.create_arc(mx+1, bar_y1-1, mx+3+2, bar_y1+2,
                              start=0, extent=90,
                              fill=C["blue"], outline="")
              self.create_arc(mx+1, bar_y2-2, mx+3+2, bar_y2+1,
                              start=270, extent=90,
                              fill=C["blue"], outline="")
  
          # text color
          fg = C["text"] if (self._active or self._hover) else C["subtext"]
  
          # icon + label
          self.create_text(mx + 16, h//2, text=self._icon,
                           font=("Segoe UI", 12), fill=fg, anchor="w")
          self.create_text(mx + 34, h//2, text=self._label,
                           font=("Segoe UI", 10, "bold" if self._active else "normal"),
                           fill=fg, anchor="w")
  
      def _rounded_rect(self, x1, y1, x2, y2, r, color):
          self.create_arc(x1,     y1,     x1+2*r, y1+2*r, start=90,  extent=90,  fill=color, outline="")
          self.create_arc(x2-2*r, y1,     x2,     y1+2*r, start=0,   extent=90,  fill=color, outline="")
          self.create_arc(x1,     y2-2*r, x1+2*r, y2,     start=180, extent=90,  fill=color, outline="")
          self.create_arc(x2-2*r, y2-2*r, x2,     y2,     start=270, extent=90,  fill=color, outline="")
          self.create_rectangle(x1+r, y1, x2-r, y2,  fill=color, outline="")
          self.create_rectangle(x1, y1+r, x2,   y2-r, fill=color, outline="")
  
  
  # ══════════════════════════════════════════════
  #  EMPTY STATE — identik HTML #empty-state
  #  Ikon mengambang naik-turun (floatIcon 3s ease-in-out infinite)
  # ══════════════════════════════════════════════
  class FloatingEmptyState(tk.Frame):
      """
      Empty state dengan animasi ikon mengambang
      identik: @keyframes floatIcon { 0%,100%: translateY(0); 50%: translateY(-6px) }
      """
      PERIOD_MS = 3000  # 3 detik identik CSS
      AMPLITUDE = 6     # pixel naik-turun identik (-6px)
      FPS       = 30
  
      def __init__(self, parent, **kw):
          super().__init__(parent, bg=C["mantle"], **kw)
          # Canvas untuk ikon mengambang
          self._canvas = tk.Canvas(self, width=80, height=80,
                                   bg=C["mantle"], highlightthickness=0)
          self._canvas.pack(pady=(0, 0))
          self._icon_id = self._canvas.create_text(40, 40, text="📂",
                                                    font=("Segoe UI", 38),
                                                    fill=C["overlay0"])
          tk.Label(self, text="Belum ada file yang dibuka",
                   font=("Segoe UI", 11, "bold"),
                   bg=C["mantle"], fg=C["overlay0"]).pack()
          tk.Label(self, text="Pilih file dari panel kiri · Double-click untuk edit",
                   font=("Consolas", 9),
                   bg=C["mantle"], fg=C["surface2"]).pack(pady=(4, 0))
  
          self._t    = 0.0
          self._anim = None
          self._start_float()
  
      def _start_float(self):
          interval = 1000 // self.FPS
          def tick():
              self._t = (self._t + interval / self.PERIOD_MS) % 1.0
              # ease-in-out: identik CSS ease-in-out
              # y = -amplitude * sin(2π·t)  → naik di t=0.25, turun di t=0.75
              raw = math.sin(2 * math.pi * self._t)
              # ease: tidak linear, tapi sin sudah mulus
              dy = -self.AMPLITUDE * raw
              # repositon ikon
              self._canvas.coords(self._icon_id, 40, 40 + dy)
              if self.winfo_exists():
                  self._anim = self.after(interval, tick)
          self._anim = self.after(0, tick)
  
      def stop(self):
          if self._anim:
              try: self.after_cancel(self._anim)
              except: pass
  
  
  # ══════════════════════════════════════════════
  #  MAIN APP
  # ══════════════════════════════════════════════
  class App:
      def __init__(self, root):
          self.root   = root
          self.root.title("⚡ RSManager — Remote Server Manager")
          self.root.geometry("1280x780")
          self.root.minsize(960, 620)
          self.root.configure(bg=C["crust"])
  
          self.ssh          = None
          self.files        = []
          self.toast        = Toast(root)
          self._edit_mode   = False
          self._cur_file    = None
          self._cur_ft      = None
          self._active_tab  = None
          self._chart_cpu   = None
          self._chart_ram   = None
          self._history_cpu = deque([0.0]*HISTORY_LEN, maxlen=HISTORY_LEN)
          self._history_ram = deque([0.0]*HISTORY_LEN, maxlen=HISTORY_LEN)
          self._bars        = {}   # name → AnimatedBar
          self._nav_btns    = {}   # key → NavButton
          self._modal       = None
  
          self._build()
          self._connect()
          self.root.bind("<Button-1>", lambda e: self.ctx_menu.unpost())
  
      # ══════════════════════════════════════════
      #  BUILD
      # ══════════════════════════════════════════
      def _build(self):
          self.sidebar = tk.Frame(self.root, bg=C["mantle"], width=200)
          self.sidebar.pack(side="left", fill="y")
          self.sidebar.pack_propagate(False)
          self._build_sidebar()
  
          right_col = tk.Frame(self.root, bg=C["crust"])
          right_col.pack(side="left", fill="both", expand=True)
  
          # ── Topbar ──
          self._topbar = tk.Frame(right_col, bg=C["surface0"], height=46)
          self._topbar.pack(fill="x")
          self._topbar.pack_propagate(False)
          self.page_title = tk.Label(self._topbar, text="",
                                     font=("Segoe UI", 12, "bold"),
                                     bg=C["surface0"], fg=C["text"])
          self.page_title.pack(side="left", padx=16, pady=10)
          self.breadcrumb = tk.Label(self._topbar, text="",
                                     font=("Consolas", 9),
                                     bg=C["surface0"], fg=C["overlay0"])
          self.breadcrumb.pack(side="left")
  
          # ── Page area ──
          self.page_area = tk.Frame(right_col, bg=C["base"])
          self.page_area.pack(fill="both", expand=True)
  
          # ── Log panel (130px) ──
          self.log = LogPanel(right_col, height=130)
          self.log.pack(fill="x")
          self.log.pack_propagate(False)
  
          self._pg_files   = tk.Frame(self.page_area, bg=C["base"])
          self._pg_monitor = tk.Frame(self.page_area, bg=C["base"])
          self._build_files_page()
          self._build_monitor_page()
  
          # ── Context menu ──
          self.ctx_menu = tk.Menu(
              self.root, tearoff=0,
              bg=C["surface0"], fg=C["text"],
              activebackground=C["surface1"],
              activeforeground=C["text"],
              font=("Segoe UI", 10), relief="flat",
              bd=1
          )
          self.ctx_menu.add_command(label="📄  Buka & Lihat Isi",   command=self._open_file)
          self.ctx_menu.add_separator()
          self.ctx_menu.add_command(label="⬇️  Download ke Lokal",  command=self._download_file)
          self.ctx_menu.add_separator()
          self.ctx_menu.add_command(label="🗑️  Hapus File",         command=self._delete_file,
                                    foreground=C["red"], activeforeground=C["red"],
                                    activebackground="#4e2030")
  
          self._switch_tab("files")
  
      def _build_sidebar(self):
          # ── Logo bar (56px identik HTML) ──
          logo = tk.Frame(self.sidebar, bg=C["crust"], height=56)
          logo.pack(fill="x")
          logo.pack_propagate(False)
          tk.Label(logo, text="⚡ RSManager",
                   font=("Segoe UI", 13, "bold"),
                   bg=C["crust"], fg=C["blue"]).pack(anchor="w", padx=14, pady=16)
          # garis gradien biru di bawah logo (identik ::after)
          tk.Frame(self.sidebar, bg=C["blue"], height=1).pack(fill="x")
          tk.Frame(self.sidebar, bg=C["mantle"], height=1).pack(fill="x")
  
          # ── Connection box ──
          info_frame = tk.Frame(self.sidebar, bg=C["surface0"],
                                relief="flat", bd=1)
          info_frame.pack(fill="x", padx=10, pady=(8, 0))
  
          conn_row = tk.Frame(info_frame, bg=C["surface0"])
          conn_row.pack(fill="x", padx=8, pady=(6, 0))
  
          # AnimatedConnDot — bg harus sama dengan parent (surface0)
          self._conn_dot_widget = AnimatedConnDot(conn_row, bg=C["surface0"])
          self._conn_dot_widget.pack(side="left")
  
          self._conn_text = tk.Label(conn_row, text=" Menghubungkan…",
                                     font=("Segoe UI", 8, "bold"),
                                     bg=C["surface0"], fg=C["yellow"])
          self._conn_text.pack(side="left", padx=4)
  
          self._conn_host = tk.Label(info_frame,
                                     text="—",
                                     font=("Consolas", 8),
                                     bg=C["surface0"], fg=C["overlay1"])
          self._conn_host.pack(anchor="w", padx=8, pady=(0, 6))
  
          # ── Divider ──
          tk.Frame(self.sidebar, bg=C["surface1"], height=1).pack(fill="x", pady=10, padx=10)
  
          tk.Label(self.sidebar, text="NAVIGASI", font=("Segoe UI", 8),
                   bg=C["mantle"], fg=C["overlay0"]).pack(anchor="w", padx=14, pady=(0, 4))
  
          # ── Nav buttons (NavButton custom widget) ──
          for key, icon, label_text in [
              ("files",   "📁", "File Explorer"),
              ("monitor", "📊", "Monitor Server"),
          ]:
              btn = NavButton(self.sidebar, icon, label_text,
                              command=lambda k=key: self._switch_tab(k))
              btn.pack(fill="x", padx=0, pady=1)
              self._nav_btns[key] = btn
  
          # ── Divider ──
          tk.Frame(self.sidebar, bg=C["surface1"], height=1).pack(fill="x", pady=10, padx=10)
          tk.Label(self.sidebar, text="AKSI", font=("Segoe UI", 8),
                   bg=C["mantle"], fg=C["overlay0"]).pack(anchor="w", padx=14, pady=(0, 4))
  
          # ── Action buttons (RoundedButton) ──
          # Wrapper frame agar fill="x" bekerja untuk RoundedButton
          for btn_text, btn_cmd, btn_bg, btn_fg in [
              ("＋  Buat File Baru", self._dlg_create,  C["green"],   C["crust"]),
              ("⟳   Refresh",        self._refresh,     C["surface1"], C["text"]),
          ]:
              wrap = tk.Frame(self.sidebar, bg=C["mantle"])
              wrap.pack(fill="x", padx=10, pady=2)
              rb = RoundedButton(wrap, text=btn_text, command=btn_cmd,
                                 bg=btn_bg, fg=btn_fg, padx=10, pady=7)
              rb.pack(fill="x")
  
          tk.Frame(self.sidebar, bg=C["mantle"]).pack(fill="both", expand=True)
          tk.Frame(self.sidebar, bg=C["surface1"], height=1).pack(fill="x", padx=10)
          tk.Label(self.sidebar, text=f"Client: {SSH_HOST.rsplit('.',1)[0]}.20",
                   font=("Consolas", 8), bg=C["mantle"],
                   fg=C["overlay0"]).pack(anchor="w", padx=14, pady=6)
  
      def _build_files_page(self):
          p = self._pg_files
  
          paned = tk.PanedWindow(p, orient="horizontal", bg=C["base"],
                                 sashwidth=4, sashrelief="flat", handlesize=0)
          paned.pack(fill="both", expand=True)
  
          # ── Left panel (220px) ──
          left = tk.Frame(paned, bg=C["mantle"])
          paned.add(left, minsize=180, width=220)
  
          search_frame = tk.Frame(left, bg=C["surface0"])
          search_frame.pack(fill="x", padx=8, pady=8)
          tk.Label(search_frame, text="🔍", bg=C["surface0"],
                   fg=C["overlay0"], font=("Segoe UI", 10)).pack(side="left", padx=(6, 0))
          self.search_var = tk.StringVar()
          self.search_var.trace("w", lambda *a: self._render_files())
          tk.Entry(search_frame, textvariable=self.search_var,
                   bg=C["surface0"], fg=C["text"],
                   insertbackground=C["text"], relief="flat",
                   font=("Segoe UI", 10),
                   highlightthickness=0).pack(fill="x", padx=6, pady=4)
  
          lb_frame = tk.Frame(left, bg=C["mantle"])
          lb_frame.pack(fill="both", expand=True, padx=4)
          sb = tk.Scrollbar(lb_frame, bg=C["surface1"],
                            troughcolor=C["mantle"], relief="flat", width=5)
          sb.pack(side="right", fill="y")
          self.file_lb = tk.Listbox(lb_frame, bg=C["mantle"], fg=C["text"],
                                    font=("Segoe UI", 10),
                                    selectbackground=C["surface0"],
                                    selectforeground=C["blue"],
                                    yscrollcommand=sb.set,
                                    relief="flat", activestyle="none",
                                    borderwidth=0, highlightthickness=0,
                                    cursor="hand2")
          self.file_lb.pack(fill="both", expand=True)
          sb.config(command=self.file_lb.yview)
          self.file_lb.bind("<Double-Button-1>", lambda e: self._open_file())
          self.file_lb.bind("<Button-3>",        self._show_ctx)
          self.file_lb.bind("<<ListboxSelect>>", self._on_select)
  
          # Info bar bawah
          self.file_info = tk.Label(left, text="", font=("Consolas", 8),
                                    bg=C["crust"], fg=C["overlay0"],
                                    anchor="w", pady=3)
          self.file_info.pack(fill="x", padx=4)
  
          # ── Right: viewer ──
          right = tk.Frame(paned, bg=C["base"])
          paned.add(right, minsize=400)
  
          # Viewer header (40px)
          vhdr = tk.Frame(right, bg=C["surface0"], height=40)
          vhdr.pack(fill="x")
          vhdr.pack_propagate(False)
  
          self.viewer_title = tk.Label(vhdr, text="Pilih file untuk melihat isinya",
                                       font=("Segoe UI", 10),
                                       bg=C["surface0"], fg=C["overlay0"])
          self.viewer_title.pack(side="left", padx=12, pady=10)
  
          self.edit_badge = tk.Label(vhdr, text="",
                                     font=("Consolas", 8, "bold"),
                                     bg=C["surface0"], fg=C["yellow"],
                                     padx=4, pady=2)
          self.edit_badge.pack(side="left")
  
          self.shortcut_hint = tk.Label(
              vhdr,
              text="  Ctrl+S  Simpan  ·  Esc  Batal  ·  Double-click  Edit",
              font=("Consolas", 8),
              bg=C["surface1"], fg=C["overlay0"],
              padx=8, pady=2)
  
          # Viewer buttons (RoundedButton) — bg harus surface0 (ikut vhdr)
          self._vbtn_dl = RoundedButton(vhdr, "⬇️ Download",
                                         command=self._download_file,
                                         bg=C["surface1"], fg=C["text"],
                                         padx=8, pady=3)
          self._vbtn_cancel = RoundedButton(vhdr, "✕ Batal",
                                             command=self._cancel_edit,
                                             bg=C["surface1"], fg=C["text"],
                                             padx=8, pady=3)
          self._vbtn_save   = RoundedButton(vhdr, "💾 Simpan",
                                             command=self._save_inline,
                                             bg=C["green"], fg=C["crust"],
                                             padx=8, pady=3)
          self._vbtn_edit   = RoundedButton(vhdr, "✏️ Edit",
                                             command=self._toggle_edit,
                                             bg=C["blue"], fg=C["crust"],
                                             padx=8, pady=3)
          self._vbtn_edit.pack(side="right", padx=8, pady=5)
  
          # ── Viewer body ──
          vbody = tk.Frame(right, bg=C["mantle"])
          vbody.pack(fill="both", expand=True)
  
          # Empty state (FloatingEmptyState dengan animasi ikon)
          self._empty_state = FloatingEmptyState(vbody)
          self._empty_state.place(relx=0.5, rely=0.5, anchor="center")
  
          self.line_nums = tk.Text(vbody, width=4, bg=C["crust"], fg=C["overlay0"],
                                   font=("Consolas", 10), relief="flat",
                                   state="disabled", cursor="arrow",
                                   highlightthickness=0, selectbackground=C["crust"])
          self.line_nums.pack(side="left", fill="y")
  
          self.viewer = tk.Text(vbody, bg=C["mantle"], fg=C["text"],
                                font=("Consolas", 10), relief="flat",
                                insertbackground=C["blue"],   # caret biru identik HTML
                                wrap="none",
                                state="disabled", highlightthickness=0,
                                selectbackground=C["surface1"],
                                padx=8, pady=4)
          vsb = tk.Scrollbar(vbody, bg=C["surface0"],
                             troughcolor=C["mantle"], relief="flat", width=5)
          vsb.pack(side="right", fill="y")
          hsb = tk.Scrollbar(right, orient="horizontal",
                             bg=C["surface0"], troughcolor=C["mantle"], relief="flat")
          hsb.pack(fill="x")
          self.viewer.pack(fill="both", expand=True)
          self.viewer.config(yscrollcommand=vsb.set, xscrollcommand=hsb.set)
          vsb.config(command=self._sync_scroll)
          hsb.config(command=self.viewer.xview)
  
          self.viewer.bind("<KeyRelease>",      self._update_line_nums)
          self.viewer.bind("<MouseWheel>",      self._sync_scroll_mouse)
          self.viewer.bind("<Double-Button-1>", self._on_viewer_double_click)
          self.viewer.bind("<Control-s>",       self._shortcut_save)
          self.viewer.bind("<Control-S>",       self._shortcut_save)
          self.viewer.bind("<Escape>",          self._shortcut_cancel)
  
          # Status bar
          self.status_bar = tk.Label(right, text="",
                                     font=("Consolas", 8),
                                     bg=C["crust"], fg=C["overlay0"],
                                     anchor="w", pady=3)
          self.status_bar.pack(fill="x", padx=8)
  
      def _build_monitor_page(self):
          p = self._pg_monitor
  
          canvas = tk.Canvas(p, bg=C["base"], highlightthickness=0)
          vsb    = tk.Scrollbar(p, orient="vertical", command=canvas.yview)
          canvas.configure(yscrollcommand=vsb.set)
          vsb.pack(side="right", fill="y")
          canvas.pack(fill="both", expand=True)
  
          inner = tk.Frame(canvas, bg=C["base"])
          win_id = canvas.create_window((0, 0), window=inner, anchor="nw")
  
          def _resize(e): canvas.itemconfig(win_id, width=e.width)
          canvas.bind("<Configure>", _resize)
          inner.bind("<Configure>",
                     lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
  
          # ── Header row ──
          hdr_row = tk.Frame(inner, bg=C["base"])
          hdr_row.pack(fill="x", padx=20, pady=(16, 4))
          tk.Label(hdr_row, text="Server Monitor",
                   font=("Segoe UI", 15, "bold"),
                   bg=C["base"], fg=C["text"]).pack(side="left")
          self.monitor_ts = tk.Label(hdr_row, text="",
                                     font=("Consolas", 9),
                                     bg=C["base"], fg=C["overlay0"])
          self.monitor_ts.pack(side="left", padx=12)
  
          # Live dot label (teks + titik)
          self._live_frame = tk.Frame(hdr_row, bg=C["base"])
          self._live_frame.pack(side="right")
          self._live_dot_canvas = AnimatedConnDot(self._live_frame, bg=C["base"])
          self._live_dot_canvas.pack(side="left")
          tk.Label(self._live_frame, text=" Live",
                   font=("Segoe UI", 9, "bold"),
                   bg=C["base"], fg=C["overlay0"]).pack(side="left")
  
          # ── 4 stat cards ──
          card_row = tk.Frame(inner, bg=C["base"])
          card_row.pack(fill="x", padx=20, pady=(4, 8))
          card_row.columnconfigure((0,1,2,3), weight=1, uniform="c")
  
          self._monitor_labels = {}
          stat_cards = [
              ("Hostname",   "🖥",  C["blue"]),
              ("IP Address", "🌐",  C["teal"]),
              ("Uptime",     "⏱",  C["green"]),
              ("Total File", "📁",  C["yellow"]),
          ]
          for col, (name, icon, color) in enumerate(stat_cards):
              cf = tk.Frame(card_row, bg=C["surface0"])
              cf.grid(row=0, column=col, padx=5, sticky="nsew")
              # Accent bar (3px) — identik HTML .card-accent
              tk.Frame(cf, bg=color, height=3).pack(fill="x")
              inn = tk.Frame(cf, bg=C["surface0"])
              inn.pack(fill="both", padx=12, pady=10)
              tk.Label(inn, text=f"{icon}  {name}",
                       font=("Segoe UI", 9),
                       bg=C["surface0"], fg=C["overlay1"]).pack(anchor="w")
              lbl = tk.Label(inn, text="—",
                             font=("Consolas", 12, "bold"),
                             bg=C["surface0"], fg=color)
              lbl.pack(anchor="w", pady=(4, 0))
              self._monitor_labels[name] = (lbl, color)
  
          # ── Metric cards (CPU, RAM, Disk) ──
          chart_metrics = [
              ("CPU Usage",  "⚙️",  C["peach"],    True),
              ("RAM Usage",  "💾",  C["mauve"],    True),
              ("Disk Usage", "🗄",  C["sapphire"], False),
          ]
  
          for name, icon, color, has_chart in chart_metrics:
              row_frame = tk.Frame(inner, bg=C["surface0"])
              row_frame.pack(fill="x", padx=20, pady=5)
              # Accent bar
              tk.Frame(row_frame, bg=color, height=3).pack(fill="x")
  
              body = tk.Frame(row_frame, bg=C["surface0"])
              body.pack(fill="both", padx=16, pady=10)
  
              info = tk.Frame(body, bg=C["surface0"])
              info.pack(side="left", fill="y", padx=(0, 16))
              tk.Label(info, text=f"{icon}  {name}",
                       font=("Segoe UI", 9),
                       bg=C["surface0"], fg=C["overlay1"],
                       width=14, anchor="w").pack(anchor="w")
              lbl = tk.Label(info, text="—",
                             font=("Consolas", 16, "bold"),
                             bg=C["surface0"], fg=color)
              lbl.pack(anchor="w", pady=(4, 8))
              self._monitor_labels[name] = (lbl, color)
  
              # AnimatedBar — identik HTML .bar-fill
              abar = AnimatedBar(info, base_color=color, width=140)
              abar.pack(anchor="w")
              self._bars[name] = abar
  
              if has_chart:
                  chart = MiniChart(body, color=color, height=70)
                  chart.pack(side="left", fill="both", expand=True)
                  if name == "CPU Usage":
                      self._chart_cpu = chart
                  elif name == "RAM Usage":
                      self._chart_ram = chart
  
          tk.Frame(inner, bg=C["base"], height=20).pack()
  
      # ══════════════════════════════════════════
      #  TAB SWITCH
      # ══════════════════════════════════════════
      def _switch_tab(self, key):
          self._pg_files.pack_forget()
          self._pg_monitor.pack_forget()
          for k, btn in self._nav_btns.items():
              btn.set_active(k == key)
          self._active_tab = key
          if key == "files":
              self._pg_files.pack(fill="both", expand=True)
              self.page_title.config(text="📁  File Explorer")
          elif key == "monitor":
              self._pg_monitor.pack(fill="both", expand=True)
              self.page_title.config(text="📊  Monitor Server")
  
      # ══════════════════════════════════════════
      #  CONNECTION
      # ══════════════════════════════════════════
      def _connect(self):
          def task():
              try:
                  self.ssh = connect_ssh()
                  def _ok():
                      self._conn_dot_widget.set_state("ok")
                      self._conn_text.config(text=" Terhubung", fg=C["green"])
                      self._conn_host.config(text=f"{SSH_USER}@{SSH_HOST}:{SSH_PORT}")
                      self.toast.show(f"Terhubung ke {SSH_HOST}", "ok")
                  self.root.after(0, _ok)
                  self.log.log(f"SSH terhubung ke {SSH_HOST}:{SSH_PORT} sebagai {SSH_USER}")
                  self._refresh()
                  self._start_monitor()
              except Exception as e:
                  def _fail():
                      self._conn_dot_widget.set_state("fail")
                      self._conn_text.config(text=" Gagal terhubung", fg=C["red"])
                      self._conn_host.config(text=str(e)[:30])
                      self.toast.show(f"Koneksi gagal: {e}", "error", 4000)
                  self.root.after(0, _fail)
                  self.log.log(f"ERROR koneksi: {e}", C["red"])
          threading.Thread(target=task, daemon=True).start()
  
      # ══════════════════════════════════════════
      #  FILE LIST
      # ══════════════════════════════════════════
      def _refresh(self):
          def task():
              try:
                  resp = json.loads(socket_request("list_files"))
                  if resp["status"] == "ok":
                      self.files = sorted(resp["files"])
                      self.root.after(0, self._render_files)
                      self.log.log(f"Refresh: {len(self.files)} file")
              except Exception as e:
                  self.root.after(0, lambda: self.toast.show(f"Refresh gagal: {e}", "error"))
                  self.log.log(f"ERROR refresh: {e}", C["red"])
          threading.Thread(target=task, daemon=True).start()
  
      def _render_files(self):
          self.file_lb.delete(0, "end")
          q = self.search_var.get().lower()
          shown = [f for f in self.files if q in f.lower()]
          for f in shown:
              icon = ("📋" if f.endswith(".json") else
                      "📄" if f.endswith(".xml")  else
                      "🐍" if f.endswith(".py")   else "📁")
              self.file_lb.insert("end", f"  {icon}  {f}")
          self.breadcrumb.config(text=f"  {len(shown)} file")
          self.file_info.config(text=f"  {len(shown)} file")
  
      def _get_selected(self):
          sel = self.file_lb.curselection()
          if not sel: return None
          text = self.file_lb.get(sel[0]).strip()
          parts = text.split("  ")
          return parts[-1].strip() if len(parts) > 1 else text
  
      def _on_select(self, event):
          f = self._get_selected()
          if f: self.file_info.config(text=f"  {f}")
  
      def _show_ctx(self, event):
          idx = self.file_lb.nearest(event.y)
          self.file_lb.selection_clear(0, "end")
          self.file_lb.selection_set(idx)
          try:
              self.ctx_menu.tk_popup(event.x_root, event.y_root)
          finally:
              self.ctx_menu.grab_release()
  
      # ══════════════════════════════════════════
      #  OPEN FILE
      # ══════════════════════════════════════════
      def _open_file(self):
          filename = self._get_selected()
          if not filename: return
          if self._edit_mode:
              if not messagebox.askyesno("Edit Aktif",
                                         "Ada perubahan belum disimpan. Keluar?"):
                  return
              self._cancel_edit()
  
          def task():
              try:
                  out, err = remote_exec(self.ssh, f"cat ~/server/{filename}")
                  if err and not out:
                      self.root.after(0, lambda: self.toast.show(f"Error: {err}", "error"))
                      return
                  ft = ("json" if filename.endswith(".json") else
                        "xml"  if filename.endswith(".xml")  else
                        "py"   if filename.endswith(".py")   else "txt")
                  content = out
                  if ft == "json":
                      try:
                          content = json.dumps(json.loads(out), indent=2, ensure_ascii=False)
                      except: pass
  
                  def _do():
                      self._cur_file = filename
                      self._cur_ft   = ft
  
                      # Sembunyikan empty state
                      self._empty_state.place_forget()
  
                      self.viewer_title.config(text=f"  {filename}", fg=C["text"])
                      is_editable = ft in ("json", "xml", "py")
                      if is_editable:
                          self._vbtn_edit.pack(side="right", padx=8, pady=5)
                          self.shortcut_hint.pack(side="left", padx=4)
                      else:
                          self._vbtn_edit.pack_forget()
                          self.shortcut_hint.pack_forget()
                      self._vbtn_dl.pack(side="right", padx=4, pady=5)
  
                      lines = content.count("\n") + 1
                      ft_color = {"json": C["blue"], "xml": C["peach"],
                                  "py": C["mauve"], "txt": C["overlay0"]}.get(ft, C["overlay0"])
                      self.status_bar.config(
                          text=f"  {lines} baris  ·  {len(content)} karakter  "
                               f"·  {ft.upper()}  ·  Double-click untuk edit cepat",
                          fg=C["overlay0"])
  
                      highlight(self.viewer, content, ft)
                      self._update_line_nums()
  
                  self.root.after(0, _do)
                  self.log.log(f"Buka: {filename}")
              except Exception as e:
                  self.root.after(0, lambda: self.toast.show(f"Error: {e}", "error"))
                  self.log.log(f"ERROR buka: {e}", C["red"])
          threading.Thread(target=task, daemon=True).start()
  
      # ══════════════════════════════════════════
      #  EDIT MODE
      # ══════════════════════════════════════════
      def _on_viewer_double_click(self, event):
          if self._cur_file and not self._edit_mode:
              if self._cur_ft in ("json", "xml", "py"):
                  self._toggle_edit()
  
      def _shortcut_save(self, event=None):
          if self._edit_mode: self._save_inline()
          return "break"
  
      def _shortcut_cancel(self, event=None):
          if self._edit_mode: self._cancel_edit()
          return "break"
  
      def _toggle_edit(self):
          if not self._cur_file: return
          self._edit_mode = True
          self.viewer.config(state="normal", bg=C["base"],
                             insertbackground=C["blue"])
  
          # Title → yellow + [EDIT] badge
          self.viewer_title.config(text=f"  ✏️  {self._cur_file}", fg=C["yellow"])
          self.edit_badge.config(text=" [EDIT]",
                                 bg=C["surface0"],
                                 fg=C["yellow"])
  
          # Swap buttons
          self._vbtn_edit.pack_forget()
          self._vbtn_dl.pack_forget()
          self._vbtn_save.pack(side="right", padx=4, pady=5)
          self._vbtn_cancel.pack(side="right", padx=2, pady=5)
  
          # Update shortcut hint
          self.shortcut_hint.config(text="  Ctrl+S  Simpan  ·  Esc  Batal")
          self.shortcut_hint.pack(side="left", padx=4)
  
          self.viewer.focus_set()
  
      def _cancel_edit(self):
          self._edit_mode = False
          self.edit_badge.config(text="")
          self._vbtn_save.pack_forget()
          self._vbtn_cancel.pack_forget()
          self.shortcut_hint.pack_forget()
          self._vbtn_edit.pack(side="right", padx=8, pady=5)
          self._vbtn_dl.pack(side="right", padx=4, pady=5)
          if self._cur_file: self._open_file()
  
      def _save_inline(self):
          if not self._cur_file: return
          content  = self.viewer.get("1.0", "end-1c")
          filename = self._cur_file
          ft       = self._cur_ft
  
          if ft == "json":
              try: json.loads(content)
              except json.JSONDecodeError as e:
                  self.toast.show(f"JSON tidak valid: {e}", "err", 4000)
                  messagebox.showerror("JSON Error",
                                       f"Syntax error:\n{e}\n\nPerbaiki sebelum simpan.")
                  return
          elif ft == "xml":
              try: ET.fromstring(content)
              except ET.ParseError as e:
                  self.toast.show(f"XML tidak valid: {e}", "err", 4000)
                  messagebox.showerror("XML Error",
                                       f"Syntax error:\n{e}\n\nPerbaiki sebelum simpan.")
                  return
  
          def task():
              try:
                  write_to_server(self.ssh, filename, content)
                  def _done():
                      self._edit_mode = False
                      self.edit_badge.config(text="")
                      self._vbtn_save.pack_forget()
                      self._vbtn_cancel.pack_forget()
                      self.shortcut_hint.pack_forget()
                      self._vbtn_edit.pack(side="right", padx=8, pady=5)
                      self._vbtn_dl.pack(side="right", padx=4, pady=5)
                      self.viewer_title.config(text=f"  {filename}", fg=C["text"])
                      self.viewer.config(bg=C["mantle"])
                      highlight(self.viewer, content, ft)
                      self._update_line_nums()
                      self.toast.show(f"✓  {filename} tersimpan", "ok")
                      self.shortcut_hint.config(
                          text="  Ctrl+S  Simpan  ·  Esc  Batal  ·  Double-click  Edit")
                      self.shortcut_hint.pack(side="left", padx=4)
                  self.root.after(0, _done)
                  self.log.log(f"Disimpan: {filename}")
              except Exception as e:
                  self.root.after(0, lambda: self.toast.show(f"Gagal simpan: {e}", "error"))
                  self.log.log(f"ERROR simpan: {e}", C["red"])
          threading.Thread(target=task, daemon=True).start()
  
      def _update_line_nums(self, event=None):
          content = self.viewer.get("1.0", "end-1c")
          lines   = content.count("\n") + 1
          nums    = "\n".join(str(i) for i in range(1, lines + 1))
          self.line_nums.config(state="normal")
          self.line_nums.delete("1.0", "end")
          self.line_nums.insert("1.0", nums)
          self.line_nums.config(state="disabled")
  
      def _sync_scroll(self, *args):
          self.viewer.yview(*args)
          self.line_nums.yview(*args)
  
      def _sync_scroll_mouse(self, event):
          self.viewer.yview_scroll(int(-1*(event.delta/120)), "units")
          self.line_nums.yview_scroll(int(-1*(event.delta/120)), "units")
          return "break"
  
      # ══════════════════════════════════════════
      #  DOWNLOAD / DELETE
      # ══════════════════════════════════════════
      def _download_file(self):
          filename = self._get_selected() or self._cur_file
          if not filename: return
          def task():
              try:
                  raw  = socket_request("get_file", filename)
                  hdr, _, body = raw.partition(b"\n")
                  meta = json.loads(hdr)
                  if meta["status"] == "ok":
                      path = f"hasil/hasil_{filename}"
                      with open(path, "wb") as f: f.write(body)
                      self.root.after(0, lambda: self.toast.show(f"⬇  {path}", "ok"))
                      self.log.log(f"Download: {path}", C["blue"])
                  else:
                      self.root.after(0, lambda: self.toast.show("Download gagal", "error"))
              except Exception as e:
                  self.root.after(0, lambda: self.toast.show(f"Error: {e}", "error"))
                  self.log.log(f"ERROR download: {e}", C["red"])
          threading.Thread(target=task, daemon=True).start()
  
      def _delete_file(self):
          filename = self._get_selected()
          if not filename: return
          if not messagebox.askyesno("Hapus File",
                                     f"Yakin hapus '{filename}'?\nTidak bisa dibatalkan."):
              return
          def task():
              try:
                  remote_exec(self.ssh, f"rm ~/server/{filename}")
                  self.root.after(0, lambda: (
                      self._refresh(),
                      self.toast.show(f"Dihapus: {filename}", "warn"),
                      self._clear_viewer()
                  ))
                  self.log.log(f"Hapus: {filename}", C["yellow"])
              except Exception as e:
                  self.root.after(0, lambda: self.toast.show(f"Error: {e}", "error"))
          threading.Thread(target=task, daemon=True).start()
  
      def _clear_viewer(self):
          self._cur_file = None
          self._cur_ft   = None
          self._edit_mode = False
          self.viewer_title.config(text="Pilih file untuk melihat isinya",
                                   fg=C["overlay0"])
          self.viewer.config(state="normal")
          self.viewer.delete("1.0", "end")
          self.viewer.config(state="disabled")
          self.line_nums.config(state="normal")
          self.line_nums.delete("1.0", "end")
          self.line_nums.config(state="disabled")
          self.status_bar.config(text="")
          self.edit_badge.config(text="")
          self._vbtn_edit.pack_forget()
          self._vbtn_dl.pack_forget()
          self._vbtn_save.pack_forget()
          self._vbtn_cancel.pack_forget()
          self.shortcut_hint.pack_forget()
          # Tampilkan empty state kembali
          self._empty_state.place(relx=0.5, rely=0.5, anchor="center")
  
      # ══════════════════════════════════════════
      #  CREATE FILE MODAL — AnimatedModal
      # ══════════════════════════════════════════
      def _dlg_create(self):
          if self._modal:
              try: self._modal.close()
              except: pass
  
          modal = AnimatedModal(self.root, "Buat File Baru",
                                width=520, height=510)
          self._modal = modal
          win = modal.win
  
          # ── Body ──
          body = tk.Frame(win, bg=C["base"])
          body.pack(fill="both", expand=True, padx=20, pady=10)
  
          # Format selector
          tk.Label(body, text="FORMAT FILE",
                   font=("Segoe UI", 8, "bold"),
                   bg=C["base"], fg=C["subtext"]).pack(anchor="w", pady=(0, 5))
  
          fmt_var = tk.StringVar(value="JSON")
          fmt_row = tk.Frame(body, bg=C["base"])
          fmt_row.pack(anchor="w", pady=(0, 10))
  
          fmt_btns = {}
          def set_fmt(f):
              fmt_var.set(f)
              for ff, rb in fmt_btns.items():
                  rb.config_btn(
                      bg=C["blue"]    if ff == f else C["surface0"],
                      fg=C["crust"]   if ff == f else C["subtext"])
  
          for fmt_name in ("JSON", "XML"):
              rb = RoundedButton(fmt_row, text=f"  {fmt_name}  ",
                                 command=lambda f=fmt_name: set_fmt(f),
                                 bg=C["blue"] if fmt_name == "JSON" else C["surface0"],
                                 fg=C["crust"] if fmt_name == "JSON" else C["subtext"],
                                 padx=12, pady=5)
              rb.pack(side="left", padx=(0, 6))
              fmt_btns[fmt_name] = rb
  
          # Nama file
          tk.Label(body, text="NAMA FILE  (tanpa ekstensi)",
                   font=("Segoe UI", 8, "bold"),
                   bg=C["base"], fg=C["subtext"]).pack(anchor="w", pady=(0, 4))
          nama_e = tk.Entry(body, bg=C["surface0"], fg=C["text"],
                            insertbackground=C["blue"], relief="flat",
                            font=("Segoe UI", 10),
                            highlightthickness=1,
                            highlightcolor=C["blue"],
                            highlightbackground=C["surface1"])
          nama_e.pack(fill="x", pady=(0, 10), ipady=6)
  
          # Isi file
          tk.Label(body, text="ISI FILE  (nama=nilai per baris · [grup] untuk grouping)",
                   font=("Segoe UI", 8, "bold"),
                   bg=C["base"], fg=C["subtext"]).pack(anchor="w")
          tk.Label(body,
                   text="Contoh:   [mahasiswa]  ·  nama=Syaiful  ·  nim=09011282328111",
                   font=("Consolas", 8), bg=C["surface0"], fg=C["overlay0"],
                   anchor="w", padx=8, pady=4).pack(fill="x", pady=(3, 6))
  
          fields_txt = tk.Text(body, bg=C["surface0"], fg=C["text"],
                               insertbackground=C["blue"], relief="flat",
                               font=("Consolas", 10), height=8,
                               highlightthickness=0, padx=6, pady=4)
          fields_txt.pack(fill="both", expand=True)
  
          # ── Footer ──
          foot = tk.Frame(win, bg=C["surface0"], height=52)
          foot.pack(fill="x", side="bottom")
          foot.pack_propagate(False)
          tk.Frame(foot, bg=C["surface1"], height=1).pack(fill="x")
  
          btn_row = tk.Frame(foot, bg=C["surface0"])
          btn_row.pack(anchor="e", padx=16, pady=10)
  
          RoundedButton(btn_row, text="Batal",
                        command=lambda: modal.close(),
                        bg=C["surface1"], fg=C["text"],
                        padx=14, pady=5).pack(side="left", padx=(0, 8))
  
          def do_create():
              nama = nama_e.get().strip()
              fmt  = fmt_var.get()
              raw  = fields_txt.get("1.0", "end").strip()
              if not nama:
                  self.toast.show("Nama file tidak boleh kosong", "warn")
                  return
              data, cur_grp = {}, None
              for line in raw.splitlines():
                  line = line.strip()
                  if not line: continue
                  if line.startswith("[") and line.endswith("]"):
                      cur_grp = line[1:-1]; data[cur_grp] = {}
                  elif "=" in line:
                      k, _, v = line.partition("=")
                      if cur_grp: data[cur_grp][k.strip()] = v.strip()
                      else:       data[k.strip()] = v.strip()
              ext      = ".json" if fmt == "JSON" else ".xml"
              filename = nama + ext
              if fmt == "JSON":
                  content = json.dumps(data, indent=2, ensure_ascii=False)
              else:
                  r_el = ET.Element("root")
                  dict_to_xml(r_el, data)
                  ET.indent(r_el)
                  content = ('<?xml version="1.0" encoding="UTF-8"?>\n'
                             + ET.tostring(r_el, encoding="unicode"))
  
              def task():
                  try:
                      existing, _ = remote_exec(self.ssh, "ls ~/server/")
                      if filename in existing.split():
                          self.root.after(0, lambda: self.toast.show(
                              f"{filename} sudah ada", "warn"))
                          return
                      write_to_server(self.ssh, filename, content)
                      self.root.after(0, lambda: (
                          modal.close(),
                          self._refresh(),
                          self.toast.show(f"✓  File dibuat: {filename}", "ok")
                      ))
                      self.log.log(f"File baru: {filename}")
                  except Exception as e:
                      self.root.after(0, lambda: self.toast.show(f"Error: {e}", "error"))
                      self.log.log(f"ERROR buat file: {e}", C["red"])
              threading.Thread(target=task, daemon=True).start()
  
          RoundedButton(btn_row, text="💾  Simpan ke Server",
                        command=do_create,
                        bg=C["green"], fg=C["crust"],
                        padx=14, pady=5).pack(side="left")
  
          # PENTING: start() dipanggil SETELAH semua widget di-pack
          # agar konten sudah ada sebelum window ditampilkan
          modal.start()
          nama_e.focus_set()
  
      # ══════════════════════════════════════════
      #  MONITOR
      # ══════════════════════════════════════════
      def _start_monitor(self):
          def loop():
              cmds = {
                  "Hostname":   "hostname",
                  "IP Address": "hostname -I | awk '{print $1}'",
                  "Uptime":     "uptime -p",
                  "CPU Usage":  "grep 'cpu ' /proc/stat | awk '{u=$2+$4; t=$2+$3+$4+$5; if(t>0) printf \"%.1f\", u*100/t}'",
                  "RAM Usage":  "free -m | awk '/^Mem:/ {printf \"%.1f/%.1f\", $3, $2}'",
                  "Disk Usage": "df / | awk 'NR==2 {printf \"%s/%s (%s)\", $3, $2, $5}'",
                  "Total File": "ls ~/server/ | wc -l",
              }
              while True:
                  try:
                      r = {m: remote_exec(self.ssh, c)[0] or "—"
                           for m, c in cmds.items()}
  
                      try:  cpu_pct = float(r["CPU Usage"])
                      except: cpu_pct = 0.0
  
                      try:
                          used_mb, total_mb = map(float, r["RAM Usage"].split("/"))
                          ram_pct = (used_mb / total_mb * 100) if total_mb else 0
                          r["RAM Usage"] = f"{used_mb:.0f}/{total_mb:.0f} MB"
                      except: ram_pct = 0.0
  
                      try:
                          disk_pct = float(r["Disk Usage"].split("(")[1].strip("%)"))
                      except: disk_pct = 0.0
  
                      r["CPU Usage"]  = f"{cpu_pct:.1f}%"
                      r["Total File"] = r["Total File"] + " file"
  
                      def _upd(r=r, cpu=cpu_pct, ram=ram_pct, disk=disk_pct):
                          ts = time.strftime("%H:%M:%S")
                          self.monitor_ts.config(text=f"update {ts}")
                          # Live dot → ok (ripple green)
                          self._live_dot_canvas.set_state("ok")
  
                          for name, (lbl, base_color) in self._monitor_labels.items():
                              val = r.get(name, "—")
                              # Color threshold identik HTML colorByPct()
                              if name == "CPU Usage":
                                  color = (C["red"]   if cpu  > 80 else
                                           C["peach"] if cpu  > 50 else C["peach"])
                              elif name == "RAM Usage":
                                  color = (C["red"]   if ram  > 80 else
                                           C["peach"] if ram  > 50 else base_color)
                              else:
                                  color = base_color
                              lbl.config(text=val, fg=color)
  
                          # AnimatedBar — smooth transition identik HTML
                          for name, abar in self._bars.items():
                              pct = (cpu  if name == "CPU Usage"  else
                                     ram  if name == "RAM Usage"  else disk)
                              # color threshold
                              fc = (C["red"]   if pct > 80 else
                                    C["peach"] if pct > 50 else abar.base_color)
                              abar.set_value(pct, override_color=fc)
  
                          if self._chart_cpu: self._chart_cpu.push(cpu)
                          if self._chart_ram: self._chart_ram.push(ram)
  
                      self.root.after(0, _upd)
                  except Exception:
                      self.root.after(0, lambda: self._live_dot_canvas.set_state("fail"))
                  time.sleep(5)
          threading.Thread(target=loop, daemon=True).start()
  
  
  # ══════════════════════════════════════════════
  #  RUN
  # ══════════════════════════════════════════════
  if __name__ == "__main__":
      root = tk.Tk()
      try: root.tk.call("tk", "scaling", 1.0)
      except: pass
      App(root)
      root.mainloop()
  ```
- Jalankan dengan 
  ```bash
  cd ~/client
  python3 client_gui.py
  ```
  <img width="1281" height="816" alt="image" src="https://github.com/user-attachments/assets/24770379-baa2-48a5-a3bd-6d7a19d7f8cb" />

## Web-based GUI
- Kode program flask_server.py di PC Client
  ```python
  """
  RSManager Web — Flask Backend
  Arsitektur IDENTIK dengan client.py:
    - Socket → list_files, get_file
    - SSH    → write, delete, create, monitor
  """
  from flask import Flask, jsonify, request, send_from_directory, Response
  from flask_cors import CORS
  import paramiko, socket as raw_socket, json, threading, time
  import xml.etree.ElementTree as ET
  
  app = Flask(__name__, static_folder="static", static_url_path="")
  CORS(app)
  
  # ══════════════════════════════════════════════
  #  KONFIGURASI — sama persis dengan client.py
  # ══════════════════════════════════════════════
  SSH_HOST    = "192.168.10.10"
  SSH_PORT    = 2005
  SSH_USER    = "capuchino"
  SSH_PASS    = "besokLibur"
  SOCKET_PORT = 5000
  
  # ══════════════════════════════════════════════
  #  SSH
  # ══════════════════════════════════════════════
  _ssh_lock = threading.Lock()
  _ssh_conn = None
  
  def get_ssh():
      global _ssh_conn
      with _ssh_lock:
          if (_ssh_conn is None
                  or not _ssh_conn.get_transport()
                  or not _ssh_conn.get_transport().is_active()):
              ssh = paramiko.SSHClient()
              ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
              ssh.connect(SSH_HOST, port=SSH_PORT,
                          username=SSH_USER, password=SSH_PASS, timeout=10)
              _ssh_conn = ssh
          return _ssh_conn
  
  def remote_exec(cmd):
      ssh = get_ssh()
      _, out, err = ssh.exec_command(cmd)
      return out.read().decode().strip(), err.read().decode().strip()
  
  # ══════════════════════════════════════════════
  #  SOCKET — sama persis dengan client.py
  # ══════════════════════════════════════════════
  def socket_request(action, filename=""):
      req = json.dumps({"action": action, "filename": filename})
      s = raw_socket.socket(raw_socket.AF_INET, raw_socket.SOCK_STREAM)
      s.settimeout(10)
      s.connect((SSH_HOST, SOCKET_PORT))
      s.sendall(req.encode())
      buf = []
      while True:
          chunk = s.recv(4096)
          if not chunk: break
          buf.append(chunk)
      s.close()
      return b"".join(buf)
  
  # ══════════════════════════════════════════════
  #  WRITE — sama persis dengan client.py
  # ══════════════════════════════════════════════
  def write_to_server(filename, content_str):
      escaped = content_str.replace("'", "'\\''")
      remote_exec(f"echo '{escaped}' > ~/server/{filename}")
  
  # ══════════════════════════════════════════════
  #  ROUTES
  # ══════════════════════════════════════════════
  @app.route("/")
  def index():
      return send_from_directory("static", "index.html")
  
  @app.route("/api/status")
  def api_status():
      try:
          get_ssh()
          return jsonify({"status": "ok", "host": SSH_HOST,
                          "port": SSH_PORT, "user": SSH_USER})
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 503
  
  @app.route("/api/files")
  def api_files():
      try:
          resp = json.loads(socket_request("list_files"))
          if resp["status"] == "ok":
              files = sorted(f for f in resp["files"]
                             if f.endswith(".json") or f.endswith(".xml"))
              return jsonify({"status": "ok", "files": files})
          return jsonify({"status": "error", "message": "Socket error"}), 500
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/files/<filename>")
  def api_get_file(filename):
      if ".." in filename or "/" in filename:
          return jsonify({"status": "error", "message": "Invalid filename"}), 400
      try:
          raw = socket_request("get_file", filename)
          hdr, _, body = raw.partition(b"\n")
          meta = json.loads(hdr)
          if meta["status"] != "ok":
              return jsonify({"status": "error", "message": meta.get("message","?")}), 404
          content = body.decode("utf-8", errors="replace")
          if filename.endswith(".json"):
              try:
                  content = json.dumps(json.loads(content), indent=2, ensure_ascii=False)
              except: pass
          return jsonify({"status": "ok", "filename": filename, "content": content})
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/files/<filename>", methods=["PUT"])
  def api_put_file(filename):
      if ".." in filename or "/" in filename:
          return jsonify({"status": "error", "message": "Invalid filename"}), 400
      try:
          content = request.get_json().get("content", "")
          if filename.endswith(".json"):
              json.loads(content)
          elif filename.endswith(".xml"):
              ET.fromstring(content)
          write_to_server(filename, content)
          return jsonify({"status": "ok"})
      except (json.JSONDecodeError, ET.ParseError) as e:
          return jsonify({"status": "error", "message": f"Syntax error: {e}"}), 422
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/files/<filename>", methods=["DELETE"])
  def api_delete_file(filename):
      if ".." in filename or "/" in filename:
          return jsonify({"status": "error", "message": "Invalid filename"}), 400
      try:
          remote_exec(f"rm ~/server/{filename}")
          return jsonify({"status": "ok"})
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/files", methods=["POST"])
  def api_create_file():
      try:
          data     = request.get_json()
          filename = data["filename"]
          content  = data["content"]
          if ".." in filename or "/" in filename:
              return jsonify({"status": "error", "message": "Invalid filename"}), 400
          existing, _ = remote_exec("ls ~/server/")
          if filename in existing.split():
              return jsonify({"status": "error", "message": f"{filename} sudah ada"}), 409
          write_to_server(filename, content)
          return jsonify({"status": "ok", "filename": filename}), 201
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/files/<filename>/download")
  def api_download_file(filename):
      if ".." in filename or "/" in filename:
          return jsonify({"status": "error", "message": "Invalid filename"}), 400
      try:
          raw = socket_request("get_file", filename)
          hdr, _, body = raw.partition(b"\n")
          meta = json.loads(hdr)
          if meta["status"] != "ok":
              return jsonify({"status": "error"}), 500
          mime = "application/json" if filename.endswith(".json") else "application/xml"
          return Response(body, mimetype=mime,
                          headers={"Content-Disposition": f'attachment; filename="{filename}"'})
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  @app.route("/api/monitor")
  def api_monitor():
      cmds = {
          "Hostname":   "hostname",
          "IP Address": "hostname -I | awk '{print $1}'",
          "Uptime":     "uptime -p",
          "CPU Usage":  "grep 'cpu ' /proc/stat | awk '{u=$2+$4; t=$2+$3+$4+$5; if(t>0) printf \"%.1f\", u*100/t}'",
          "RAM Usage":  "free -m | awk '/^Mem:/ {printf \"%.1f/%.1f\", $3, $2}'",
          "Disk Usage": "df / | awk 'NR==2 {printf \"%s/%s (%s)\", $3, $2, $5}'",
          "Total File": "ls ~/server/ | wc -l",
      }
      try:
          r = {m: remote_exec(c)[0] or "—" for m, c in cmds.items()}
  
          try:    cpu_pct = float(r["CPU Usage"])
          except: cpu_pct = 0.0
  
          try:
              used_mb, total_mb = map(float, r["RAM Usage"].split("/"))
              ram_pct = (used_mb / total_mb * 100) if total_mb else 0
              r["RAM Usage"] = f"{used_mb:.0f}/{total_mb:.0f} MB"
          except: used_mb = total_mb = ram_pct = 0
  
          try:    disk_pct = float(r["Disk Usage"].split("(")[1].strip("%)"))
          except: disk_pct = 0.0
  
          r["CPU Usage"]  = f"{cpu_pct:.1f}%"
          r["Total File"] = r["Total File"] + " file"
  
          return jsonify({
              "status":     "ok",
              "Hostname":   r["Hostname"],
              "IP Address": r["IP Address"],
              "Uptime":     r["Uptime"],
              "Total File": r["Total File"],
              "cpu_pct":    cpu_pct,
              "ram_used":   int(used_mb),
              "ram_total":  int(total_mb),
              "ram_pct":    round(ram_pct, 1),
              "RAM Usage":  r["RAM Usage"],
              "Disk Usage": r["Disk Usage"],
              "disk_pct":   disk_pct,
              "ts":         time.strftime("%H:%M:%S"),
          })
      except Exception as e:
          return jsonify({"status": "error", "message": str(e)}), 500
  
  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=8080, debug=False)
  ```
- Kode program index.html di PC Client (~/client/static/)
  ```bash
  <!DOCTYPE html>
  <html lang="id">
  <head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>⚡ RSManager — Remote Server Manager</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@300;400;500;600;700&family=Outfit:wght@300;400;500;600;700;800&display=swap" rel="stylesheet">
  <style>
  /* ══════════════════════════════════════════════
     CATPPUCCIN MOCHA
  ══════════════════════════════════════════════ */
  :root {
    --base:     #1e1e2e;
    --mantle:   #181825;
    --crust:    #11111b;
    --surface0: #313244;
    --surface1: #45475a;
    --surface2: #585b70;
    --overlay0: #6c7086;
    --overlay1: #7f849c;
    --text:     #cdd6f4;
    --subtext:  #a6adc8;
    --blue:     #89b4fa;
    --green:    #a6e3a1;
    --red:      #f38ba8;
    --yellow:   #f9e2af;
    --peach:    #fab387;
    --mauve:    #cba6f7;
    --teal:     #94e2d5;
    --sky:      #89dceb;
    --sapphire: #74c7ec;
    --lavender: #b4befe;
  }
  
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  
  html, body {
    height: 100%;
    overflow: hidden;
    background: var(--crust);
    font-family: 'Outfit', system-ui, sans-serif;
    color: var(--text);
    font-size: 13px;
  }
  
  ::-webkit-scrollbar { width: 5px; height: 5px; }
  ::-webkit-scrollbar-track { background: transparent; }
  ::-webkit-scrollbar-thumb { background: var(--surface1); border-radius: 99px; }
  ::-webkit-scrollbar-thumb:hover { background: var(--surface2); }
  
  /* ══════════════════════════════════════════════
     LAYOUT ROOT
  ══════════════════════════════════════════════ */
  #app {
    display: flex;
    height: 100vh;
    width: 100vw;
    overflow: hidden;
  }
  
  /* ══════════════════════════════════════════════
     SIDEBAR
  ══════════════════════════════════════════════ */
  #sidebar {
    width: 200px;
    min-width: 200px;
    background: var(--mantle);
    display: flex;
    flex-direction: column;
    overflow: hidden;
    border-right: 1px solid rgba(255,255,255,0.04);
    position: relative;
    z-index: 10;
    /* GPU layer — sidebar tidak pernah berubah isi, baik di-promote */
    will-change: transform;
    transform: translateZ(0);
  }
  
  #sidebar::before {
    content: '';
    position: absolute;
    inset: 0;
    background: repeating-linear-gradient(
      0deg,
      transparent,
      transparent 24px,
      rgba(255,255,255,0.012) 24px,
      rgba(255,255,255,0.012) 25px
    );
    pointer-events: none;
  }
  
  .logo-bar {
    height: 56px;
    background: var(--crust);
    display: flex;
    align-items: center;
    padding: 0 14px;
    font-family: 'Outfit', sans-serif;
    font-size: 15px;
    font-weight: 800;
    color: var(--blue);
    letter-spacing: -0.4px;
    flex-shrink: 0;
    border-bottom: 1px solid rgba(255,255,255,0.05);
    position: relative;
  }
  
  .logo-bar::after {
    content: '';
    position: absolute;
    bottom: 0; left: 14px; right: 14px;
    height: 1px;
    background: linear-gradient(90deg, var(--blue), transparent);
    opacity: 0.4;
  }
  
  .conn-box {
    margin: 8px 10px 0;
    background: var(--surface0);
    border-radius: 5px;
    padding: 7px 9px;
    border: 1px solid rgba(255,255,255,0.04);
  }
  .conn-dot {
    font-size: 11px;
    font-weight: 600;
    color: var(--overlay0);
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .conn-dot.ok   { color: var(--green); }
  .conn-dot.fail { color: var(--red); }
  .conn-dot.wait { color: var(--yellow); }
  
  .dot-pulse {
    width: 7px; height: 7px;
    border-radius: 50%;
    background: currentColor;
    flex-shrink: 0;
    position: relative;
    /* Isolate animation ke layer sendiri */
    will-change: transform;
    transform: translateZ(0);
  }
  .conn-dot.ok .dot-pulse::after {
    content: '';
    position: absolute;
    inset: -3px;
    border-radius: 50%;
    background: var(--green);
    opacity: 0.25;
    animation: ripple 2s infinite;
    /* Pastikan animasi tidak repaint parent */
    will-change: transform, opacity;
  }
  @keyframes ripple {
    0%   { transform: scale(1); opacity: 0.25; }
    100% { transform: scale(2.5); opacity: 0; }
  }
  
  .conn-host {
    font-family: 'JetBrains Mono', monospace;
    font-size: 10px;
    color: var(--overlay1);
    margin-top: 3px;
  }
  
  .sdivider {
    height: 1px;
    background: linear-gradient(90deg, transparent, var(--surface1) 30%, var(--surface1) 70%, transparent);
    margin: 10px 10px;
    flex-shrink: 0;
  }
  
  .nav-section-label {
    font-size: 9px;
    font-weight: 700;
    letter-spacing: 1.2px;
    color: var(--overlay0);
    padding: 0 14px;
    margin-bottom: 4px;
    text-transform: uppercase;
    flex-shrink: 0;
  }
  
  .nav-btn {
    display: flex;
    align-items: center;
    gap: 9px;
    padding: 9px 12px;
    margin: 1px 6px;
    border-radius: 5px;
    cursor: pointer;
    font-family: 'Outfit', sans-serif;
    font-size: 12.5px;
    font-weight: 500;
    color: var(--subtext);
    transition: background 0.15s, color 0.15s;
    user-select: none;
    position: relative;
    flex-shrink: 0;
  }
  .nav-btn:hover {
    background: var(--surface0);
    color: var(--text);
  }
  .nav-btn.active {
    background: var(--surface0);
    color: var(--text);
  }
  .nav-btn.active::before {
    content: '';
    position: absolute;
    left: 0; top: 25%; bottom: 25%;
    width: 3px;
    background: var(--blue);
    border-radius: 0 3px 3px 0;
  }
  .nav-icon { font-size: 14px; }
  
  .side-btn {
    display: flex;
    align-items: center;
    gap: 7px;
    padding: 8px 12px;
    margin: 2px 10px;
    border-radius: 5px;
    cursor: pointer;
    font-family: 'Outfit', sans-serif;
    font-size: 12px;
    font-weight: 700;
    transition: filter 0.15s, transform 0.15s;
    user-select: none;
    flex-shrink: 0;
    border: none;
    outline: none;
  }
  .side-btn.green {
    background: var(--green);
    color: var(--crust);
    box-shadow: 0 2px 8px rgba(166,227,161,0.25);
  }
  .side-btn.green:hover {
    filter: brightness(1.08);
    box-shadow: 0 3px 12px rgba(166,227,161,0.35);
    transform: translateY(-1px);
  }
  .side-btn.gray {
    background: var(--surface1);
    color: var(--text);
  }
  .side-btn.gray:hover {
    background: var(--surface2);
    transform: translateY(-1px);
  }
  
  .sidebar-spacer { flex: 1; min-height: 0; }
  
  .sidebar-footer {
    border-top: 1px solid rgba(255,255,255,0.05);
    padding: 7px 14px;
    font-family: 'JetBrains Mono', monospace;
    font-size: 9.5px;
    color: var(--overlay0);
    flex-shrink: 0;
  }
  
  /* ══════════════════════════════════════════════
     RIGHT COLUMN
  ══════════════════════════════════════════════ */
  #right-col {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    background: var(--crust);
  }
  
  #topbar {
    height: 46px;
    background: var(--surface0);
    display: flex;
    align-items: center;
    padding: 0 16px;
    gap: 8px;
    flex-shrink: 0;
    border-bottom: 1px solid rgba(0,0,0,0.2);
  }
  #page-title {
    font-family: 'Outfit', sans-serif;
    font-size: 14px;
    font-weight: 700;
    color: var(--text);
  }
  #breadcrumb {
    font-size: 11.5px;
    color: var(--overlay0);
    font-family: 'JetBrains Mono', monospace;
  }
  
  #page-area {
    flex: 1;
    background: var(--base);
    overflow: hidden;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }
  
  .page {
    display: none;
    flex: 1;
    min-height: 0;
    overflow: hidden;
  }
  .page.active {
    display: flex;
    flex-direction: column;
    flex: 1;
    min-height: 0;
  }
  
  /* ══════════════════════════════════════════════
     LOG PANEL
     OPTIM: contain:strict → browser tidak perlu
     layout/paint di luar kotak ini
  ══════════════════════════════════════════════ */
  #log-panel {
    height: 130px;
    min-height: 130px;
    background: var(--crust);
    display: flex;
    flex-direction: column;
    flex-shrink: 0;
    border-top: 1px solid rgba(0,0,0,0.3);
    /* Isolasi layout agar scroll log tidak trigger repaint global */
    contain: layout style;
  }
  .log-hdr {
    height: 26px;
    background: var(--surface0);
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 10px;
    font-size: 11px;
    font-weight: 700;
    color: var(--subtext);
    flex-shrink: 0;
    letter-spacing: 0.3px;
  }
  .log-clear-btn {
    background: none;
    border: none;
    color: var(--overlay0);
    cursor: pointer;
    font-size: 10px;
    font-family: 'Outfit', sans-serif;
    padding: 2px 6px;
    border-radius: 3px;
    transition: color 0.15s;
  }
  .log-clear-btn:hover { color: var(--text); background: var(--surface1); }
  #log-box {
    flex: 1;
    overflow-y: auto;
    padding: 3px 6px;
    font-family: 'JetBrains Mono', monospace;
    font-size: 11px;
    color: var(--green);
    line-height: 1.7;
    word-wrap: break-word;
    /* contain:strict pada scroll area mencegah layout thrashing */
    contain: strict;
    /* GPU scroll */
    transform: translateZ(0);
  }
  .log-entry { animation: logIn 0.15s ease; }
  @keyframes logIn { from { opacity: 0; transform: translateX(-4px); } }
  .log-red  { color: var(--red) !important; }
  .log-yel  { color: var(--yellow) !important; }
  .log-blue { color: var(--blue) !important; }
  
  /* ══════════════════════════════════════════════
     FILE PAGE
  ══════════════════════════════════════════════ */
  #page-files { flex-direction: row; }
  
  #file-left {
    width: 220px;
    min-width: 180px;
    background: var(--mantle);
    display: flex;
    flex-direction: column;
    flex-shrink: 0;
    border-right: 2px solid var(--base);
  }
  
  .search-frame {
    background: var(--surface0);
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 0 8px;
    margin: 8px;
    border-radius: 5px;
    border: 1px solid transparent;
    transition: border-color 0.15s;
    flex-shrink: 0;
  }
  .search-frame:focus-within { border-color: var(--blue); }
  .search-icon { color: var(--overlay0); font-size: 12px; flex-shrink: 0; }
  .search-frame input {
    flex: 1;
    background: none;
    border: none;
    outline: none;
    color: var(--text);
    font-family: 'Outfit', sans-serif;
    font-size: 12.5px;
    padding: 7px 0;
  }
  .search-frame input::placeholder { color: var(--overlay0); }
  
  #file-lb {
    flex: 1;
    overflow-y: auto;
    padding: 3px 4px;
    min-height: 0;
    /* Isolasi scroll list file */
    contain: strict;
    transform: translateZ(0);
  }
  
  .file-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 7px 10px;
    border-radius: 5px;
    cursor: pointer;
    color: var(--text);
    font-family: 'Outfit', sans-serif;
    font-size: 12.5px;
    font-weight: 400;
    transition: background 0.12s, color 0.12s;
    user-select: none;
    position: relative;
  }
  .file-item:hover { background: var(--surface0); }
  .file-item.active {
    background: var(--surface0);
    color: var(--blue);
  }
  .file-item.active::before {
    content: '';
    position: absolute;
    left: 0; top: 20%; bottom: 20%;
    width: 2px;
    background: var(--blue);
    border-radius: 0 2px 2px 0;
  }
  .file-icon { font-size: 13px; flex-shrink: 0; }
  .file-name {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  .file-info-bar {
    padding: 5px 10px;
    font-size: 10.5px;
    color: var(--overlay0);
    background: var(--crust);
    font-family: 'JetBrains Mono', monospace;
    flex-shrink: 0;
    border-top: 1px solid rgba(0,0,0,0.2);
    display: flex;
    align-items: center;
    gap: 6px;
    min-height: 26px;
  }
  #file-selected-name {
    color: var(--subtext);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  #file-right {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    background: var(--base);
    min-width: 0;
  }
  
  .viewer-hdr {
    height: 40px;
    min-height: 40px;
    background: var(--surface0);
    display: flex;
    align-items: center;
    padding: 0 10px;
    gap: 6px;
    flex-shrink: 0;
    border-bottom: 1px solid rgba(0,0,0,0.2);
  }
  #viewer-title {
    flex: 1;
    font-family: 'JetBrains Mono', monospace;
    font-size: 12px;
    color: var(--overlay0);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    min-width: 0;
  }
  #viewer-title.loaded  { color: var(--text); }
  #viewer-title.editing { color: var(--yellow); }
  
  #edit-badge {
    font-size: 10px;
    font-weight: 700;
    color: var(--yellow);
    background: rgba(249,226,175,0.12);
    padding: 2px 6px;
    border-radius: 3px;
    display: none;
    flex-shrink: 0;
    font-family: 'JetBrains Mono', monospace;
  }
  
  #hint-label {
    font-size: 10px;
    color: var(--overlay0);
    white-space: nowrap;
    display: none;
    font-family: 'JetBrains Mono', monospace;
    flex-shrink: 0;
    padding: 2px 8px;
    background: var(--surface0);
    border-radius: 3px;
    border: 1px solid var(--surface1);
    letter-spacing: 0.1px;
  }
  
  .vbtn {
    padding: 4px 10px;
    border-radius: 4px;
    border: none;
    cursor: pointer;
    font-family: 'Outfit', sans-serif;
    font-size: 11px;
    font-weight: 700;
    white-space: nowrap;
    transition: filter 0.12s, transform 0.12s;
    flex-shrink: 0;
  }
  .vbtn:hover { filter: brightness(1.1); transform: translateY(-1px); }
  .vbtn:active { transform: translateY(0); }
  #btn-edit   { background: var(--blue);    color: var(--crust); display: none; }
  #btn-save   { background: var(--green);   color: var(--crust); display: none; }
  #btn-cancel { background: var(--surface1);color: var(--text);  display: none; }
  #btn-dl     { background: var(--surface1);color: var(--text);  display: none; }
  
  .viewer-body {
    flex: 1;
    position: relative;
    overflow: hidden;
    background: var(--mantle);
    min-height: 0;
    /* Isolasi viewer body dari repaint global */
    contain: layout style;
  }
  .viewer-body.edit-mode { background: var(--base); }
  
  #line-nums {
    position: absolute;
    top: 0; left: 0;
    width: 42px;
    height: 100%;
    background: var(--crust);
    border-right: 1px solid var(--surface0);
    padding: 4px 0;
    font-family: 'JetBrains Mono', monospace;
    font-size: 11.5px;
    line-height: 1.7;
    color: var(--overlay0);
    text-align: right;
    overflow: hidden;
    user-select: none;
    z-index: 3;
    /* Layer sendiri agar tidak repaint saat scroll */
    will-change: scroll-position;
    transform: translateZ(0);
  }
  
  #empty-state {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    color: var(--overlay0);
    pointer-events: none;
    font-family: 'Outfit', sans-serif;
  }
  .empty-icon {
    font-size: 52px;
    opacity: 0.2;
    animation: floatIcon 3s ease-in-out infinite;
    /* Animasi di layer tersendiri */
    will-change: transform;
  }
  @keyframes floatIcon {
    0%,100% { transform: translateY(0); }
    50%     { transform: translateY(-6px); }
  }
  .empty-text { font-size: 14px; font-weight: 500; }
  .empty-hint {
    font-family: 'JetBrains Mono', monospace;
    font-size: 10.5px;
    color: var(--surface2);
  }
  
  .lnum {
    padding-right: 8px;
    display: block;
    transition: color 0.1s;
  }
  .lnum:hover { color: var(--subtext); }
  
  /* ══════════════════════════════════════════════
     HIGHLIGHT OVERLAY
     OPTIM: layer GPU sendiri, scroll-sync via rAF
  ══════════════════════════════════════════════ */
  #hl-overlay {
    position: absolute;
    top: 0;
    left: 42px;
    right: 0;
    bottom: 0;
    padding: 4px 10px;
    font-family: 'JetBrains Mono', monospace;
    font-size: 11.5px;
    line-height: 1.7;
    white-space: pre;
    overflow: auto;
    pointer-events: none;
    display: none;
    tab-size: 2;
    color: var(--text);
    z-index: 1;
    /* GPU layer untuk overlay — mencegah repaint textarea trigger overlay repaint */
    will-change: transform;
    transform: translateZ(0);
  }
  #hl-overlay::-webkit-scrollbar { display: none; }
  #hl-overlay { scrollbar-width: none; }
  
  #editor {
    position: absolute;
    top: 0;
    left: 42px;
    right: 0;
    bottom: 0;
    background: transparent;
    color: transparent;
    font-family: 'JetBrains Mono', monospace;
    font-size: 11.5px;
    line-height: 1.7;
    padding: 4px 10px;
    border: none;
    outline: none;
    resize: none;
    overflow: auto;
    white-space: pre;
    word-wrap: normal;
    tab-size: 2;
    caret-color: var(--blue);
    z-index: 2;
  }
  #editor[readonly] { cursor: default; }
  #editor.edit-on   { cursor: text; }
  #editor::selection { background: rgba(137,180,250,0.25); }
  
  #status-bar {
    padding: 3px 10px;
    font-size: 10.5px;
    color: var(--overlay0);
    background: var(--crust);
    font-family: 'JetBrains Mono', monospace;
    flex-shrink: 0;
    border-top: 1px solid rgba(0,0,0,0.2);
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .sb-seg { display: flex; align-items: center; gap: 3px; }
  .sb-sep { color: var(--surface2); }
  
  /* ══════════════════════════════════════════════
     SYNTAX HIGHLIGHT
  ══════════════════════════════════════════════ */
  .hj-brace { color: var(--mauve); }
  .hj-bool  { color: var(--red); }
  .hj-num   { color: var(--peach); }
  .hj-str   { color: var(--green); }
  .hj-key   { color: var(--blue); }
  .hx-pro   { color: var(--overlay0); }
  .hx-tag   { color: var(--blue); }
  .hx-attr  { color: var(--peach); }
  .hx-val   { color: var(--green); }
  
  /* ══════════════════════════════════════════════
     MONITOR PAGE
  ══════════════════════════════════════════════ */
  #page-monitor {
    overflow-y: auto;
    background: var(--base);
    padding: 16px 20px 24px;
    contain: layout style;
  }
  
  .mon-hdr-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 14px;
  }
  .mon-title {
    font-family: 'Outfit', sans-serif;
    font-size: 17px;
    font-weight: 800;
    color: var(--text);
    letter-spacing: -0.4px;
  }
  .mon-ts {
    font-size: 11px;
    color: var(--overlay0);
    font-family: 'JetBrains Mono', monospace;
    margin-left: 10px;
  }
  .live-badge {
    display: flex; align-items: center; gap: 7px;
    font-family: 'Outfit', sans-serif;
    font-size: 11px;
    font-weight: 700;
    color: var(--overlay0);
  }
  .live-indicator {
    width: 8px; height: 8px;
    border-radius: 50%;
    background: var(--overlay0);
    position: relative;
    transition: background 0.3s;
    will-change: transform;
    transform: translateZ(0);
  }
  .live-indicator.on { background: var(--green); }
  .live-indicator.on::after {
    content: '';
    position: absolute;
    inset: -4px;
    border-radius: 50%;
    border: 1.5px solid var(--green);
    opacity: 0.4;
    animation: liveRing 2s infinite;
    will-change: transform, opacity;
  }
  @keyframes liveRing {
    0%   { transform: scale(0.8); opacity: 0.5; }
    100% { transform: scale(1.6); opacity: 0; }
  }
  
  .card-row {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 10px;
    margin-bottom: 10px;
  }
  
  .stat-card {
    background: var(--surface0);
    border-radius: 6px;
    overflow: hidden;
    border: 1px solid rgba(255,255,255,0.04);
    transition: transform 0.15s;
  }
  .stat-card:hover { transform: translateY(-2px); }
  .card-accent { height: 3px; }
  .card-body { padding: 10px 12px; }
  .card-label {
    font-size: 11px;
    color: var(--overlay1);
    margin-bottom: 6px;
    font-weight: 500;
  }
  .card-val {
    font-family: 'JetBrains Mono', monospace;
    font-size: 13.5px;
    font-weight: 700;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  .metric-card {
    background: var(--surface0);
    border-radius: 6px;
    overflow: hidden;
    margin-bottom: 10px;
    border: 1px solid rgba(255,255,255,0.04);
  }
  .metric-accent { height: 3px; }
  .metric-body {
    padding: 12px 16px;
    display: flex;
    gap: 16px;
    align-items: flex-start;
  }
  .metric-left { min-width: 155px; }
  .metric-label {
    font-size: 11px;
    color: var(--overlay1);
    margin-bottom: 6px;
    font-weight: 500;
  }
  .metric-val {
    font-family: 'JetBrains Mono', monospace;
    font-size: 22px;
    font-weight: 700;
    margin-bottom: 10px;
    transition: color 0.4s;
  }
  
  .bar-bg {
    height: 6px;
    background: var(--surface1);
    border-radius: 99px;
    overflow: hidden;
    width: 140px;
  }
  .bar-fill {
    height: 100%;
    border-radius: 99px;
    width: 0%;
    transition: width 0.6s cubic-bezier(0.4,0,0.2,1), background 0.4s;
    /* Animasi width di GPU */
    will-change: width;
    transform: translateZ(0);
  }
  
  .metric-right { flex: 1; min-width: 0; }
  .mini-chart {
    width: 100%;
    height: 70px;
    display: block;
    border-radius: 4px;
  }
  
  /* ══════════════════════════════════════════════
     CONTEXT MENU
  ══════════════════════════════════════════════ */
  #ctx-menu {
    position: fixed;
    background: var(--surface0);
    border: 1px solid var(--surface1);
    border-radius: 6px;
    box-shadow: 0 8px 30px rgba(0,0,0,0.55), 0 2px 8px rgba(0,0,0,0.3);
    z-index: 9000;
    min-width: 200px;
    padding: 4px;
    display: none;
    animation: ctxIn 0.12s ease;
    /* Layer tersendiri agar tidak trigger full repaint */
    will-change: transform;
    transform: translateZ(0);
  }
  @keyframes ctxIn {
    from { opacity: 0; transform: scale(0.96) translateY(-4px); }
  }
  
  .ctx-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 8px 12px;
    border-radius: 4px;
    cursor: pointer;
    font-size: 12.5px;
    font-weight: 500;
    color: var(--text);
    transition: background 0.1s;
    user-select: none;
  }
  .ctx-item:hover { background: var(--surface1); }
  .ctx-item.danger { color: var(--red); }
  .ctx-item.danger:hover { background: rgba(243,139,168,0.15); }
  .ctx-sep { height: 1px; background: var(--surface1); margin: 3px 8px; }
  
  /* ══════════════════════════════════════════════
     MODAL
     OPTIM KRITIS: backdrop-filter:blur DIHAPUS
     → penyebab utama lag saat overlay muncul.
     Diganti dengan overlay gelap solid + opacity
     yang ringan dan tidak memerlukan compositing
     seluruh layer di belakang modal.
  ══════════════════════════════════════════════ */
  .modal-bg {
    display: none;
    position: fixed;
    inset: 0;
    /* SEBELUM (berat): background: rgba(17,17,27,0.8); backdrop-filter: blur(4px); */
    /* SESUDAH (ringan): overlay solid + gradient ringan, tanpa blur */
    background: rgba(11,11,19,0.88);
    z-index: 5000;
    align-items: center;
    justify-content: center;
    /* Modal overlay di layer tersendiri — tidak trigger repaint konten di bawah */
    will-change: opacity;
    transform: translateZ(0);
  }
  .modal-bg.open {
    display: flex;
    animation: modalBgIn 0.18s ease;
  }
  @keyframes modalBgIn { from { opacity: 0; } }
  
  .modal-win {
    background: var(--base);
    border-radius: 8px;
    border: 1px solid var(--surface1);
    box-shadow: 0 20px 60px rgba(0,0,0,0.7), 0 4px 16px rgba(0,0,0,0.4);
    width: 520px;
    max-width: 95vw;
    overflow: hidden;
    animation: modalIn 0.2s cubic-bezier(0.34,1.4,0.64,1);
    /* Layer tersendiri agar animasi tidak trigger repaint global */
    will-change: transform, opacity;
    transform: translateZ(0);
  }
  @keyframes modalIn {
    from { opacity: 0; transform: scale(0.93) translateY(10px); }
  }
  
  .modal-hdr {
    height: 48px;
    background: var(--surface0);
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 16px;
    font-family: 'Outfit', sans-serif;
    font-size: 14px;
    font-weight: 700;
    border-bottom: 1px solid rgba(0,0,0,0.2);
  }
  .modal-close {
    background: none;
    border: none;
    color: var(--overlay0);
    cursor: pointer;
    font-size: 20px;
    line-height: 1;
    transition: color 0.1s;
    padding: 0 4px;
  }
  .modal-close:hover { color: var(--red); }
  
  .modal-body { padding: 18px 20px 12px; }
  
  .form-lbl {
    display: block;
    font-family: 'Outfit', sans-serif;
    font-size: 10.5px;
    font-weight: 700;
    color: var(--subtext);
    text-transform: uppercase;
    letter-spacing: 0.6px;
    margin-bottom: 6px;
  }
  .form-hint {
    font-family: 'JetBrains Mono', monospace;
    font-size: 11px;
    color: var(--overlay0);
    background: var(--surface0);
    padding: 7px 10px;
    border-radius: 4px;
    margin-bottom: 12px;
    line-height: 1.6;
  }
  
  .form-inp {
    width: 100%;
    background: var(--surface0);
    border: 1px solid var(--surface1);
    border-radius: 5px;
    color: var(--text);
    font-family: 'Outfit', sans-serif;
    font-size: 13px;
    padding: 8px 11px;
    outline: none;
    transition: border-color 0.15s, box-shadow 0.15s;
    margin-bottom: 14px;
  }
  .form-inp:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 3px rgba(137,180,250,0.12);
  }
  textarea.form-inp {
    font-family: 'JetBrains Mono', monospace;
    font-size: 11.5px;
    resize: vertical;
    min-height: 120px;
    margin-bottom: 0;
    line-height: 1.7;
  }
  
  .fmt-row { display: flex; gap: 6px; margin-bottom: 14px; }
  .fmt-btn {
    padding: 6px 18px;
    border-radius: 5px;
    border: 1px solid var(--surface1);
    background: var(--surface0);
    color: var(--subtext);
    font-family: 'Outfit', sans-serif;
    font-size: 13px;
    font-weight: 700;
    cursor: pointer;
    transition: border-color 0.12s, color 0.12s, background 0.12s;
  }
  .fmt-btn:hover { border-color: var(--overlay1); color: var(--text); }
  .fmt-btn.active {
    background: var(--blue);
    color: var(--crust);
    border-color: var(--blue);
    box-shadow: 0 2px 8px rgba(137,180,250,0.3);
  }
  
  .modal-foot {
    padding: 12px 20px;
    background: var(--surface0);
    display: flex;
    gap: 8px;
    justify-content: flex-end;
    border-top: 1px solid rgba(0,0,0,0.15);
  }
  .btn {
    padding: 7px 18px;
    border-radius: 5px;
    border: none;
    cursor: pointer;
    font-family: 'Outfit', sans-serif;
    font-size: 12px;
    font-weight: 700;
    transition: filter 0.12s, transform 0.12s;
  }
  .btn:hover { filter: brightness(1.1); transform: translateY(-1px); }
  .btn:active { transform: translateY(0); }
  .btn-ghost   { background: var(--surface1); color: var(--text); }
  .btn-primary { background: var(--green); color: var(--crust); box-shadow: 0 2px 8px rgba(166,227,161,0.25); }
  
  /* ══════════════════════════════════════════════
     TOAST
     OPTIM: layer sendiri, tidak trigger repaint
  ══════════════════════════════════════════════ */
  #toast-wrap {
    position: fixed;
    bottom: 22px;
    right: 22px;
    z-index: 9999;
    display: flex;
    flex-direction: column;
    gap: 7px;
    pointer-events: none;
    /* Stacking context baru — tidak ganggu repaint halaman */
    will-change: transform;
    transform: translateZ(0);
  }
  .toast {
    padding: 9px 16px;
    border-radius: 5px;
    font-family: 'Outfit', sans-serif;
    font-size: 12px;
    font-weight: 700;
    box-shadow: 0 6px 20px rgba(0,0,0,0.45);
    animation: tIn 0.2s cubic-bezier(0.34,1.4,0.64,1);
    pointer-events: auto;
    max-width: 340px;
    cursor: default;
    will-change: transform, opacity;
  }
  @keyframes tIn  { from { transform: translateX(20px) scale(0.95); opacity: 0; } }
  @keyframes tOut { to   { transform: translateX(20px) scale(0.95); opacity: 0; } }
  .toast.out { animation: tOut 0.2s ease forwards; }
  .t-ok   { background: var(--green);  color: var(--crust); }
  .t-err  { background: var(--red);    color: var(--crust); }
  .t-warn { background: var(--yellow); color: var(--crust); }
  .t-info { background: var(--blue);   color: var(--crust); }
  
  /* Spinner */
  .spin {
    display: inline-block;
    width: 13px; height: 13px;
    border: 2px solid rgba(255,255,255,0.15);
    border-top-color: var(--blue);
    border-radius: 50%;
    animation: sp 0.7s linear infinite;
    vertical-align: middle;
    will-change: transform;
  }
  @keyframes sp { to { transform: rotate(360deg); } }
  
  .file-loading {
    padding: 20px 10px;
    color: var(--overlay0);
    text-align: center;
    font-size: 12px;
  }
  
  #resize-handle {
    width: 4px;
    background: var(--surface1);
    cursor: col-resize;
    transition: background 0.15s;
    flex-shrink: 0;
  }
  #resize-handle:hover { background: var(--blue); }
  </style>
  </head>
  <body>
  
  <div id="app">
  
    <!-- ══════════════ SIDEBAR ══════════════ -->
    <div id="sidebar">
      <div class="logo-bar">⚡ RSManager</div>
  
      <div class="conn-box">
        <div class="conn-dot wait" id="conn-dot">
          <span class="dot-pulse"></span>
          <span id="conn-text">Menghubungkan…</span>
        </div>
        <div class="conn-host" id="conn-host">—</div>
      </div>
  
      <div class="sdivider"></div>
      <div class="nav-section-label">Navigasi</div>
  
      <div class="nav-btn active" id="nav-files" onclick="switchTab('files')">
        <span class="nav-icon">📁</span> File Explorer
      </div>
      <div class="nav-btn" id="nav-monitor" onclick="switchTab('monitor')">
        <span class="nav-icon">📊</span> Monitor Server
      </div>
  
      <div class="sdivider"></div>
      <div class="nav-section-label">Aksi</div>
  
      <button class="side-btn green" onclick="openCreateModal()">＋ &nbsp;Buat File Baru</button>
      <button class="side-btn gray"  onclick="loadFiles()">⟳ &nbsp;&nbsp;Refresh</button>
  
      <div class="sidebar-spacer"></div>
      <div class="sidebar-footer" id="sidebar-footer">Client: 192.168.10.20</div>
    </div>
  
    <!-- ══════════════ RIGHT COLUMN ══════════════ -->
    <div id="right-col">
  
      <div id="topbar">
        <div id="page-title">📁&nbsp; File Explorer</div>
        <div id="breadcrumb"></div>
      </div>
  
      <div id="page-area">
  
        <!-- FILE PAGE -->
        <div class="page active" id="page-files">
  
          <div id="file-left">
            <div class="search-frame">
              <span class="search-icon">🔍</span>
              <input type="text" id="search-inp" placeholder="Cari file…" oninput="filterFiles()">
            </div>
            <div id="file-lb">
              <div class="file-loading">
                <div class="spin"></div><br>Memuat…
              </div>
            </div>
            <div class="file-info-bar" id="file-count">
              <span id="file-selected-name"></span>
            </div>
          </div>
  
          <div id="resize-handle"></div>
  
          <div id="file-right">
            <div class="viewer-hdr">
              <span id="viewer-title">Pilih file untuk melihat isinya</span>
              <span id="edit-badge">[EDIT]</span>
              <span id="hint-label" style="display:none">Ctrl+S  Simpan  ·  Esc  Batal  ·  Double-click  Edit</span>
              <button class="vbtn" id="btn-dl"     onclick="downloadFile()">⬇️ Download</button>
              <button class="vbtn" id="btn-cancel" onclick="cancelEdit()">✕ Batal</button>
              <button class="vbtn" id="btn-save"   onclick="saveFile()">💾 Simpan</button>
              <button class="vbtn" id="btn-edit"   onclick="enterEdit()">✏️ Edit</button>
            </div>
  
            <div class="viewer-body" id="viewer-body">
              <div id="empty-state">
                <div class="empty-icon">📂</div>
                <div class="empty-text">Belum ada file yang dibuka</div>
                <div class="empty-hint">Pilih file dari panel kiri · Double-click untuk edit</div>
              </div>
              <div id="line-nums"></div>
              <pre id="hl-overlay"></pre>
              <textarea id="editor" spellcheck="false" readonly></textarea>
            </div>
  
            <div id="status-bar" style="display:none">
              <span class="sb-seg" id="sb-lines">—</span>
              <span class="sb-sep">·</span>
              <span class="sb-seg" id="sb-chars">—</span>
              <span class="sb-sep">·</span>
              <span class="sb-seg" id="sb-ft" style="color:var(--blue)">—</span>
              <span class="sb-sep">·</span>
              <span class="sb-seg" id="sb-hint" style="color:var(--overlay0)">Double-click untuk edit cepat</span>
            </div>
          </div>
        </div>
  
        <!-- MONITOR PAGE -->
        <div class="page" id="page-monitor">
  
          <div class="mon-hdr-row">
            <div style="display:flex;align-items:center">
              <span class="mon-title">Server Monitor</span>
              <span class="mon-ts" id="mon-ts"></span>
            </div>
            <div class="live-badge">
              <div class="live-indicator" id="live-dot"></div>
              <span>Live</span>
            </div>
          </div>
  
          <div class="card-row">
            <div class="stat-card">
              <div class="card-accent" style="background:var(--blue)"></div>
              <div class="card-body">
                <div class="card-label">🖥 &nbsp;Hostname</div>
                <div class="card-val" id="m-hostname" style="color:var(--blue)">—</div>
              </div>
            </div>
            <div class="stat-card">
              <div class="card-accent" style="background:var(--teal)"></div>
              <div class="card-body">
                <div class="card-label">🌐 &nbsp;IP Address</div>
                <div class="card-val" id="m-ip" style="color:var(--teal)">—</div>
              </div>
            </div>
            <div class="stat-card">
              <div class="card-accent" style="background:var(--green)"></div>
              <div class="card-body">
                <div class="card-label">⏱ &nbsp;Uptime</div>
                <div class="card-val" id="m-uptime" style="color:var(--green);font-size:11px">—</div>
              </div>
            </div>
            <div class="stat-card">
              <div class="card-accent" style="background:var(--yellow)"></div>
              <div class="card-body">
                <div class="card-label">📁 &nbsp;Total File</div>
                <div class="card-val" id="m-files" style="color:var(--yellow)">—</div>
              </div>
            </div>
          </div>
  
          <div class="metric-card">
            <div class="metric-accent" style="background:var(--peach)"></div>
            <div class="metric-body">
              <div class="metric-left">
                <div class="metric-label">⚙️ &nbsp;CPU Usage</div>
                <div class="metric-val" id="m-cpu" style="color:var(--peach)">—</div>
                <div class="bar-bg"><div class="bar-fill" id="bar-cpu" style="background:var(--peach)"></div></div>
              </div>
              <div class="metric-right">
                <canvas class="mini-chart" id="chart-cpu"></canvas>
              </div>
            </div>
          </div>
  
          <div class="metric-card">
            <div class="metric-accent" style="background:var(--mauve)"></div>
            <div class="metric-body">
              <div class="metric-left">
                <div class="metric-label">💾 &nbsp;RAM Usage</div>
                <div class="metric-val" id="m-ram" style="color:var(--mauve)">—</div>
                <div class="bar-bg"><div class="bar-fill" id="bar-ram" style="background:var(--mauve)"></div></div>
              </div>
              <div class="metric-right">
                <canvas class="mini-chart" id="chart-ram"></canvas>
              </div>
            </div>
          </div>
  
          <div class="metric-card">
            <div class="metric-accent" style="background:var(--sapphire)"></div>
            <div class="metric-body">
              <div class="metric-left">
                <div class="metric-label">🗄 &nbsp;Disk Usage</div>
                <div class="metric-val" id="m-disk" style="color:var(--sapphire)">—</div>
                <div class="bar-bg"><div class="bar-fill" id="bar-disk" style="background:var(--sapphire)"></div></div>
              </div>
              <div class="metric-right" id="m-disk-detail"
                   style="padding-top:6px;font-family:'JetBrains Mono',monospace;font-size:11px;color:var(--overlay0);line-height:1.9">—</div>
            </div>
          </div>
  
        </div>
  
      </div>
  
      <!-- Log Panel -->
      <div id="log-panel">
        <div class="log-hdr">
          📋&nbsp; Log Aktivitas
          <button class="log-clear-btn" onclick="clearLog()">✕ Hapus</button>
        </div>
        <div id="log-box"></div>
      </div>
  
    </div>
  </div>
  
  <!-- Context Menu -->
  <div id="ctx-menu">
    <div class="ctx-item" onclick="ctxOpen()">📄 &nbsp;Buka &amp; Lihat Isi</div>
    <div class="ctx-sep"></div>
    <div class="ctx-item" onclick="ctxDownload()">⬇️ &nbsp;Download ke Lokal</div>
    <div class="ctx-sep"></div>
    <div class="ctx-item danger" onclick="ctxDelete()">🗑️ &nbsp;Hapus File</div>
  </div>
  
  <!-- Modal -->
  <div class="modal-bg" id="modal-create">
    <div class="modal-win">
      <div class="modal-hdr">
        ＋&nbsp; Buat File Baru
        <button class="modal-close" onclick="closeModal()">×</button>
      </div>
      <div class="modal-body">
        <label class="form-lbl">Format File</label>
        <div class="fmt-row">
          <button class="fmt-btn active" id="fmt-json" onclick="setFmt('JSON')">JSON</button>
          <button class="fmt-btn"        id="fmt-xml"  onclick="setFmt('XML')">XML</button>
        </div>
        <label class="form-lbl" for="new-name">
          Nama File
          <span style="font-weight:400;text-transform:none;letter-spacing:0;color:var(--overlay0)">(tanpa ekstensi)</span>
        </label>
        <input class="form-inp" type="text" id="new-name" placeholder="contoh: mahasiswa">
        <label class="form-lbl">
          Isi File
          <span style="font-weight:400;text-transform:none;letter-spacing:0;color:var(--overlay0)">(nama=nilai per baris · [grup] untuk grouping)</span>
        </label>
        <div class="form-hint">Contoh:&nbsp;&nbsp;[mahasiswa]&nbsp; · &nbsp;nama=Syaiful&nbsp; · &nbsp;nim=09011282328111</div>
        <textarea class="form-inp" id="new-content" placeholder="[grup]&#10;kunci=nilai&#10;kunci2=nilai2"></textarea>
      </div>
      <div class="modal-foot">
        <button class="btn btn-ghost"   onclick="closeModal()">Batal</button>
        <button class="btn btn-primary" onclick="createFile()">💾&nbsp; Simpan ke Server</button>
      </div>
    </div>
  </div>
  
  <div id="toast-wrap"></div>
  
  <script>
  /* ══════════════════════════════════════════════
     CONFIG
  ══════════════════════════════════════════════ */
  const API         = "http://127.0.0.1:8080/api";
  const HISTORY_LEN = 30;
  
  /* ══════════════════════════════════════════════
     STATE
  ══════════════════════════════════════════════ */
  let allFiles  = [];
  let curFile   = null;
  let curFt     = null;
  let isEdit    = false;
  let ctxTarget = null;
  let selFmt    = "JSON";
  let monTimer  = null;
  const cpuHist = Array(HISTORY_LEN).fill(0);
  const ramHist = Array(HISTORY_LEN).fill(0);
  
  /* ══════════════════════════════════════════════
     OPTIM: Debounce highlight
     Saat user mengetik cepat, highlight tidak
     dijalankan tiap keystroke — tunggu 80ms idle.
     Ini mencegah jank saat file besar diedit.
  ══════════════════════════════════════════════ */
  let hlTimer = null;
  function scheduleHL(content, ft) {
    clearTimeout(hlTimer);
    hlTimer = setTimeout(() => applyHL(content, ft), 80);
  }
  
  /* ══════════════════════════════════════════════
     OPTIM: rAF scroll sync
     Gunakan requestAnimationFrame agar sync scroll
     tidak memblokir main thread saat render frame.
  ══════════════════════════════════════════════ */
  let rafScroll = null;
  function syncScroll() {
    if (rafScroll) return;
    rafScroll = requestAnimationFrame(() => {
      const ed = document.getElementById("editor");
      const ln = document.getElementById("line-nums");
      const ov = document.getElementById("hl-overlay");
      ln.scrollTop = ed.scrollTop;
      ov.scrollTop  = ed.scrollTop;
      ov.scrollLeft = ed.scrollLeft;
      rafScroll = null;
    });
  }
  
  /* ══════════════════════════════════════════════
     INIT
  ══════════════════════════════════════════════ */
  document.addEventListener("DOMContentLoaded", () => {
    checkStatus();
    loadFiles();
  
    const ed = document.getElementById("editor");
  
    ed.addEventListener("keydown", e => {
      if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "s") {
        e.preventDefault();
        if (isEdit) saveFile();
      }
      if (e.key === "Escape" && isEdit) cancelEdit();
    });
  
    ed.addEventListener("dblclick", () => {
      if (curFile && !isEdit && (curFt === "json" || curFt === "xml")) {
        enterEdit();
      }
    });
  
    /* OPTIM: debounced highlight saat input */
    ed.addEventListener("input", () => {
      const content = ed.value;
      updateLineNums(content);
      if (curFt) scheduleHL(content, curFt);
    });
  
    /* OPTIM: rAF scroll sync */
    ed.addEventListener("scroll", syncScroll);
  
    document.addEventListener("click", () => {
      document.getElementById("ctx-menu").style.display = "none";
    });
    document.addEventListener("keydown", e => {
      if (e.key === "Escape") document.getElementById("ctx-menu").style.display = "none";
    });
  
    initResizeHandle();
  
    /* OPTIM: ResizeObserver untuk canvas chart
       Hindari forced layout saat window di-resize */
    const ro = new ResizeObserver(() => {
      drawMiniChart("chart-cpu", cpuHist, "#fab387");
      drawMiniChart("chart-ram", ramHist, "#cba6f7");
    });
    const cpuCanvas = document.getElementById("chart-cpu");
    const ramCanvas = document.getElementById("chart-ram");
    if (cpuCanvas) ro.observe(cpuCanvas);
    if (ramCanvas) ro.observe(ramCanvas);
  });
  
  /* ══════════════════════════════════════════════
     STATUS / CONNECTION
  ══════════════════════════════════════════════ */
  async function checkStatus() {
    try {
      const d = await apiFetch("/status");
      setConn(true, `${d.user}@${d.host}:${d.port}`);
      addLog(`SSH terhubung ke ${d.host}:${d.port} sebagai ${d.user}`);
    } catch (e) {
      setConn(false, String(e.message || e));
      addLog(`ERROR koneksi: ${e.message}`, "red");
    }
  }
  
  function setConn(ok, detail) {
    const dot  = document.getElementById("conn-dot");
    const txt  = document.getElementById("conn-text");
    const host = document.getElementById("conn-host");
    dot.className = "conn-dot " + (ok ? "ok" : "fail");
    txt.textContent = ok ? "Terhubung" : "Gagal terhubung";
    host.textContent = detail;
  }
  
  /* ══════════════════════════════════════════════
     FETCH HELPER
  ══════════════════════════════════════════════ */
  async function apiFetch(path, opts = {}) {
    const r = await fetch(API + path, opts);
    const d = await r.json();
    if (d.status !== "ok") throw new Error(d.message || "Server error");
    return d;
  }
  
  /* ══════════════════════════════════════════════
     NAVIGATION
  ══════════════════════════════════════════════ */
  function switchTab(tab) {
    document.querySelectorAll(".page").forEach(p => p.classList.remove("active"));
    document.querySelectorAll(".nav-btn").forEach(b => b.classList.remove("active"));
    document.getElementById("page-" + tab).classList.add("active");
    document.getElementById("nav-" + tab).classList.add("active");
  
    const titles = {
      files:   "📁\u00a0\u00a0 File Explorer",
      monitor: "📊\u00a0\u00a0 Monitor Server"
    };
    document.getElementById("page-title").innerHTML = titles[tab];
  
    if (tab === "monitor") startMonitor();
    else stopMonitor();
  }
  
  /* ══════════════════════════════════════════════
     FILE LIST
  ══════════════════════════════════════════════ */
  async function loadFiles() {
    document.getElementById("file-lb").innerHTML =
      `<div class="file-loading"><div class="spin"></div><br>Memuat…</div>`;
    try {
      const d = await apiFetch("/files");
      allFiles = d.files || [];
      renderFiles(allFiles);
      addLog(`Refresh: ${allFiles.length} file`);
    } catch (e) {
      document.getElementById("file-lb").innerHTML =
        `<div style="padding:12px;color:var(--red);font-size:11px;font-family:'JetBrains Mono',monospace">Gagal: ${e.message}</div>`;
      addLog(`ERROR refresh: ${e.message}`, "red");
      toast(`Refresh gagal: ${e.message}`, "err");
    }
  }
  
  /* OPTIM: Render file list via DocumentFragment
     Menghindari multiple DOM reflow — semua item
     dibangun dulu di memory, baru dimasukkan sekali.
  */
  function renderFiles(files) {
    const lb = document.getElementById("file-lb");
    document.getElementById("breadcrumb").textContent = `  ${files.length} file`;
  
    if (!files.length) {
      lb.innerHTML = `<div style="padding:20px 10px;color:var(--overlay0);text-align:center;font-size:12px">Folder kosong</div>`;
      document.getElementById("file-count").querySelector("#file-selected-name").textContent = "0 file";
      return;
    }
  
    const frag = document.createDocumentFragment();
    files.forEach(f => {
      const icon = f.endsWith(".json") ? "📋"
                  : f.endsWith(".xml")  ? "📄"
                  : f.endsWith(".py")   ? "🐍"
                  :                      "📁";
      const div = document.createElement("div");
      div.className = "file-item" + (f === curFile ? " active" : "");
      div.dataset.f = f;
      div.innerHTML = `<span class="file-icon">${icon}</span><span class="file-name">${f}</span>`;
      div.addEventListener("click",       () => onFileSelect(f));
      div.addEventListener("contextmenu", e  => showCtx(e, f));
      frag.appendChild(div);
    });
  
    lb.innerHTML = "";
    lb.appendChild(frag);
  
    document.getElementById("file-selected-name").textContent = `  ${files.length} file`;
  }
  
  function filterFiles() {
    const q = document.getElementById("search-inp").value.toLowerCase();
    renderFiles(allFiles.filter(f => f.toLowerCase().includes(q)));
  }
  
  function onFileSelect(filename) {
    document.getElementById("file-selected-name").textContent = `  ${filename}`;
    openFile(filename);
  }
  
  /* ══════════════════════════════════════════════
     OPEN FILE
  ══════════════════════════════════════════════ */
  async function openFile(filename) {
    if (isEdit) {
      if (!confirm("Ada perubahan belum disimpan. Keluar?")) return;
      cancelEditState();
    }
  
    document.querySelectorAll(".file-item").forEach(el =>
      el.classList.toggle("active", el.dataset.f === filename)
    );
  
    const ed = document.getElementById("editor");
    ed.value = "";
    document.getElementById("empty-state").style.display = "none";
    document.getElementById("status-bar").style.display = "flex";
    document.getElementById("sb-lines").textContent = "Memuat…";
  
    try {
      const d = await apiFetch(`/files/${encodeURIComponent(filename)}`);
      curFile = filename;
      curFt   = filename.endsWith(".json") ? "json"
              : filename.endsWith(".xml")  ? "xml"
              : filename.endsWith(".py")   ? "py"
              :                             "txt";
  
      ed.value = d.content;
      applyHL(d.content, curFt);
      updateLineNums(d.content);
      updateStatusBar(d.content, curFt);
  
      const vt = document.getElementById("viewer-title");
      vt.textContent = `  ${filename}`;
      vt.className = "loaded";
  
      const isEditable = (curFt === "json" || curFt === "xml" || curFt === "py");
      document.getElementById("btn-edit").style.display = isEditable ? "" : "none";
      document.getElementById("btn-dl").style.display   = "";
      document.getElementById("hint-label").style.display = isEditable ? "" : "none";
      document.getElementById("hint-label").textContent = "Ctrl+S  Simpan  ·  Esc  Batal  ·  Double-click  Edit";
  
      addLog(`Buka: ${filename}`);
    } catch (e) {
      toast(`Error: ${e.message}`, "err");
      addLog(`ERROR buka: ${e.message}`, "red");
    }
  }
  
  /* ══════════════════════════════════════════════
     SYNTAX HIGHLIGHT
  ══════════════════════════════════════════════ */
  function esc(s) {
    return s.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
  }
  
  function hlJson(code) {
    let out = esc(code);
    out = out.replace(/([{}\[\]])/g,          '<span class="hj-brace">$1</span>');
    out = out.replace(/\b(true|false|null)\b/g,'<span class="hj-bool">$1</span>');
    out = out.replace(/\b(-?\d+(?:\.\d+)?(?:[eE][+\-]?\d+)?)\b/g,'<span class="hj-num">$1</span>');
    out = out.replace(/(:\s*)("(?:[^"\\]|\\.)*")/g,'$1<span class="hj-str">$2</span>');
    out = out.replace(/("(?:[^"\\]|\\.)*")(\s*:)/g,'<span class="hj-key">$1</span>$2');
    return out;
  }
  
  function hlXml(code) {
    let out = "";
    const TOKEN = /(<\?[\s\S]*?\?>)|(<!--[\s\S]*?-->)|(<\/[\w:.-]+\s*>)|(<[\w:.-][\s\S]*?\/?>)|([^<]+)/g;
    let m;
    while ((m = TOKEN.exec(code)) !== null) {
      if (m[1]) {
        out += '<span class="hx-pro">' + esc(m[1]) + '</span>';
      } else if (m[2]) {
        out += '<span class="hx-pro">' + esc(m[2]) + '</span>';
      } else if (m[3]) {
        const t = esc(m[3]);
        out += t.replace(/(&lt;\/)([\w:.-]+)/, '$1<span class="hx-tag">$2</span>');
      } else if (m[4]) {
        let t = m[4];
        const nameM = t.match(/^<([\w:.-]+)([\s\S]*)$/);
        if (nameM) {
          let attrs = nameM[2];
          let tagOut = '&lt;<span class="hx-tag">' + esc(nameM[1]) + '</span>';
          attrs = esc(attrs);
          attrs = attrs.replace(/([\w:.-]+)(=)(&quot;[^&]*?&quot;|&#\d+;[^<]*?)/g,
            '<span class="hx-attr">$1</span>$2<span class="hx-val">$3</span>');
          attrs = attrs.replace(/([\w:.-]+)(=)("(?:[^"])*"|'(?:[^']*)')/g,
            '<span class="hx-attr">$1</span>$2<span class="hx-val">$3</span>');
          attrs = attrs.replace(/(\/&gt;|&gt;)$/, '<span class="hx-tag">$1</span>');
          out += tagOut + attrs;
        } else {
          out += esc(t);
        }
      } else if (m[5]) {
        out += esc(m[5]);
      }
    }
    return out;
  }
  
  function applyHL(content, ft) {
    const ov = document.getElementById("hl-overlay");
    if (ft === "json") {
      ov.innerHTML = hlJson(content);
    } else if (ft === "xml") {
      ov.innerHTML = hlXml(content);
    } else {
      ov.innerHTML = esc(content);
    }
    ov.style.display = "block";
  }
  
  /* ══════════════════════════════════════════════
     LINE NUMBERS
  ══════════════════════════════════════════════ */
  function updateLineNums(content) {
    const n   = content.split("\n").length;
    const box = document.getElementById("line-nums");
    /* OPTIM: bangun string sekali, set innerHTML sekali */
    let html = "";
    for (let i = 1; i <= n; i++) html += `<span class="lnum">${i}</span>`;
    box.innerHTML = html;
  }
  
  /* ══════════════════════════════════════════════
     STATUS BAR
  ══════════════════════════════════════════════ */
  function updateStatusBar(content, ft) {
    const lines = content.split("\n").length;
    document.getElementById("status-bar").style.display = "flex";
    document.getElementById("sb-lines").textContent = `${lines} baris`;
    document.getElementById("sb-chars").textContent = `${content.length} karakter`;
    document.getElementById("sb-ft").textContent    = ft.toUpperCase();
    document.getElementById("sb-hint").textContent  = "Double-click untuk edit cepat";
  }
  
  /* ══════════════════════════════════════════════
     EDIT MODE
  ══════════════════════════════════════════════ */
  function enterEdit() {
    if (!curFile) return;
    isEdit = true;
  
    const ed = document.getElementById("editor");
    ed.removeAttribute("readonly");
    ed.classList.add("edit-on");
    document.getElementById("viewer-body").classList.add("edit-mode");
  
    const vt = document.getElementById("viewer-title");
    vt.textContent = `  ✏️  ${curFile}`;
    vt.className = "editing";
    document.getElementById("edit-badge").style.display = "";
    document.getElementById("hint-label").textContent = "Ctrl+S  Simpan  ·  Esc  Batal";
  
    document.getElementById("btn-edit").style.display   = "none";
    document.getElementById("btn-dl").style.display     = "none";
    document.getElementById("btn-save").style.display   = "";
    document.getElementById("btn-cancel").style.display = "";
  
    ed.focus();
  }
  
  function cancelEditState() {
    isEdit = false;
    const ed = document.getElementById("editor");
    ed.setAttribute("readonly", "");
    ed.classList.remove("edit-on");
    document.getElementById("viewer-body").classList.remove("edit-mode");
  
    document.getElementById("edit-badge").style.display  = "none";
    document.getElementById("btn-save").style.display    = "none";
    document.getElementById("btn-cancel").style.display  = "none";
  
    const isEditable = (curFt === "json" || curFt === "xml" || curFt === "py");
    document.getElementById("btn-edit").style.display = isEditable ? "" : "none";
    document.getElementById("btn-dl").style.display   = curFile ? "" : "none";
    document.getElementById("hint-label").textContent = "Ctrl+S  Simpan  ·  Esc  Batal  ·  Double-click  Edit";
  }
  
  function cancelEdit() {
    cancelEditState();
    if (curFile) openFile(curFile);
  }
  
  async function saveFile() {
    if (!curFile) return;
    const content = document.getElementById("editor").value;
  
    if (curFt === "json") {
      try { JSON.parse(content); }
      catch (e) {
        toast(`JSON tidak valid: ${e.message}`, "err", 4000);
        alert(`Syntax error JSON:\n${e.message}\n\nPerbaiki sebelum simpan.`);
        return;
      }
    } else if (curFt === "xml") {
      try {
        const doc = new DOMParser().parseFromString(content, "application/xml");
        const err = doc.querySelector("parseerror");
        if (err) throw new Error(err.textContent);
      } catch (e) {
        toast(`XML tidak valid: ${e.message}`, "err", 4000);
        return;
      }
    }
  
    try {
      await apiFetch(`/files/${encodeURIComponent(curFile)}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ content })
      });
  
      cancelEditState();
      applyHL(content, curFt);
      updateLineNums(content);
      updateStatusBar(content, curFt);
  
      const vt = document.getElementById("viewer-title");
      vt.textContent = `  ${curFile}`;
      vt.className = "loaded";
  
      toast(`✓  ${curFile} tersimpan`, "ok");
      addLog(`Disimpan: ${curFile}`);
    } catch (e) {
      toast(`Gagal simpan: ${e.message}`, "err");
      addLog(`ERROR simpan: ${e.message}`, "red");
    }
  }
  
  /* ══════════════════════════════════════════════
     DOWNLOAD
  ══════════════════════════════════════════════ */
  function downloadFile(fname) {
    const f = fname || curFile;
    if (!f) return;
    const a = document.createElement("a");
    a.href = `${API}/files/${encodeURIComponent(f)}/download`;
    a.download = f;
    a.click();
    addLog(`Download: ${f}`, "blue");
    toast(`⬇  ${f}`, "ok");
  }
  
  /* ══════════════════════════════════════════════
     CONTEXT MENU
  ══════════════════════════════════════════════ */
  function showCtx(e, filename) {
    e.preventDefault();
    e.stopPropagation();
    ctxTarget = filename;
    const m = document.getElementById("ctx-menu");
    m.style.display = "block";
    const x = Math.min(e.clientX, window.innerWidth  - 210);
    const y = Math.min(e.clientY, window.innerHeight - 130);
    m.style.left = x + "px";
    m.style.top  = y + "px";
  }
  function ctxOpen()     { if (ctxTarget) onFileSelect(ctxTarget); hideCtx(); }
  function ctxDownload() { if (ctxTarget) downloadFile(ctxTarget); hideCtx(); }
  async function ctxDelete() {
    if (!ctxTarget) return;
    hideCtx();
    if (!confirm(`Yakin hapus '${ctxTarget}'?\nTidak bisa dibatalkan.`)) return;
    try {
      await apiFetch(`/files/${encodeURIComponent(ctxTarget)}`, { method: "DELETE" });
      if (ctxTarget === curFile) clearViewer();
      toast(`Dihapus: ${ctxTarget}`, "warn");
      addLog(`Hapus: ${ctxTarget}`, "yel");
      loadFiles();
    } catch (e) {
      toast(`Error: ${e.message}`, "err");
    }
    ctxTarget = null;
  }
  function hideCtx() { document.getElementById("ctx-menu").style.display = "none"; }
  
  /* ══════════════════════════════════════════════
     CLEAR VIEWER
  ══════════════════════════════════════════════ */
  function clearViewer() {
    curFile = null; curFt = null; isEdit = false;
    const ed = document.getElementById("editor");
    ed.value = "";
    ed.setAttribute("readonly", "");
    ed.classList.remove("edit-on");
  
    document.getElementById("hl-overlay").style.display  = "none";
    document.getElementById("hl-overlay").innerHTML      = "";
    document.getElementById("viewer-body").classList.remove("edit-mode");
    document.getElementById("empty-state").style.display = "flex";
    document.getElementById("viewer-title").textContent  = "Pilih file untuk melihat isinya";
    document.getElementById("viewer-title").className    = "";
    document.getElementById("line-nums").innerHTML       = "";
    document.getElementById("edit-badge").style.display  = "none";
    document.getElementById("hint-label").style.display  = "none";
    document.getElementById("status-bar").style.display  = "none";
    ["btn-edit","btn-dl","btn-save","btn-cancel"].forEach(id =>
      document.getElementById(id).style.display = "none"
    );
  }
  
  /* ══════════════════════════════════════════════
     CREATE FILE MODAL
  ══════════════════════════════════════════════ */
  function openCreateModal() {
    document.getElementById("new-name").value    = "";
    document.getElementById("new-content").value = "";
    setFmt("JSON");
    document.getElementById("modal-create").classList.add("open");
    setTimeout(() => document.getElementById("new-name").focus(), 120);
  }
  function closeModal() {
    document.getElementById("modal-create").classList.remove("open");
  }
  function setFmt(fmt) {
    selFmt = fmt;
    document.getElementById("fmt-json").classList.toggle("active", fmt === "JSON");
    document.getElementById("fmt-xml").classList.toggle("active",  fmt === "XML");
  }
  
  async function createFile() {
    const nama = document.getElementById("new-name").value.trim();
    const raw  = document.getElementById("new-content").value.trim();
    if (!nama) { toast("Nama file tidak boleh kosong", "warn"); return; }
  
    let data = {}, curGrp = null;
    for (const line of raw.split("\n")) {
      const l = line.trim();
      if (!l) continue;
      if (l.startsWith("[") && l.endsWith("]")) {
        curGrp = l.slice(1, -1);
        data[curGrp] = {};
      } else if (l.includes("=")) {
        const idx = l.indexOf("=");
        const k = l.slice(0, idx).trim();
        const v = l.slice(idx + 1).trim();
        if (curGrp) data[curGrp][k] = v;
        else        data[k] = v;
      }
    }
  
    const ext      = selFmt === "JSON" ? ".json" : ".xml";
    const filename = nama + ext;
    const content  = selFmt === "JSON" ? JSON.stringify(data, null, 2) : buildXml(data);
  
    try {
      await apiFetch("/files", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ filename, content })
      });
      closeModal();
      toast(`✓ File dibuat: ${filename}`, "ok");
      addLog(`File baru: ${filename}`);
      loadFiles();
    } catch (e) {
      toast(`Error: ${e.message}`, "err");
      addLog(`ERROR buat file: ${e.message}`, "red");
    }
  }
  
  function buildXml(data, depth = 0) {
    const pad = "  ".repeat(depth + 1);
    let out = depth === 0 ? '<?xml version="1.0" encoding="UTF-8"?>\n<root>\n' : "";
    for (const [k, v] of Object.entries(data)) {
      if (typeof v === "object") {
        out += `${pad}<${k}>\n${buildXml(v, depth + 1)}${pad}</${k}>\n`;
      } else {
        out += `${pad}<${k}>${v}</${k}>\n`;
      }
    }
    if (depth === 0) out += "</root>";
    return out;
  }
  
  /* ══════════════════════════════════════════════
     MONITOR
  ══════════════════════════════════════════════ */
  function colorByPct(pct, base) {
    const v = parseFloat(pct) || 0;
    if (v > 80) return "#f38ba8";
    if (v > 50) return "#fab387";
    return base;
  }
  
  function drawMiniChart(id, hist, color) {
    const c = document.getElementById(id);
    if (!c) return;
    const W = c.offsetWidth, H = c.offsetHeight;
    if (!W || !H) return;
    c.width  = W;
    c.height = H;
    const ctx = c.getContext("2d");
    ctx.clearRect(0, 0, W, H);
    if (hist.length < 2) return;
  
    const step = W / (hist.length - 1);
  
    ctx.setLineDash([2, 4]);
    ctx.strokeStyle = "#45475a";
    ctx.lineWidth   = 1;
    [25, 50, 75].forEach(pct => {
      const y = H - (pct / 100) * H;
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
    });
    ctx.setLineDash([]);
  
    const r = parseInt(color.slice(1, 3), 16);
    const g = parseInt(color.slice(3, 5), 16);
    const b = parseInt(color.slice(5, 7), 16);
  
    const grad = ctx.createLinearGradient(0, 0, 0, H);
    grad.addColorStop(0, `rgba(${r},${g},${b},0.28)`);
    grad.addColorStop(1, `rgba(${r},${g},${b},0.02)`);
    ctx.fillStyle = grad;
    ctx.beginPath();
    ctx.moveTo(0, H);
    hist.forEach((v, i) => { ctx.lineTo(i * step, H - (v / 100) * (H - 4)); });
    ctx.lineTo((hist.length - 1) * step, H);
    ctx.closePath();
    ctx.fill();
  
    ctx.strokeStyle = color;
    ctx.lineWidth   = 2;
    ctx.lineJoin    = "round";
    ctx.beginPath();
    hist.forEach((v, i) => {
      const x = i * step;
      const y = H - (v / 100) * (H - 4);
      i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
    });
    ctx.stroke();
  
    const lx = (hist.length - 1) * step;
    const ly = H - (hist[hist.length - 1] / 100) * (H - 4);
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.arc(lx, ly, 3, 0, Math.PI * 2);
    ctx.fill();
  }
  
  async function fetchMonitor() {
    try {
      const d = await apiFetch("/monitor");
  
      document.getElementById("live-dot").classList.add("on");
      document.getElementById("mon-ts").textContent = `update ${d.ts}`;
  
      document.getElementById("m-hostname").textContent = d["Hostname"] || "—";
      document.getElementById("m-ip").textContent       = d["IP Address"] || "—";
      document.getElementById("m-uptime").textContent   = d["Uptime"] || "—";
      document.getElementById("m-files").textContent    = d["Total File"] || "—";
  
      const cpuC = colorByPct(d.cpu_pct, "#fab387");
      document.getElementById("m-cpu").textContent      = `${(d.cpu_pct||0).toFixed(1)}%`;
      document.getElementById("m-cpu").style.color      = cpuC;
      document.getElementById("bar-cpu").style.width      = Math.min(d.cpu_pct||0, 100) + "%";
      document.getElementById("bar-cpu").style.background = cpuC;
  
      const ramC = colorByPct(d.ram_pct, "#cba6f7");
      document.getElementById("m-ram").textContent      = d["RAM Usage"] || "—";
      document.getElementById("m-ram").style.color      = ramC;
      document.getElementById("bar-ram").style.width      = Math.min(d.ram_pct||0, 100) + "%";
      document.getElementById("bar-ram").style.background = ramC;
  
      document.getElementById("m-disk").textContent   = d["Disk Usage"] || "—";
      document.getElementById("bar-disk").style.width = Math.min(d.disk_pct||0, 100) + "%";
      document.getElementById("m-disk-detail").innerHTML =
        `Used: ${d.ram_used || "—"} MB<br>Total: ${d.ram_total || "—"} MB`;
  
      cpuHist.push(d.cpu_pct || 0); if (cpuHist.length > HISTORY_LEN) cpuHist.shift();
      ramHist.push(d.ram_pct || 0); if (ramHist.length > HISTORY_LEN) ramHist.shift();
  
      drawMiniChart("chart-cpu", cpuHist, "#fab387");
      drawMiniChart("chart-ram", ramHist, "#cba6f7");
  
    } catch (e) {
      document.getElementById("live-dot").classList.remove("on");
      addLog(`ERROR monitor: ${e.message}`, "red");
    }
  }
  
  function startMonitor() {
    if (monTimer) return;
    fetchMonitor();
    monTimer = setInterval(fetchMonitor, 5000);
  }
  function stopMonitor() {
    clearInterval(monTimer);
    monTimer = null;
  }
  
  /* ══════════════════════════════════════════════
     LOG PANEL
     OPTIM: batasi maksimal 200 entry di DOM
     Terlalu banyak node di log = scroll berat
  ══════════════════════════════════════════════ */
  const LOG_MAX = 200;
  function addLog(msg, color) {
    const box = document.getElementById("log-box");
    const ts  = new Date().toLocaleTimeString("id-ID", { hour12: false });
    const el  = document.createElement("div");
    el.className = "log-entry" + (color === "red" ? " log-red" : color === "yel" ? " log-yel" : color === "blue" ? " log-blue" : "");
    el.textContent = `[${ts}]  ${msg}`;
    box.appendChild(el);
  
    /* Buang entry lama jika melebihi LOG_MAX */
    while (box.childElementCount > LOG_MAX) {
      box.removeChild(box.firstChild);
    }
  
    box.scrollTop = box.scrollHeight;
  }
  function clearLog() {
    document.getElementById("log-box").innerHTML = "";
  }
  
  /* ══════════════════════════════════════════════
     TOAST
  ══════════════════════════════════════════════ */
  function toast(msg, kind = "info", dur = 2500) {
    const cls = { ok: "t-ok", err: "t-err", warn: "t-warn", info: "t-info" }[kind] || "t-info";
    const el  = document.createElement("div");
    el.className = `toast ${cls}`;
    el.textContent = msg;
    document.getElementById("toast-wrap").appendChild(el);
    setTimeout(() => {
      el.classList.add("out");
      setTimeout(() => el.remove(), 220);
    }, dur);
  }
  
  /* ══════════════════════════════════════════════
     RESIZE HANDLE
  ══════════════════════════════════════════════ */
  function initResizeHandle() {
    const handle = document.getElementById("resize-handle");
    const panel  = document.getElementById("file-left");
    let dragging = false, startX = 0, startW = 0;
  
    handle.addEventListener("mousedown", e => {
      dragging = true;
      startX   = e.clientX;
      startW   = panel.offsetWidth;
      document.body.style.cursor = "col-resize";
      document.body.style.userSelect = "none";
    });
    document.addEventListener("mousemove", e => {
      if (!dragging) return;
      const newW = Math.max(160, Math.min(400, startW + (e.clientX - startX)));
      panel.style.width = newW + "px";
    });
    document.addEventListener("mouseup", () => {
      if (!dragging) return;
      dragging = false;
      document.body.style.cursor = "";
      document.body.style.userSelect = "";
    });
  }
  </script>
  </body>
  </html>
  ```
  <img width="1718" height="1090" alt="image" src="https://github.com/user-attachments/assets/1114377d-9f42-4ed7-8051-e273e03a43c1" />
