# Aplikasi Web Chibisafe

## Sekilas Tentang

**Chibisafe** adalah aplikasi web open-source untuk mengunggah, menyimpan, mengelola, dan membagikan file melalui tautan. Aplikasi ini dapat digunakan sebagai layanan penyimpanan file pribadi yang di-host pada server sendiri (*self-hosted*).

Chibisafe mendukung berbagai jenis file, seperti gambar, video, dokumen, dan berkas lainnya. Pengguna juga dapat mengelompokkan file ke dalam album, membuat tautan berbagi, serta mengelola file melalui dashboard web.

Pada proyek ini, Chibisafe dijalankan pada VPS dengan sistem operasi Ubuntu 24.04 LTS menggunakan Docker Compose.

### Teknologi yang Digunakan

- Ubuntu 24.04 LTS sebagai sistem operasi server.
- Docker Engine untuk menjalankan aplikasi dalam container.
- Docker Compose untuk mengelola beberapa layanan aplikasi.
- Chibisafe sebagai aplikasi utama.
- Caddy sebagai web server dan pengatur routing aplikasi.
- Penyimpanan lokal VPS untuk database, file upload, dan log aplikasi.

## Instalasi

### 1. Prasyarat

Sebelum melakukan instalasi, pastikan data berikut telah tersedia dari penyedia VPS:

- Alamat IP Publik VPS (contoh: `103.xxx.xxx.xxx`).
- Username SSH (`root` atau pengguna dengan hak akses `sudo`).
- Port SSH (standar: `22`, atau port khusus dari penyedia VPS).
- Password SSH atau berkas SSH Private Key (`.pem` / `.id_rsa`).
- Port TCP `24424` dapat diakses, atau tersedia SSH tunnel untuk pengujian lokal.

### 2. Menghubungkan ke Server VPS (SSH)

Buka aplikasi terminal pada komputer lokal Anda (PowerShell, Command Prompt, Git Bash pada Windows, atau Terminal pada Linux/macOS).

**a. Menghubungkan dengan port standar (22):**

```bash
ssh root@IP-VPS
```

> Ganti `root` dengan username Anda (jika bukan root) dan `IP-VPS` dengan alamat IP server Anda.

**b. Menghubungkan dengan port SSH khusus:**

Jika penyedia VPS Anda menggunakan port non-standar (misal port `20217`):

```bash
ssh -p 20217 root@IP-VPS
```

**c. Menghubungkan menggunakan berkas SSH Key:**

Jika VPS menggunakan autentikasi key:

```bash
ssh -i /path/ke/private-key root@IP-VPS
```

**Proses saat pertama kali terhubung:**

1. Sistem akan menampilkan peringatan verifikasi *host fingerprint*. Ketik `yes` lalu tekan **Enter**:
   ```text
   The authenticity of host '103.xxx.xxx.xxx' can't be established.
   ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   ```
2. Masukkan password akun VPS Anda saat diminta (teks password memang tidak akan tampil di layar saat diketik).
3. Setelah login berhasil, prompt terminal server akan aktif:
   ```bash
   root@vps-ubuntu:~#
   ```

![Koneksi SSH ke VPS](screenshots/01-ssh-connect.png)  
*Gambar 1. Tampilan saat berhasil terhubung ke server VPS melalui SSH.*

### 3. Instalasi Docker

Setelah terhubung ke terminal VPS, perbarui paket sistem dan pasang dependensi yang diperlukan:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

Tambahkan repository resmi Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Pasang Docker Engine dan plugin Compose:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Referensi lengkap: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

### 4. Memeriksa Docker

Periksa versi Docker:

```bash
docker --version
```

Periksa Docker Compose:

```bash
docker compose version
```

Pastikan Docker sedang berjalan:

```bash
systemctl status docker
```

Jika Docker belum berjalan, aktifkan dengan:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

![Status Layanan Docker](screenshots/02-docker-status.png)  
*Gambar 2. Memeriksa status dan versi Docker yang berjalan di server.*

### 5. Membuat Direktori Instalasi

Buat direktori khusus untuk Chibisafe:

```bash
mkdir -p /opt/chibisafe
cd /opt/chibisafe
```

Direktori ini digunakan untuk menyimpan konfigurasi Docker Compose, konfigurasi Caddy, database, file upload, dan log aplikasi.

Struktur direktori yang dibutuhkan:

```
└── chibisafe
    ├── docker-compose.yml
    └── Caddyfile
```

> **Catatan:** Direktori `database/`, `uploads/`, dan `logs/` akan dibuat secara otomatis oleh Docker saat container pertama kali dijalankan.

### 6. Membuat File Docker Compose

Buat file konfigurasi:

```bash
nano docker-compose.yml
```

Isi dengan konfigurasi berikut:

```yaml
services:
  chibisafe:
    image: chibisafe/chibisafe:latest
    environment:
      - BASE_API_URL=http://chibisafe_server:8000
    expose:
      - "8001"
    restart: unless-stopped

  chibisafe_server:
    image: chibisafe/chibisafe-server:latest
    volumes:
      - ./database:/app/database:rw
      - ./uploads:/app/uploads:rw
      - ./logs:/app/logs:rw
    expose:
      - "8000"
    restart: unless-stopped

  caddy:
    image: caddy:2-alpine
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./uploads:/app/uploads:ro
    ports:
      - "24424:80"
    environment:
      - BASE_URL=":80"
    restart: unless-stopped
```

Simpan file dengan `Ctrl + O`, tekan `Enter`, lalu keluar menggunakan `Ctrl + X`.

**Penjelasan konfigurasi:**

| Layanan | Image | Fungsi |
|---|---|---|
| `chibisafe` | `chibisafe/chibisafe:latest` | Frontend Chibisafe, melayani antarmuka web pada port internal 8001 |
| `chibisafe_server` | `chibisafe/chibisafe-server:latest` | Backend API Chibisafe pada port internal 8000 |
| `caddy` | `caddy:2-alpine` | Web server yang menggabungkan frontend dan backend menjadi satu port (24424) |

> **Penting:** Meskipun sudah menggunakan NGINX Proxy Manager atau Caddy pada host, Caddy di dalam Docker Compose ini tetap diperlukan. Jangan mengubah konfigurasi ini kecuali memahami cara kerja routing Chibisafe, karena dapat menyebabkan aplikasi tidak berfungsi.

### 7. Membuat File Caddyfile

Buat file konfigurasi Caddy:

```bash
nano Caddyfile
```

Masukkan konfigurasi berikut:

```caddyfile
{
    servers {
        trusted_proxies static private_ranges
        client_ip_headers X-Forwarded-For X-Real-IP
    }
}

{$BASE_URL} {
    route {
        file_server * {
            root /app/uploads
            pass_thru
        }

        @api path /api/*
        reverse_proxy @api http://chibisafe_server:8000 {
            header_up Host {http.reverse_proxy.upstream.hostport}
            header_up X-Real-IP {http.request.header.X-Real-IP}
        }

        @docs path /docs*
        reverse_proxy @docs http://chibisafe_server:8000 {
            header_up Host {http.reverse_proxy.upstream.hostport}
            header_up X-Real-IP {http.request.header.X-Real-IP}
        }

        reverse_proxy http://chibisafe:8001 {
            header_up Host {http.reverse_proxy.upstream.hostport}
            header_up X-Real-IP {http.request.header.X-Real-IP}
        }
    }
}
```

Simpan file dengan `Ctrl + O`, tekan `Enter`, lalu keluar menggunakan `Ctrl + X`.

**Penjelasan routing Caddy:**

| Pola URL | Tujuan |
|---|---|
| File di `/app/uploads` | Disajikan langsung oleh Caddy (file_server) |
| `/api/*` | Diteruskan ke backend Chibisafe (port 8000) |
| `/docs*` | Diteruskan ke backend Chibisafe (dokumentasi API) |
| Semua path lainnya | Diteruskan ke frontend Chibisafe (port 8001) |

### 8. Menjalankan Chibisafe

Pastikan berada di direktori `/opt/chibisafe`, lalu jalankan:

```bash
docker compose up -d
```

Perintah ini mengunduh image yang dibutuhkan dan menjalankan tiga layanan, yaitu frontend Chibisafe, backend Chibisafe, dan Caddy.

Periksa container yang berjalan:

```bash
docker compose ps
```

Pastikan ketiga layanan (`chibisafe`, `chibisafe_server`, `caddy`) memiliki status `Up`. Jika salah satu layanan tidak berjalan, periksa log:

```bash
docker compose logs --tail=100
```

Untuk melihat log layanan tertentu:

```bash
docker compose logs chibisafe_server --tail=50
```

![Status Container Docker](screenshots/03-docker-compose-ps.png)  
*Gambar 3. Status container Chibisafe yang berhasil dijalankan.*

### 9. Mengakses Aplikasi

Jika port dapat diakses dari luar, buka alamat berikut melalui browser:

```text
http://IP-VPS:24424
```

Ganti `IP-VPS` dengan alamat IP publik server.

Jika port tidak dapat diakses secara langsung (misalnya karena firewall VPS), gunakan SSH tunnel dari komputer lokal:

```bash
ssh -L 12424:127.0.0.1:24424 root@IP-VPS
```

> **Catatan:** Jika server menggunakan port SSH non-standar (bukan 22), tambahkan opsi `-p PORT_SSH`. Contoh: `ssh -p 20217 -L 12424:127.0.0.1:24424 root@IP-VPS`

Setelah terhubung, buka:

```text
http://localhost:12424
```

### 10. Login Pertama Kali

Pada instalasi baru, kredensial awal Chibisafe adalah:

- Username: `admin`
- Password: `admin`

**Segera ubah kata sandi bawaan setelah login pertama.** Jangan biarkan kredensial default digunakan pada layanan yang dapat diakses publik.

Langkah mengubah kata sandi:

1. Login menggunakan kredensial default.
2. Klik ikon profil di pojok kanan atas dashboard.
3. Masuk ke halaman pengaturan akun.
4. Ubah kata sandi dengan kata sandi baru yang kuat.

![Halaman Login Chibisafe](screenshots/04-login-page.png)  
*Gambar 4. Tampilan halaman login Chibisafe.*

![Dashboard Utama Chibisafe](screenshots/05-dashboard-utama.png)  
*Gambar 5. Tampilan dashboard utama setelah berhasil masuk.*

## Konfigurasi

### 1. Batas Upload File

Chibisafe menyediakan pengaturan melalui dashboard administrasi untuk mengatur batas ukuran file, ekstensi file yang diizinkan, dan konfigurasi layanan lainnya.

Setelah login sebagai administrator, buka menu **Settings** pada sidebar untuk mengakses pengaturan berikut:

- **Service name**: Nama layanan yang ditampilkan pada halaman utama.
- **Maximum file size**: Batas ukuran file maksimum per upload.
- **Chunk size**: Ukuran potongan file untuk upload berukuran besar.
- **Allowed file extensions**: Ekstensi file yang diizinkan untuk diunggah.
- **User registration**: Mengaktifkan atau menonaktifkan pendaftaran pengguna baru.

![Halaman Pengaturan Chibisafe](screenshots/06-settings.png)  
*Gambar 6. Tampilan halaman pengaturan (Settings) pada dashboard administrator.*

### 2. Penyimpanan Data

Data persisten disimpan pada direktori berikut:

| Direktori | Fungsi |
|---|---|
| `database/` | Data database aplikasi (SQLite) |
| `uploads/` | File yang diunggah pengguna |
| `logs/` | Log aplikasi |

Direktori tersebut terhubung ke container melalui konfigurasi volume Docker Compose sehingga data tetap tersimpan ketika container dimulai ulang atau diperbarui.

### 3. Keamanan Server

Beberapa hal yang perlu diperhatikan:

- Ganti kredensial bawaan (`admin`/`admin`) segera setelah instalasi.
- Gunakan HTTPS jika aplikasi akan diakses melalui domain publik.
- Jangan membuka port internal backend (8000 dan 8001) secara langsung ke internet.
- Batasi akses SSH dan gunakan kata sandi yang kuat atau autentikasi kunci SSH.
- Lakukan backup database dan file upload secara berkala.

### 4. Pengaturan Memori

VPS yang digunakan memiliki sumber daya terbatas, sehingga penggunaan CPU dan RAM perlu dipantau selama pengujian.

Periksa penggunaan memori dengan:

```bash
free -h
```

Periksa penggunaan CPU dan memori container:

```bash
docker stats
```

Pengaturan pembatasan memori container dapat dipertimbangkan jika diperlukan, tetapi harus diuji agar aplikasi tidak sering mengalami restart akibat kehabisan memori.

### 5. Plugin dan Integrasi

Chibisafe mendukung beberapa integrasi, antara lain:

- **ShareX** untuk mengunggah tangkapan layar dari komputer.
- **Ekstensi browser** untuk membantu proses upload.
- **API key** untuk integrasi dengan aplikasi lain.

Konfigurasi integrasi dapat diakses melalui dashboard Chibisafe pada menu **Uploaders** setelah login.

## Maintenance

### 1. Memeriksa Status Aplikasi

```bash
cd /opt/chibisafe
docker compose ps
```

### 2. Melihat Log Aplikasi

```bash
docker compose logs --tail=100
```

Untuk mengikuti log secara langsung:

```bash
docker compose logs -f
```

Tekan `Ctrl + C` untuk berhenti melihat log.

### 3. Memperbarui Aplikasi

Periksa pembaruan image resmi, lalu jalankan:

```bash
cd /opt/chibisafe
docker compose pull
docker compose up -d
```

Sebelum melakukan pembaruan, buat backup database dan file upload terlebih dahulu.

### 4. Backup Data

Contoh membuat arsip backup lokal:

```bash
cd /opt/chibisafe
tar -czf "/root/chibisafe-backup-$(date +%F).tar.gz" database uploads
```

Arsip tersebut menyimpan database dan file upload. Untuk perlindungan yang lebih baik, pindahkan salinan backup ke lokasi lain di luar VPS. Backup lokal saja tidak melindungi data apabila VPS mengalami kerusakan.

Pastikan ruang penyimpanan mencukupi sebelum membuat arsip berukuran besar.

### 5. Menghentikan Aplikasi

```bash
cd /opt/chibisafe
docker compose down
```

Perintah ini menghentikan dan menghapus container proyek, tetapi mempertahankan direktori data lokal (`database/`, `uploads/`, `logs/`).

Jangan menggunakan opsi penghapusan volume (`docker compose down -v`) atau menghapus direktori database dan upload jika data masih diperlukan.

## Otomatisasi

Untuk memudahkan pemeliharaan, beberapa perintah dapat dijalankan secara berkala menggunakan cron atau systemd timer.

### Backup Otomatis dengan Cron

Contoh menjadwalkan backup harian pada pukul 02:00:

```bash
crontab -e
```

Tambahkan baris berikut:

```cron
0 2 * * * cd /opt/chibisafe && tar -czf "/root/chibisafe-backup-$(date +\%F).tar.gz" database uploads
```

### Pembaruan Manual

Pembaruan otomatis tanpa pemeriksaan tidak disarankan sebelum aplikasi stabil. Sebaiknya lakukan backup dan uji pembaruan terlebih dahulu menggunakan perintah:

```bash
cd /opt/chibisafe
docker compose pull
docker compose up -d
```

## Cara Pemakaian

### 1. Login

Buka alamat aplikasi melalui browser (`http://IP-VPS:24424` atau `http://localhost:12424` jika menggunakan SSH tunnel), lalu masuk menggunakan akun administrator. Ubah kata sandi awal sebelum menggunakan layanan secara rutin.

### 2. Mengunggah File

1. Setelah login, klik tombol **Upload** atau seret file ke area upload pada halaman utama.
2. Pilih satu atau beberapa file yang ingin diunggah.
3. Tunggu proses upload selesai. File berukuran besar akan secara otomatis diunggah menggunakan mekanisme *chunked upload*.
4. Setelah upload berhasil, file akan muncul di halaman pengelolaan file.

![Proses Upload File](screenshots/07-upload-file.png)  
*Gambar 7. Proses upload file ke antarmuka web Chibisafe.*

![Hasil Upload File](screenshots/08-daftar-file.png)  
*Gambar 8. Tampilan file yang berhasil diunggah dan muncul di daftar file.*

### 3. Membagikan File

1. Klik pada file yang ingin dibagikan di halaman pengelolaan file.
2. Salin tautan (*link*) yang dihasilkan aplikasi.
3. Bagikan tautan tersebut kepada pengguna lain. Pengguna yang memiliki tautan dapat mengunduh file tanpa perlu login.

![Tautan Berbagi File](screenshots/09-share-link.png)  
*Gambar 9. Mengambil tautan (link) publik untuk membagikan file.*

### 4. Mengelola Album dan File

1. Buat album baru melalui menu **Albums** pada sidebar.
2. Tambahkan file ke album untuk mengelompokkan file berdasarkan kategori.
3. Gunakan fitur pencarian untuk menemukan file tertentu.
4. File dapat dihapus melalui halaman pengelolaan file dengan mengklik ikon hapus.

![Pengelolaan Album](screenshots/10-album.png)  
*Gambar 10. Tampilan halaman pengelolaan album file.*

### 5. Mengelola Pengguna (Administrator)

1. Buka menu **Users** pada sidebar (hanya tersedia untuk administrator).
2. Lihat daftar pengguna yang terdaftar.
3. Administrator dapat mengelola akun pengguna dan melihat file yang diunggah oleh setiap pengguna.

![Manajemen Pengguna](screenshots/11-users-management.png)  
*Gambar 11. Halaman manajemen pengguna oleh administrator.*

## Pembahasan

### Kelebihan

- Dapat di-host pada server sendiri sehingga data tetap berada di bawah kontrol pengguna.
- Mendukung berbagai jenis file tanpa batasan format bawaan.
- Menyediakan tautan langsung untuk membagikan file dengan mudah.
- Mendukung album, pengelolaan file, dan pencarian.
- Mendukung upload berukuran besar melalui mekanisme *chunking*.
- Dapat diintegrasikan dengan aplikasi lain melalui API dan uploader seperti ShareX.
- Menggunakan Docker sehingga proses deployment relatif mudah dan konsisten.
- Antarmuka web yang modern dan responsif.

### Kekurangan dan Hal yang Perlu Diperhatikan

- Memerlukan konfigurasi dan pemeliharaan server sendiri.
- Pengguna bertanggung jawab atas backup dan keamanan data.
- Kapasitas penyimpanan dibatasi oleh sumber daya VPS.
- Penggunaan memori dan performa perlu diuji pada server dengan spesifikasi rendah.
- Akses publik memerlukan perhatian khusus terhadap HTTPS, autentikasi, dan batas upload.
- Tidak memiliki fitur kolaborasi seperti penyuntingan dokumen secara bersamaan.

### Perbandingan dengan Aplikasi Sejenis

| Aspek | Chibisafe | Nextcloud | File Browser |
|---|---|---|---|
| Fokus utama | Upload dan berbagi file melalui tautan | Kolaborasi dan penyimpanan cloud | Mengelola file melalui browser |
| Pengelolaan file | Upload, album, tagging, dan tautan berbagi | File, folder, dan fitur kolaborasi | Navigasi dan pengelolaan file |
| Penggunaan | Berbagi file dengan cepat | Cloud pribadi dengan fungsi lebih luas | Antarmuka web untuk file server |
| Kebutuhan konfigurasi | Memerlukan deployment dan pemeliharaan sendiri | Memerlukan deployment dan pemeliharaan sendiri | Memerlukan deployment dan pemeliharaan sendiri |

Pemilihan aplikasi bergantung pada kebutuhan. Chibisafe lebih sesuai apabila tujuan utamanya adalah mengunggah dan membagikan file melalui tautan, sedangkan aplikasi lain dapat lebih sesuai untuk kebutuhan cloud pribadi atau pengelolaan file server.

## Referensi

1. Chibisafe. *Official GitHub Repository*. https://github.com/chibisafe/chibisafe
2. Chibisafe Guides. *Running Chibisafe with Docker*. https://github.com/chibisafe/guides/blob/main/docs/installation/running-with-docker.md
3. Chibisafe. *Official Documentation*. https://chibisafe.app/docs/intro
4. Docker. *Docker Documentation*. https://docs.docker.com/
5. Docker. *Install Docker Engine on Ubuntu*. https://docs.docker.com/engine/install/ubuntu/
6. Ubuntu. *Ubuntu Server Documentation*. https://documentation.ubuntu.com/server/
7. Caddy. *Caddy Documentation*. https://caddyserver.com/docs/
