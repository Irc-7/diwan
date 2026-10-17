# Polish Notes — Arah A "Terminal Operasi" (dir-a → dir-a-polished)

Basis: `../dir-a/index.html` (struktur & UX dipertahankan penuh). Token warna diganti
ke palet dark terukur ShipMate. Font tetap **Inter + JetBrains Mono**. Impeccable tidak dipakai.

## 1. Ganti token warna (Tugas 1)

| Token lama | Token baru | Dipakai untuk |
|---|---|---|
| `--alas #0B0E14` | `--bg #161311` | latar body, input cari, cmdk |
| `--panel #0E1219` | `--surface #201c18` | panel, topbar, rail, metrik, tabel |
| `--panel2 #141A24` | `--surface2 #262019` | hover, kbd, tombol jeda, toast |
| `--tinta #C9D0D9` | `--ink #f4efe8` (14.8:1 di surface) | teks utama/heading/angka besar |
| teks isi tabel `#C9D0D9` | `--body #d8d1c6` (12.2:1 di bg) | isi sel tabel, item palet |
| `--redup #7E8794` | `--muted #b3aa9c` (8.06:1 di bg) | teks sekunder: label metrik, th, dim, placeholder |
| `--samar #4E5765` | **dihapus → `--muted`** | semua kemunculan (lihat §3.1) |
| `--garis #1C2330` | `--line #332c24` | border panel, thead |
| `--garis2 #151B26` | `--line-soft #2b251e` | border antar-baris tabel/modem |
| scrollbar `#232B38` | `--line #332c24` | thumb scrollbar |
| `#2A3342` (bar sinyal kosong) | `--faint #7f776b` @35% opacity | bar sinyal belum terisi (elemen grafis, bukan teks) |
| tooltip rail `#1A212D` | `--chip #2a241e` | tooltip hover rail |
| `--sehat #4FA06E` | `--ok #54c47e` | dot live, sdot ok, bar sinyal penuh |
| `--sehat-t #79BE93` | `--ok-ink #8fdcab` (8.74:1 di ok_bg) | prompt `>_` brand, dot live, teks sukses |
| badge SUK `rgba(79,160,110,.13)` | `--ok-bg #183024` | latar badge SUK |
| `--waspada #D1A44B` | `--warn #eeb04a` | state jeda, sdot warn |
| `--waspada-t #DDBB7E` | `--warn-ink #f2cc82` (9.20:1 di warn_bg) | teks peringatan, tombol "Lanjutkan" |
| `--bahaya #C65A50` | `--err #ef6f6f` | badge GGL, sdot bad, M4 |
| `--bahaya-t #D08A82` | `--err-ink #f49c9c` (7.40:1 di err_bg) | teks error |
| badge GGL `rgba(198,90,80,.14)` | `--err-bg #3a1c1c` | latar badge GGL |
| `--biru #7E97B5` (tab aktif, kode, rx log) | `--info #8b96f0` | kode TX, `< +CSQ` rx, tab count aktif |
| tab underline `--biru` | `--orange #ff8f52` | indikator tab aktif + rail aktif (aksen ShipMate) |
| `--biru` (focus ring) | `--orange #ff8f52` (2px) | `:focus-visible` |
| sparkline `#4FA06E` | `--ok #54c47e` | sparkline metrik + laris |
| flash sukses `rgba(79,160,110,.22)` | `rgba(84,196,126,.20)` | animasi kilat baris SUK |
| flash gagal `rgba(198,90,80,.25)` | `rgba(239,111,111,.22)` | animasi kilat baris GGL |

**Perubahan semantik penting:** badge **PRC** sebelumnya memakai biru (`--biru`, mirip dengan
info) → sekarang **amber** (`warn_ink #f2cc82` di `warn_bg #38290f`, 9.20:1). Status kini
tiga warna yang jelas berbeda: SUK = hijau, PRC = amber, GGL = merah — sesuai mandat tugas.

**Token yang sengaja tidak dipakai di teks kecil:** `--faint #7f776b` (4.19:1) hanya untuk
bar sinyal kosong (elemen grafis + didampingi teks "CSQ n/31", bukan satu-satunya pembawa info).

## 2. UX Arah A dipertahankan (Tugas 2)

Tidak ada perubahan struktur: command palette (Ctrl+K), strip 5 KPI, tabel live 20 baris
@31px/baris, panel Terminal & modem + Log AT, hint keyboard di statusbar, dark-first.
Simulasi live 2-detik + tombol "Jeda live" tetap berfungsi (aria-pressed ikut status).

## 3. Lima temuan kritik (Tugas 3)

### 3.1 Teks samar `#4E5765` (2.64:1 — di bawah ambang baca)
Seluruh kemunculan `--samar` diganti `--muted #b3aa9c` (8.06:1 di `--bg`):
`brand small`, `clock span`, `.metric .sb`, `.metric .vl .unit` ("jt", "/4"),
`.laris .ct`, `.tabs .n`, `cari::placeholder`, timestamp Log AT. Token `--samar` dihapus.

### 3.2 Breakpoint mobile ≤900px + bottom nav 5 item
Pola ditiru dari `shipmate-improved`: `.rail` disembunyikan, grid body jadi satu kolom,
`.bottomnav` fixed di bawah (5 item: Dashboard, Transaksi, Terminal, Inbox, Pengaturan —
ikon memakai ulang SVG rail, item aktif oranye + `aria-current`). Penyesuaian lain:
- strip metrik jadi swipeable horizontal (`flex`, tiap kartu 196px) — cocok untuk konsol ops;
- `.col-side` menumpuk di bawah `.col-main`; `padding-bottom:96px` agar konten tidak tertutup nav;
- tabel `min-width:680px` dalam `.tbl-wrap` scroll horizontal (ux-guideline #71);
- `.shift` dan hint keyboard disembunyikan, statusbar membungkus, legend badge disembunyikan
  (tooltip `title` tetap tersedia);
- target sentuh nav ≥48px (ux-guideline #22/#66).

### 3.3 Badge SUK/PRC/GGL: legend + tooltip
- Setiap badge mendapat `title`: "SUK = Sukses", "PRC = Diproses", "GGL = Gagal".
- Legend kecil di bilah tab: `SUK = Sukses, PRC = Diproses, GGL = Gagal` (disembunyikan ≤1180px
  agar tidak berdesakan; tooltip tetap ada).

### 3.4 `aria-label` tombol ikon
Semua 10 tombol rail (sebelumnya hanya tooltip visual via `::after`) kini punya `aria-label`
yang bermakna. Tombol × banner krisis: `aria-label="Tutup peringatan krisis"`. SVG dekoratif
(rail, sparkline, ikon jeda) disembunyikan dari screen reader (`aria-hidden`). Bar sinyal
modem mendapat `role="img"` + `aria-label` ("Sinyal M1: CSQ 24/31") agar tidak hanya
mengandalkan warna (ux-guideline #37).

### 3.5 `prefers-reduced-motion`
Blok lama hanya mematikan 2 animasi. Sekarang: semua `animation`/`transition` dipaksa
`.001s` + `iteration-count:1` (pola shipmate-improved), kilat baris (flash-suk/flash-ggl),
denyut badge PRC, dan transisi tooltip dimatikan eksplisit. State akhir tetap benar
tanpa bergantung pada `animationend` (ux-guideline #119): class kilat dihapus via timeout
yang tetap jalan.

## 4. Strip krisis ala Arah B (Tugas 4)

Satu-satunya elemen bold baru: banner solid `crisis #cf4636` di atas tabel —
"M4 (Tri) tidak ada sinyal — 12 transaksi antre tertunda, dialihkan otomatis ke M1/M2"
+ tombol aksi **"Lihat antrean"** (membuka tab "Perlu tindakan") + tombol × dismiss
(`role="alert"`). Identitas A tidak diubah; tidak ada elemen bold lain yang ditambahkan.

**Keputusan kontras:** teks `#f4efe8` di atas `#cf4636` hanya 4.01:1 (< 4.5).
Dipakai putih murni `#ffffff` → **4.58:1**, lolos AA tanpa mengubah hex token krisis.
Tombol aksi memakai `rgba(0,0,0,.30)` di atas crisis → kontras putih ≈5.9:1.

## 5. Catatan restraint (frontend-design)

- Satu elemen berani = strip krisis; sisanya disiplin (tidak ada fade-slide-up,
  tidak ada SaaS-card kit, tidak ada eyebrow ALL-CAPS baru).
- Legend memakai koma, bukan pola "A · B · C".
- Copy tetap dari sudut pandang operator: "Lihat antrean", "Jeda live", "uji AT".
- Token dipakai apa adanya; tidak ada hex campuran/turunan.

## 7. Ronde perbaikan temuan review (2026-10-17, F1–F4)

Tidak ada perubahan selain 4 temuan; tidak ada redesign, tidak ada token baru.

- **F1** — `.cr-txt span`: `opacity:.92` dihapus → putih penuh `#fff` di atas crisis
  `#cf4636` = 4.58:1 (lolos AA). Terverifikasi via `getComputedStyle` → opacity `1`.
- **F2** — `.cr-x`: `opacity:.85` → `opacity:1` permanen (4.58:1); efek hover
  disederhanakan ke `background` saja (aturan `:hover` yang redundan `opacity:1` dibuang).
  Terverifikasi via `getComputedStyle` → opacity `1`.
- **F3** — overflow horizontal 390px (scrollWidth 730 → **390**):
  - `@media (max-width:900px)`: `.tabs{flex-wrap:wrap}`,
    `.tab-tools{margin-left:0;flex-basis:100%}` (baris sendiri full-width),
    `.cari{flex:1}`; `.crisis{flex-wrap:wrap;row-gap:8px}` (tombol "Lihat antrean" + ×
    turun ke baris kedua).
  - Temuan saat verifikasi: saran reviewer saja tidak cukup — pemicu sebenarnya adalah
    `.cmdk` (min-content ~492px dari teks hint `white-space:nowrap`) yang memaksa track
    grid body `1fr` (= `minmax(auto,1fr)`) melebar ke 728px. Perbaikan: track jadi
    `minmax(0,1fr)` + `.cmdk{min-width:0}` (teks `.ph` sudah `ellipsis`, tombol menyusut
    ke ~154px di 390px tanpa terpotong visual).
  - Tombol "Jeda live" terukur di x=261, lebar 106 → tepi kanan 367 < 390: selalu dalam
    viewport tanpa scroll horizontal.
- **F4** — `#palIn` mendapat `aria-label="Ketik perintah"` (sebelumnya hanya placeholder).

**Hasil verifikasi** (headless Chromium, `--dump-dom` + JS ukur, `virtual-time-budget`):
- 390×844: `document.documentElement.scrollWidth` = 390 (≤ 390 ✓); "Jeda live" dalam
  viewport ✓; tidak ada JS error di console ✓.
- 1440×900: scrollWidth = 1440 ✓; layout desktop tidak berubah (aturan wrap hanya di
  ≤900px; `.cmdk` kembali 492px) ✓; tidak ada JS error ✓.
- Screenshot: `/tmp/shot-390.png`, `/tmp/shot-1440.png` (sementara, tidak diarsipkan).
