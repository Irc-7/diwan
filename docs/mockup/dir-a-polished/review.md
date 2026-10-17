# Review Audit — dir-a-polished (mockup dashboard OtomaX)

Tanggal: 2026-10-17 · Reviewer: subagent auditor · Impeccable TIDAK dipakai.
Metode: hitung rasio kontras WCAG dari hex aktual di CSS (bukan tebakan), render headless
Chromium 1440×900 & 390×844, ukur rect/DOM via Playwright-core, cek checklist
ux-guidelines.csv yang relevan (#22/#28/#36/#37/#40/#43/#69/#71/#104/#108).

**Yang lolos verifikasi (bukan temuan):** hierarki — strip krisis memang satu-satunya elemen
bold/solid saturated, dominan secara visual (krisis > KPI > tabel tercapai secara salience);
semua pasangan kontras lain ≥4.5:1 (terverifikasi: ink/surface 14.80, body/bg 12.20,
muted/bg 8.06, badge SUK 8.74, PRC 9.20, GGL 7.40, teks utama krisis putih/crisis 4.58,
kode TX info/surface 6.22, bottomnav aktif oranye/surface 7.50); token hex semua terdaftar
di `:root` (duplikasi `stroke="#54c47e"` di `spark()` hanya nit, nilai = token `--ok`);
status semantik konsisten (SUK=hijau, PRC=amber, GGL=merah); aria-label 10 tombol rail +
tombol × krisis + bottomnav ada; legend + `title` badge ada; `role="img"`+aria-label pada
bar sinyal ada; `:focus-visible` 2px oranye ada; `prefers-reduced-motion` mematikan semua
animasi; bottomnav tampil ≤900px dengan target 48px + `aria-current`.

---

## Temuan kritis

### F1 — Kontras gagal: sub-baris strip krisis (4.11:1 < 4.5)
- **Lokasi:** `index.html` baris 112 — `.cr-txt span{display:block;font-size:11.5px;opacity:.92}`
- **Masalah:** teks "12 transaksi antre tertunda — dialihkan otomatis ke M1/M2" adalah teks
  normal 11.5px (bukan teks besar) sehingga butuh ≥4.5:1. Opacity .92 menurunkan putih di
  atas `#cf4636` menjadi efektif `#fbf0ef`.
- **Bukti:** hitungan relative-luminance WCAG dari nilai aktual → **4.11:1** (gagal AA).
  Catatan: polish-notes §4 hanya menghitung teks utama (4.58:1) dan melewatkan opacity ini.
- **Saran:** hapus `opacity:.92` (pakai `#fff` penuh → 4.58:1, lolos AA tanpa ubah token),
  atau naikkan ke teks besar (≥18.66px bold / 24px) bila opacity dipertahankan.

### F2 — Kontras gagal: tombol × dismiss krisis (3.72:1 < 4.5)
- **Lokasi:** `index.html` baris 116 — `.cr-x{...font-size:18px;...opacity:.85}`
- **Masalah:** glif "×" 18px non-bold = teks normal (ambang teks besar butuh ≥14pt bold
  ≈ 18.66px), butuh ≥4.5:1. Opacity .85 → putih efektif `#f8e3e1` di atas `#cf4636`.
- **Bukti:** hitungan WCAG dari nilai aktual → **3.72:1** (gagal; bahkan di bawah 4.5 dengan
  margin besar). Kondisi hover/focus `opacity:1` lolos (4.58:1), tapi state default gagal.
- **Saran:** `opacity:1` permanen pada `.cr-x` (4.58:1), ganti efek hover ke background saja
  (sudah ada `background:rgba(0,0,0,.25)`).

### F3 — Overflow horizontal di mobile: 4 kontrol inti keluar viewport (≤900px)
- **Lokasi:** `index.html` baris 106 (`.crisis` flex nowrap), 133 (`.tab-tools` +
  `margin-left:auto`), media query baris 233.
- **Masalah:** pada viewport 390px, `.tabs` (flex, nowrap) dan `.crisis` (flex, nowrap)
  melebar hingga ~710px. Hasil ukur Playwright (390×844): `document.scrollWidth` = **730px**
  vs viewport 390px. Kontrol yang jatuh di luar viewport: `.cari` (x=401, w=190),
  `.btn-jeda` "Jeda live" (x=601), `.cr-btn` "Lihat antrean" (x=570), `.cr-x` dismiss (x=684).
  Ini melanggar guideline #69 (hindari horizontal scroll, High) dan #108: tombol
  jeda/lanjut — kontrol wajib untuk konten auto-update 2-detik — tidak terlihat/tersentuh
  tanpa menggeser horizontal seluruh kolom (yang berkonflik dengan scroll horizontal tabel).
  Klaim polish-notes §6 ("tidak ada overflow horizontal tak disengaja") terbukti salah.
- **Bukti:** angka rect di atas + screenshot mobile (`crop-tabs.png`, `crop-crisis.png`):
  baris tab hanya menampilkan 3 tab; strip krisis hanya ikon + teks.
- **Saran:** di `@media (max-width:900px)`: `.tabs{flex-wrap:wrap}` dan pindahkan
  `.tab-tools` ke baris sendiri full-width (`.tab-tools{margin-left:0;flex-basis:100%}`,
  `.cari{flex:1}`); `.crisis{flex-wrap:wrap}` atau jadikan `.cr-btn` full-width di bawah teks.

### F4 — Input palet perintah tanpa label aksesibel
- **Lokasi:** `index.html` baris 414 —
  `<input id="palIn" placeholder="Ketik perintah… (mis. jeda, gagal, produk)" autocomplete="off">`
- **Masalah:** tidak ada `aria-label`, `aria-labelledby`, maupun `<label>` terasosiasi —
  hanya placeholder. Melanggar guideline #43 (Input Labels, High): screen reader
  mengumumkan input ini tanpa nama ("edit text" kosong). Input `.cari` di file yang sama
  sudah benar (`aria-label="Cari transaksi"`), jadi ini inkonsistensi.
- **Bukti:** DOM check via Playwright: `aria-label: null, aria-labelledby: null`.
- **Saran:** tambah `aria-label="Ketik perintah"` pada `#palIn` (atau `<label class="sr-only"
  for="palIn">`).

---

## Catatan untuk parent
- Tidak ada temuan kritis pada: hierarki visual, konsistensi token/status semantik,
  aria-label tombol ikon (lolos), legend badge, `prefers-reduced-motion`, bottom nav
  (render benar, 5 item, target ≥48px), dan 17 pasangan kontras lain (semua ≥4.5:1).
- F1 & F2 adalah regresi dari keputusan kontras polish-notes §4 yang hanya dihitung untuk
  teks utama tanpa memperhitungkan `opacity`.
- F3 adalah temuan terberat: 4 kontrol operasional hilang dari viewport mobile.
- Tidak ada perbaikan dilakukan — hanya pelaporan, sesuai mandat.
