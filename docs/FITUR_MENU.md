# FITUR_MENU.md — Penyidikan Fitur Menu OtomaX vs Rancangan Linux

**Tanggal:** 2026-10-17 | **Metode:** 4 agen spesialis paralel, bukti RE wajib (BEDAH_LOGIKA_BISNIS.md, SCHEMA_DEEPDIVE.md, FINDINGS.md, CLI_METADATA_INVENTORY.md, EDISI_DAN_BATASAN.md, TARGETED_DECRYPTION.md, ARSITEKTUR_LINUX.md, arch_engine/db/ui.md + revision log `Version.txt`/`Version_en.txt` + `NotaBill.prt`).
**Konvensi:** [B] terbukti dari bukti · [I] inferensi · [R] rekonstruksi. Status: SUDAH TERCAKUP / SEBAGIAN / BELUM ADA (dibanding kolom "Fungsi inti" arch_ui.md). Prioritas: P0 wajib MVP · P1 fase 2 · P2 opsional.

## Ringkasan eksekutif

| Metrik | Jumlah | % |
|---|---|---|
| Total fitur teridentifikasi | 225 | 100% |
| SUDAH TERCAKUP di rancangan Linux | 40 | 18% |
| SEBAGIAN tercakup | 86 | 38% |
| BELUM ADA di rancangan | 99 | 44% |
| Keputusan eksplisit TIDAK diporting | 2 | — |

**Per area:**

| Area | SUDAH | SEBAGIAN | BELUM |
|---|---|---|---|
| A — Dashboard & Monitoring | 8 | 8 | 11 |
| B — Transaksi | 6 | 8 | 5 |
| C — Master Data | 8 | 2 | 21 |
| D — Reseller & Keanggotaan | 2 | 9 | 16 |
| E — Modul & Terminal | 5 | 14 | 13 |
| F — Inbox / Outbox / Pesan | 5 | 4 | 14 |
| G — Keuangan | 4 | 25 | 13 |
| H — Laporan | 1 | 4 | 1 |
| I — Pengaturan & Sistem | 1 | 12 | 5 |

**Kesimpulan pola:** rancangan Linux menangkap *struktur* (layar, kontrak API, DDL inti) tapi kehilangan *perilaku operasional halus* yang justru membedakan OtomaX: parameter per-tipe-modul, katalog placeholder jawaban, aturan Suspect/Terjadwal, proteksi fraud, locking saldo, dan puluhan setting key-value. Gap terbesar: Area C (master data lanjutan), Area E (parameter modul), Area G (skema keuangan lanjutan).

### Top 10 gap P0 (wajib MVP)

1. **Wizard "Tambah Modul" dengan field per tipe + kolom JSON `parameter`** (E) — tanpa ini daftar 15 TipeModul tidak bisa dikonfigurasi; skema `modul` saat ini hanya `kode,nama,tipe`.
2. **Katalog placeholder jawaban resmi + editor PolaParam regex + tombol "uji"** (E) — balasan adalah nyawa server pulsa; daftar placeholder (`[trxcount]`, `[pin]`, `[saldo]` vs `[saldoawal]`, …) belum terpetakan penuh.
3. **Rumus Harga + Filter Tujuan (RegEx) + Qty Yang Diizinkan** (C) — logika harga & validasi inti transaksi; fitur Ultimate-only yang wajib dibuka; mekanika evaluasi belum di-spec.
4. **Kolom `tgl_jadwal` + aturan eksekusi status `Terjadwal`** (B) — tanpa kolom skema, status Terjadwal tidak bisa diimplementasikan; plus aturan "kecualikan dari hitungan dalam-proses".
5. **Aksi operasional harian**: kirim-ulang per-modul pending + bulk, proses-ulang (modul sama) dengan reset `perintah`, ubah-status multi-select + edit keterangan saat Gagal + audit log (A).
6. **Keamanan uang**: locking saldo (`SELECT … FOR UPDATE`), refund idempoten, idempotensi deposit bank (unique constraint), generator angka unik thread-safe, engine menulis `saldo_awal` (G).
7. **RBAC sejak MVP**: tabel `pengguna`/`peran`/`peran_menu`, proteksi API + sembunyikan menu di UI (I) — arsitektur web multi-user, tidak bisa disusulkan.
8. **Proteksi fraud**: Daftar Hitam + kolom Fraud → auto-nonaktif reseller, validasi terpusat cek blacklist + duplikat nomor WA/SMS (normalisasi 62/0/+62) di 4 titik (D).
9. **Fondasi keuangan**: enum `jenis_mutasi` eksplisit, spec pemicu komisi (`SettingTundaKomisi`, `KomisiFull`), pipeline auto-claim mutasi bank berurutan (G).
10. **Keputusan multi-dompet sebelum DDL final** (G) — single vs multi dompet mengubah skema `mutasi`/`transaksi`; salah pilih = migrasi mahal.

---

## Area A — Dashboard & Monitoring

### Layar: Dashboard Operasional
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Monitor Modul (Alat Bantu): pantau saldo tiap modul + toggle "Active modules only" | SEBAGIAN | Version.txt:1085, :458, :278 | [B] | Tambahkan kolom saldo modul + warning threshold (setting "Menangkap Saldo… warning dan forward CS jika saldo mencapai angka tertentu"). **P0** — saldo modul = nyawa tembak transaksi |
| Monitor Saldo Reseller (Alat Bantu): daftar saldo reseller + checkbox Auto Update | SEBAGIAN | Version.txt:80, :63, :54 (tabel `monitor_saldo`, kolom `saldo_reseller`) | [B] | Aksi UI: auto-refresh toggle + daftar saldo real-time (`reseller.saldo`, `tgl_aktivitas`). **P0** |
| CS Monitor: jendela monitor pesan CS + kolom "jawaban" + limit data | BELUM ADA | Version.txt:1190, :691, :881 | [B] | Layar/modal "Monitor CS": kolom jawaban per baris + limit baris + filter non-member. **P1** |
| Log Sistem dengan fitur Silent (popup/beep dimatikan) | SEBAGIAN | Version.txt:869 | [B] | Setting Silent sejak awal di Log Sistem; dashboard menghormati mode silent. **P1** |
| Dashboard ringkasan terpadu (transaksi hari ini per status + saldo reseller total) | BELUM ADA | Tidak ada di bukti — OtomaX asli pakai form utama + monitor terpisah; `[trxcount]` ada di balasan (Version.txt:897) | [I] | Fitur BARU khas web: `GET /dashboard/ringkasan` (count per status hari ini, total saldo, terminal up/down). **P1** |

### Layar: Monitor Transaksi (live)
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Auto-refresh data transaksi | SUDAH TERCAKUP | Version.txt:143, :157 | [B] | Pastikan bisa dimatikan (beban DB). **P0** |
| Menu "Proses Ulang (modul sama)" per baris | SUDAH TERCAKUP | Version.txt:810, :910, :932 | [B] | Perilaku asli: reset `perintah`, increment `kirim_ulang`/`counter`, aturan cache REFID (status Kosong/Gangguan/ProdukSalah tetap sama bila produk dibuka — :811). **P0** |
| Kirim ulang (per modul pending) + auto kirim ulang per modul (tiap 30 dtk, rentang 10 mnt dari tgl_entri) | SEBAGIAN | Version.txt:361, :486, :816, :1087, :1146, :895 | [B] | Aksi "Kirim ulang" terpisah (re-send tanpa revalidasi penuh) + tombol bulk per modul + indikator auto aktif. Catatan: auto hanya untuk modul IM/IP (:816). **P0** |
| Ubah status manual (multi-select; ubah ke gagal → edit keterangan) | SUDAH TERCAKUP | Version.txt:203, :13, :949, :950 | [B] | Tambahkan bulk multi-select + dialog keterangan + log audit. **P0** |
| Dialog Input SN/Ref: kotak Harga Beli + Qty, bisa ubah harga postpaid | SUDAH TERCAKUP | Version.txt:360, :586, :972, :144 | [B] | Perilaku asli: ubah harga jual TIDAK otomatis memotong saldo (postpaid) — replikasi eksplisit. **P0** |
| "Batalkan Semua Transaksi Yang Belum Diproses" (bulk) | BELUM ADA | Version.txt:1040 | [B] | Aksi bulk + konfirmasi + daftar status yang dicakup (pernah bug untuk `TidakAdaParsing`). **P1** |
| Status Suspect: mirip MenungguJawaban tapi disembunyikan dari daftar pending | SEBAGIAN | Version.txt:62 | [B] | Default view "pending" mengecualikan `Suspect` + toggle "tampilkan suspect". **P0** |
| Aturan restart: "sedang diproses" saat program ditutup → `MenungguJawaban` saat loading | SEBAGIAN | Version.txt:231 | [B] | Aturan engine; tampilkan badge "dipulihkan dari restart" bila perlu. **P0** |
| Notifikasi SN ganda pada transaksi sukses | BELUM ADA | Version.txt:917 | [B] | Indikator/warning bila `sn` duplikat pada transaksi sukses. **P1** |
| Auto-update SN: sukses tanpa SN otomatis diupdate saat tertangkap | BELUM ADA | Version.txt:918 | [B] | Aturan engine; monitor live-update kolom SN. **P0** |
| Menangkap >1 reply jawaban sukses (mis. MKios Dial/SMS) | BELUM ADA | Version.txt:688 | [B] | Detail tampilkan daftar jawaban 1:N (dampak skema: `kode_jawaban` tunggal tidak cukup). **P1** |
| Dobel-klik sel "Jawaban Provider" → copy jawaban lengkap | BELUM ADA | Version.txt:248 | [B] | Copy-to-clipboard di kolom jawaban (AG Grid mendukung). **P2** |
| Kolom "Keterangan (status)" di tabel Transaksi | SUDAH TERCAKUP | Version.txt:687 | [B] | Pastikan tampil default di monitor. **P0** |
| Export data transaksi | BELUM ADA | Version.txt:169, :184, :237 | [B] | Tombol Export (Excel/CSV) dengan pilihan data terfilter. **P1** |
| Ringkasan Outbox pada halaman Transaksi + statistik outbox (checkbox, bisa dimatikan) | BELUM ADA | Version.txt:826, :641 | [B] | Panel ringkasan outbox per modul + toggle on/off (alasan performa). **P1** |

### Layar: Monitor Modul & Terminal
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Menu "Putuskan Semua Terminal" | SUDAH TERCAKUP | Version.txt:330; arch_ui.md §1 | [B] | Tambahkan konfirmasi + status per terminal real-time via SignalR. **P0** |
| Menu "Duplikat modul" (klik kanan) | SUDAH TERCAKUP | Version.txt:212; arch_ui.md §1 | [B] | Clone konfigurasi ke kode baru. **P1** |
| "Kill All Chromes" + setting KillAllChromesOnExit | SEBAGIAN | Version.txt:459, :358 | [B] | Tombol "Kill proses Chrome/Selenium" (worker Internet Banking & WA Web macet) + setting kill-on-exit. **P0** |
| Monitor Modul: daftar modul + saldo + hanya modul aktif | SUDAH TERCAKUP | Version.txt:1085, :458, :1076 | [B] | Kolom: saldo modul, threshold warning, jam aktif, limit counter. **P0** |
| Jam aktif + limit counter per modul | BELUM ADA | Version.txt:871 | [B] | Indikator modul di luar jam aktif; counter vs limit. **P1** |
| Kotak "Modul Terkait" pada setting terminal (balasan SMS diproses modul lain) | BELUM ADA | Version.txt:504 | [B] | Field `modul_terkait[]` di Setting Terminal → routing ReplyDispatch. **P1** |
| Batas 255 terminal; label terminal s.d. 50 karakter | SEBAGIAN | Version.txt:916, :989 | [B]/[I] | Validasi jumlah terminal di form (batas teknis asli; uji apakah 255 batas array). **P2** |

## Area B — Transaksi

### Layar: Daftar Transaksi
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Menu "Pilih kolom" + ingat layout (urutan & lebar kolom) | SUDAH TERCAKUP | Version.txt:182, :485 | [B] | Persistensi layout per user (localStorage/backend). **P0** |
| Filter Serial Number + reset mengosongkan kotak SN | SEBAGIAN | Version.txt:198, :186, :187 | [B] | Filter SN eksplisit + perilaku reset (semua kotak ikut kosong). **P0** |
| Pencarian kode produk `'S10'` (exact) vs `S10` (prefix) | BELUM ADA | Version.txt:335, :336 | [B] | Sintaks petik di kotak produk + tooltip dokumentasi. **P1** |
| Rentang kode transaksi (dari–sampai) | SEBAGIAN | Pola `cek_stok`: `t.kode BETWEEN 445863 AND 448361` (SCHEMA_DEEPDIVE.md §1) | [I] | Filter `kode_dari`–`kode_sampai`. **P0** |
| Filter tanggal (dari–sampai) | SEBAGIAN | Filter tanggal eksplisit hanya di Log Sistem (Version.txt:200); kolom `tgl_entri` terindeks | [I] | Pertahankan; **P0** |
| Filter tujuan & reseller | SEBAGIAN | Kolom `tujuan`, `kode_reseller` di skema (SCHEMA_DEEPDIVE.md §1); bukti UI eksplisit tidak ditemukan | [I] | Filter `q` generik + kolom spesifik. **P0** |
| Halaman "Transaksi - Mutasi Komisi" (pintasan ke mutasi komisi terkait) | BELUM ADA | Version.txt:173 | [B] | Aksi "Lihat mutasi komisi" → filter mutasi by `kode_transaksi`. **P1** |
| Data voucher tampil di halaman transaksi | BELUM ADA | Version.txt:316 | [B] | Kolom info voucher (produk `fisik=1`). **P2** |

### Layar: Detail Transaksi
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| SMS asal: `transaksi.kode_inbox` → isi SMS perintah asli | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §2 (relasi inbox 1:N transaksi) | [B] | Panel "SMS asal" (isi, pengirim, waktu). **P0** |
| SMS balasan: `transaksi.kode_jawaban` → `inbox.kode` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §2; pola view `cek_stok` | [B] | Panel "Jawaban/balasan" + dukung >1 jawaban. **P0** |
| Timeline/riwayat perubahan status | BELUM ADA | Tidak ada bukti riwayat status (hanya `status` + `tgl_status`; "Lihat Riwayat Data" hanya di mutasi bank — Version.txt:75/77) | [I] | Fitur baru khas web: tabel audit `transaksi_histori` ditulis engine tiap transisi. **P1** |
| SN + modul pemroses (`sn`, `kode_modul`, `kode_terminal`, `ket_modul`, `modul_proses`) | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1 (27 kolom) | [B] | Blok "Pemrosesan". **P0** |
| Kirim ulang dari detail | SUDAH TERCAKUP | Version.txt:361 dkk. | [B] | Tersedia di drawer detail. **P0** |
| Ubah status + edit keterangan dari detail | SUDAH TERCAKUP | Version.txt:13, :203 | [B] | **P0** |
| Harga beli / harga / saldo_awal / qty / counter / kirim_ulang / perintah / prioritas | SEBAGIAN | SCHEMA_DEEPDIVE.md §1 | [B] | Section "Keuangan & teknis" di detail dengan daftar field eksplisit. **P0** |

### Layar: Transaksi Manual / Terjadwal
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Form manual: histori data, proteksi klik-dobel Simpan, bawa kode area reseller | SEBAGIAN | Version.txt:1110, :1209, :1109 | [B] | Panel histori (paginasi), proteksi double-submit (disable + idempotency key), auto-isi kode area. Field: kode_produk, tujuan, kode_reseller, perintah, prioritas, qty. **P0** |
| Status `Terjadwal` (tidak langsung diproses) | SEBAGIAN | BEDAH_LOGIKA_BISNIS.md §2; Version.txt:1018 | [B] | Simpan dengan `status=Terjadwal` + kolom jadwal; SchedulerService (Quartz) eksekusi. **P0** |
| Field jadwal (tanggal/jam eksekusi, pengulangan) | BELUM ADA | Tidak ada kolom jadwal di 27 kolom `transaksi` | [R] | Kolom baru `tgl_jadwal` (+ opsional `jadwal_rrule`); date-time picker + opsi ulangi. **P0** skema; UI picker P1 |
| "Terjadwal" dikecualikan dari hitungan "masih dalam proses" | SEBAGIAN | Version.txt:1018 | [B] | Filter khusus "Terjadwal" + kolom waktu eksekusi + aksi batalkan/jalankan-sekarang. **P0** |

### Catatan khusus: 49 ItemStatus yang butuh aksi/aturan UI
Sumber: BEDAH_LOGIKA_BISNIS.md §2, SCHEMA_DEEPDIVE.md §3 (`status<20`=pending, `status>0`=terproses), Version.txt.

| Status | Kebutuhan khusus UI | Label |
|---|---|---|
| `Suspect` | Sembunyikan dari daftar pending default; toggle "tampilkan suspect" (Version.txt:62) | [B] |
| `Terjadwal` | Kecualikan dari hitungan "dalam proses" (Version.txt:1018); kolom waktu jadwal + aksi batalkan/jalankan | [B] |
| `ProsesUlang` | Badge khusus; filter terpisah dari `BelumDiproses` | [I] |
| `GagalKirim` | Retryable → tombol kirim ulang aktif; bedakan dari `Gagal` final | [I] |
| `Alihkan` | Trigger failover otomatis (arch_engine.md §4); tampilkan modul asal→tujuan | [B]+[R] |
| `MenungguJawaban` | Status pemulihan default setelah restart (Version.txt:231); badge "dipulihkan" | [B] |
| `Dibatalkan` / `Diabaikan` | Final; aksi proses-ulang/kirim-ulang dinonaktifkan | [I] |
| 21 status gagal validasi (`BukanReseller`, `FormatSalah`, `SaldoTidakCukup`, …) | Grup filter "-- Gagal --" (pola filter Inbox, Version.txt:52) + keterangan alasan | [B] pola, [I] grup |
| `Sukses*` (4 varian) | Bedakan `Sukses` vs `SuksesMasukOutbox`/`SuksesMasukTransaksi`/`SuksesMasukCS` (alur balasan berbeda) | [B] |

## Area C — Master Data

### Layar: Produk
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| CRUD produk, 14 kolom `admindgProduk`: aktif, kode, nama, kode_operator, fisik, tanpa_kode, nominal, harga_beli, harga_jual, kosong, gangguan, stok, qty, prefix_tujuan | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#7; arch_ui.md §1C | [B] | Grid AG Grid + EntityForm. **P0** — tanpa produk & harga, transaksi tidak bisa di-routing |
| Kolom **Rumus Harga** (Ultimate-only): `Rumus harga (Qty * Nominal * 1000) + HrgMarkup`; nilai 0=default, 2=bebas nominal/qty | SUDAH TERCAKUP (disebut, mekanika belum diuraikan) | Version.txt:564, :600, :555, :279; EDISI_DAN_BATASAN.md | [B] definisi; [I] evaluasi saat penentuan harga | Engine: selector rumus_harga; harga dihitung saat validasi/pembuatan transaksi. Kolom via editor numerik terbatas. **P0** |
| Kolom **Filter Tujuan (RegEx)** (Ultimate-only) | SUDAH TERCAKUP (disebut, mekanika belum diuraikan) | Version_en.txt:18; EDISI_DAN_BATASAN.md | [B] keberadaan; [I] divalidasi saat intake (→ `TujuanSalah`/`TujuanDiluarWilayah`) | Engine: validasi regex vs `tujuan` setelah parsing format, sebelum routing. UI: editor + tombol "uji regex". **P0** |
| Qty Yang Diizinkan (pengganti Min/Max Qty), validasi qty saat transaksi masuk | SUDAH TERCAKUP (disebut) | Version.txt:847, :342; Version_en.txt:32, :27 | [B] | Enforce di validasi (postpaid qty>1 tanpa kolom ini → qty dipaksa 1). **P0** |
| 2 kolom hidden (index 13–14) di `admindgProduk`, nama tak diketahui | BELUM ADA | SCHEMA_DEEPDIVE.md §1#7; arch_db.md (`kolom_hidden_13/14` placeholder) | [B] | 2 kolom ekstensi generik di DDL/UI; identifikasi saat spike porting UI. **P1** |
| Kolom Harga Awal (postpaid, potong saldo di awal transaksi) | BELUM ADA | Version.txt:590 | [B] | Kolom `harga_awal` + potong saldo saat transaksi dibuat; refund jika gagal. **P1** |
| Kolom Poin di tabel Produk (Ultimate, maks 32.000) | BELUM ADA | Version.txt:597, :609–611 | [B] | Kolom `poin` + akumulasi poin reseller. **P1** |
| Kolom Postpaid di tabel Produk (modul khusus) | BELUM ADA | Version.txt:1124 | [B] | Kolom `postpaid` + flag routing ke modul postpaid. **P1** |
| Kotak filter Catatan di halaman Produk; kolom Keterangan/Alamat bisa diurut | BELUM ADA | Version.txt:17; Version_en.txt:22 | [B] | Kolom `catatan` + filter box + sortir. **P2** |
| Tombol "Aktifkan Semua" di tabel Produk & Reseller; filter Nominal | BELUM ADA | Version.txt:246, :968 | [B] | Aksi massal di toolbar grid; filter nominal. **P2** |

### Layar: Provider / HLR / Operator
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| 12 kolom `admindgProvider` + `admindgHLR` (kode, prefix_tujuan) + tabel `operator` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#11/#14; arch_ui.md §1C; arch_db.md | [B] | CRUD generik; FK hlr→provider, level→provider. **P0** — mapping prefix→operator & validasi panjang/kode area = inti validasi transaksi |
| Validasi Kode Area: provider↔produk, status `KodeAreaTidakCocok`, checkbox per-modul | SEBAGIAN | Version.txt:732, :829; BEDAH_LOGIKA_BISNIS.md | [B] | Engine: validasi kode_area → `KodeAreaTidakCocok`; setting per-modul "aktifkan filter kode area". **P0** |
| `blok_produk` di Operator/Produk/Harga Grup (blok per operator) | BELUM ADA | Version.txt:696–697 | [B] | Kolom `blok_produk` di operator; produk terblok → skip dari Cek Harga & transaksi. **P1** |
| Kolom Struk Templat Path per Provider (dipakai Kiosk) | BELUM ADA | Version.txt:587–588 | [B] | Kolom `struk_templat_path`. **P1** |
| Centang "Auto Print Receipt on Kiosk" per Provider | SEBAGIAN | Version.txt:431; arch_ui.md §1C (kolom `struk_auto_cetak` ada, perilaku belum dirancang) | [B] | Event-driven: transaksi Sukses & `struk_auto_cetak=1` → job cetak via template provider. **P1** |

### Layar: Parsing (Routing)
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| 8 kolom `admindgParsing`: aktif, kode_modul, kode_produk, prioritas, harga_beli, _harga_jual, perintah, keterangan; routing berdasar prioritas | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#13; arch_ui.md §1C; arch_db.md | [B] | Editor CRUD; engine: prioritas + stok + failover (`Alihkan`). **P0** |
| Kolom **Auto Update Harga** & **Markup** (Ultimate) | BELUM ADA | Version.txt:138–140, :915 | [B] | Kolom `auto_update_harga`, `markup`; engine update harga parsing saat jawaban supplier berubah (bila flag aktif). **P1** |
| Filter Provider di halaman Parsing | BELUM ADA | Version.txt:191 | [B] | Filter provider (join produk→operator). **P2** |
| Auto-generate parsing dari setting modul (modul IP: cukup kode produk) | BELUM ADA | Version.txt:216–217 | [B] | Tombol "Auto Generate Parsing" di Setting Modul. **P2** |

### Layar: Cek Harga (+ Reseller, + Grup)
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Simulasi/cek harga per produk/level | SUDAH TERCAKUP (konsep) | arch_ui.md §1C + §3 (`GET /produk/cek-harga`); SCHEMA_DEEPDIVE.md §1#12 | [B] kolom; [I] rumus = harga_jual + total markup | Endpoint: input kode produk + reseller/level → harga final. **P1** |
| Halaman **Cek Harga Reseller** & **Cek Harga Grup** (filter provider, kolom nama produk, cari berdasar operator) | BELUM ADA | Version.txt:1024, :138, :723, :107 | [B] | Dua layar read-only; reuse endpoint cek-harga. **P1** |
| **Tabel Harga Grup**: harga beli+jual per grup; harga kosong = blok; baca grup upline; normalisasi harga | BELUM ADA | Version.txt:685, :326, :349, :202, :301 | [B] | Tabel `harga_grup` (grup, kode_produk, harga_beli, harga_jual); kosong → blok (bukan 0/-1); baca hierarki upline; validasi normalisasi. **P1** |
| View `harga_agen`: `harga_jual+200` per produk aktif + pola exclude (S50P, %H, %F, FT%, FK%, HF%, Z%, M%, IN%, A[^X]%) | BELUM ADA (view direkonstruksi, belum jadi endpoint/layar) | SCHEMA_DEEPDIVE.md §1#16 + §3; arch_db.md | [B] | Port view ke PG persis (termasuk pola exclude). **P1** |
| `reseller.blok_produk` dihormati format CekHarga & halaman Cek Harga | BELUM ADA | Version_en.txt:21 | [B] | Engine cek-harga filter produk terblok. **P1** |
| **Markup Produk per Reseller**: format CekMarkupProduk/UbahMarkupProduk/HapusMarkupProduk; akhiran `.*` = semua downline; kolom Grup & Upline; bisa minus (Ultimate) | BELUM ADA | Version.txt:664, :619, :501, :393, :386 | [B] | Tabel `markup_produk` (wildcard `OR0001.*`); izinkan markup negatif. **P1** |
| Kolom Total Markup di Reseller + tombol Hitung Ulang Total Markup | BELUM ADA | Version.txt:807–809 | [B] | Kolom `total_markup` + tombol aksi; dipakai engine sebagai harga final. **P1** |
| Proteksi ubah markup: tolak jika markup baru > lama & masih ada transaksi pending | BELUM ADA | Version.txt:73 | [B] | Validasi di endpoint ubah markup. **P2** |

### Layar: Level & Grup
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| 12 kolom `admindgLevel` + hierarki MLM self-reference | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#12; arch_ui.md §1C; arch_db.md | [B] | CRUD + tree view hierarki (upline). **P0** |
| **Grup Reseller** (entitas tersendiri): Batas Harian Trx Per Nomor, Saldo Maksimal, No Trx (Ultimate), bonus, no_ubah_markup | BELUM ADA | Version.txt:97, :53, :124, :121; Version_en.txt:440; Version.txt:532 | [B] | Modelkan `grup_reseller` terpisah dari `level`; enforce di engine (limit harian → `LimitHarian`; saldo maksimal → tolak klaim tiket). **P1** |
| Setting DaftarSesamaLevel; [kodelevel]/[markup]/[selisih] di format daftar downline | BELUM ADA | Version.txt:975–976, :986, :208, :827 | [B] | Setting key-value + param format pendaftaran downline. **P1** |
| Alat Bantu "Ubah Grup Reseller" (ubah massal) | BELUM ADA | Version.txt:1025–1026 | [B] | Aksi batch di Alat Bantu (fase 3). **P2** |

## Area D — Reseller & Keanggotaan

### Layar: Reseller
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| CRUD reseller pola `OR####`; **tidak bisa tambah langsung — hanya via Pendaftaran** (keamanan); edit kode butuh akses Delete | SEBAGIAN | SCHEMA_DEEPDIVE.md §1#6 + §4; arch_ui.md §1D; Version.txt:365–366 | [B] | Enforce: POST /reseller langsung ditolak (atau RBAC Delete); kreasi hanya via endpoint pendaftaran. **P0** |
| Kolom **suspend** (server blok walau upline mengaktifkan) | SEBAGIAN | Version.txt:532–533; BEDAH_LOGIKA_BISNIS.md | [B] | Kolom `suspend` terpisah dari `aktif`; engine cek suspend dulu. **P0** |
| Reset PIN upline→downline; setting HarusUbahPin; MinPanjangPin; PIN disembunyikan di inbox | BELUM ADA | Version.txt:270, :444, :446, :299; BEDAH_LOGIKA_BISNIS.md (HarusUbahPin, PinSalah) | [B] | Aksi "Reset PIN" per baris (otorisasi upline); flag `harus_ubah_pin` → transaksi ditolak sampai PIN diganti. **P0** |
| Transfer saldo antar reseller (format TransferSaldo + [keterangan]) | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §2 (`mutasi.kode_reseller_2`); Version.txt:432 | [B] | **P0** — aliran uang, fase 1 |
| Kolom Berita Bank per reseller (multi keyword `;`, regex `regex:...`) | BELUM ADA | Version.txt:123, :98 | [B] | Kolom `berita_bank`; engine cocokkan pola ke keterangan mutasi bank. **P1** |
| Alamat IP multi (koma; kosong = tidak dicek), Password IP, Report URL | BELUM ADA | Version.txt:494, :734, :766 | [B] | Kolom `alamat_ip`, `password_ip`, `report_url`; validasi IP di endpoint IP Center. **P1** |
| Kode Referral per reseller | BELUM ADA | Version.txt:196–197 | [B] | Kolom `kode_referral`. **P2** |
| Filter Alamat & Kode Area; kolom Keterangan/Alamat bisa diurut; Save Numbers To Vcf | BELUM ADA | Version.txt:971, :25, :247 | [B] | Filter box + sortir + export VCF. **P2** |
| Downline (default Kode Area = Kode Area upline) | SEBAGIAN | Version.txt:1059, :11, :21 | [B] | View downline per reseller + inherit kode_area saat pendaftaran. **P1** |
| Pencatatan log: tambah saldo, registrasi, ubah kode, hapus reseller/produk | BELUM ADA (masuk Area I) | Version.txt:1148–1149 | [B] | Audit trail otomatis aksi sensitif (lihat Area I – Log Sistem). **P1** |

### Layar: Pendaftaran / Registrasi Member
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Form: tombol Refresh daftar upline; **kotak saldo dihilangkan** (keamanan); input Password IP, IP Address, Report URL; Kode Area default = kode area upline | SEBAGIAN | Version.txt:216, :395, :506, :1213, :1059, :645 | [B] | Form: upline picker + refresh, kode_area (inherit upline), password_ip/ip_address/report_url; **tanpa field saldo**; validasi blacklist & duplikat. **P0** |
| Format **DaftarNonMember** (registrasi via inbox): [kodereferral], setting NonMemberRegGroup, Default Markup | BELUM ADA | Version.txt:827, :197, :208, :20, :633 | [B] | Engine parsing → buat reseller di grup default + referral; UI: editor format + setting. **P1** |
| Cek daftar hitam + cek nomor WA/SMS dobel di 4 titik (Pengirim, tambah nomor, downline, member baru) | BELUM ADA | Version.txt:11, :21; Version_en.txt:10, :19 | [B] | Validasi terpusat `cek_nomor(nomor)` dipakai di semua titik. **P0** — anti-fraud inti |
| Alur approval pendaftaran | SEBAGIAN | arch_ui.md §1D menulis "+ approval"; **tidak ditemukan di Version.txt/skema** | [I] | Jangan over-engineer: "daftar member baru" sebagai list monitoring + flag verifikasi, bukan workflow multi-langkah. **P1** |
| Pola ID `OR####` (auto-number) | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §4 | [B] | Generator OR+4 digit. **P0** |

### Layar: Pengirim (Sender)
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Daftar nomor pengirim; tampil berdasar grup; Centang Check Semua; Save to VCF | SEBAGIAN | SCHEMA_DEEPDIVE.md §1#1/#8; Version.txt:218, :100, :247 | [B] | Grid + filter grup + bulk check + export VCF. Tabel `pengirim` perlu direkonstruksi [R]. **P0** |
| Cek nomor WA/SMS dobel (+ fix duplikat awalan 62) | SEBAGIAN | Version.txt:21, :255 | [B] | Normalisasi nomor (62/0/+62) sebelum cek duplikat semua channel. **P0** |
| Cek daftar hitam di halaman Pengirim | SEBAGIAN | Version.txt:11 | [B] | Integrasi cek blacklist saat tambah/edit pengirim. **P0** |
| **Daftar Hitam**: kolom **Fraud** → nomor fraud kena hit = reseller otomatis dinonaktifkan | SEBAGIAN | Version.txt:598; Version_en.txt:504 | [B] | Tabel `daftar_hitam` (nomor, fraud, kadaluarsa, keterangan); engine: hit → `reseller.aktif=0` + log. **P0** |
| Nomor masuk daftar hitam otomatis via setting NomorBlacklist, **kadaluarsa 30 hari** | BELUM ADA | Version.txt:117 | [B] | Kolom `kadaluarsa`; job harian; setting jawaban NomorBlacklist → auto-insert. **P1** |
| Setting DefaultAksesPengirim (+ varian Y/G/W) | BELUM ADA | Version.txt:72, :81 | [B] | Setting key-value; default terisi otomatis di form. **P1** |
| Regex Pengirim + Parameter Balasan di tabel Dompet (eWallet) | BELUM ADA (masuk Area G) | Version.txt:58, :82 | [B] | Kolom `regex_pengirim`, `parameter_balasan` di `dompet`; routing balasan per dompet. **P1** |

### Layar: Kiosk
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Modul Kiosk: kasir toko + cetak struk; cari transaksi berdasar **nomor tujuan** (bebas format, bukan hanya angka) | BELUM ADA | Version.txt:587, :561, :563; arch_engine.md (fase 2) | [B] | Layar kasir: search box → daftar transaksi → cetak. **P1** (fase 2) |
| Sembunyikan baris: klik kepala baris + Delete; auto-update daftar | BELUM ADA | Version.txt:562, :327 | [B] | Row-header click + Delete = soft-hide per sesi; auto-refresh via SignalR. **P1** |
| Cetak struk otomatis saat sukses (per provider) + Struk Templat Path; via IE → diganti QuestPDF→CUPS | BELUM ADA | Version.txt:431, :588, :126; arch_engine.md | [B] | Pipeline: template per provider → PDF (QuestPDF) → CUPS; auto-print bila `struk_auto_cetak=1`. **P1** |
| Template `NotaBill.prt` ("BUKTI TRANSFER DEPOSIT"): placeholder [keterangan], [date], [time], [kode], [nama], [jumlah], [saldolama], [saldo] | BELUM ADA | extracted/NotaBill.prt (338 B); FINDINGS.md | [B] | 8 placeholder wajib didukung renderer struk. **P1** |
| Panel promo di modul Kiosk | BELUM ADA | Version.txt:120 | [B] | Banner promo di layar kasir (konten dari setting). **P2** |

## Area E — Modul & Terminal

> **Koreksi dari agen:** daftar tipe di tugas menduplikat `SMSCenter` dan menghilangkan `Fisik`. Enum asli `YAR.OtomaX.TipeModul` = 15 nilai: `Fisik, SMSSender, _GsmUssd, Dial, SMS, Keypad, IM, SMSCenter, STK, WebSMSSender, IP, SpecialHth, OH, ApiSiDompul, ApiAxis` [B: arch_engine.md §2]. "Internet Banking", "WA Center", "SMS Order/H2H", "IM Hybrid HTH" di menu UI **bukan** nilai TipeModul — melainkan modul menu yang dipetakan ke worker engine.

### Layar: Daftar Modul
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Daftar 15 tipe modul dikenali (enum `TipeModul`) | SUDAH TERCAKUP | arch_ui.md §1; arch_db.md §2.5; arch_engine.md §2 | [B] | UI menampilkan 15 tipe persis dari enum |
| Tabel pemetaan "modul menu UI" → `TipeModul` + worker engine (Internet Banking/WA Center = worker Selenium, bukan TipeModul) | BELUM ADA | arch_ui.md Area E; BEDAH_LOGIKA_BISNIS.md §2 | [I] | **P0**: tanpa ini "Tambah Modul" ambigu |
| **Parameter per-tipe-modul** (form setting berbeda per tipe) | SEBAGIAN | arch_ui.md: hanya "Konfigurasi per tipe modul" (tanpa skema); Version.txt:7,12,15,118,217,249–251,254,409,436,486,509–510,518,575–580 | [B] fitur; [R] skema | **P0**: wizard "Tambah Modul" dengan field per tipe; simpan JSON `parameter` + kolom typed untuk yang sering di-query |
| SMSSender: Message Regex (Exclusion), Auto Off jika gagal kirim, kirim-ulang pending | SEBAGIAN | Version.txt:12, :580, :361; arch_engine.md §2 | [B] | Field `message_regex_exclusion`, toggle `auto_off_on_fail`, aksi kirim-ulang per sender. **P1** |
| WebSMSSender: HTTP GET; parameter `to/to2/to3`, `[kodereseller]`, `[namareseller]` | SEBAGIAN | Version.txt:589, :575–580, :178; arch_engine.md §2 | [B] | **P0**: form wajib URL template + parameter auto-terisi; test-send dari UI |
| IP Client: HTTP Request Timeout + HTTP URL Report (page 3); Pause Per Transaksi; auto-generate parsing | SEBAGIAN | Version.txt:509, :510, :375, :217; arch_engine.md §2 | [B] | Tab "Page 3": `http_timeout`, `http_url_report`, `pause_per_trx`; tombol "Auto-generate parsing". **P1** |
| SpecialHth/OH: OH dukung **parsing kosong**; `[keterangan]`/`[description]` pada reply | SEBAGIAN | Version.txt:257, :868; TARGETED_DECRYPTION.md §2 | [B] | Field `allow_empty_parsing`, `reply_param_map`; UI endpoint+header per supplier. **P1** |
| ApiSiDompul: edit ID/Nomor Sender Supplier; terima jawaban dari IP Center; saldo di Monitor Modul | SEBAGIAN | Version.txt:254, :199, :278, :6; arch_engine.md §2 | [B] | Form kredensial + endpoint; kartu saldo di monitor. **P1** |
| ApiAxis: tipe modul API Axis | SEBAGIAN | Version.txt:116; arch_engine.md §2 | [B] | Form kredensial (endpoint dari sniff). **P1** |
| IM Center: Filter Server Exclusion | BELUM ADA | Version.txt:652; arch_engine.md §2 | [B] | Field `server_exclusion` (via sidecar slixmpp). **P2** |
| Internet Banking: username/ID (min 4 char), "Hanya cek jika ada tiket open", Pola berita auto deposit + Filter Mutasi, Cek semua rekening, butuh Selenium | SEBAGIAN | Version.txt:304, :180, :122, :355, :474, :86, :544; arch_engine.md §2 | [B] | Form per bank: kredensial (vault), pola berita, filter mutasi; worker terisolasi + fallback manual (R1). **P1** |
| Parameter umum semua tipe: Parameter Parsing s.d. 8000 karakter + Parsing 2; Batas Antrian s.d. 32000; Jam Aktif (awal/akhir + detik); Filter qty; Filter Regex tujuan; simbol desimal; saldo warning; toggle "Update harga parsing"; catatan | SEBAGIAN | Version.txt:15, :7, :581, :251, :258, :249, :250, :436, :409, :518, :118; arch_ui.md hanya menyebut 2 di antaranya | [B] | **P0**: `parameter_parsing`, `parameter_parsing2`, `batas_antrian`. **P1**: jam aktif (awal > akhir didukung), filter qty/regex, desimal, saldo warning, update-harga-parsing, catatan |
| Auto kirim ulang per modul (interval min 10 detik); pilih modul saat kirim ulang pending | BELUM ADA | Version.txt:486, :478 | [B] | Field `auto_resend_interval` + dialog kirim-ulang dengan pemilih modul. **P1** |
| Duplikat modul (klik kanan) | BELUM ADA | Version.txt:212 | [I] | Aksi "Duplikat" (copy parameter + parsing, nama baru). **P1** |
| "Active modules only" pada monitor (saldo) modul | BELUM ADA | Version.txt:458, :278 | [B] | Filter tampilan + kolom saldo per modul. **P2** |

### Layar: Setting Modul & Terminal
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Menu "Tambah Modul" | SUDAH TERCAKUP | Version.txt:960; arch_ui.md §3 (`PUT /modul/{kode}`) | [B] | Tambahkan wizard per tipe |
| Tambah/edit terminal: port modem, label s.d. 50 karakter | SEBAGIAN | Version.txt:989; arch_db.md §2.5 (`port`, `nama`) [R] | [B] fitur; [R] skema | **P0**: form `port` (/dev/ttyUSB0), `label` (≤50 char); udev naming stabil sudah di ARSITEKTUR_LINUX.md §3 |
| **Modul Terkait** pada setting terminal (balasan SMS diproses modul lain) | BELUM ADA | Version.txt:504 (+314: tidak bekerja untuk ApiSiDompul) | [B] | **P0**: field `modul_terkait[]` (multi-select) → routing ReplyDispatch; memengaruhi engine |
| Nilai kadaluarsa SMS per terminal | BELUM ADA | Version.txt:1155 | [B] | Field `sms_expiry`. **P2** |
| Menu "Putuskan Semua Terminal" | SUDAH TERCAKUP | Version.txt:330; arch_ui.md §1 + §3 | [B] | Konfirmasi + progress (bug "lambat" aslinya) |
| Modul Hapus Data: hapus terminal/modul/sender tak terpakai (soft-delete) + purge | BELUM ADA | Version.txt:1086, :1106 | [B] | Soft-delete + job "Hapus Data" (purge `status_terhapus`). **P1** |
| Pengecekan kode terminal pada inbox (longgar, dukung API) | SEBAGIAN | Version.txt:235 | [B] | Validasi `kode_terminal` di IngressHub jangan strict-match (sumber API/WA tidak punya terminal). **P1** |
| **Pemetaan 18 token AT + 10 expected responses + 1 regex → parameter modul** | SEBAGIAN | TARGETED_DECRYPTION.md §1 (builder [2450]/[2468]/[2473], expected di [2424], regex -160994090); arch_engine.md §3 | [B] pola; [R] isi string | Engine AtChannel sudah dirancang (**P0**). **P1-UI**: tab "AT / Modem" per terminal: init sequence, expected responses editable, timeout per command, mode SMS text/PDU. Catatan jujur: isi exact belum terdekripsi — pakai standar GSM 07.05/07.07 |
| Raw AT log di Monitor Modul (debug modem) | SUDAH TERCAKUP | arch_ui.md §5; PRD.md §4.5 | [I] | Tab log per terminal (ring buffer). **P1** |

### Layar: Administrasi Jawaban
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Template format & balasan (`Balasan0`, `Format0` sebagai setting key-value) | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §4; arch_ui.md; arch_db.md §2.5 #8 | [B] | `GET/PUT /jawaban` sudah di kontrak API |
| PolaParam0/20 + regex parsing jawaban supplier | SEBAGIAN | SCHEMA_DEEPDIVE.md §4; arch_engine.md R8; Version.txt:310 | [B] | **P0**: editor PolaParam sebagai daftar regex bernama + tombol "uji" |
| Katalog placeholder: `[trxcount]` (tidak menghitung trx itu sendiri), `[pin]`, `[qty]`, `[nomorseri]`, `[keterangan]`, `[markuptotal]`, `[saldo]` vs `[saldoawal]`, `[waktu]`, `[bank]`, `[refid]`/`[trxid]` (alias), `[kodereseller]`, `[namareseller]` | SEBAGIAN | Version.txt:35, :36, :571, :59, :151, :359, :417, :345, :181, :965, :966, :178, :136, :137 | [B] | **P0**: dokumentasikan katalog resmi di UI (tooltip/panel referensi) + uji substitusi; daftar lengkap masih [R] |
| Kolom "Awalan dan akhiran nominal" | BELUM ADA | Version.txt:1199 | [B] | Kolom `awalan_nominal`, `akhiran_nominal` per template. **P2** |
| Kolom catatan + pencarian mencakup catatan | BELUM ADA | Version.txt:92 | [B] | Kolom `catatan` + search. **P2** |
| **Penguji Jawaban** (simulator: pilih modul, tempel jawaban supplier → parameter tertangkap & balasan terkirim) | BELUM ADA | Version.txt:201, :377, :1004 | [B] | **P1**: layar/modal "Uji Jawaban" — alat debug utama operator |
| Setting jawaban per status (NomorBlacklist → blacklist 30 hari; DaftarNonMember kosong → error) | BELUM ADA | Version.txt:117, :633, :20 | [B] | Seksi "Jawaban per status" (mapping status → template + aksi sampingan). **P1** |
| Parameter Balasan di tabel Grup/Dompet (bedakan reply; dipakai TiketDeposit) | BELUM ADA | Version.txt:82, :243, :228, :353 | [B] | Field `parameter_balasan` di Grup & Dompet. **P2** |

## Area F — Inbox / Outbox / Pesan

### Layar: Inbox
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Grid SMS masuk: `kode, pesan, kode_reseller, pengirim, tipe_pengirim` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1 obj 8; arch_ui.md; API `GET/POST /inbox` | [B] | — |
| Cleanup: hapus inbox yang reseller-nya tidak dikenal | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §3; arch_ui.md | [B] | Jadikan job terjadwal (cron), bukan di akhir SP — arch_db.md §5.4. **P1** |
| Kolom "Service Center" di Inbox | BELUM ADA | Version.txt:1120 (+880: SC kosong SMS bisa masuk) | [B] | Kolom `service_center` + filter; SC kosong tetap diterima. **P1** |
| Filter status "-- Gagal --" di tabel Inbox | BELUM ADA | Version.txt:52 | [B] | Filter status termasuk nilai gagal. **P2** |
| Sembunyikan nilai PIN pada inbox | BELUM ADA | Version.txt:299 | [B] | Masking PIN di kolom pesan (penting untuk web multi-operator). **P1** |
| Report URL reseller boleh tidak lengkap → dilengkapi otomatis dari IP/pengirim | BELUM ADA | Version.txt:766 | [B] | Auto-complete report URL saat proses inbox. **P2** |
| Inbox IM Center format query-string → verifikasi WrongSignature | SEBAGIAN | Version.txt:320; BEDAH_LOGIKA_BISNIS.md | [B] | Parser inbox per `tipe_pengirim` (SMS vs IM/API query-string). **P1** |
| Setting `AutoResponFirst` (cek Auto Respon dulu sebelum format lain) | BELUM ADA | Version.txt:119 | [B] | Toggle urutan pipeline di Setting; IngressHub menghormatinya. **P1** |
| "Ignore all inbox" pada setting WA Center | BELUM ADA | Version.txt:125 | [B] | Toggle per modul WA Center. **P2** |
| Modul Hapus Data: hapus inbox/outbox/komisi (+ optimasi index) | BELUM ADA | Version.txt:1106, :372 | [B] | Dialog "Hapus Data" dengan rentang tanggal + tipe data. **P1** |
| Inbox paralel (Ultimate) | SUDAH TERCAKUP | Version.txt:190; ARSITEKTUR_LINUX.md (Channel<T> bounded) | [I] | Engine channel-based sudah mencakup |

### Layar: Outbox
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Antrian SMS keluar + kirim ulang per baris | SUDAH TERCAKUP | arch_ui.md + §3 (`POST /outbox/{kode}/kirim-ulang`); arch_db.md §2.5 #5 [R] | [B] fitur; [R] skema | — |
| Status outbox berwarna (kuning = belum terkirim/pending) | SEBAGIAN | Version.txt:425, :997; arch_ui.md hanya "status outbox" | [B] | **P0**: legenda status warna di UI — operator membaca warna, bukan kode |
| Setting `BatalkanOutboxKuning` (batalkan outbox lama belum diproses) | BELUM ADA | Version.txt:84 | [B] | Setting `batalkan_outbox_kuning_menit` + job pembatalan. **P1** |
| Kirim ulang TIDAK bekerja saat "sedang diproses" (bug yang di-fix) | SEBAGIAN | Version.txt:566 | [B] | Guard di API kirim-ulang (tolak/queue jika processing) + pesan jelas. **P1** |
| Fallback otomatis: alihkan outbox SMS → WA jika tidak ada SMS Sender aktif (`SendWaIfNoSmsSender`) | BELUM ADA | Version.txt:385 | [B] | Setting `send_wa_if_no_sms_sender` + indikator saat fallback terjadi. **P1** |
| Kolom sender pada halaman Outbox | BELUM ADA | Version.txt:387 (bug: tidak muncul data) | [B] | Kolom `sender`/`kode_modul` tampil benar (join modul). **P1** |
| Statistik Outbox berupa checkbox (matikan untuk ringan query) | BELUM ADA | Version.txt:641 | [B] | Toggle "tampilkan statistik" (perf DB besar). **P2** |
| Ringkasan Outbox pada halaman Transaksi | BELUM ADA | Version.txt:826 | [B] | Panel ringkasan di Detail Transaksi. **P2** |
| Index `kode_inbox` & `kode_transaksi` pada tabel Outbox | SUDAH TERCAKUP | Version.txt:219, :372; arch_db.md | [I] | Pastikan index ada di DDL |

### Layar: Auto Respon
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Aturan balasan otomatis (CRUD aturan) | SEBAGIAN | arch_ui.md (tabel `auto_respon`); Version.txt:37, :46 (kolom `balasan` NOT NULL, `format_balasan` NOT NULL, `timestamp`) | [B] ada; [R] skema | **P1**: desain tabel `auto_respon` dari kebutuhan + layar CRUD |
| Penanganan `\r\n` pada Auto Respon | BELUM ADA | Version.txt:898, :284 | [B] | Normalisasi newline di editor template. **P2** |
| `AutoResponFirst` dievaluasi sebelum format lain | BELUM ADA | Version.txt:119 | [B] | Aturan auto-respon dievaluasi di tahap awal pipeline. **P1** |

## Area G — Keuangan

### Layar: Mutasi Saldo
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Riwayat mutasi 10 kolom; `saldo_akhir` = snapshot per baris | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#3; arch_db.md §2.1; arch_ui.md | [B] | Validasi tipe NUMERIC(19,2) via restore `.bak` ke SQL Express. **P0** — fondasi rekonsiliasi |
| Filter Kode Dompet di halaman Mutasi | SEBAGIAN | Version.txt:28 | [B] | Buat tabel `dompet` + kolom `kode_dompet`; filter chip di grid. **P1** |
| Enum `jenis` mutasi (masuk/keluar, deposit, transfer, refund, tukar komisi…) | SEBAGIAN | arch_db.md §2.1 (`jenis SMALLINT` [I]) | [I] | Definisikan enum `jenis_mutasi` eksplisit + seed sejak MVP. **P0** |
| Link `kode_reseller_2`, `kode_transaksi`, `kode_outbox` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#3, §2 | [B] | Jadikan link drill-down di UI. **P1** |
| Gaya Excel: copy-paste, sortir; export formatting decimal benar | SEBAGIAN | Version.txt 4.2.8; arch_ui.md §5 | [B] | Acceptance test export (decimal tidak jadi scientific). **P1** |
| Cleanup: hapus mutasi milik reseller yang sudah tidak ada (`#rsmati`) | BELUM ADA | SCHEMA_DEEPDIVE.md §3 | [B] | Spec: soft-delete vs hard-delete + cascade; jangan hard-delete diam-diam. **P1** |

### Layar: Mutasi Komisi / Tukar Komisi
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Halaman Mutasi Komisi (Ultimate-only; dibuka semua) — kolom produk & tujuan, label Jumlah Total | SEBAGIAN | EDISI_DAN_BATASAN.md; Version.txt:45, :60, :78, :173; arch_db.md §2.5 | [B] | View `mutasi_komisi` = `komisi` JOIN `transaksi` (produk, tujuan) + agregat total. **P1** |
| Komisi tunda MLM: `SettingTundaKomisi=1`; SP `AddKomisiTunda` upsert | SEBAGIAN | SCHEMA_DEEPDIVE.md §3; arch_engine.md; arch_db.md §3.1 | [B] | Spec pemicu: status sukses saja? definisi `jumlah`; seed `SettingTundaKomisi=0`. **P0** — uang |
| Penukaran via SMS `TUKAR.[jumlah].[pin]` / menu Alat Bantu | SEBAGIAN | SCHEMA_DEEPDIVE.md §3; arch_db.md §3.1; Version.txt:165, :176 | [B] | Validasi PIN, kurangi `komisi` → tambah `mutasi` + `saldo` atomik; form admin untuk tukar manual. **P1** |
| Bug: tukar komisi timeout bila data terlalu banyak; notifikasi modul salah | BELUM ADA | Version.txt:165, :1010, :1108 | [B] | Penukaran sebagai job batch ber-paging + progress; notifikasi pakai snapshot saldo benar. **P1** |
| Setting `KomisiFull` (bagi komisi penuh walau mengurangi margin) | SEBAGIAN | Version.txt:585; arch_ui.md §1 | [B] | Cabang kalkulasi di engine + toggle di Setting MLM. **P1** |

### Layar: Mutasi Poin / Unit
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Halaman Mutasi Poin | SEBAGIAN | Version.txt:42; arch_ui.md §1 | [B] | Tabel `mutasi_poin` + kolom `poin` di `reseller` (cermin `mutasi`). **P1** |
| Dialog Tambah Poin (+ opsi "Kirim laporan" via SMS/WA) | SEBAGIAN | Version.txt:302; arch_db.md — | [B] | Endpoint terima `kirim_laporan` + kanal; reuse ReplyDispatch. **P1** |
| Menu Alat Bantu – Reset Poin | BELUM ADA | Version.txt:845 | [B] | `POST /poin/reset` + konfirmasi destruktif + audit log. **P1** |
| Sistem Unit (Ultimate-only): Mutasi Unit + Tambah Unit | BELUM ADA | Version.txt:60, :399, :500; EDISI_DAN_BATASAN.md | [B] | Tabel `mutasi_unit` + `POST /unit/tambah`; definisikan semantik unit dulu. **P1** |
| Tabel Hadiah Poin + Penukaran Poin (Sudah Diserahkan, tanggal serah) + format `LihatHadiahPoin` | BELUM ADA | Version.txt:599, :608, :616, :617, :621 | [B] | Tabel `hadiah_poin` + `penukaran_poin`; layar daftar penukaran. **P2** |
| Kolom Poin di Produk & Level; field poin di transaksi; balasan `KurangPoin`/`TukarPoin` | BELUM ADA | Version.txt:609–615 | [B] | Kolom `poin` di `produk`, `level`, `transaksi`; format balasan → Administrasi Jawaban. **P1** skema, **P2** balasan |

### Layar: Tiket Deposit
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Klaim tiket: status `'O'`=Open, `tgl_status`, link `kode_inbox` & `kode_data_bank` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#2, §3; arch_db.md §2.1; arch_ui.md | [B] | State `Cancelled`/`Settled` eksplisit; tampilkan umur tiket open. **P0** |
| Angka unik deposit + optimasi pembuatan angka unik | SEBAGIAN | Version.txt:259 | [B] | Generator angka unik thread-safe (hindari dobel saat klaim paralel). **P0** — uang |
| Parameter `[bank]`: tiket hanya di-settle modul bank yang sesuai (Ultimate-only) | SEBAGIAN | Version.txt:181 | [B] | Validasi bank tiket vs bank modul; tolak + log bila mismatch. **P1** |
| Urutan proses mutasi kredit: 1) Tiket (angka unik), 2) PolaAutoDeposit, 3) Berita bank | SEBAGIAN | Version.txt:68 (4.2.6) | [B] | Spec pipeline auto-claim berurutan di worker Internet Banking; tiap tahap tercatat di `data_bank.catatan`. **P0** |
| Kolom Saldo Maksimal di Grup Reseller, dicek saat pengambilan tiket | SEBAGIAN | Version.txt:124 | [B] | Tambah `saldo_maksimal`; tolak tiket bila saldo + jumlah > maksimal. **P1** |
| Tiket Cancelled bila ada input saldo manual jumlah sama | SEBAGIAN | Version.txt:315 | [B] | Anti-dobel: input manual jumlah sama → auto-cancel tiket open + notifikasi. **P1** |
| Optimasi kolom Reseller pada modul Tiket Deposit | SEBAGIAN | Version.txt:103 | [B] | Tampilkan & filter by reseller dari berita bank. **P2** |

### Layar: Data Bank
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Daftar mutasi bank 11 kolom, flag `terklaim`, link `kode_tiket` | SUDAH TERCAKUP | SCHEMA_DEEPDIVE.md §1#4; arch_db.md §2.1 | [B] | Tambah kolom `approved` + index (bank, tgl_bank). **P0** |
| Kolom Approved + "Lihat Riwayat Data" | SEBAGIAN | Version.txt:75, :77 (4.2.5) | [B] | `approved BOOLEAN` + tabel audit `data_bank_riwayat`; drawer riwayat. **P1** |
| Klaim otomatis via Internet Banking (BCA/BNI/BRI/Mandiri; dibuka semua) | SEBAGIAN | EDISI_DAN_BATASAN.md; arch_engine.md; ARSITEKTUR_LINUX.md §2 | [B] | MVP: input mutasi manual via UI dulu; worker Selenium = P1 (polling 300 dtk) |
| Kolom Berita Bank di Reseller (multi keyword `;`, regex `regex:…`) + Pola berita auto deposit | SEBAGIAN | Version.txt:122, :123, 4.2.3 | [B] | `berita_bank TEXT` di `reseller`; engine compile regex saat auto-claim; editor pola. **P1** |
| Filter: mutasi mengandung teks tertentu → terbaca tapi tidak diproses | BELUM ADA | Version.txt:296 | [B] | Setting pola ignore per bank; badge "diabaikan" di grid. **P2** |
| Mutasi debit/keluar mencurigakan → Log Sistem, tanpa auto correction | SEBAGIAN | Version.txt:876 | [B] | Debit tak dikenal → log warning + tandai baris; jangan auto-koreksi. **P1** |
| Auto deposit hanya untuk data bank hari ini/PEND; cek data dobel | SEBAGIAN | Version.txt:112, :113 | [B] | Unique constraint `(bank, tgl_bank, jumlah, keterangan)` + filter tanggal. **P0** — cegah deposit ganda |
| Setting "Hanya cek jika ada tiket open" | BELUM ADA | Version.txt:180 | [B] | Toggle di Setting modul Internet Banking. **P2** |

### Layar: Dompet / Dompet Reseller
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Menu Administrasi – Dompet; kolom Regex Pengirim, Parameter Balasan, Refund Ke | SEBAGIAN | Version.txt:44, :58, :82; arch_ui.md §1 + §3 | [B] | Tabel `dompet` (kode, nama, regex_pengirim, parameter_balasan, refund_ke); CRUD admin. **P1** |
| Multi-dompet (edisi eWallet): `kodedompet` di format transaksi, validasi `kode_dompet` | SEBAGIAN | Version.txt:94, :110, :194 | [B] | **Keputusan P0**: single vs multi dompet sebelum DDL final — mengubah `mutasi`/`transaksi` |
| Refund: auto-refund jawaban (syarat Dibatalkan + trxid); Refund Ke mendukung `.` | SEBAGIAN | Version.txt:19, :230, :286 | [B] | Spec refund idempoten (cek SN/trxid ganda); mutasi refund ter-link ke transaksi asal. **P0** |
| Locking pemotongan saldo untuk transaksi dan refund | SEBAGIAN | Version.txt:1044 | [B] | `SELECT … FOR UPDATE` pada `reseller.saldo` di semua path; uji race paralel. **P0** — bug asli bikin saldo minus |
| Hak akses menu Administrasi – Dompet & Dompet Reseller | → Area I | Version.txt:44 | [B] | Menu dompet = objek RBAC prioritas. **P0** bersama RBAC |

### Layar: Voucher Fisik
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Stok & status voucher; bug 4.3.1: USED padahal transaksi gagal | SEBAGIAN | Version.txt:8; arch_ui.md §1; arch_engine.md (fase 2) | [B] | Tabel `voucher` (kode, kode_produk, nomor_seri, harga_beli, status, tgl_entri, tgl_kadaluarsa); state machine ketat: USED hanya setelah sukses terkonfirmasi — bug asli jadi acceptance test. **P1** |
| Kolom Harga Beli + Nomor Seri; label Total Harga Beli | BELUM ADA | Version.txt:153, :163 | [B] | Termasuk di DDL `voucher`. **P1** |
| Prioritas pemakaian berdasar tgl kadaluarsa lalu tgl entri (FIFO) | BELUM ADA | Version.txt:512 | [B] | `ORDER BY tgl_kadaluarsa, tgl_entri`; lock baris saat alokasi. **P1** |
| Pencarian keterangan; tampil berdasar tgl entri; kolom Kadaluarsa | BELUM ADA | Version.txt:145, :215, :751 | [B] | Filter tgl entri + pencarian keterangan. **P2** |
| Relasi voucher↔transaksi | BELUM ADA | Version.txt:316 | [B] | `kode_voucher` nullable di `transaksi`; detail tampilkan voucher terpakai. **P1** |
| Voucher Hybrid | BELUM ADA | Version.txt:1029 | [B] | Riset KB resmi dulu; parkir. **P2** |

## Area H — Laporan

### Layar: Rekap Komisi Reseller
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Laporan komisi **termasuk data tukar komisi** (pernah bug: tidak termasuk) | SEBAGIAN | Version.txt:164; arch_ui.md §1 | [B] | Spec: `komisi` (tunda) UNION mutasi jenis=tukar_komisi; kolom: reseller, tunda, sudah ditukar, sisa; bug asli jadi test case. **P1** |

### Layar: Saldo Awal Reseller
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Menu + **warning bila perhitungan saldo awal ada selisih** | SEBAGIAN | Version.txt:76, :79; arch_ui.md §1 | [B] | Tabel `saldo_snapshot_harian` via scheduler; bandingkan snapshot vs hitung-ulang `mutasi`; tampilkan selisih sebagai warning. **P1** |
| Kolom Saldo Awal di transaksi (`[saldoawal]` = saldo saat terinput) | SUDAH TERCAKUP | Version.txt:418; SCHEMA_DEEPDIVE.md §1#1 | [B] | Engine menulis `saldo_awal` sebelum potong saldo. **P0** — data audit |

### Layar: Laporan Transaksi / Umum
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Modul Laporan (ikut Free Edition) — rekap per produk/periode | SEBAGIAN | FINDINGS.md; arch_ui.md §1 | [B]/[I] | Spec: dimensi produk × periode; metrik qty, omzet, harga_beli, margin, per status; parameter tanggal + produk + reseller. **P1** |
| Data `[qty]` di balasan `LaporanTransaksi` & `CekTransaksiAkhir` (via SMS) | BELUM ADA | Version.txt 4.2.6 | [B] | Format balasan SMS → Administrasi Jawaban. **P2** |
| Export laporan formatting decimal benar | SEBAGIAN | Version.txt 4.2.8 | [B] | Acceptance test export (sama seperti Mutasi). **P1** |

## Area I — Pengaturan & Sistem

### Layar: Setting Umum & Lain-Lain
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Setting key-value: `Setting0`, `SettingMLM`, `Balasan0`, `Format0`, `PolaParam0/20`, "Seting Lain-Lain" | SEBAGIAN | SCHEMA_DEEPDIVE.md §4; arch_db.md §4 #8 (tabel `setting` [R]); arch_ui.md §3 | [B]/[R] | Tabel + seeder default + UI per grup (Umum/MLM/Lain-Lain); tiap key terdokumentasi (tipe, default, efek) — menggantikan ratusan checkbox WinForms. **P0** fondasi |
| Night Mode (Menu File) | SUDAH TERCAKUP (desain) | Version.txt:653; arch_ui.md §6 | [B] | Dark theme sejak awal; preferensi per pemakai. **P1** |
| AutoDeleteSistemLog & AutoDeleteTiketDeposit (disisakan 31 hari) | SEBAGIAN | Version.txt:43, :83 | [B] | Scheduler retensi: hapus > N hari (default 31); **kecualikan log audit**. **P1** |
| Optimasi: `TurnOffModuleLogs`, `TurnOffBlacklist` | SEBAGIAN | Version.txt:534, :535 | [B] | Flag engine; `TurnOffBlacklist` beri peringatan keamanan + butuh hak admin. **P1** |
| `DisableXpSslValidation` | TIDAK DIPORTING (obsolete — workaround Windows XP) | Version.txt:922 | [B] | Keputusan, bukan gap |
| Setting `BatalkanOutboxKuning` | BELUM ADA (lintas Area F) | Version.txt 4.2.5 | [B] | Masuk spec Outbox: job batalkan outbox > threshold. **P2** |

### Layar: Hak Akses Pemakai
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| RBAC per menu (pernah bug khusus Administrasi–Dompet) | SEBAGIAN | Version.txt:44; arch_ui.md §3 + §5 risiko #5 | [B] | Tabel `pengguna`, `peran`, `peran_menu`; seed admin; proteksi API + sembunyikan menu; menu Dompet = test case. **P0** — tidak bisa disusulkan |
| Mode `-client` / SafeMode | TIDAK DIPORTING (keputusan arsitektur) | FINDINGS.md; ARSITEKTUR_LINUX.md §2 | [B] | Digantikan web UI; keputusan, bukan gap |

### Layar: Alat Bantu
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Monitor Saldo Reseller (tabel `monitor_saldo`; checkbox Auto Update) | SEBAGIAN | Version.txt:63, :80, 4.2.7 | [B] | Tabel `monitor_saldo` + `GET /monitor-saldo` (live via SignalR); job periodik; watchlist. **P1** |
| Tukar Komisi (menu) | SEBAGIAN | Version.txt:176 | [B] | Form admin: pilih reseller → jumlah → PIN → eksekusi; tampilkan sisa tunda. **P1** |
| Tambah Unit (menu, Ultimate-only) | BELUM ADA | Version.txt:399 | [B] | `POST /unit/tambah` + dialog. **P1** |
| Reset Poin (menu) | BELUM ADA | Version.txt:845 | [B] | `POST /poin/reset` + konfirmasi + audit. **P1** |
| Kill All Chromes | BELUM ADA | Version.txt:459; arch_engine.md | [B] | `POST /admin/kill-chrome`: hentikan konteks browser per worker dengan aman; konfirmasi + hak admin + log. **P1** |
| Penukaran Hadiah Poin (menu) | BELUM ADA | Version.txt:617 | [B] | Lihat Area G – Hadiah Poin. **P2** |

### Layar: Log Sistem
| Fitur OtomaX asli | Status di rancangan Linux | Bukti sumber | Label | Rekomendasi + Prioritas |
|---|---|---|---|---|
| Tabel `oto_histori` / `oto_histori_update` | SEBAGIAN | SCHEMA_DEEPDIVE.md §1#9-10; arch_db.md §2.4 | [B] | Lengkapi skema: `waktu, modul, level, kode_pengguna, pesan, data_json`; bedakan operasional vs watermark sync. **P1** |
| Entri log wajib: login; tambah saldo (+keterangan deposit); registrasi/ubah/hapus reseller; tambah/hapus produk & modul; mutasi bank debit mencurigakan | SEBAGIAN | Version.txt:690, :794, :876, :1148, :1149 | [B] | Helper `audit_log()` di engine untuk semua path; level info/warning/error; jangan pernah hapus via "Hapus Data". **P1** (sebagian **P0** untuk tambah-saldo) |
| Filter Tanggal di Log Sistem | SEBAGIAN | Version.txt:200 | [B] | `GET /log-sistem?dari=&sampai=&modul=&level=` + filter UI. **P1** |
| "Hapus Data tidak menghapus Log Sistem" (pernah bug) | SEBAGIAN | Version.txt:818 | [B] | Spec Hapus Data: whitelist tabel yang dipertahankan; bug asli jadi test case. **P0** — integritas audit |
| Mode hening: log modul dimatikan (no popup/beep) | SEBAGIAN | arch_ui.md §1; Version.txt:534 | [I] | Bedakan: (a) matikan *pencatatan* (engine), (b) mode hening *tampilan* (preferensi per pemakai). **P2** |
| Log Sistem tidak auto-refresh (pernah bug) | SEBAGIAN | Version.txt:693, :891 | [B] | Live tail via SignalR; pagination server-side (pernah "not responding" — :879). **P1** |

---

## Catatan metodologi & batas bukti

1. **Sumber bukti utama** adalah revision log resmi 4.3.1 (`Version.txt` 82 versi, 1213 baris + `Version_en.txt`), skema dari `db_otomax.bak`, enum dari inner assembly, dan 28 token hasil targeted decryption. Tidak ada screenshot/form WinForms yang bisa dibuka (string terenkripsi) — yang terverifikasi adalah *keberadaan fitur dan perilakunya*, bukan detail visual (posisi tombol, nama menu persis).
2. **Gap skema terbesar**: tabel `dompet`, `voucher`, `mutasi_poin`/`mutasi_unit`, `hadiah_poin`, `pengguna`/`peran`, `pengirim`, `daftar_hitam`, `harga_grup`, `markup_produk`, `grup_reseller`, `auto_respon`, `saldo_snapshot_harian`, `data_bank_riwayat`, `monitor_saldo` — semuanya BELUM ADA di `arch_db.md` dan harus masuk DDL final sebelum Fase 1 dikunci. Mitigasi termurah: restore `.bak` ke SQL Server Express lalu generate DDL eksak (arch_db.md §7#1).
3. **Kolom jadwal Terjadwal** (`tgl_jadwal`) dan **tabel audit `transaksi_histori`** adalah kolom/tabel BARU (tidak ada di aslinya) yang dibutuhkan agar fitur web setara/tidak-buta.
4. **Keputusan edisi** (EDISI_DAN_BATASAN.md): limit "maks 10 Web SMS Sender", "maks 255 terminal", "maks 30 modul IP" **tidak direplikasi** di clone (tanpa batasan edisi).
5. **SP `DownloadFromMaster`** (sync live→backup) adalah utilitas backup internal, bukan fitur produk — digantikan `pg_dump` terjadwal.
6. Skill `local-deep-research` tidak dipakai para agen: bukti primer (backup DB + changelog resmi + metadata binary) sudah mencukupi dan lebih kuat daripada riset web.
