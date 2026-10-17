# Kritik Impeccable — ShipMate-Improved Dashboard

**Target:** `docs/mockup/shipmate-improved/index.html` (+ `preview-light.png`, `dna.json`)
**Mode:** Operate (operator menyelesaikan tugas: scanability, konsistensi, ekspektasi native)
**Tanggal:** 2026-10-17 | **Metode:** ⚠️ DEGRADED: single-context (subagent tidak dapat spawn subagen penilaian terpisah; Assessment A+B dikerjakan sekuensial dalam satu konteks)

**Skor: 23/32 (72%) — Good.** Fondasi solid; perbaiki area lemah sebelum naik ke sistem desain.

## Tabel Heuristik Nielsen

| # | Heuristik | Skor | Isu kunci |
|---|-----------|------|-----------|
| 1 | Visibility of System Status | 4 | Badge Live + pulse, jam WIB, "8 dtk lalu", AT ticker, `role="alert"` — status sistem sangat terbaca |
| 2 | Match System / Real World | 3 | Bahasa operator Indonesia natural; estetika "buku log" mono pas untuk NOC pulsa |
| 3 | User Control and Freedom | 3 | Jeda live, filter chips, pencarian — kontrol monitoring cukup |
| 4 | Consistency and Standards | 2 | Penamaan modem tidak konsisten antara strip krisis vs panel terminal; legend hantu |
| 5 | Error Prevention | 3 | Read-only monitoring; satu-satunya aksi (retry modem) memberi umpan balik |
| 6 | Recognition Rather Than Recall | 3 | Semua opsi terlihat; label M1–M4 butuh pemetaan mental ke nama operator |
| 7 | Flexibility and Efficiency | 2 | Tanpa keyboard shortcut; pause/search/filter semuanya butuh mouse |
| 8 | Aesthetic and Minimalist Design | 3 | Disiplin satu elemen berani; tombol "Jeda live" putih di dark mode menyaingi strip krisis |
| 9 | Error Recovery | n/a | Tidak ada input/pengisian form di permukaan ini |
| 10 | Help and Documentation | n/a | Tool operasional; hint inline ("4 port aktif") sudah cukup sebagai dokumentasi |

## Design Specificity Verdict

**Dibuat untuk produk ini, bukan SaaS generik.** Metafora "ruang kontrol jam 2 pagi" terbaca: JetBrains Mono untuk seluruh data ("suara buku log"), rel modem + AT ticker sebagai kolom kanan (bukan widget generik), strip krisis sebagai satu-satunya momen hero, dan copy Indonesia aktif ("Pindahkan beban ke modem cadangan", "Coba sambungkan ulang"). Risiko interchangeability ada di baris KPI + donat + line chart (pola kategori standar), tapi diselamatkan oleh konteks log/terminal. Klaim kontras di `dna.json` **terverifikasi benar via perhitungan** kecuali dua celah: `--faint` (3.71 light / 3.83 dark) dan badge nav di dark mode (2.93) — lihat temuan KRITIS 2 & 3.

## Kesan Keseluruhan

Alur-F bekerja persis seperti DNA: alarm → angka KPI → log live → grafik pendukung. Dalam 3 detik pertama mata menangkap satu-satunya hal yang penting (modem down, 47 antrean). Ini dashboard yang menghormati operator shift malam. Lima temuan KRITIS di bawah semuanya soal **kepercayaan pada data** (nama modem, angka yang jujur, teks yang terbaca) — bukan soal selera.

## Yang Berhasil

1. **Strip krisis = satu-satunya elemen berani.** Solid merah + pulse 2.6s + tombol aksi "Buka terminal" + `aria-live="assertive"`. Tidak ada hero kedua. Copy-nya instruktif, bukan dekoratif.
2. **Semantik warna disiplin.** Satu hue per makna di badge, donat, dan bar sinyal (hijau=ok, amber=lemah/pending, merah=gagal/down, abu=mati). Bar sinyal 4-segmen dengan `aria-label` ("Sinyal kuat/sedang/lemah") — tidak mengandalkan warna saja.
3. **Densitas tabel jujur.** Baris 31px, header sticky, font 12.5px, ~20 baris dalam 624px — scanability Excel-like sesuai brief, plus flash 1.6s untuk baris baru dan `prefers-reduced-motion` dihormati.

## Temuan KRITIS (wajib diperbaiki — 5)

### K1. Nama modem tidak konsisten antara strip krisis dan panel terminal
- **Lokasi:** `.crisis .msg` ("**Modem 3 (Tri)** down 12 menit") dan `.kpi .note` ("47 di antaranya menunggu **modem 3**") vs `.modem` rows: M3 = **XL** (Lemah, /dev/ttyUSB2), M4 = **Tri** (Down, /dev/ttyUSB3).
- **Dampak:** Di momen krisis, operator klik "Buka terminal" lalu mencari "Modem 3 (Tri)" — yang ia temukan: M3=XL lemah dan M4=Tri down. Dua skema penamaan bertabrakan tepat saat keputusan harus cepat. Ini merusak kepercayaan pada strip krisis, elemen terpenting layar ini.
- **Perbaikan:** Samakan skema — strip krisis harus berkata "**M4 (Tri)** down 12 menit" (atau ganti label rel jadi nama operator saja tanpa M1–M4). Satu sumber kebenaran untuk penamaan.

### K2. Teks `--faint` gagal WCAG AA di kedua tema
- **Lokasi:** `--faint: #8a847b` (light, `:root` baris ±17) = **3.71:1** di atas surface; `--faint: #7f776b` (dark, baris ±33) = **3.83:1** di atas surface. Dipakai untuk info operasional nyata: label nav ("Menu", "Lainnya"), hint kartu ("Semua modul · diperbarui tiap jam", "4 port aktif"), port modem (`/dev/ttyUSB0 · M1`), CSQ ("CSQ 24,99 · 8 dtk lalu"), persen donat, `tfoot`, note KPI, label sumbu grafik.
- **Dampak:** DNA menargetkan ≥4.5:1; teks sekunder yang memuat data (CSQ, timestamp, port) tidak terbaca nyaman — operator shift malam dengan mata lelah adalah kasus terburuk untuk kontras rendah.
- **Perbaikan:** Gelapkan ke ±`#6e6860` (light) / terangkan ke ±`#a39a8b` (dark), verifikasi ulang hingga ≥4.5:1. (Catatan: audit DNA sudah memperbaiki `--muted` dari `#767169` tapi `--faint` lolos.)

### K3. Badge angka nav tidak terbaca di dark mode
- **Lokasi:** `.nav-badge` (baris ±120) — teks putih di atas `var(--err)` yang di dark = `#ef6f6f` → **2.93:1** (terukur). Di light (`#cf3d3d`) masih 4.79:1, lolos.
- **Dampak:** Angka "1" (jumlah alarm) di "Modul & terminal" adalah sinyal penting; di dark mode nyaris tak terbaca. Audit kontras DNA tidak mencakup kasus ini.
- **Perbaikan:** Ikuti pola badge status yang sudah benar: teks gelap di atas tint (`--err-ink` di `--err-bg`), atau pertahankan merah solid dengan teks `#1a0b0b`.

### K4. Legend "Rata-rata 6.480/hari" tanpa garis di grafik
- **Lokasi:** `.chart-legend` (HTML ±baris 440) menampilkan swatch + "Rata-rata 6.480/hari", tetapi `drawChart()` (JS) tidak menggambar garis rata-rata apa pun — terkonfirmasi di kode dan di kedua screenshot.
- **Dampak:** Legend menjanjikan elemen visual yang tidak ada; operator mencari garis acuan yang tak pernah muncul. Grafik berbohong secara pasif.
- **Perbaikan:** Gambar garis rata-rata (dashed, warna `--faint` yang sudah diperbaiki di K2) pada nilai 6480, atau hapus item legend.

### K5. Tidak ada navigasi sama sekali di ≤900px
- **Lokasi:** `@media (max-width:900px){ .sidebar{display:none} }` (±baris 300) — tanpa hamburger, bottom bar, atau alternatif apa pun.
- **Dampak:** Brief mewajibkan "HP operator untuk monitoring". Di 390px (terverifikasi via screenshot) user terkunci di satu layar: tidak bisa ke Transaksi, Produk, atau Pengaturan. Untuk produk 22-layar, ini bukan degradasi anggun — ini jalan buntu.
- **Perbaikan:** Tambahkan bottom navigation bar (5 item: Dashboard, Transaksi, Produk, Reseller, Modul) khusus ≤900px, atau hamburger yang membuka drawer.

## SARAN (opsional, berdampak)

- **S1. Garis grafik light gagal kontras non-teks.** `#f66e27` di putih = **2.93:1** (< 3:1 WCAG 1.4.11 untuk grafis data). Pakai `--orange-deep` (`#d95a15` → 3.87:1) untuk stroke garis di light mode; dark sudah 7.50:1, aman.
- **S2. "Jeda live" jadi elemen paling terang di dark mode.** `btn-pause` memakai `background:var(--ink)` → putih terang di atas warm-coal; ia menyaingi strip krisis sebagai "satu-satunya elemen berani" (DNA). Pertimbangkan `background:var(--chip)` + border di dark.
- **S3. Delta hijau untuk "Gagal ↓ 18%".** Benar secara "kabar baik", tapi meregangkan aturan DNA "satu hue per makna" (hijau=sukses di badge/donat). Operator yang memindai cepat bisa salah baca "61 gagal = hijau = aman". Pertimbangkan badge netral untuk tren, atau tambahkan kata "membaik".
- **S4. Hint "4 port aktif" tidak jujur.** Tri Down, XL Lemah — yang benar-benar aktif hanya 2–3. Ganti jadi "4 port terpasang · 1 down" atau hitung dinamis.
- **S5. Chips filter memakai `role=tab` tanpa tabpanel dan tanpa navigasi keyboard panah.** Ganti ke `role="group"` + `aria-pressed`, atau lengkapi pola tabs yang benar.
- **S6. Default tema = light, padahal brief menetapkan dark** ("operator kerja malam; konsisten dengan mockup v1"). Toggle persist membantu, tapi kesan pertama untuk audiens shift-malam seharusnya dark. Pertimbangkan default dark (atau `prefers-color-scheme`).
- **S7. `rowflash` oranye untuk semua baris baru** termasuk yang Gagal — warna flash tidak membedakan status. Pertimbangkan flash netral, atau biarkan badge status yang berbicara.

## CATATAN

- **C1.** Hierarki F-flow terverifikasi di 1600px: judul → strip krisis → 4 KPI → grafik (2:1) → tabel live + rel modem (2:1). Urutan baca benar.
- **C2.** Tabel 8 kolom di 390px menjadi scroll horizontal di dalam `.table-scroll` (`overflow-x` terkomputasi `auto`) — pola yang dapat diterima untuk tabel padat; bila ingin tanpa scroll, sembunyikan kolom Reseller/Modul di <480px.
- **C3.** Copy Indonesia sudah sesuai brand voice DNA: kalimat aktif, sentence case di mana-mana, tanpa eyebrow ALL-CAPS. ("Jeda live", "Coba sambungkan ulang", "Pindahkan beban ke modem cadangan").
- **C4.** Angka KPI ikut berubah tiap 2.6 detik karena feed simulasi — di produksi, throttle update agregat (mis. 30 detik) agar angka tidak berkedip dan mengganggu pemindaian.
- **C5.** `th` tanpa `scope="col"`; bar sinyal dan badge status sudah punya label non-warna — fondasi a11y baik, tinggal poles.

## Persona Red Flags

- **Alex (power user):** Tidak ada keyboard shortcut — "Jeda live", filter chips, dan search semuanya butuh mouse. Tidak ada `/` untuk fokus pencarian, tidak ada `Esc`/spasi untuk pause. Untuk operator yang memantau 8 jam, ini memperlambat respons krisis.
- **Sam (aksesibilitas):** Kontras `--faint` gagal (K2), badge nav dark gagal (K3), pola `role=tab` tanpa tabpanel membingungkan screen reader (S5). Bar sinyal punya `aria-label` teks — bagus, pertahankan.
- **Operator shift malam (persona proyek):** Default light theme (S6) + teks faint berkontras rendah (K2) = kombinasi terburuk untuk mata lelah di ruangan gelap. Dark mode-nya sendiri sudah baik (warm-coal, bukan hitam pekat).

## Pertanyaan untuk dipertimbangkan

- Jika strip krisis adalah satu-satunya elemen berani, apakah tombol "Jeda live" putih di dark mode melanggar aturan itu — atau justru pause adalah aksi darurat kedua yang layak menonjol?
- Apakah penamaan modem harus mengikuti label hardware (M1–M4) atau nama operator (Telkomsel/Indosat/XL/Tri) sebagai identitas utama — mana yang operator ucapkan saat menelepon teknisi?
- Grafik 7-hari: apakah operator benar-benar butuh garis rata-rata, atau legend dua-baris itu sisa dari iterasi sebelumnya yang bisa dihapus saja?
