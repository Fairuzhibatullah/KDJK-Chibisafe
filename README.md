# Aplikasi Web Chibisafe

## Sekilas Tentang

**Chibisafe** adalah aplikasi web open-source untuk mengunggah, menyimpan, mengelola, dan membagikan file melalui tautan. Aplikasi ini dapat digunakan sebagai layanan penyimpanan file pribadi yang di-host pada server sendiri (*self-hosted*).

Chibisafe mendukung berbagai jenis file, seperti gambar, video, dokumen, dan berkas lainnya. Pengguna juga dapat mengelompokkan file ke dalam album, membuat tautan berbagi, serta mengelola file melalui dashboard web.

Pada proyek ini, Chibisafe direncanakan untuk dijalankan pada VPS dengan sistem operasi Ubuntu 24.04 LTS menggunakan Docker Compose.

### Teknologi yang Digunakan

- Ubuntu 24.04 LTS sebagai sistem operasi server.
- Docker Engine untuk menjalankan aplikasi dalam container.
- Docker Compose untuk mengelola beberapa layanan aplikasi.
- Chibisafe sebagai aplikasi utama.
- Caddy sebagai web server dan pengatur routing aplikasi.
- Penyimpanan lokal VPS untuk database, file upload, dan log aplikasi.

## Instalasi

### 1. Prasyarat

Sebelum melakukan instalasi, pastikan tersedia:

- VPS dengan Ubuntu 24.04 LTS.
- Akses SSH dan hak akses `root` atau pengguna dengan hak `sudo`.
- Docker Engine.
- Docker Compose plugin.
- Koneksi internet untuk mengunduh image Docker.
- Port TCP `24424` dapat diakses, atau tersedia SSH tunnel untuk pengujian lokal.

### 2. Memeriksa Docker

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

### 3. Membuat Direktori Instalasi

Buat direktori khusus untuk Chibisafe:

```bash
mkdir -p /opt/chibisafe
cd /opt/chibisafe
```

Direktori ini digunakan untuk menyimpan konfigurasi Docker Compose, konfigurasi Caddy, database, file upload, dan log aplikasi.

### 4. Membuat File Docker Compose

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

### 5. Membuat File Caddyfile

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

### 6. Menjalankan Chibisafe

Pastikan berada di direktori `/opt/chibisafe`, lalu jalankan:

```bash
docker compose up -d
```

Perintah ini mengunduh image yang dibutuhkan dan menjalankan tiga layanan, yaitu frontend Chibisafe, backend Chibisafe, dan Caddy.

Periksa container yang berjalan:

```bash
docker compose ps
```

Periksa log apabila terjadi masalah:

```bash
docker compose logs --tail=100
```

### 7. Mengakses Aplikasi

Jika port dapat diakses dari luar, buka alamat berikut melalui browser:

```text
http://IP-VPS:24424
```

Ganti `IP-VPS` dengan alamat IP publik server.

Jika port tidak dapat diakses secara langsung, gunakan SSH tunnel dari komputer lokal:

```bash
ssh -p 20217 -L 12424:127.0.0.1:24424 root@IP-VPS
```

Setelah terhubung, buka:

```text
http://localhost:12424
```

Pada instalasi baru, kredensial awal Chibisafe adalah:

- Username: `admin`
- Password: `admin`

**Segera ubah kata sandi bawaan setelah login pertama.** Jangan biarkan kredensial default digunakan pada layanan yang dapat diakses publik.

## Konfigurasi

### 1. Batas Upload File

Chibisafe menyediakan pengaturan melalui dashboard administrasi untuk mengatur batas ukuran file, ekstensi file yang diizinkan, dan konfigurasi layanan lainnya.

Setelah instalasi berhasil, periksa pengaturan tersebut dan sesuaikan dengan kebutuhan server.

### 2. Penyimpanan Data

Data persisten disimpan pada direktori berikut:

| Direktori | Fungsi |
|---|---|
| `database/` | Data database aplikasi |
| `uploads/` | File yang diunggah pengguna |
| `logs/` | Log aplikasi |

Direktori tersebut terhubung ke container melalui konfigurasi volume Docker Compose sehingga data tetap tersimpan ketika container dimulai ulang atau diperbarui.

### 3. Keamanan Server

Beberapa hal yang perlu diperhatikan:

- Ganti kredensial bawaan setelah instalasi.
- Gunakan HTTPS jika aplikasi akan diakses melalui domain publik.
- Jangan membuka port internal backend secara langsung ke internet.
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

- ShareX untuk mengunggah tangkapan layar dari komputer.
- Ekstensi browser untuk membantu proses upload.
- API key untuk integrasi dengan aplikasi lain.
- Aplikasi mobile yang kompatibel dengan instance Chibisafe.

Ketersediaan dan konfigurasi setiap integrasi perlu disesuaikan dengan versi aplikasi yang digunakan.

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
docker compose pull
docker compose up -d
```

Sebelum melakukan pembaruan, buat backup database dan file upload.

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

Perintah ini menghentikan dan menghapus container proyek, tetapi mempertahankan direktori data lokal.

Jangan menggunakan opsi penghapusan volume atau menghapus direktori database dan upload jika data masih diperlukan.

## Otomatisasi

Untuk memudahkan pemeliharaan, beberapa perintah dapat dijalankan secara berkala menggunakan cron atau systemd timer.

Contoh perintah pembaruan manual:

```bash
cd /opt/chibisafe
docker compose pull
docker compose up -d
```

Pembaruan otomatis tanpa pemeriksaan tidak disarankan sebelum aplikasi stabil. Sebaiknya lakukan backup dan uji pembaruan terlebih dahulu.

Backup juga dapat dijadwalkan, tetapi jadwal dan lokasi penyimpanannya perlu disesuaikan dengan kapasitas disk serta kebutuhan pengguna.

## Cara Pemakaian

*Bagian ini dilengkapi setelah aplikasi berhasil dipasang dan diuji.*

### 1. Login

Buka alamat aplikasi melalui browser, lalu masuk menggunakan akun administrator. Ubah kata sandi awal sebelum menggunakan layanan secara rutin.

### 2. Mengunggah File

Pilih file yang ingin diunggah melalui antarmuka aplikasi. Setelah upload berhasil, periksa apakah file muncul di halaman pengelolaan file.

### 3. Membagikan File

Gunakan tautan yang dihasilkan aplikasi untuk membagikan file kepada pengguna lain. Pastikan pengaturan akses sesuai dengan tujuan berbagi.

### 4. Mengelola Album dan File

Gunakan fitur album untuk mengelompokkan file. Fitur pengelolaan, pencarian, dan penandaan file dapat digunakan untuk mempermudah pengorganisasian data.

### 5. Screenshot Pengujian

Tambahkan screenshot asli setelah pengujian selesai, misalnya:

- Halaman login.
- Dashboard utama.
- Proses upload file.
- Hasil upload beserta tautan berbagi.
- Halaman pengaturan administrator.
- Status container Docker yang berjalan.

Contoh format dokumentasi screenshot:

```markdown
![Dashboard Chibisafe](screenshots/dashboard.png)

Gambar 1. Tampilan dashboard Chibisafe setelah login.
```

Buat direktori `screenshots/` dan isi dengan tangkapan layar aktual dari aplikasi.

## Pembahasan

*Bagian ini merupakan evaluasi awal berdasarkan fitur yang tersedia dan perlu diperbarui setelah pengujian langsung.*

### Kelebihan

- Dapat di-host pada server sendiri.
- Mendukung berbagai jenis file.
- Menyediakan tautan untuk membagikan file.
- Mendukung album, pengelolaan file, dan pencarian.
- Mendukung upload berukuran besar melalui mekanisme chunking.
- Dapat diintegrasikan dengan aplikasi lain melalui API.
- Menggunakan Docker sehingga proses deployment relatif terstruktur.

### Kekurangan dan Hal yang Perlu Diperhatikan

- Memerlukan konfigurasi dan pemeliharaan server sendiri.
- Pengguna bertanggung jawab atas backup dan keamanan data.
- Kapasitas penyimpanan dibatasi oleh sumber daya VPS.
- Penggunaan memori dan performa perlu diuji pada server dengan spesifikasi rendah.
- Akses publik memerlukan perhatian khusus terhadap HTTPS, autentikasi, dan batas upload.

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
3. Docker. *Docker Documentation*. https://docs.docker.com/
4. Ubuntu. *Ubuntu Server Documentation*. https://documentation.ubuntu.com/server/
5. Caddy. *Caddy Documentation*. https://caddyserver.com/docs/

---

**Catatan:** README ini merupakan dokumentasi rencana instalasi. Status keberhasilan instalasi, hasil pengujian, screenshot, serta evaluasi berdasarkan pengalaman penggunaan akan ditambahkan setelah Chibisafe berhasil dijalankan pada VPS.
