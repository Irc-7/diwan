# Sketsa Arsitektur UI Web — Clone Linux OtomaX 4.3.1

**Tanggal:** 2026-10-17 | **Peran:** arsitek frontend, migrasi WinForms → web
**Metode:** forge, evidence-based. Sumber bukti: `SCHEMA_DEEPDIVE.md` (18 objek DB + definisi grid `admindg*`), revision log `Version.txt` 4.3.1 (nama halaman/modul/menu dari 82 versi changelog), enum `ItemStatus` (49 status) & `TipeModul` (15 tipe) dari inner assembly, `LAPORAN_GAP_RISET.md` (Opsi B), `EDISI_DAN_BATASAN.md` (keputusan: tanpa batasan edisi, setara Ultimate, tanpa lisensi).

**Keputusan desain yang mengikat:** semua batasan edisi hilang, semua fitur terbuka (setara Ultimate). Tidak ada sistem lisensi/kunci edisi di clone.

---

## 1. Inventaris layar (dari bukti)

### Area A — Dashboard & Monitoring
| Layar | Fungsi inti | Data |
|---|---|---|
| Dashboard Operasional | Ringkasan live: transaksi hari ini per status, saldo reseller total, status terminal/modul | Agregat `transaksi` (status, tgl_entri), `reseller.saldo`, status terminal |
| Monitor Transaksi (live) | Tabel transaksi berjalan, auto-refresh; aksi: proses ulang, ubah status, input SN/Ref | `transaksi` 27 kolom; filter status (49 nilai `ItemStatus`), kode_produk, tujuan |
| Monitor Modul & Terminal | Status tiap terminal/modem + antrian modul; aksi: putuskan semua terminal, kill proses, duplikat modul | `transaksi.kode_modul/kode_terminal/ket_modul`; menu "Monitor Modul", "Putuskan Semua Terminal" |

### Area B — Transaksi
| Layar | Fungsi inti | Data |
|---|---|---|
| Daftar Transaksi | Riwayat + pencarian (rentang kode, tanggal, tujuan, reseller); pilih kolom tampil | `transaksi` full 27 kolom; view `cek_stok` sebagai pola query operasional |
| Detail Transaksi | Timeline status, SMS asal/balasan, SN, modul pemroses, kirim ulang | `transaksi` + `inbox` (kode_inbox, kode_jawaban) |
| Transaksi Manual / Terjadwal | Input manual + penjadwalan (`Terjadwal`) | Form → `transaksi` (perintah, prioritas, qty) |

### Area C — Master Data
| Layar | Fungsi inti | Data (dari definisi grid di backup) |
|---|---|---|
| Produk | CRUD produk + kolom Ultimate: Rumus Harga, Filter Tujuan (RegEx), Qty yang diizinkan | `admindgProduk`: aktif, kode, nama, kode_operator, fisik, tanpa_kode, nominal, harga_beli, harga_jual, kosong, gangguan, stok, qty, prefix_tujuan (+2 hidden) |
| Provider / HLR / Operator | Mapping prefix → operator; area & HLR | `admindgProvider`: kode, nama, kosong, gangguan, prioritas, prefix_tujuan, kode_area, kode_hlr, panjang_min/maks, kode_level, struk_auto_cetak; `admindgHLR`: kode, prefix_tujuan |
| Parsing (Routing) | Aturan routing modul→produk berdasar prioritas | `admindgParsing`: aktif, kode_modul, kode_produk, prioritas, harga_beli, _harga_jual, perintah, keterangan |
| Cek Harga (+ Reseller, + Grup) | Simulasi/cek harga per produk/level | `produk.harga_jual`, `level.selisih_harga/default_markup`, view `harga_agen` |
| Level & Grup | Hierarki MLM, markup, blokir produk, deposit min/maks | `admindgLevel`: kode, nama, kode_upline, selisih_harga, default_markup, no_ubah_markup, deposit_minimal/maksimal, jumlah_ym, jumlah_sms, blok_produk, keterangan |

### Area D — Reseller & Keanggotaan
| Layar | Fungsi inti | Data |
|---|---|---|
| Reseller | CRUD, saldo, aktivitas, downline (pendaftaran/aktivasi/transfer/reset PIN) | `reseller`: kode, saldo, tgl_aktivitas (+ kolom lain); pola ID `OR####` |
| Pendaftaran / Registrasi Member | Form pendaftaran manual + approval | Form pendaftaran; `oto_user` (nomor PK, imsi, serial_number) |
| Pengirim (Sender) | Daftar nomor pengirim + cek duplikat WA/SMS + blacklist | Nomor pengirim; kolom Fraud di Daftar Hitam → auto-nonaktif reseller |
| Kiosk | Kasir cetak struk (cari transaksi by tujuan, sembunyikan baris) | `transaksi` + template struk (`NotaBill.prt` sebagai contoh) |

### Area E — Modul & Terminal
| Layar | Fungsi inti | Data |
|---|---|---|
| Daftar Modul | 15 tipe (`TipeModul`): SMS Center, SMS Sender, Dial/USSD, STK, SMS Order/H2H, Keypad, IM Center, IM Hybrid HTH, IP Client/Center, Internet Banking, WA Center, Voucher Fisik, Web SMS Sender, API SiDompul, API Axis | Konfigurasi per tipe modul |
| Setting Modul & Terminal | Tambah/edit terminal (port modem), parameter parsing (s.d. 8000 karakter), batas antrian | Setting terminal; "Tambah Modul" |
| Administrasi Jawaban | Format & balasan transaksi (template `[trxcount]`, `[pin]`, dsb.) | Setting rows: `Balasan0`, `Format0`, `PolaParam0/20` |

### Area F — Inbox / Outbox / Pesan
| Layar | Fungsi inti | Data |
|---|---|---|
| Inbox | SMS masuk + cleanup (hapus inbox reseller tak dikenal) | `inbox`: kode, pesan, kode_reseller, pengirim, tipe_pengirim |
| Outbox | Antrian SMS keluar + kirim ulang | `outbox` (diimplikasikan via `mutasi.kode_outbox`); status outbox |
| Auto Respon | Aturan balasan otomatis | Tabel `auto_respon` (disebut di fix 4.2.8) |

### Area G — Keuangan
| Layar | Fungsi inti | Data |
|---|---|---|
| Mutasi Saldo | Riwayat mutasi + filter dompet | `mutasi`: kode, kode_reseller, tanggal, jumlah, keterangan, kode_reseller_2, jenis, kode_transaksi, kode_outbox, saldo_akhir |
| Mutasi Komisi / Tukar Komisi | Komisi tunda (MLM) → tukar jadi deposit (`TUKAR.[jumlah].[pin]`) | `komisi`: kode_transaksi, kode_reseller, jumlah; SP `AddKomisiTunda`; Setting `SettingTundaKomisi`, `KomisiFull` |
| Mutasi Poin / Unit | Poin & unit reseller (dialog Tambah Poin/Unit) | Halaman Mutasi Poin; menu Reset Poin, Tambah Unit |
| Tiket Deposit | Klaim tiket deposit + tambah saldo manual | `tiket_deposit`: kode, waktu, jumlah, status ('O'=Open), tgl_status, kode_inbox, kode_data_bank |
| Data Bank | Mutasi bank + klaim otomatis (Internet Banking) | `data_bank`: kode, bank, tgl_proses, tgl_bank, jumlah, tipe, saldo, keterangan, kode_tiket, terklaim, catatan |
| Dompet / Dompet Reseller | Administrasi dompet (refund, dsb.) | Menu Administrasi - Dompet |
| Voucher Fisik | Stok & status voucher (USED pada transaksi gagal = bug yang di-fix 4.3.1) | Produk `fisik=1`; halaman Voucher Fisik |

### Area H — Laporan
| Layar | Fungsi inti | Data |
|---|---|---|
| Rekap Komisi Reseller | Laporan komisi termasuk data tukar komisi | Agregat `komisi` + `mutasi` |
| Saldo Awal Reseller | Laporan saldo awal periode | Snapshot `reseller.saldo` |
| Laporan Transaksi / Umum | Rekap per produk/periode (modul Laporan ikut di paket Free) | Agregat `transaksi` |

### Area I — Pengaturan & Sistem
| Layar | Fungsi inti | Data |
|---|---|---|
| Setting Umum & Lain-Lain | Night mode, auto-delete log/tiket, optimasi (matikan log modul, matikan cek blacklist) | Setting rows: `Setting0`, `SettingMLM`, `Seting Lain-Lain` |
| Hak Akses Pemakai | RBAC per menu (fix 4.2.8: Administrasi - Dompet) | Tabel user/role (diimplikasikan) |
| Alat Bantu | Monitor Saldo Reseller, Tukar Komisi, Tambah Unit, Reset Poin, Kill All Chromes | Aksi operasional |
| Log Sistem | Log modul (bisa diheningkan: no popup/beep) | `oto_histori`, `oto_histori_update` |

**Total: 66 form WinForms → ±22 layar web** (konsolidasi: tiap grup CRUD satu-tabel menjadi satu layar generik; dialog kecil menjadi modal/drawer; menu konteks menjadi aksi baris).

---

## 2. Rekomendasi stack web UI

**Pilihan utama: React + TypeScript SPA + AG Grid Community, di belakang ASP.NET Core (.NET 8) REST API + SignalR hub.**

Alasan (terikat bukti, bukan preferensi):
1. **Real-time monitoring adalah nyawa aplikasi** — Monitor Transaksi dan Monitor Modul butuh live update (di desktop: popup log + beep). SignalR (.NET 8 native) memberi channel real-time tanpa polling; React berlangganan hub untuk patch inkremental ke grid.
2. **Ekspektasi "gaya Excel"** — revision log & materi marketing menekankan "manajemen data gaya Excel: copy-paste, sortir". AG Grid memberi perilaku itu (sorting, filtering, column chooser — "menu pilih kolom di Transaksi" ada di changelog) tanpa membangun dari nol.
3. **Decoupling dari engine** — Opsi B memisahkan engine service dan UI; SPA murni membuat UI bisa di-host di mana saja (termasuk HP operator), sementara engine tetap systemd service.
4. **66 form → 22 layar** dimungkinkan karena pola UI-nya repetitif (grid + form CRUD); komponen generik `DataGridPage` + `EntityForm` menutup ~70% layar master.

**Alternatif (bila tim murni C#):** Blazor Server — real-time via circuit SignalR built-in, satu bahasa. Konsekuensi: grid Excel-like lebih lemah dari AG Grid, dan scaling per-circuit lebih berat untuk banyak operator concurrent. Tetap valid untuk MVP internal.

**Keputusan yang TIDAK direkomendasikan:** port 1:1 tiap WinForms (66 layar) — membuang keuntungan web (navigasi, deep-link, responsif) dan melipatgandakan effort tanpa nilai bisnis.

---

## 3. API contract garis besar (UI ↔ engine service)

**REST** (`/api/v1`) — resource-oriented, JSON:
- `GET/POST /transaksi?status=&dari=&sampai=&q=` · `GET /transaksi/{kode}` · `POST /transaksi/{kode}/proses-ulang` · `POST /transaksi/{kode}/status`
- `GET/PUT /produk`, `/produk/{kode}` · `GET /produk/cek-harga?kode=&level=`
- CRUD generik: `/provider`, `/hlr`, `/level`, `/parsing`, `/reseller`, `/pengirim`
- `GET/POST /inbox`, `/outbox`, `POST /outbox/{kode}/kirim-ulang`
- `GET /mutasi`, `/mutasi-komisi`, `POST /komisi/tukar` · `GET /mutasi-poin`, `POST /poin/tambah`, `/unit/tambah`
- `GET/POST /tiket-deposit`, `POST /tiket-deposit/{kode}/klaim` · `GET /data-bank`
- `GET /voucher-fisik` · `GET /dompet`
- `GET /modul`, `PUT /modul/{kode}` · `GET /terminal`, `POST /terminal`, `POST /terminal/putuskan-semua`
- `GET /laporan/komisi`, `/laporan/saldo-awal`, `/laporan/transaksi`
- `GET/PUT /setting/{grup}` · `GET/PUT /jawaban` (format & balasan)
- `GET /dashboard/ringkasan` (agregat untuk dashboard)
- Auth: `POST /auth/login` → JWT; RBAC per menu (bukti: "hak akses pemakai" di changelog).

**Real-time — SignalR hub `/hub/monitor`:**
- Server→client events: `transaksi.baru`, `transaksi.status` (kode, status lama→baru, 49 nilai `ItemStatus`), `inbox.baru`, `modul.status` (kode_modul, online/antrian), `saldo.berubah` (kode_reseller, saldo_akhir), `tiket-deposit.baru`.
- Client→server: `subscribe(filter)` — mis. hanya status pending/diproses untuk Monitor Transaksi.
- Kontrak payload memakai nama kolom DB apa adanya (`kode`, `tgl_entri`, `kode_produk`, …) agar mapping 1:1 dengan skema hasil RE.

---

## 4. Prioritas pembangunan (MVP)

**Fase 1 — Inti operasional (server bisa jalan):**
1. Dashboard + Monitor Transaksi live (jantung operasional; 49 status → tampilkan sebagai badge + filter).
2. Daftar & Detail Transaksi (aksi: proses ulang, ubah status, input SN/Ref).
3. Master Produk + Parsing (tanpa routing produk→modul, tidak ada transaksi).
4. Reseller + Mutasi + Tiket Deposit + Data Bank (aliran uang harus benar sejak hari pertama).
5. Monitor Modul & Terminal + Setting terminal (operator harus bisa melihat/menambah modem).

**Fase 2 — Kelengkapan operasional:**
6. Inbox/Outbox + Auto Respon; 7. Komisi/Mutasi Komisi/Tukar Komisi + Mutasi Poin/Unit; 8. Provider/HLR/Level/Cek Harga; 9. Voucher Fisik + Kiosk; 10. Laporan (komisi, saldo awal).

**Fase 3 — Administrasi:**
11. Pengaturan (umum/lain-lain/jawaban), Hak Akses Pemakai, Alat Bantu, Log Sistem, Administrasi Dompet.

Rasional: urutan mengikuti aliran nilai — transaksi hidup → produk bisa di-routing → uang tercatat benar → sisanya administratif.

---

## 5. Risiko & mitigasi

| # | Risiko | Bukti | Mitigasi |
|---|---|---|---|
| 1 | State machine 49 status salah dimodelkan → transaksi nyangkut | Enum `ItemStatus` lengkap dari binary | Modelkan sebagai state machine eksplisit di engine + uji transisi per status; UI hanya menampilkan, tidak menebak alur |
| 2 | Real-time tidak scalable saat ratusan transaksi/menit | Monitor live adalah fitur inti | SignalR dengan filter subscribe per layar; Redis backplane bila multi-instance; mulai single-node |
| 3 | Ekspektasi grid "seperti Excel" tidak terpenuhi | Changelog & marketing menekankan copy-paste/sortir | AG Grid (bukan tabel HTML buatan sendiri); uji dengan operator memakai data sample |
| 4 | Skema DB hasil RE tidak lengkap (tak ada CREATE TABLE; tipe data unknown) | `SCHEMA_DEEPDIVE.md` §5 | Fase 1: buat DDL dari inferensi + validasi terhadap sample data; siapkan migrasi skema sejak awal |
| 5 | Hak akses: UI web = multi-user, desktop = single-user | "Hak akses pemakai" ada di changelog | RBAC sejak MVP (menu-level), JWT; jangan tunda ke Fase 3 |
| 6 | Perilaku modem di Linux berbeda dari Windows (timing AT-command) | Targeted decryption: 18 token modem terpetakan | Layar Monitor Modul menampilkan raw AT log (membantu debug); engine pakai timeout/antrian yang dapat di-tuning dari UI |
| 7 | Scope creep: meniru 66 form 1:1 | 66 WinForms di binary | Patokan: 22 layar; tiap permintaan layar baru harus memetakan ke salah satu area §1 atau ditolak/ditunda |
| 8 | Tanpa lisensi = tanpa pembeda edisi; risiko "fitur Ultimate" dianggap beban | Keputusan desain 2026-10-17 | Semua fitur Ultimate (Rumus Harga, Filter Tujuan RegEx, mutasi komisi, Web SMS Sender) masuk sejak Fase 1/2 sebagai bagian normal — bukan add-on |

---

## 6. Catatan arsitektur silang (untuk engine service)

- UI tidak pernah menyentuh DB langsung; semua lewat REST engine (memudahkan ganti SQL Server → PostgreSQL di fase 2 tanpa menyentuh UI).
- Kontrak kolom memakai nama kolom DB hasil RE (`kode`, `tgl_entri`, …) — bila engine menormalisasi nama, sediakan alias, jangan ubah kontrak.
- Night mode ("Menu File - Night Mode" di changelog) → dukung tema gelap sejak awal (murah di web, mahal bila disusulkan).
