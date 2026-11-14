# Panduan Setup Backend GAS & Deploy Frontend GitHub Pages — Academy Studio

Dokumen ini memandu langkah lengkap untuk menjalankan **Academy Studio** secara penuh di produksi dengan arsitektur decoupled:
- **Backend REST API JSON Murni**: Google Apps Script (`gas/`) + Google Sheets & Google Drive
- **Frontend SPA Statis**: HTML5 + Tailwind CSS + Vanilla JS (`index.html`, `css/`, `js/`) di **GitHub Pages**

> **Cara Langsung Membuka Aplikasinya Sekarang (Tanpa Setup Server Dulu):**  
> Aplikasi web ini **sudah ada** di folder proyek ini! Cukup **klik ganda file `index.html`** di File Explorer untuk membukanya langsung di Google Chrome / Microsoft Edge.  
> Selama `GAS_URL` di `js/config.js` masih berisi `XXXX`, seluruh 20 layar dan fitur langsung berjalan otomatis dalam **Mode Simulasi Lokal (`localStorage`)** agar Anda bisa langsung mencoba tampilannya terlebih dahulu.
>
> Barulah ketika Anda ingin aplikasi tersimpan *online* di Google Sheets / Google Drive dan bisa diakses dari HP/laptop mana saja, ikuti Bagian 1 s.d. Bagian 3 di bawah ini.

---

## Bagian 1 — Setup Google Apps Script (Backend)

### 1.1 Apakah Harus Membuat Folder Drive & Spreadsheet Secara Manual?
**TIDAK PERLU.** Fungsi `setupAppEnvironment()` di dalam `gas/Kode.gs` sudah dirancang untuk **membuat semuanya secara otomatis** dalam 1 kali klik:
- **1 Folder Induk Google Drive** bernama `AcademyStudio_Root_Drive`, lengkap dengan **4 Subfolder otomatis** di dalamnya:
  1. `01_Proyek_Naskah_Mahasiswa`
  2. `02_Jurnal_Acuan_Upload`
  3. `03_Laporan_Integritas_PDF`
  4. `04_Bukti_Bayar_QRIS`
- **1 File Google Sheets** bernama `Database_AcademyStudio` (dipindahkan otomatis ke dalam `AcademyStudio_Root_Drive`) lengkap dengan seluruh tabel inti (`Users`, `Projects_Skripsi`, `Documents_Content`, `Literature_References`, `Synthesis_Matrix`, `Slide_Decks`, `Transactions_Billing`, `Defense_Simulations`, `Integrity_Reports`, `Config_System`, `System_Logs`), 7 sheet DocEngine (`DE_*`), 4 sheet Tagihan (`Tagihan`, `Pembayaran`, `PotonganTagihan`, `ReminderLog`), dan 4 sheet Notifikasi/CRM (`AntrianNotif`, `BlastWA`, `CRM_Kontak`, `CRM_Interaksi`).
- Seluruh ID folder (`ROOT_FOLDER_ID`, `PROYEK_FOLDER_ID`, `JURNAL_FOLDER_ID`, `INTEGRITY_FOLDER_ID`, `BUKTI_FOLDER_ID`) dan `SPREADSHEET_ID` langsung tersimpan otomatis ke **Script Properties**.

> *Kapan perlu membuat manual?* Hanya jika Anda sudah punya Spreadsheet/Folder lama dan ingin memakai ID yang sudah ada. Jika baru mulai dari nol, langsung saja ke langkah 1.2!

### 1.2 Masukkan File Backend `.gs` ke Editor Apps Script
1. Buka [script.google.com](https://script.google.com) -> klik **Project baru** -> beri nama `Academy Studio API`.
2. Buat 5 file skrip di sidebar kiri (ikon `+` -> **Skrip**), lalu salin-tempel isi masing-masing file dari folder `gas/`:
   - `Kode.gs` (timpa isi `Code.gs` bawaan)
   - `Migrasi.gs`
   - `DocEngine.gs`
   - `Tagihan.gs`
   - `Notifikasi_CRM.gs`
3. Salin manifes **`gas/appsscript.json`**:
   - Buka **Setelan Project (Project Settings)** (ikon roda gigi di sidebar kiri) -> centang **Tampilkan file manifes "appsscript.json" di editor** (*Show "appsscript.json" manifest file in editor*).
   - Kembali ke **Editor**, klik file `appsscript.json` di sidebar kiri, lalu ganti isinya dengan isi `gas/appsscript.json` (mengatur timezone `Asia/Jakarta`, runtime `V8`, konfigurasi Web App, dan seluruh `oauthScopes` yang dibutuhkan).
4. *(Opsional — untuk hasil aplikasi paling maksimal)* Di **Setelan Project (Project Settings)** -> bagian **Properti Skrip (Script Properties)** -> klik **Edit script properties** -> **Add script property** untuk menambahkan kunci API berikut (jangan pernah menaruh kunci di `js/config.js` atau `js/app.js`):
   - `GEMINI_API_KEY` *(Gratis, paling direkomendasikan — buat di `aistudio.google.com/apikey`; otomatis menjadi fallback seluruh fitur AI)*
   - `ANTHROPIC_API_KEY` *(Opsional berbayar — buat di `console.anthropic.com`)*
   - `OPENAI_API_KEY` *(Opsional berbayar — buat di `platform.openai.com/api-keys`)*
   - `FONNTE_TOKEN` *(Opsional — ambil Device Token di `fonnte.com` untuk WA otomatis & blast CRM; tanpa ini sistem tetap kirim lewat Email Gmail)*
   - `GOOGLE_CLIENT_ID` *(Opsional — ambil dari Google Cloud Console -> Credentials -> OAuth 2.0 Web Client ID; tambahkan domain GitHub Pages Anda ke Authorized JavaScript origins)*
   - `PUBLIC_APP_URL` *(Gratis — isi URL live GitHub Pages Anda setelah deploy agar QR Code kwitansi & laporan mengarah ke domain Anda)*

> Catatan Keamanan: `SESSION_SECRET`, `QR_SECRET`, dan kredensial akses darurat dibuat otomatis secara acak kriptografis oleh `setupAppEnvironment()`.

### 1.3 Inisialisasi Otomatis (`setupAppEnvironment`) & Trigger Latar Belakang
1. Di Editor Apps Script, pastikan file `Kode.gs` terbuka, pilih fungsi **`setupAppEnvironment`** pada dropdown fungsi di toolbar atas, lalu klik **Run (Jalankan)**.
2. Saat muncul jendela izin Google (*Authorization Required*), klik **Review permissions -> Pilih akun Google Anda -> Lanjutan (Advanced) -> Buka Academy Studio API (tidak aman) -> Izinkan (Allow)**.
3. Tunggu hingga log eksekusi menampilkan pesan selesai (folder Drive, Spreadsheet, dan akun demo Admin/Mahasiswa otomatis dibuat; periksa **Execution Log** bila ingin melihat catatan setup).
4. Buka menu **Pemicu (Triggers)** (ikon jam di sidebar kiri Apps Script) -> klik **+ Tambah Pemicu (Add Trigger)**:
   - Pilih fungsi yang akan dijalankan: `prosesAntrianNotif`
   - Pilih sumber acara: **Dari waktu (Time-driven)** -> **Penghitung menit (Minutes timer)** -> **Setiap menit (Every minute)**.
5. *(Opsional)* Tambahkan pemicu harian untuk `svcTgReminderRun` bila ingin pengingat jatuh tempo H-3 & H-0 otomatis berjalan setiap hari.

### 1.4 Deploy Web App (REST API JSON Murni)
1. Klik **Terapkan (Deploy)** -> **Deployment baru (New deployment)**.
2. Pilih jenis: **Aplikasi Web (Web app)**.
3. Konfigurasi wajib:
   - **Jalankan sebagai (Execute as):** `Saya (Me)`
   - **Yang memiliki akses (Who has access):** `Siapa saja (Anyone)`
4. Klik **Terapkan**, lalu salin **URL Aplikasi Web** (`https://script.google.com/macros/s/.../exec`).

---

## Bagian 2 — Hubungkan Frontend ke GAS

Buka file `js/config.js` dan ganti nilai `GAS_URL` dengan URL Web App hasil deploy:

```js
const GAS_URL = 'https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec';
```

Pastikan **tidak ada** API Key LLM, token Fonnte, maupun rahasia apa pun di dalam `js/config.js` atau `js/app.js`.

---

## Bagian 3 — Deploy Frontend Statis ke GitHub Pages (via Terminal Git)

> **Aturan Penting:** Folder `gas/` hanya untuk Editor Apps Script dan **tidak** dipublikasikan ke situs statis GitHub Pages. Untuk deploy bersih ke GitHub Pages, gunakan cabang khusus `gh-pages` yang hanya berisi `index.html`, `css/`, `js/`, dan aset publik.

### 3.1 Siapkan Personal Access Token (PAT) GitHub
1. Buka **GitHub -> Settings -> Developer settings -> Personal access tokens -> Tokens (classic)**.
2. Klik **Generate new token (classic)**, centang scope **`repo`**, lalu salin token Anda.

### 3.2 Inisialisasi & Push ke Branch `gh-pages` Bersih
Jalankan di PowerShell / Git Bash dari folder root proyek:

```bash
# 1. Pastikan working tree utama rapi (jika belum inisialisasi git)
git init
git add .
git commit -m "Initial commit: Academy Studio full-stack"
git branch -M main
git remote add origin https://github.com/<USERNAME>/<NAMA_REPO>.git
git push -u origin main

# 2. Buat & push branch gh-pages khusus berkas frontend statis (tanpa folder gas/ atau file .md internal)
git checkout --orphan gh-pages
git rm -rf --cached gas PRD_*.md WIREFRAME_*.md UIUX_*.md PANDUAN_*.md "Desain Ui Ux" 2>$null
git add index.html css js
git commit -m "Deploy frontend statis Academy Studio ke GitHub Pages"
git push -u origin gh-pages --force

# Kembali ke cabang main untuk pengembangan berikutnya
git checkout main
```

Saat diminta password Git pada proses push, masukkan **Personal Access Token (PAT)** GitHub Anda (bukan password akun).

### 3.3 Aktifkan GitHub Pages
1. Buka repositori di GitHub -> **Settings -> Pages**.
2. Pada bagian **Build and deployment**:
   - Source: **Deploy from a branch**
   - Branch: **`gh-pages`** | Folder: **`/(root)`** -> Klik **Save**.
3. Tunggu 1–2 menit hingga URL live `https://<USERNAME>.github.io/<NAMA_REPO>/` aktif.

---

## Bagian 4 — Checklist Verifikasi Pasca-Deploy
1. Buka URL GitHub Pages Anda, buka **DevTools -> Console & Network**.
2. Coba daftar akun baru atau login: request POST menuju URL `.../exec` harus mengembalikan HTTP `200` dengan `Content-Type: application/json` tanpa error CORS.
3. Coba buat draf makalah atau humanizer text untuk menguji gateway AI + mutasi kredit di sheet `Users` & `KreditMutasi`.
4. Coba ekspor Docs/Slides di `DocEngine` dan pindai QR verifikasi pada kwitansi PDF di modul `Tagihan & Pembayaran`.
