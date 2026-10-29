# NADEL STORE: pembayaran otomatis (Vercel + Midtrans QRIS + Discord + antrian bersama)

Alur: pembeli buat pesanan → muncul QRIS dinamis (nominal pas) → pembeli bayar → Midtrans memberi tahu server → pesan masuk ke #pesanan dan admin di-tag (No. Antrian, username, jumlah Robux). Tidak perlu tombol "sudah bayar" atau bukti transfer.

## Isi folder
- `index.html` website, `qris.jpg` (gambar QRIS NADEL STORE; QR dinamis tiap pesanan otomatis ditempel di atas kotak QR-nya)
- `api/pay.js` buat tagihan QRIS (harga dihitung di server)
- `api/status.js` cek status bayar
- `api/midtrans-webhook.js` terima notifikasi bayar dari Midtrans (tanda tangan diverifikasi)
- `api/_discord.js`, `api/_midtrans.js` fungsi bantu

## 1. Akun Midtrans (payment gateway)
1. Daftar di midtrans.com. Untuk menerima uang asli, Midtrans meminta verifikasi data (KYC) dan ada biaya per transaksi; syarat dan tarif bisa berubah, cek langsung di midtrans.com.
2. Selama menunggu verifikasi, pakai mode **Sandbox** untuk uji coba (tanpa uang asli).
3. Aktifkan metode **QRIS (GoPay)** di Settings → Payment.
4. Ambil **Server Key** di Settings → Access Keys. Rahasia, jangan dikirim ke siapa pun.

## 2. Webhook Discord
Channel #pesanan → Edit Channel → Integrasi → Webhook → Webhook Baru → Salin URL. Rahasia juga.

## 3. Upload ke GitHub dan deploy ke Vercel
1. github.com → New repository → Add file → Upload files (semua isi folder, termasuk `api`).
2. vercel.com → Add New → Project → pilih repo → Deploy.

## 3b. Antrian bersama (Upstash Redis, gratis)
Supaya nomor antrian berurutan (#1, #2, #3) dan antrian, statistik, top buyer tampil sama untuk SEMUA pengunjung:
1. Daftar di upstash.com → **Create Database** (Redis, pilih region terdekat, mis. Singapore).
2. Di halaman database, tab **REST API**: salin `UPSTASH_REDIS_REST_URL` dan `UPSTASH_REDIS_REST_TOKEN` (rahasia).
3. Tambahkan keduanya ke Environment Variables Vercel, lalu tambah juga:
   - `ADMIN_SECRET`: kata acak panjang buatanmu sendiri (rahasia)
   - `SITE_URL`: alamat websitemu, mis. `https://nama-tokomu.vercel.app`
4. Redeploy.

Cara kerja: saat pembayaran terkonfirmasi, pesanan masuk antrian bersama dengan nomor urut, lalu pesan Discord berisi link **✅ Tandai pesanan selesai**. Setelah Robux kamu kirim, klik link itu (hanya terlihat di channel privat adminmu). Status antrian berubah jadi Selesai, dan angka **Pesanan Selesai / Robux Terkirim / Pembeli** serta **Top Buyer** ikut bertambah untuk semua orang. Username di halaman publik disamarkan (Ald***2).
Tanpa Upstash, website tetap jalan tapi antrian/statistik hanya tersimpan di browser masing-masing pembeli.

## 4. Environment Variables (Vercel → Settings → Environment Variables)
| Name | Isi |
|---|---|
| `MIDTRANS_SERVER_KEY` | Server Key Midtrans |
| `MIDTRANS_IS_PRODUCTION` | `false` untuk Sandbox, `true` untuk uang asli |
| `DISCORD_WEBHOOK_URL` | URL webhook #pesanan |
| `DISCORD_ADMIN_ID` | ID Discord admin yang di-tag (banyak: pisah koma) |
| `DISCORD_ADMIN_ROLE_ID` | (opsional) ID role admin |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `ADMIN_SECRET`, `SITE_URL` | Lihat langkah 3b |
| `PRICE_U` / `PRICE_P` | Harga per 100 Robux: Via Username / Payout (bawaan 16000 / 13000) |

ID Discord: Pengaturan → Lanjutan → aktifkan Mode Pengembang → klik kanan nama/role → Salin ID.
Setelah mengisi, lakukan **Redeploy**.

## 5. Daftarkan alamat notifikasi di Midtrans
Midtrans → Settings → Configuration → **Payment Notification URL**:
`https://NAMA-DOMAINMU.vercel.app/api/midtrans-webhook`

## 6. Tes (di Sandbox)
Buat pesanan → QRIS muncul → bayar lewat simulator Midtrans → pesan "PEMBAYARAN DITERIMA" muncul di #pesanan.

## Supaya hanya admin yang bisa melihat #pesanan
Ini pengaturan di Discord, bukan di kode. Pesan dari webhook tetap masuk walau channel dibuat privat.
1. Klik kanan channel **#pesanan** → **Edit Channel** → **Izin** → aktifkan **Channel Pribadi**.
   (Atau: pilih `@everyone` → **Lihat Channel** = ✗ / tolak.)
2. Tambahkan **role admin** atau akun admin saja → **Lihat Channel** ✔ dan **Baca Riwayat Pesan** ✔.
3. Pastikan role lain (member, booster, bot) tidak punya izin **Administrator** atau **Lihat Channel** di sini.
4. Batasi izin **Kelola Webhook** dan **Kelola Channel** hanya untuk dirimu. Siapa pun yang punya izin itu bisa melihat URL webhook.
5. Cek hasilnya: Pengaturan Server → Peran → **Lihat Server Sebagai Peran** (pilih `@everyone`). Channel #pesanan harus tidak terlihat.
6. Link invite khusus channel #pesanan (yang dulu dipakai pembeli) tidak diperlukan lagi di versi otomatis ini. Hapus invite itu di Pengaturan Server → Undangan.
Pembeli tidak melihat apa pun di Discord. Mereka hanya melihat status pesanan di website.

## Catatan
- Kalau harga diubah, samakan angka `mk(16000)` / `mk(13000)` di `index.html` dengan `PRICE_U` / `PRICE_P` di Vercel. Harga yang ditagih selalu dari server.
- Nomor antrian berurutan (#1, #2, ...) dan dibuat saat pembayaran terkonfirmasi. Kode pesanan NS-XXXXX tetap dipakai sebagai ID.
- Akun, riwayat pribadi, dan ulasan masih tersimpan di browser pembeli. Antrian, statistik, dan top buyer sudah bersama.
- Notifikasi dari Midtrans yang dikirim ulang tidak menimbulkan pesan ganda (diproses satu kali per pesanan).
- Transfer bank manual tidak dideteksi otomatis. QRIS menerima pembayaran dari semua aplikasi bank dan e-wallet.
- Kalau mau pakai gateway lain (Tripay, Xendit, iPaymu), `api/pay.js`, `api/status.js`, dan `api/midtrans-webhook.js` yang perlu disesuaikan.
