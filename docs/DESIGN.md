# DESIGN.md — OtomaX Linux Web UI · Design System Terkunci

**Tanggal kunci:** 2026-10-17 · **Arah:** "Terminal Operasi" (Arah A), dikunci user ·
**Sumber:** mockup `docs/mockup/dir-a-polished/index.html` (di-polish + diaudit),
`polish-notes.md`, `review.md`, `docs/mockup/shipmate-improved/dna.json` ·
**Status:** ini adalah referensi tunggal komponen — semua 22 layar mengacu ke sini.
impeccable **tidak dipakai** untuk dashboard ini (keputusan user); review memakai
ux-guidelines ui-ux-pro-max + verifikasi screenshot.

Catatan kejujuran: mockup hanya menunjukkan layar dashboard. Komponen yang
**tidak muncul** di mockup (form select/datetime/textarea/checkbox, modal/wizard,
bulk action bar, filter chips, toggle, empty/error state) dispesifikasikan dari
token & konvensi DNA yang sama dan ditandai `SPEC` — bukan temuan dari mockup.

---

## 1. Token warna (dark — tema utama, sudah live)

Semua hex dari `:root` mockup final. Kontras diukur WCAG dari nilai aktual
(bukan tebakan); hasil audit: 17 pasangan ≥ 4.5:1 lolos, 4 temuan diperbaiki.

| Token | Hex | Peran | Kontras (atas latar) |
|---|---|---|---|
| `--bg` | `#161311` | Latar body, input cari, command palette | — |
| `--surface` | `#201c18` | Panel, topbar, rail, kartu KPI, tabel | — |
| `--surface2` | `#262019` | Hover baris, kbd, tombol jeda, toast | — |
| `--line` | `#332c24` | Border panel, thead, thumb scrollbar | — (bukan teks) |
| `--line-soft` | `#2b251e` | Border antar-baris tabel/modem | — (bukan teks) |
| `--frame` | `#2c2721` | Bingkai struktur (dari DNA, cadangan) | — |
| `--chip` | `#2a241e` | Tooltip rail, chip | — |
| `--ink` | `#f4efe8` | Teks utama: heading, angka besar, clock | **14.80:1** di `--surface` ✓ |
| `--body` | `#d8d1c6` | Isi sel tabel, item palet | **12.20:1** di `--bg` ✓ |
| `--muted` | `#b3aa9c` | Teks sekunder: label KPI, th, placeholder, timestamp | **8.06:1** di `--bg` ✓ |
| `--faint` | `#7f776b` | **Hanya grafis** — bar sinyal kosong @35% opacity + teks CSQ pendamping | 4.19:1 — **dilarang untuk teks** |
| `--orange` | `#ff8f52` | Aksen ShipMate: indikator tab aktif, rail aktif, `:focus-visible` 2px, tombol teks nav aktif | **7.50:1** di `--surface` ✓ |
| `--orange-deep` | `#e07a35` | Hover border (tombol/cari) | — |
| `--orange-ink` | `#ffb98a` | Count tab aktif | ✓ lolos |
| `--orange-soft` | `#3d2817` | Tint latar aksen (cadangan) | — |
| `--info` | `#8b96f0` | Kode TX, pesan RX di log AT | **6.22:1** di `--surface` ✓ |

### Status semantik (1 hue per makna — aturan DNA)

| Status | Token teks/bg | Teks | Latar | Kontras | Arti |
|---|---|---|---|---|---|
| SUK = Sukses | `--ok-ink #8fdcab` / `--ok-bg #183024` | hijau terang | hijau gelap | **8.74:1** ✓ | Sukses |
| PRC = Diproses | `--warn-ink #f2cc82` / `--warn-bg #38290f` | amber terang | amber gelap | **9.20:1** ✓ | Diproses (dulu biru — diganti karena bentrok dengan info) |
| GGL = Gagal | `--err-ink #f49c9c` / `--err-bg #3a1c1c` | merah terang | merah gelap | **7.40:1** ✓ | Gagal |

Token mentah status (dot live, bar sinyal, pulse): `--ok #54c47e`, `--warn #eeb04a`,
`--err #ef6f6f`. Legend wajib di bilah tab: `SUK = Sukses, PRC = Diproses, GGL = Gagal`
(koma, bukan pola titik tengah; disembunyikan ≤1180px, tooltip `title` tetap ada);
tiap badge punya `title` ("SUK = Sukses", "PRC = Diproses", "GGL = Gagal").

### Strip krisis — satu-satunya elemen bold

| Token | Hex | Aturan kontras |
|---|---|---|
| `--crisis` | `#cf4636` | Solid merah; **teks putih murni `#fff` → 4.58:1 ✓** (dilarang `opacity < 1` pada teks krisis — temuan F1/F2 audit). Tombol aksi: `rgba(0,0,0,.30)` + putih ≈ 5.9:1. |

Keputusan: teks di atas krisis memakai `#ffffff`, **bukan** `--ink` (4.01:1, gagal).

### Light theme — BELUM live (toggle masa depan)

Light theme **tidak ada di mockup** (mockup hanya dark). Nilai dari
`shipmate-improved/dna.json` dicadangkan untuk toggle `localStorage 'ox-theme'`
(knob + label "Mode gelap"/"Mode terang" di sidebar — pola dari DNA shipmate).
Implementasi = 22 layar dulu dark-only; token light di bawah adalah kontrak:

| Token | Hex (light) | Token | Hex (light) |
|---|---|---|---|
| `--bg` | `#fbfbfa` | `--crisis` | `#b3261e` (putih ≈ 6.5:1) |
| `--surface` | `#ffffff` | `--ok-ink` / `--ok-bg` | `#1c6a3a` / `#e1f2e7` |
| `--surface2` | `#f5f3f1` | `--warn-ink` / `--warn-bg` | `#7a4f00` / `#fae7bd` |
| `--ink` | `#191919` | `--err-ink` / `--err-bg` | `#9c2b2b` / `#fadada` |
| `--body` | `#33302c` | `--info` | `#4f5fd6` |
| `--muted` | `#5b564e` (6.8:1 — bukan `#767169` yang 4.1:1) | `--orange` | `#f66e27` |
| `--line` | `#e7e2dd` | `--ok` | `#2e9e5b` |
| `--line-soft` | `#efe9e4` | `--warn` | `#cf8617` |
| `--chip` | `#f4f1ee` | `--err` | `#cf3d3d` |
| `--faint` | `#8a847b` | `--frame` | `#f0eeed` |

---

## 2. Tipografi

Mockup memakai **Inter + JetBrains Mono** (Google Fonts, gratis; system-stack fallback).
Catatan: dna.json menulis "Plus Jakarta Sans" sebagai cita-cita DNA shipmate — **mockup
terkunci memakai Inter** (`--sans:'Inter',system-ui,sans-serif`); DNA JSON harus
diperbarui bila ingin selaras. Gunakan Inter.

| Peran | Ukuran / weight | Contoh pakai |
|---|---|---|
| KPI display | `20px / 600 / mono` | Nilai metrik ("4.812", "98,2%") |
| H1 | `24px / 800 / sans` | Judul layar |
| Brand terminal | `14px / 700 / mono` | `>_` otomaX (prompt `>_` warna `--ok-ink`) |
| Body UI | `13px / 400 / sans` | Body default, item palet (12.5px) |
| H2 / judul seksi | `12.5px / 600 / sans` (mockup) · DNA: `15.5px / 700` | Header panel `.sect-h h2` |
| Tabel data | `12px / 400–500 / mono, tabular-nums` | Sel tabel; kode TX 500 + `--info` |
| Label | `11px / 500–600 / sans` | Label KPI, th tabel (600), timestamp log AT 10.8px mono |
| Caption | `11px / 600 / sans` | Caption dari DNA (10.5px mono untuk sub-label KPI, legend) |
| Small chip | `10.5px / 700 / mono, +5% letter-spacing` | Badge SUK/PRC/GGL |
| Unit | `13px / 400 / muted` | "jt", "/4" di samping angka KPI |

Aturan:
- **Data mesin = JetBrains Mono** (kode transaksi, waktu, nomor HP, nominal, port,
  log AT, clock, search input) — "suara buku log", pembeda dari SaaS generik.
- **Sentence case** di semua label, tombol, badge — tidak ada eyebrow ALL-CAPS
  (kecuali badge status 3-huruf yang memang kode).
- Bahasa Indonesia aktif: "Jeda live", "Lihat antrean", "Tutup peringatan krisis".

---

## 3. Spacing / radius / elevasi

Nilai persis dari CSS mockup:

| Kategori | Nilai |
|---|---|
| Layout shell | grid `top 46px / nav 58px / foot 28px`; konten `grid-template-columns: minmax(0,1fr) 322px`, gap `12px`, padding `12px` |
| Panel padding | `10px 12px` (seksi), `10px 14px` (strip krisis, strip laris), `14px 16px` (input palet) |
| KPI | `padding:10px 14px`, border-right `1px --line-soft`; tabel `padding:8px 10px` th, baris `height:31px`, sel `0 10px` |
| Radius | kartu/panel/metric/strip/tabel: `7px`; tombol/input/tab-tools: `6px`; rail: `7px`; tooltip: `6px`; kbd: `4px`; badge: `4px`; command palette: `9px`; bottomnav item: `10px`; pill: `999px` (DNA) |
| Border | `1px solid --line` (panel), `1px --line-soft` (antar-baris); scrollbar thumb `10px`/`radius 5px` |
| Elevasi (dark) | `0 24px 70px rgba(0,0,0,.6)` (palette), `0 10px 30px rgba(0,0,0,.5)` (toast); DNA: `0 8px 24px rgba(0,0,0,.38)` untuk light `0 4px 16px rgba(31,30,29,.06)` |
| Focus | `:focus-visible` — `outline:2px solid --orange; outline-offset:1px` (wajib di semua kontrol) |
| Selection | `background:rgba(255,143,82,.28)` |
| Mobile ≤900px | body jadi 1 kolom, `padding-bottom:96px`, KPI swipe horizontal (kartu `flex:0 0 196px`), tabel `min-width:680px` dalam scroll-x, rail disembunyikan, bottom nav fixed |

Densitas: tabel 31px/baris, ~20 baris terlihat — kepadatan adalah fitur.

---

## 4. Komponen

> `LIVE` = diamati di mockup. `SPEC` = spesifikasi dari token DNA, belum ada di mockup
> (wajib diuji di 22 layar).

### 4.1 Badge SUK/PRC/GGL + legend — LIVE
Anatomi: `inline-flex`, ikon dot `6px`, font mono `10.5px/700`, `letter-spacing:.05em`,
`padding:3px 8px`, radius `4px`; `b-suk/b-prc/b-ggl` = `{warna}-ink` + `{warna}-bg`.
PRC punya `denyut` pulse (lihat §6). Aturan: badge selalu punya `title`
("SUK = Sukses" dsb.); legend kecil `SUK = Sukses, PRC = Diproses, GGL = Gagal`
mono `10.5px` di bilah tab (hide ≤1180px); status tidak pernah mengandalkan warna
saja — selalu ada teks/badge/tooltip.

### 4.2 Kartu KPI — LIVE
Anatomi: grid 5 kolom dalam 1 panel; tiap metrik label `11px/500/muted`,
nilai mono `20px/600/ink` tabular-nums, sub-label mono `10.5px/muted` berisi delta
(`▲/▼` warna `{warna}-ink`: up ok, warn, bad err), unit inline `13px/muted`
("jt", "/4"). Aturan: satu metrik = satu fakta; delta selalu bandingkan
("vs kemarin", "7 hari terakhir"); kartu borderless, pemisah `line-soft`.
(DNA shipmate menyebut aksen 3px kiri + display 29px — mockup terkunci memakai
20px tanpa aksen; ikut mockup.)

### 4.3 Tabel live — LIVE
Anatomi: baris `31px`, font mono `12px` tabular-nums, thead sticky (`position:sticky;
top:0`, bg surface, z-index 2), th sans `11px/600/muted`, td border-b `line-soft`,
hover `--surface2`. Baris baru: class `flash-suk`/`flash-ggl` — `animation fls/flg .9s
ease-out` dari `rgba(ok,.20)` / `rgba(err,.22)` → transparan (DNA: 1.6s; mockup: .9s —
ikuti mockup). Kode TX: `td.kode` `--info` 500. Toolbar: badge Live berdenyut +
jam WIB mono + chips filter berhitung + pencarian + tombol "Jeda live". Mobile:
`min-width:680px` + scroll horizontal (ux-guideline #71). Selalu `aria-label` tabel.

### 4.4 Command palette (Ctrl+K) — LIVE
Anatomi: overlay fixed, `rgba(10,7,5,.62)`, padding-top `14vh`; kotak `560px`
(radius `9px`, border `line`, shadow `0 24px 70px rgba(0,0,0,.6)`); input mono
`13.5px`, `padding:14px 16px`, border-b `line`; daftar item `padding:9px 10px`,
radius `6px`, hover/selected `--surface2`; shortcut kanan pakai `<kbd>`. Tombol
pemanggil di topbar (bg `--bg`, border `line`, radius `6px`, `padding:7px 10px`,
hint ellipsis). Aturan: selalu ada input kosong (`aria-label="Ketik perintah"`,
F4 audit) + state kosong; `aria-label="Buka palet perintah"` pada tombol.
Item: ikon kiri + label + kbd kanan.

### 4.5 Strip krisis — LIVE (SATU-SATUNYA elemen bold per layar)
Anatomi: solid `--crisis`, teks `#ffffff`, radius `7px`, `padding:10px 14px`,
ikon `!` bulat border putih, judul `12.5px/700`, sub-baris `11.5px`
(**tanpa opacity**), tombol aksi `rgba(0,0,0,.30)` + border putih 35% radius `6px`,
tombol × putih `18px` opacity `1`. `role="alert"`. Aturan: maksimal satu per layar;
tidak boleh ada elemen bold/solid kedua; × wajib `aria-label` bermakna; dismiss
benar-benar menutup (bukan memudarkan).

### 4.6 Panel modem + bar sinyal — LIVE
Anatomi: grid 2 kolom (`1fr auto`), nama mono `12px/600/ink` + operator `11px/muted`,
sub `10.5px/muted` (port, CSQ), `sdot` `7px` (ok=glow, warn, bad), bar sinyal 4 segmen
`width:4px` tinggi bertingkat `18px` — terisi `--ok/--warn/--err`, kosong `--faint @35%`.
Aturan: bar sinyal punya `role="img"` + `aria-label` ("Sinyal M1: CSQ 24/31") —
tidak mengandalkan warna (ux-guideline #37); angka CSQ mono selalu ditampilkan.

### 4.7 Log AT — LIVE
Anatomi: mono `10.8px`, `line-height:1.75`, warna dasar `muted`; TX (kita kirim) =
`--ink`, RX (diterima) = `--info`, OK = `--ok-ink`, ER = `--err-ink`; tinggi
`264px` (desktop), scroll-y, auto-scroll ke bawah saat data baru (mockup: interval
2 dtk). Aturan: tidak ada interpretasi — tampilkan apa adanya; error berwarna
`err-ink` plus langkah perbaikan di §5.

### 4.8 Tombol — LIVE (primer/sekunder/danger SPEC mengikuti token)
- **Sekunder/ghost** (LIVE): "Jeda live" — bg `--surface2`, border `line`,
  radius `6px`, mono `11.5px/600`, `padding:6px 11px`; hover border `--orange-deep`;
  state jeda = teks/border amber + `aria-pressed`. "Lihat antrean" (di krisis)
  — `rgba(0,0,0,.30)`, putih, border putih 35%.
- **Primer** (SPEC): bg `--orange` + teks `--bg` (kontras ≥7:1, konsisten token),
  radius `6px`, padding `8px 14px`, hover `--orange-deep`; focus ring oranye 2px
  (bukan ring putih).
- **Danger** (SPEC): bg `--err-bg`, teks `--err-ink`, border `rgba(239,111,111,.45)` —
  aksi destruktif selalu butuh konfirmasi (§5); dilarang danger solid besar
  kecuali di dalam dialog konfirmasi.

### 4.9 Form field — `.cari` LIVE, sisanya SPEC
- **Input/search** (LIVE): bg `--bg`, border `line`, radius `6px`, mono `11.5px`,
  `padding:6px 10px`; placeholder `--muted`; focus: border `--orange-deep`,
  `outline:none` (ring diwakili `:focus-visible` oranye). Wajib `aria-label`.
- **Select / datetime** (SPEC): pakai anatomi input yang sama; ikon chevron/calendar
  `--muted` 15px; menu dropdown = surface, border line, radius 7px, shadow toast.
- **Textarea** (SPEC): anatomi input; min-height 3 baris; mono bila untuk template/AT.
- **Checkbox** (SPEC): kotak `16px`, radius `4px`, border `line`; checked = bg
  `--orange` + centang `--bg`; label `--body` 13px, klik seluruh baris.

### 4.10 Modal / wizard multi-step — SPEC
Anatomi mengikuti command palette: overlay `rgba(10,7,5,.62)`, kotak surface
radius `9px`, border `line`, shadow `0 24px 70px rgba(0,0,0,.6)`; header sans
`13px/700/ink` + langkah ("Langkah 2 dari 4", mono `11px/muted`) + × 18px;
body `14px`; footer: tombol sekunder kiri ("Kembali"), primer kanan ("Lanjut") —
urutan Indonesia. Aturan: fokus masuk ke modal + trap; Esc menutup (kecuali wizard
dengan data kotor → tanya dulu); tombol primer dinonaktifkan sampai valid, bukan
error misterius.

### 4.11 Bulk action bar — SPEC
Anatomi: bar mengambang di atas tabel saat ≥1 baris dicentang — bg `--chip`,
border `line`, radius `7px`, padding `8px 12px`; teks "3 dipilih" mono `12px/ink`;
aksi sekunder (label/arsip) + danger (hapus) kanan. Aturan: muncul hanya saat ada
pilihan; Esc / batal = pilihan dihapus; aksi destruktif → dialog konfirmasi §5.

### 4.12 Filter chips — SPEC
Anatomi: pill radius `999px`, `padding:4px 10px`, mono `11px`; idle: bg `--chip`,
teks `muted`; aktif: bg `--orange-soft` atau `{warna}-bg`, teks `{warna}-ink`
(bila chip status, warna mengikuti makna). Dengan hitungan: "Gagal (3)" tabular-nums.
Aturan: chip = filter, bukan tombol aksi; reset filter selalu tersedia bila ada
filter aktif.

### 4.13 Bottom nav mobile (≤900px) — LIVE
Anatomi: fixed bawah, bg `--surface`, border-top `line`, padding hormat safe-area;
5 item (Dashboard, Transaksi, Terminal, Inbox, Pengaturan), ikon `20px`,
label `10.5px/600`, `min-height:48px` per item (target sentuh #22/#66), aktif =
`--orange` + `aria-current`. Aturan: rail kiri disembunyikan di ≤900px; konten
`padding-bottom:96px` agar tak tertutup; ≤5 item.

### 4.14 Toggle — SPEC
Anatomi: track `36×20px` radius `999px`, bg `--line` (off) / `--ok` atau `--orange`
(on — pakai `--ok` untuk fitur on/off, `--orange` untuk preferensi non-biner);
knob putih `16px`; label di kanan, `--body` 13px. Aturan: state selalu tercermin
di label/status teks (bukan warna saja); `role="switch"` + `aria-checked`.
(Contoh dari DNA: theme toggle dengan knob + label "Mode gelap/Mode terang",
persist `localStorage 'ox-theme'` — untuk masa depan, bukan sekarang.)

---

## 5. Pola

### Empty state
Struktur: ikon outline `--faint` (bukan `--muted`, agar tidak dibaca sebagai teks),
kalimat sans `13px/--body` yang menjelaskan **apa yang kosong**,
satu kalimat tindakan berikutnya ("Tambahkan modem dulu di Modul & terminal"),
satu tombol sekunder/primer. Dilarang: ilustrasi besar, teks kosong tanpa CTA.

### Error state — menjelaskan cara perbaiki, tanpa minta maaf
Struktur: ikon/sdot `--err`, judul sans `13px/700/--err-ink` (bukan "Maaf…",
bukan "Terjadi kesalahan"), kalimat berisi **penyebab + langkah**: contoh
"Gagal kirim via M4 — tidak ada sinyal. Pindahkan beban ke modem cadangan,
atau coba lagi setelah sinyal kembali." + tombol aksi ("Pindahkan ke M1/M2",
"Coba lagi"). Tone brand: tegas, bukan sopan — merah solid + angka + instruksi,
bukan teks pink lembut. Log AT menampilkan baris ER apa adanya sebagai bukti.

### Konfirmasi aksi destruktif
Dialog modal (§4.10): judul "Hapus 3 transaksi?", isi menyebut dampak
("Tidak bisa dibatalkan. Saldo reseller tidak terpengaruh." — sebutkan yang
**tidak** ikut terhapus bila relevan), input ketik-nama untuk aksi masif
(≥50 item / hapus akun / reset pabrik), tombol danger kanan ("Hapus"),
sekunder kiri ("Batal"), default fokus = "Batal". Esc = batal. Tidak ada
konfirmasi untuk aksi reversibel (pin/unpin, jeda live).

---

## 6. Motion

Aturan: **motion = sinyal, bukan dekorasi** — setiap animasi menjawab aksi user
atau perubahan data. Yang diizinkan (semua dari mockup/DNA, tanpa tambahan):

| Efek | Trigger | Nilai |
|---|---|---|
| Flash baris baru | transaksi masuk (SUK/GGL) | `.9s ease-out`, dari `rgba(84,196,126,.20)` / `rgba(239,111,111,.22)` → transparan |
| Pulse badge Live | selalu saat mode live | dot `8px` + `box-shadow` denyut (dna: `ping 1.6s`) |
| Denyut PRC | badge PRC aktif | `denyut 1.2s` (opacity 1 ↔ .25) |
| Tooltip rail | hover | fade `.12s` |
| Crisis pulse | alarm aktif | box-shadow pulse `2.6s` (DNA; mockup: solid — pulse opsional bila tidak mengganggu kontras) |

**Dilarang:** entrance fade-slide-up per section, parallax, gradient dekoratif,
glassmorphism, partikel, custom cursor, spinner dekoratif tanpa status teks.

`prefers-reduced-motion`: semua `animation`/`transition` → `.001s` +
`iteration-count:1`; flash baris + denyut PRC + transisi tooltip dimatikan
eksplisit. **State akhir tetap benar tanpa `animationend`**: class flash dihapus
via timeout yang tetap berjalan (bukan menunggu event animasi) — ux-guideline #119.

---

## 7. Aturan anti-slop (mengikat)

1. **Satu elemen bold per layar.** Di dashboard = strip krisis. Tidak ada hero kedua,
   tidak ada kartu "hero" lain yang bersaing.
2. **Hindari default AI-generik:** dilarang SaaS-card kit (grid kartu putih seragam
   dengan ikon pastel), dilarang eyebrow ALL-CAPS, dilarang fade-slide-up massal,
   dilarang aksen acid-green tunggal sebagai hiasan.
3. **Kepadatan adalah fitur:** tabel 31px/baris, font data 12px mono — jangan
   "dibersihkan" jadi kartu lapang. Kartu tetap bernapas (padding 12–16px) tapi
   tabel dikompresi.
4. **Satu hue per makna:** sukses=hijau, pending=amber, gagal=merah, info=biru —
   berlaku di badge, donat, bar sinyal, log AT. Tidak pernah 4 shade satu hue.
5. **Copy operator, bukan copy marketing:** kalimat aktif, sentence case,
   Indonesia: "Jeda live", "Lihat antrean", "uji AT". Error memberi langkah,
   bukan permintaan maaf.
6. **Token dipakai apa adanya:** tidak ada hex campuran/turunan; `--faint`
   tidak pernah untuk teks; tidak ada `opacity < 1` pada teks di atas `--crisis`.
7. **Hierarki salience:** Alarm > angka KPI > log live > grafik. Grafik pendukung,
   bukan pameran.
8. **Keyboard-first:** command palette (Ctrl+K), `kbd` bergaya terminal
   (`surface2`, border bawah 2px), hint shortcut di statusbar.

---

### Perubahan dari brief → terkunci (jejak keputusan)

- Badge PRC: biru → amber (bentrok dengan info). — polish-notes §1
- Token samar `#4E5765` (2.64:1) → `--muted #b3aa9c` (8.06:1). — polish-notes §3.1
- Strip krisis teks putih penuh `#fff` (4.58:1), × opacity 1 — audit F1/F2.
- Mobile ≤900px: overflow 730px → 390px; grid track `minmax(0,1fr)` + `.cmdk{min-width:0}`. — audit F3
- `aria-label` pada `#palIn`; `role="img"`+`aria-label` pada bar sinyal; 10 `aria-label` rail. — audit F3.4/F4
- Font terkunci Inter (bukan Plus Jakarta Sans — DNA JSON shipmate perlu diperbarui).
- Flash baris terkunci `.9s` (mockup), bukan `1.6s` (DNA lama).
