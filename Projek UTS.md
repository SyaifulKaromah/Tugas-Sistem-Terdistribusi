[**(←) Back**](https://github.com/SyaifulKaromah/Tugas-Sistem-Terdistribusi/blob/main/README.md)

# **PROJECT UTS — Remote Server & Data Exchange (Client-Server Linux)**

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

def sftp_download(ssh, remote_path, local_path):
    sftp = ssh.open_sftp()
    sftp.get(remote_path, local_path)
    sftp.close()
    print(f"[SFTP] {remote_path} → {local_path}")


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


def list_remote_files():
    resp = json.loads(socket_request("list_files"))
    if resp["status"] == "ok":
        print("[FILES] Daftar file di server:")
        for f in resp["files"]:
            print(f"  - {f}")

def get_file(filename):
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

def get_json_data():
    resp = json.loads(socket_request("get_json_data"))
    if resp["status"] == "ok":
        data = resp["data"]
        print("[JSON] Data diterima:")
        print(json.dumps(data, indent=2, ensure_ascii=False))
        with open("hasil/data_olahan.json", "w") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        print("[JSON] Disimpan: hasil/data_olahan.json")

def get_xml_data():
    resp = json.loads(socket_request("get_xml_data"))
    if resp["status"] == "ok":
        data = resp["data"]
        print("[XML] Data diterima (parsed dari server):")
        print(json.dumps(data, indent=2, ensure_ascii=False))
        # Rekonstruksi XML di sisi client
        root = ET.Element("root")
        for k, v in data.items():
            el = ET.SubElement(root, k)
            if isinstance(v, dict):
                for ck, cv in v.items():
                    sub = ET.SubElement(el, ck)
                    sub.text = str(cv) if cv else ""
            else:
                el.text = str(v) if v else ""
        ET.indent(root)
        ET.ElementTree(root).write("hasil/data_rekonstruksi.xml", encoding="unicode", xml_declaration=True)
        print("[XML] Disimpan: hasil/data_rekonstruksi.xml")


# ====== MAIN ======
def main():
    ssh = connect_ssh()

    # Cek isi folder server via Paramiko remote exec
    out, _ = remote_exec(ssh, "ls ~/server/")
    print(f"[REMOTE] Isi folder server: {out}")

    while True:
        print("\n========================================")
        print("  MENU CLIENT")
        print("========================================")
        print("1. List file di server")
        print("2. Download file dari server")
        print("3. Ambil data JSON dari server")
        print("4. Ambil data XML dari server")
        print("5. Jalankan perintah CLI di server (Paramiko)")
        print("6. Download file via SFTP (Paramiko)")
        print("0. Keluar")
        print("========================================")
        choice = input("Pilihan: ").strip()

        if choice == "1":
            list_remote_files()
        elif choice == "2":
            get_file(input("Nama file: ").strip())
        elif choice == "3":
            get_json_data()
        elif choice == "4":
            get_xml_data()
        elif choice == "5":
            out, err = remote_exec(ssh, input("Perintah: ").strip())
            if out: print("[OUT]\n" + out)
            if err: print("[ERR]\n" + err)
        elif choice == "6":
            sftp_download(ssh, input("Remote path: ").strip(), input("Local path: ").strip())
        elif choice == "0":
            break

    ssh.close()
    print("[SSH] Koneksi ditutup")


if __name__ == "__main__":
    main()
```

---

# D. Demonstrasi Projek

> Pastikan bagian A, B, dan C sudah selesai dikonfigurasi sebelum memulai demonstrasi.

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

> Biarkan terminal ini tetap berjalan.

## D.2 Jalankan Client (Terminal 2 — di mesin client)

Buka terminal baru di mesin client:

```bash
cd ~/client
python3 client.py
```

Output awal:

```
[SSH] Terhubung ke server
[REMOTE] Isi folder server: data.json  data.xml  server.py
```

## D.3 Demonstrasi Tiap Fitur

### List File di Server

```
Pilihan: 1

[FILES] Daftar file di server:
  - data.json
  - data.xml
  - server.py
```

### Ambil Data JSON

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

### Ambil Data XML

```
Pilihan: 4

[XML] Data diterima (parsed dari server):
{
  "mahasiswa": { ... },
  "server": { ... }
}
[XML] Disimpan: hasil/data_rekonstruksi.xml
```

### Remote CLI via Paramiko

```
Pilihan: 5
Perintah: cat ~/server/data.json

[OUT]
{
  "mahasiswa": { ... }
}
```

### Download File via SFTP

```
Pilihan: 6
Remote path: /home/capuchino/server/data.json
Local path: hasil/data.json

[SFTP] /home/capuchino/server/data.json → hasil/data.json
```

## D.4 Verifikasi Hasil di Client

```bash
ls ~/client/hasil/
```

Output:

```
data.json
data_olahan.json
data_rekonstruksi.xml
```

## D.5 Verifikasi SSH Filter (Uji Keamanan)

Coba koneksi dari IP selain `192.168.10.20`, server akan menolak:

```
[BLOCKED] 192.168.10.99
```

Coba login SSH dengan user selain `capuchino`:

```
Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```
