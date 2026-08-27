# SILINGKAR DLH Kota Palu

**SILINGKAR DLH Kota Palu (Sistem Informasi Lingkungan dan Kebersihan Dinas Lingkungan Hidup Kota Palu)** — portal layanan DLH Kota Palu. Aplikasi ini menyediakan layanan publik (pengaduan, permohonan, pelacakan tiket), informasi dinas, dan panel administrasi internal **SILINGKAR DLH ADMIN** berbasis peran untuk empat bidang: Pengendalian, Sampah & LB3, Tata Penataan, dan RTH. Seluruh antarmuka (portal publik, panel admin, dan aplikasi desktop) memakai splash loading SILINGKAR yang tampil saat aplikasi dibuka dan setiap perpindahan halaman.

## Fitur utama

**Portal publik**

- Pengaduan terpadu di `/pengaduan` untuk bidang Pengendalian, Sampah, RTH, dan Tata Penataan (form Livewire multi-foto + titik lokasi peta). URL lama per bidang (`/pengaduan-pengendalian`, dll.) redirect ke form terpadu.
- Permohonan rekomendasi, pinjam taman (penyewaan taman), registrasi usaha LB3, dan pengajuan RINTEK/PERTEK — masing-masing dengan halaman cek status dan bukti PDF.
- Pelacakan satu pintu di `/lacak` dengan riwayat status (timeline) dan umpan balik/ulasan masyarakat per tiket.
- Berita (`/berita`), profil dinas, halaman UPTD (Lab Lingkungan, TPA Kawatuna), kebijakan privasi, dan syarat & ketentuan.
- Dokumen Tata Lingkungan dari Google Drive, peta persampahan publik, dan monitoring armada GPS (`/armada`, `/api/armada-aktif`).
- Chatbot AI streaming dengan provider OpenAI-compatible (failover otomatis antar provider) yang dikelola dari Pengaturan Admin; kunci provider disimpan terenkripsi di database.
- Mode pemeliharaan yang bisa diaktifkan dari Pengaturan Admin (halaman publik diblokir 503, panel admin tetap bisa diakses, pratinjau via `?preview=1`).

**Panel admin — SILINGKAR DLH ADMIN (`/admin`)**

- Dashboard per peran: KPI per bidang, grafik tren (Chart.js), tugas tertunda, aktivitas terbaru, dan peta sebaran laporan.
- CRUD berbasis registry (`AdminRegistry` + `ResourceController`) dengan filter, pencarian, bulk action, ekspor CSV/XLSX (sinkron atau antrean via `QUEUE_EXPORTS`), dan detail record.
- Modul GIS: layer peta, impor Shapefile, digitasi fitur, bulk visibility/delete, soft delete + restore.
- Audit log, notifikasi internal (polling), ulasan masyarakat, backup/restore database, monitoring kuota Neon/B2, dan manajemen pengguna.

**Aplikasi desktop Windows (Tauri 2)**

- Shell Windows terpisah di folder `desktop/` yang menampilkan panel admin produksi (`https://www.silingkardlhpalu.web.id`) lewat WebView2 — tanpa menduplikasi UI maupun backend.
- Splash loading saat aplikasi dibuka dan pada setiap perpindahan halaman; halaman error + tombol "Coba Lagi" bila server tidak terjangkau.
- Single instance, judul window terkunci "SILINGKAR DLH ADMIN", link/popup eksternal otomatis dibuka di browser default Windows, drag & drop upload berperilaku seperti browser.

**Keamanan bawaan**

- reCAPTCHA v2 Invisible pada form publik, throttling endpoint, security headers (nosniff, frame options, referrer policy, permissions policy, HSTS), trusted proxy dari env, login throttle per identitas + IP, kebijakan password minimal 10 karakter, dan endpoint `/.well-known/security.txt`.
- Sertifikat sosialisasi diakses via token acak (anti-IDOR); API armada hanya mengekspos kolom publik.

## Teknologi

| Area           | Teknologi                                                        |
| -------------- | ---------------------------------------------------------------- |
| Backend        | PHP 8.2+, Laravel 12                                             |
| UI             | Blade, Livewire 4 (single-file components), Alpine.js, Tailwind CSS 4 |
| Frontend libs  | MapLibre GL, Chart.js, Flatpickr, Jodit editor, @resvg/resvg-js  |
| Build          | Vite 7, TypeScript                                               |
| Desktop        | Tauri 2 + WebView2 (shell Windows untuk panel admin produksi)    |
| Database       | PostgreSQL; Neon untuk produksi atau PostgreSQL 16 di Docker     |
| Storage/backup | Local storage dan Backblaze B2 (S3-compatible)                   |
| Paket utama    | spatie/laravel-permission, DomPDF, Intervention Image, HTMLPurifier, Flysystem S3 |
| Integrasi      | portal.gps.id, Google Drive API, provider AI OpenAI-compatible   |

## Arsitektur singkat

Portal publik memakai route Blade + komponen Livewire single-file di `resources/views/components/public/` (form pengaduan, lacak, cek status, armada). Panel admin memakai Blade + Livewire; sebagian besar CRUD ditangani `ResourceController` yang membaca definisi resource dari `app/Support/Admin/AdminRegistry.php`. Model, policy, observer, dan service memisahkan aturan bisnis, otorisasi, nomor tiket, notifikasi, unggahan, GIS, GPS, AI, statistik, serta backup.

Nomor tiket dibuat oleh `TicketGenerator` dengan prefix per bidang: `PDL` (Pengendalian), `SMP` (Sampah & LB3), `RTH`, `TTP` (Tata Penataan), `PJM` (pinjam taman), format `PREFIX-XXXX-XXXX`. Status pengaduan dinormalisasi menjadi dua nilai: **Belum Ditindaklanjuti** dan **Ditindaklanjuti**.

Splash loading bersama dipakai lewat komponen Blade `resources/views/components/splash.blade.php` (`<x-splash />`) di layout publik, layout admin, dan halaman login; aplikasi desktop menyuntikkan splash setara dari shell Tauri untuk halaman yang tidak punya splash sendiri.

Direktori penting:

```text
app/
  Console/Commands/       # GPS, seeder, impor SHP, pembersihan B2, unduh gambar
  Enums/                  # bidang, status, jenis pengaduan, role admin
  Http/Controllers/       # endpoint publik dan admin
  Jobs/                   # backup, restore, ekspor antrean, proses foto
  Livewire/               # ChatBot + trait reCAPTCHA/throttle
  Models/                 # 30+ model domain (pengaduan, permohonan, GIS, GPS, dll.)
  Observers/              # audit log & notifikasi admin otomatis
  Policies/               # otorisasi per resource
  Services/               # AI, GPS, Drive, upload, statistik, GIS, monitoring
  Support/Admin/          # registry, akses, feed notifikasi admin
database/migrations/      # skema PostgreSQL
resources/views/
  components/public/      # komponen Livewire form publik (SFC)
  admin/                  # view panel admin
  pdf/                    # template PDF (bukti, sertifikat, laporan, surat sanksi)
routes/web.php            # rute HTTP
routes/console.php        # scheduler
desktop/                  # aplikasi Windows Tauri 2 (terpisah dari Laravel)
```

### Peran dan akses admin

Role didefinisikan di `App\Enums\AdminRole` (spatie/laravel-permission):

| Role                  | Akses                                                        |
| --------------------- | ------------------------------------------------------------ |
| `admin Website`       | Semua grup (Admin Website): audit log, backup, reset password|
| `bidang-pengendalian` | Grup Pengendalian                                            |
| `bidang-sampah-lb3`   | Grup Sampah & LB3                                            |
| `bidang-tata-penataan`| Grup Tata Penataan                                           |
| `bidang-rth`          | Grup RTH                                                     |

Selain role, kolom `additional_access` pada user memberikan akses per menu (slug resource) di luar grup rolenya. Menu admin terbagi dalam grup: Pengendalian, Sampah & LB3, Tata Penataan, RTH, dan Konten & Sistem (artikel, pengguna, ulasan masyarakat).

## Prasyarat

- PHP 8.2+ dengan `pdo_pgsql`, `gd`, `mbstring`, `zip`, `bcmath`, `intl`, dan `exif`.
- Composer, Node.js, dan npm.
- PostgreSQL atau akun Neon.
- Opsional: Docker Desktop/Compose, Backblaze B2, Google Drive API, akun GPS.id, dan kunci reCAPTCHA.
- Untuk aplikasi desktop: Rust (toolchain `x86_64-pc-windows-msvc`), Node.js, dan WebView2 Runtime (sudah bawaan Windows 10/11).

## Setup lokal

```bash
composer install
copy .env.example .env
php artisan key:generate
```

Atur koneksi PostgreSQL pada `.env`, lalu buat skema dan aset:

```bash
php artisan migrate
php artisan storage:link
npm install
npm run build
```

Untuk role dan akun admin awal, jalankan seeder secara eksplisit:

```bash
php artisan db:seed
# atau setup lengkap (folder storage + seeding)
php artisan dlh:setup-seeder
```

Seeder membuat 5 akun (username: `admin`, `pengendalian`, `sampah-lb3`, `tata-penataan`, `rth`) dengan password acak 16 karakter yang **hanya ditampilkan sekali di console** — simpan segera. Menjalankan ulang seeder tidak menimpa password akun yang sudah ada.

Menjalankan pengembangan:

```bash
composer run dev
```

Perintah tersebut menjalankan web server, queue listener, Pail, dan Vite. Alternatifnya, jalankan `php artisan serve`, `php artisan queue:work`, dan `npm run dev` secara terpisah; `php artisan queue:restart` untuk me-restart queue listener.

## Aplikasi desktop Windows (Tauri 2)

Folder `desktop/` berisi shell Windows yang terpisah dari project Laravel. Aplikasi ini hanya "jendela" WebView2 yang membuka panel admin produksi di `https://www.silingkardlhpalu.web.id` (prefix `ADMIN_PATH`), sehingga login/logout, session/cookie, CSRF, seluruh CRUD, upload, unduh PDF/Excel/CSV, peta GIS, notifikasi polling, dan semua fitur admin berjalan persis seperti di browser.

```bash
cd desktop
npm install
npm run dev     # mode pengembangan
npm run build   # build release + installer NSIS
```

Hasil build release: `desktop/src-tauri/target/release/bundle/nsis/SILINGKAR DLH ADMIN_1.0.0_x64-setup.exe`.

Perilaku shell:

- **Splash loading** tampil saat aplikasi dibuka (halaman `ui/index.html`) dan disuntikkan otomatis pada setiap perpindahan halaman yang belum punya splash sendiri.
- **Probe koneksi**: sebelum masuk panel, aplikasi mengecek server (`GET {origin}/up`). Online → panel dimuat; offline → halaman `ui/error.html` dengan tombol **Coba Lagi**. Selagi sesi berjalan, server dipantau tiap 15 detik; gangguan ≥45 detik berturut-turut memindahkan sesi ke halaman error (pemulihan selalu manual agar form pengguna tidak ter-reload tiba-tiba).
- **Link eksternal**: hanya domain produksi dan halaman internal yang boleh dimuat di webview; URL lain, `mailto:`, dan `tel:` dibuka di browser default Windows.
- **Single instance**: instans kedua otomatis difokuskan ke window yang sudah terbuka.

Konfigurasi khusus desktop (variabel environment, untuk dev/uji):

- `DLH_ADMIN_URL` — mengganti target panel, mis. `DLH_ADMIN_URL=http://127.0.0.1:8000/admin` untuk uji terhadap server lokal.
- `DLH_WEBVIEW_ARGS` — argumen tambahan WebView2, mis. `DLH_WEBVIEW_ARGS="--remote-debugging-port=9223"` untuk debugging.

Domain produksi dan daftar host yang diizinkan dikode di `desktop/src-tauri/src/main.rs` (`ADMIN_BASE_URL`, `ALLOWED_HOSTS`) — ubah di sana bila domain berubah. Profil release sengaja memakai `strip = true` saja (tanpa LTO/`codegen-units=1`) agar `rustc` stabil di mesin Windows dengan RAM terbatas.

## Docker dan produksi

`docker-compose.yml` menyediakan `app` (PHP-FPM multi-stage), `nginx` (port 80, gzip, `client_max_body_size 512M`), `db` (PostgreSQL 16), `queue` (`queue:work --tries=3 --timeout=1900`), dan `scheduler`. Kode aplikasi + vendor + aset build berasal dari image Docker, bukan bind-mount — deploy cukup `git pull && docker compose up -d --build`. Yang di-bind dari host hanya `.env`, `storage/`, dan `bootstrap/cache/`.

```bash
copy .env.example .env
# isi APP_KEY serta konfigurasi produksi
docker compose up -d --build
docker compose exec app php artisan migrate --force
docker compose exec app php artisan storage:link
```

Untuk produksi, set `APP_ENV=production`, `APP_DEBUG=false`, `APP_URL` ke URL HTTPS, `SESSION_SECURE_COOKIE=true`, dan isi `TRUSTED_PROXIES` hanya dengan alamat reverse proxy yang benar. Nginx bawaan melayani HTTP; TLS harus ditangani reverse proxy di depannya (misalnya Caddy, Cloudflare, atau Nginx host).

Setelah deployment:

```bash
php artisan optimize
php artisan queue:restart
```

## Konfigurasi lingkungan

Mulai dari `.env.example`; jangan commit `.env`. Bagian inti:

```dotenv
APP_NAME="SILINGKAR DLH Kota Palu"
APP_ENV=local
APP_DEBUG=false
APP_URL=http://localhost
TRUSTED_PROXIES=172.16.0.0/12,127.0.0.1

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=dlh_palu
DB_USERNAME=dlh_palu
DB_PASSWORD=
DB_SSLMODE=prefer
# DB_NEON_ENDPOINT=ep-xxx-yyy   # wajib diisi untuk Neon

FILESYSTEM_DISK=local
QUEUE_CONNECTION=database
CACHE_STORE=file
SESSION_DRIVER=file

B2_KEY_ID=
B2_APPLICATION_KEY=
B2_BUCKET=
B2_REGION=us-west-004
B2_ENDPOINT=https://s3.us-west-004.backblazeb2.com
BACKUP_DISK=b2

RECAPTCHA_SITE_KEY=
RECAPTCHA_SECRET_KEY=
```

Untuk Neon, gunakan host endpoint Neon, `DB_SSLMODE=require`, dan isi `DB_NEON_ENDPOINT`. `NeonDatabaseProvider` akan menambahkan parameter endpoint ke koneksi PostgreSQL.

Disk utama aplikasi tetap `local`; B2 digunakan untuk backup dan objek cloud terkait. Jangan mengubah `FILESYSTEM_DISK` ke B2 tanpa memahami konsekuensi upload sementara Livewire (upload sementara sudah dipaksa ke disk `local` via `config/livewire.php`). `B2_STORAGE_LIMIT_GB`, `NEON_STORAGE_LIMIT_GB`, `B2_PLAN`, dan `NEON_PLAN` mengatur informasi monitoring pada admin.

Konfigurasi tambahan tersedia di `.env.example`:

- `GPS_*` untuk data armada dari portal.gps.id.
- `GOOGLE_DRIVE_*` untuk dokumen Tata Lingkungan (API key + folder ID, cache TTL 900 detik).
- `RECAPTCHA_*` untuk verifikasi form publik.
- `QUEUE_EXPORTS=true` untuk memproses ekspor besar lewat queue (notifikasi + tautan unduh) alih-alih sinkron.

Provider chatbot tidak memakai variabel `.env`: kelola melalui **Admin → Pengaturan**. API key diproteksi oleh enkripsi Laravel; pastikan `APP_KEY` tidak berubah setelah provider dibuat.

## Operasional data dan storage

Backup dibuat dari **Admin → Backup Database** (khusus superadmin) dan diproses lewat queue (`RunBackupJob`). Arsip berisi dump PostgreSQL dan file storage, lalu disimpan pada disk `BACKUP_DISK` (biasanya B2). Restore (`RunRestoreJob`) bersifat merge/non-destruktif dan membuat backup sebelum pemulihan. Progress dan pembatalan bisa dipantau dari halaman backup.

Perintah artisan yang tersedia:

| Perintah                                              | Fungsi                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------- |
| `gps:fetch`                                           | Mengambil posisi armada dan menyimpannya ke cache database.   |
| `dlh:setup-seeder [--fresh]`                          | Menyiapkan data awal; `--fresh` mereset skema lebih dahulu.   |
| `dlh:cleanup-orphan-files [--delete] [--disk=] [--days=]` | Menghapus objek storage yang tidak lagi direferensikan database (default dry-run). |
| `dlh:cleanup-b2-orphans [--all]`                      | Membersihkan upload sementara Livewire pada B2 (`livewire-tmp`). |
| `dlh:download-images`                                 | Mengunduh gambar artikel dari situs sumber.                   |
| `shp:import-bulk {folder} {bidang} [--dry-run] [--skip-existing] [--color=]` | Mengimpor file Shapefile secara massal ke layer GIS. |

Scheduler menjalankan `gps:fetch` setiap 30 detik dan `dlh:cleanup-orphan-files --delete` setiap Minggu pukul 03:00. Pada server tanpa container scheduler, pasang cron berikut:

```cron
* * * * * cd /path/to/project && php artisan schedule:run >> /dev/null 2>&1
```

Queue memakai driver database secara default. Jalankan worker terus-menerus di produksi:

```bash
php artisan queue:work --tries=3 --timeout=1900
```

### Reset total lingkungan

`php artisan migrate:fresh` menghapus seluruh tabel lalu membuat ulang **skema kosong** dari migration. Ini tidak menghapus objek Backblaze B2. Sebelum reset produksi, buat dan unduh backup jika data masih dibutuhkan.

Untuk mengosongkan B2, hapus seluruh object version dan delete marker pada bucket melalui kredensial B2 yang tepat. Perintah pembersih bawaan hanya menghapus file yatim atau upload sementara, bukan seluruh bucket. Operasi ini permanen dan harus dibatasi ke bucket aplikasi.

Setelah reset tanpa seeder, tidak ada akun admin, role, konfigurasi provider AI, atau konten. Jalankan `php artisan db:seed` hanya jika memang ingin menambahkan data awal.

## Rute penting

**Publik**

| Area              | Rute                                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------- |
| Beranda           | `/`                                                                                                        |
| Pengaduan terpadu | `/pengaduan` (alias `/lapor`; per bidang: `?bidang=pengendalian\|sampah\|rth\|tata-penataan`)              |
| Pelacakan         | `/lacak` + `POST /feedback/{nomor_tiket}`                                                                  |
| Permohonan        | `/permohonan-rekomendasi`, `/pinjam-taman`, `/registrasi-usaha-lb3`, `/pengajuan-rintek-pertek` (+ halaman cek & bukti PDF masing-masing) |
| Armada            | `/armada`, `/api/armada-aktif`                                                                             |
| Peta persampahan  | `/peta-persampahan`, `/api/peta-persampahan/layers`                                                        |
| Tata Lingkungan   | `/tata-lingkungan`, `/api/tata-lingkungan/folders`, `/api/tata-lingkungan/files`                           |
| Berita            | `/berita` dan `/berita/{slug}`                                                                             |
| Profil & UPTD     | `/profil`, `/tentang`, `/uptd/lab-lingkungan`, `/uptd/tpa-kawatuna` (+ sejarah), `/uptd/jurnal-lab`        |
| Legal             | `/kebijakan-privasi`, `/syarat-ketentuan`                                                                  |
| Chatbot stream    | `POST /api/chatbot/stream`                                                                                 |
| OG image proxy    | `/file/og`                                                                                                 |
| Sertifikat        | `/sosialisasi/{id}/sertifikat/{token}.pdf` (token acak)                                                    |

**Admin** (prefix `/admin`, middleware `auth` + `admin.access` + `no-store`)

| Area              | Rute                                                            |
| ----------------- | --------------------------------------------------------------- |
| Login/panel       | `/admin/login`, `/admin` (dashboard)                             |
| CRUD registry     | `/admin/{resource}`, `/admin/{resource}/create`, `/{record}`, `/{record}/edit` |
| Ekspor            | `/admin/{resource}/export`, `/export-all`, `/bulk-export`, `/exports/download/{token}` |
| Peta GIS          | `/admin/peta` (+ import, layer CRUD, draw, feature edit)        |
| Peta laporan      | `/admin/peta-laporan/data`                                       |
| Audit log         | `/admin/activity-log` (superadmin)                               |
| Backup            | `/admin/backup` (superadmin; progress, cancel, restore, unduh)  |
| Notifikasi        | `/admin/notifications` (+ poll, read)                            |
| Profil/pengaturan | `/admin/profile`, `/admin/settings` (termasuk provider AI & mode pemeliharaan), `/admin/help` |
| Ulasan            | `/admin/ulasan-masyarakat`                                       |
| Berkas            | `/admin/file/download`, `/admin/{resource}/{file}` (preview), `/admin/upload-image` |

## Keamanan dan troubleshooting

- Jangan aktifkan `APP_DEBUG` di produksi atau menyimpan kredensial pada repository.
- Batasi `TRUSTED_PROXIES`; jangan gunakan `*` di produksi (aplikasi mencatat warning log bila terdeteksi).
- Semua endpoint publik penting diberi throttle; jangan hapus middleware ini tanpa mitigasi pengganti. Endpoint update Livewire juga di-throttle 60/menit.
- Pastikan folder Google Drive publik memang dibagikan untuk dibaca dan API Google Drive aktif.
- Bila unggahan tidak bisa diakses lokal, jalankan kembali `php artisan storage:link` dan cek permission `storage/` serta `bootstrap/cache/`.
- Bila job tertahan, periksa `jobs`/`failed_jobs`, jalankan worker, kemudian `php artisan queue:restart` setelah deploy.
- Lihat log aplikasi dengan `php artisan pail` atau `storage/logs/laravel.log`.
- Aplikasi desktop menampilkan halaman error bila server produksi tidak terjangkau — klik **Coba Lagi** setelah koneksi pulih; bila domain produksi berubah, sesuaikan `ADMIN_BASE_URL` dan `ALLOWED_HOSTS` di `desktop/src-tauri/src/main.rs`.

## Pengembangan

```bash
npm run typecheck
npm run build
php artisan optimize:clear
```

Entrypoint Vite: `resources/css/app.css`, `resources/js/app.js`, `admin-common.js`, `map-bundle.js`, `dashboard-charts.js`, `flatpickr-init.js`, dan `tata-lingkungan.ts`. Build produksi menyertakan source map dan memaksa `font-display: optional` untuk mencegah layout shift.

Panduan kontribusi internal:

- Jaga migration bersifat aman untuk database yang sudah ada.
- Gunakan policy dan `AdminAccess` untuk resource baru, serta daftarkan modulnya di `AdminRegistry`.
- Form publik baru sebaiknya memakai komponen Livewire SFC di `resources/views/components/public/` dengan trait `VerifiesGoogleRecaptcha` dan `ThrottlesPublic`.
- Terjemahan validasi/auth/pagination bahasa Indonesia ada di `lang/id/`.
- Halaman web baru sebaiknya menyertakan `<x-splash />` (komponen splash SILINGKAR) di layoutnya; aplikasi desktop otomatis menambahkan splash setara untuk halaman yang belum memilikinya.
