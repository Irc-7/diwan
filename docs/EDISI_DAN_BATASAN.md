# Edisi OtomaX & Batasan Free Edition

**Tanggal riset:** 2026-10-17 | **Sumber:** situs resmi otomax-software.com (halaman Harga, Download, EULA, produk), revision log 4.3.1, Knowledge Base resmi.

## Temuan kunci

Situs resmi **tidak mempublikasikan daftar batasan fungsional rinci** untuk Free Edition (tidak ada angka terminal/modem maksimal, daftar modul terkunci, batas transaksi, atau masa berlaku khusus Free). Free Edition diposisikan murni sebagai **versi coba** — "dicoba dahulu sebelum memesan" — dan tidak masuk tabel perbandingan edisi berbayar.

Yang terverifikasi tentang paket Free Edition (halaman Download):
- Tidak termasuk petunjuk restore database dan user manual.
- Berisi: backup database, petunjuk instalasi database, aplikasi OtomaX, dan modul Laporan.

## Struktur edisi (tabel harga resmi Feb 2025)

**Kategori Umum (Server Induk):** Mini (Rp1jt) → Lite (Rp2jt) → Standard (Rp3jt) → Pro (Rp5jt) → Advanced (Rp7jt) → Enterprise (Rp8,5jt) → Ultimate (Rp9,5jt).
**Kategori Server Cabang:** Mars (Rp1jt) → Venus (Rp2jt) → Jupiter (Rp3jt).
**Free Edition:** di luar tabel, hanya sebagai trial.

Kuota per edisi (Umum) — Terminal = jumlah modem/HP/dongle/SIM terlisensi:
| Edisi | Terminal | IM Center | IM Hybrid | IP Client | Internet Banking | WA Center | IP Center | Postpaid |
|---|---|---|---|---|---|---|---|---|
| Mini | 2 | 2 | 2 | 1 | – | – | – | – |
| Lite | 5 | 2 | 5 | 2 | – | – | – | – |
| Standard | 10 | 4 | 10 | 5 | 1 | – | – | – |
| Pro | 20 | 4 | 20 | 10 | 2 | – | – | – |
| Advanced | 40 | 6 | 30 | 20 | 4 | 1 | – | – |
| Enterprise | unlimited | 10 | unlimited | 30 | 5 | 2 | Ya | – |
| Ultimate | unlimited | unlimited | unlimited | unlimited | 10 | 3 | Ya | Ya |

Server Cabang (Mars/Venus/Jupiter): Terminal 8/16/32; TANPA IM Hybrid, IP Client, Postpaid, Internet Banking, WA Center; hanya boleh 1 reseller (induknya); tanpa voucher fisik, markup, komisi, cek/transfer saldo downline.

**Berlaku semua edisi:** jumlah transaksi UNLIMITED. Modul standar: SMS Center, SMS Sender, IM Center, IM Hybrid HTH, USSD/Dial, STK, SMS Order/H2H, Keypad, Voucher Fisik.
**Lisensi:** masa berlaku 1 tahun, wajib perpanjangan tahunan (ada iuran tahunan per edisi: Mini Rp240rb … Ultimate Rp1,5jt). Upgrade = selisih harga + Rp100rb (hanya dalam masa berlaku). Garansi 15 hari uang kembali.

## Fitur yang dikunci per edisi (dari revision log resmi)

- **Ultimate saja:** kolom Rumus Harga di tabel Produk; kolom Filter Tujuan (RegEx) di halaman Produk; halaman mutasi komisi; modul Web SMS Sender (kirim SMS via HTTP GET).
- **Advanced ke atas:** penyempurnaan modul Internet Banking BNI.
- **Standard ke atas:** modul Internet Banking (BCA, BNI, Mandiri, BRI) — artinya Free/Mini/Lite tidak dapat auto-deposit via mutasi bank.

## Implikasi untuk clone Linux

1. **Satu binary untuk semua edisi** — pembedanya adalah kunci lisensi, bukan build terpisah. Di binary Free yang kita bedah, seluruh kode modul (termasuk yang terkunci) ada; yang membedakan adalah gate lisensi.
2. Untuk clone, batasan edisi Free **tidak perlu direplikasi** kecuali ingin model freemium — bangun setara Ultimate (semua modul terbuka, tanpa kuota terminal).
3. Jika ingin meniru model bisnisnya, yang perlu dibangun adalah **sistem lisensi** (kunci edisi + masa berlaku tahunan), bukan memangkas fitur.
4. Modul prioritas dari sisi teknis (sudah dipetakan di teardown): SMS Center/Sender, Dial/USSD, STK, SMS Order/H2H, Keypad, Voucher Fisik, IM Center, IM Hybrid, IP Center/Client, Internet Banking, WA Center, Postpaid.

Sumber: https://www.otomax-software.com/harga/, /download/, /eula/, /produk/, revision log Free_Edition.zip 4.3.1.

## Keputusan desain clone (2026-10-17, disetujui user)

**Semua batasan edisi DIHILANGKAN, semua fitur DIBUKA.** Clone dibangun setara edisi Ultimate:
- Tanpa kuota terminal/modem (unlimited).
- Semua modul terbuka: SMS Center/Sender, Dial/USSD, STK, SMS Order/H2H, Keypad, Voucher Fisik, IM Center, IM Hybrid HTH, IP Center/Client, Internet Banking, WA Center, Postpaid, Web SMS Sender.
- Fitur Ultimate-only ikut dibuka: kolom Rumus Harga, Filter Tujuan (RegEx), halaman mutasi komisi.
- Tanpa sistem lisensi/kunci edisi, tanpa masa berlaku tahunan, tanpa iuran.
- Satu-satunya yang TIDAK direplikasi: mekanisme proteksi/lisensi itu sendiri.
