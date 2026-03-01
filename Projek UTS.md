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
