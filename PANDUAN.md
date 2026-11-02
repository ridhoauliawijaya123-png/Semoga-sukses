# Monitoring Teknisi — Panduan Pemasangan

Sistem ini terdiri dari tiga bagian:

| Bagian | Isi | Dipakai oleh |
|---|---|---|
| **APK Teknisi** (`app/`) | Langkah 1 absen masuk (selfie + radius GPS) → Langkah 2 laporan pekerjaan (foto, ODP, SN ONT, redaman) → Langkah 3 absen pulang | Teknisi di HP Android |
| **Dashboard** (`dashboard/`) | Monitoring realtime, peta posisi teknisi, verifikasi laporan, rekap absensi, kelola teknisi & lokasi, unduh CSV | Admin / supervisor lewat browser (GitHub Pages) |
| **Database** (`supabase/schema.sql`) | Tabel, validasi di server, keamanan RLS, penyimpanan foto | Supabase |

Semua bisa memakai paket **gratis** (Supabase Free + GitHub). APK dibangun otomatis oleh GitHub, jadi Anda **tidak perlu Android Studio**.

---

## Langkah 1 — Siapkan Supabase (±10 menit)

1. Daftar di **supabase.com** → **New project**. Region pilih **Southeast Asia (Singapore)**. Simpan password database.
2. Buka **SQL Editor → New query**. Tempel seluruh isi `supabase/schema.sql` → **Run**. Harus muncul *Success*.
3. Buka **Authentication → Sign In / Providers → Email**. Matikan **Confirm email** agar teknisi bisa langsung masuk setelah mendaftar.
   *Jika ingin tetap memakai verifikasi email, biarkan menyala; teknisi harus klik link di email dulu.*
4. Buka **Project Settings → API**. Salin dua nilai berikut:
   - **Project URL** (contoh `https://abcd1234.supabase.co`)
   - **anon public key** (teks panjang diawali `eyJ…`)

> Anon key memang boleh terlihat publik. Data tetap aman karena dilindungi aturan RLS: teknisi hanya bisa melihat datanya sendiri, dan hanya admin yang bisa melihat semua data.

## Langkah 2 — Isi `config.js`

Buka `config.js` di folder utama, lalu ganti nilainya:

```js
window.APP_CONFIG = {
  SUPABASE_URL: 'https://abcd1234.supabase.co',
  SUPABASE_ANON_KEY: 'eyJhbGciOi...',
  NAMA_TIM: 'Tim Teknisi FTTH Subang'
};
```

File ini dipakai oleh APK dan dashboard sekaligus.

## Langkah 3 — Upload ke GitHub

1. Buat repository baru di **github.com**, misalnya `monitoring-teknisi` (Public).
2. Upload **seluruh isi** folder ini, termasuk folder tersembunyi `.github`.
   Cara termudah: **Add file → Upload files**, lalu seret semua file dan folder → **Commit**.
   > Jika folder `.github` tidak ikut terupload lewat browser, buat filenya manual: **Add file → Create new file**, isi nama `.github/workflows/build-apk.yml`, lalu tempel isinya. Ulangi untuk `deploy-dashboard.yml`.
3. Aktifkan dashboard: **Settings → Pages → Build and deployment → Source: GitHub Actions**.
4. Buka tab **Actions**. Dua proses akan berjalan:
   - **Deploy Dashboard** (±1 menit). Alamatnya `https://USERNAME.github.io/monitoring-teknisi/`.
   - **Build APK Teknisi** (±5–8 menit). File APK muncul di **Releases** (kolom kanan halaman repo) dengan nama `teknisi-lapangan.apk`.

   Jika kedua proses tidak berjalan otomatis, buka **Actions**, pilih workflow-nya, lalu klik **Run workflow**.

## Langkah 4 — Buat akun admin pertama

1. Instal APK di HP Anda, lalu **Daftar** (atau buat akun lewat Supabase **Authentication → Add user**).
2. Di Supabase **SQL Editor**, jalankan perintah ini dengan email Anda:
   ```sql
   update public.profiles set role = 'admin', aktif = true where email = 'email-anda@contoh.com';
   ```
3. Login ke dashboard dengan email dan password tersebut.

## Langkah 5 — Isi data awal di dashboard

1. **Lokasi & radius**: klik peta di titik kantor, gudang, atau base camp, atur radiusnya (umumnya 100–300 m untuk teknisi lapangan), lalu simpan.
2. **Pengaturan**: atur jam masuk, jam pulang, toleransi terlambat, dan batas akurasi GPS.
3. **Teknisi**: teknisi mendaftar sendiri dari APK. Klik **Aktifkan** pada akunnya. Lewat **Edit**, Anda bisa membatasi lokasi absen per teknisi.

## Membagikan APK ke teknisi

- Kirim link halaman **Releases** atau file `teknisi-lapangan.apk` lewat WhatsApp.
- Di HP Android, izinkan **Instal aplikasi tidak dikenal** untuk WhatsApp atau browser saat diminta.
- Saat aplikasi pertama dibuka, izinkan **Kamera** dan **Lokasi** (pilih "Saat aplikasi digunakan" dan **Lokasi akurat**).

## Alur kerja teknisi

1. **Absen masuk**: tombol aktif hanya jika teknisi berada di dalam lingkaran radius. Teknisi mengambil selfie, lalu server mencatat jam server dan statusnya (Tepat Waktu atau Terlambat).
2. **Laporan pekerjaan**: baru bisa diisi setelah absen masuk. Isinya jenis pekerjaan, no. tiket/WO, pelanggan, ODP/port, SN ONT, redaman, status (Selesai/Pending/Gagal), keterangan, dan 1–8 foto. Setiap foto otomatis diberi cap waktu, koordinat, dan no. tiket. Teknisi bisa mengirim laporan berkali-kali dalam sehari.
3. **Absen pulang**: selfie lagi di dalam radius.
4. Jika admin meminta **revisi**, catatan admin muncul di daftar laporan teknisi.

## Mengubah aplikasi

- Setiap perubahan pada `app/` atau `config.js` yang di-commit akan otomatis membangun APK baru di Releases.
- Setiap perubahan pada `dashboard/` akan otomatis memperbarui website dashboard.
- APK baru bisa langsung diinstal menimpa yang lama, tanpa perlu uninstall, karena tanda tangannya tetap sama (`app/keystore/debug.keystore`).
  *Opsional, untuk keamanan lebih:* buat keystore sendiri, ubah ke base64, lalu simpan di **Settings → Secrets → Actions** dengan nama `KEYSTORE_B64`.
- Mengganti nama atau ID aplikasi: ubah `app/capacitor.config.json` (`appName`, `appId`).

## Batasan yang perlu diketahui

- **Fake GPS**: aplikasi ini tidak bisa memastikan HP tidak memakai aplikasi pemalsu lokasi. Sebagai gantinya, selfie dan foto ber-cap koordinat tersimpan di server sebagai bukti audit. Radius, urutan langkah, dan jam tetap divalidasi di server, sehingga tidak bisa diakali dengan mengubah jam HP.
- **Posisi di peta** adalah posisi terakhir saat teknisi absen atau mengirim laporan, bukan pelacakan terus-menerus.
- **Kuota Supabase Free**: 500 MB database dan 1 GB penyimpanan foto. Foto sudah dikompres (±100–250 KB per foto); kira-kira cukup untuk 4.000–8.000 foto. Setelah itu, hapus foto lama atau naik ke paket Pro.
- Project Supabase Free akan **dijeda jika tidak dipakai selama 7 hari**. Selama teknisi aktif setiap hari, hal ini tidak terjadi.

## Struktur folder

```
config.js                     ← isi URL & anon key Supabase
supabase/schema.sql           ← jalankan di SQL Editor
app/                          ← aplikasi HP (Capacitor)
  www/index.html              ← tampilan & logika aplikasi teknisi
  www/vendor/                 ← library Supabase & peta (offline di APK)
  capacitor.config.json       ← nama & ID aplikasi
  keystore/debug.keystore     ← kunci tanda tangan APK
dashboard/index.html          ← website monitoring admin
.github/workflows/            ← otomatisasi build APK & deploy dashboard
```
