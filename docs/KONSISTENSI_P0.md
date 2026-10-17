# KONSISTENSI_P0.md — Audit Konsistensi arch_db.md × arch_engine.md

**Tanggal:** 2026-10-17 | **Peran:** consistency reviewer | **Metode:** baca penuh kedua file + `docs/FITUR_MENU.md` (sumber kebenaran gap P0), verifikasi silang via grep atas setiap identifier.
**Konvensi label:** [B] terbukti · [I] inferensi · [R] rekonstruksi. Dimensi: (a) referensi silang · (b) kontradiksi · (c) label [B]/[I]/[R] · (d) duplikasi.

---

## Ringkasan

| Tingkat | Jumlah |
|---|---|
| KRITIS | 5 (K1–K5) |
| Non-kritis | 10 (N1–N10) |
| Cek duplikasi (d) | bersih — tidak ada definisi ganda |

---

## Temuan KRITIS

### K1 — (b) `approved` ditaruh di tabel yang salah: `tiket_deposit`, bukti menunjuk `data_bank`

| | |
|---|---|
| **Lokasi** | `arch_db.md:105` (DDL `tiket_deposit.approved`), `arch_db.md:860` (§4 #18: "`tiket_deposit.approved`+`berita_bank`"); `arch_engine.md:424` (§11.2 langkah 1: "`tiket_deposit.approved` menandai persetujuan") |
| **Bukti** | `docs/FITUR_MENU.md:364–365`, keduanya di bawah heading **"### Layar: Data Bank"** (Area G): baris 364 "Daftar mutasi bank 11 kolom, flag `terklaim`, link `kode_tiket`" → rekomendasi "**Tambah kolom `approved`** + index (bank, tgl_bank). **P0**"; baris 365 "Kolom Approved + "Lihat Riwayat Data"" → bukti `Version.txt:75, :77 (4.2.5)` [B] → rekomendasi "`approved BOOLEAN` + tabel audit `data_bank_riwayat`". Tidak satu pun baris di seksi Tiket Deposit menyebut "approved". |
| **Masalah** | DB Specialist menaruh `approved` di `tiket_deposit` murni sebagai [R] ("persetujuan operator sebelum settle") — tebakan lokasi yang **salah menurut bukti**. Engine Specialist lalu mengonsumsinya apa adanya di §11.2. Persetujuan terjadi di modul Data Bank (atas baris mutasi bank), bukan di tiket. |
| **Rekomendasi** | 1. `arch_db.md`: hapus `approved` dari DDL `tiket_deposit` (:105); tambahkan `approved BOOLEAN NOT NULL DEFAULT FALSE -- [B] FITUR_MENU Area G, Version.txt:75,:77` ke DDL `data_bank`. Index `(bank, tgl_bank)` sudah ter-cover oleh prefix `uq_data_bank_idempoten` — catat itu, tak perlu index tambahan. 2. Perbaiki §4 #18 (:860): `tiket_deposit.approved` → `data_bank.approved`. 3. `arch_engine.md:424`: ubah menjadi "`data_bank.approved` menandai persetujuan (aksi di modul Data Bank, tercatat di `data_bank_riwayat`)". |

### K2 — (b) Engine §1 + §5 masih bilang SQL Server; seluruh portingan = PostgreSQL

| | |
|---|---|
| **Lokasi** | `arch_engine.md:45` (diagram: "SQL Server (container)"), `arch_engine.md:296–297` (tabel: DAL "**Dapper** + `Microsoft.Data.SqlClient`"; Database "**SQL Server 2022 di container Linux**"; alternatif ditolak: "PostgreSQL: fase 2 saja") |
| **Bukti** | Seluruh `arch_db.md` adalah rencana porting **PostgreSQL** (JSONB, `CREATE TYPE jenis_mutasi`, TIMESTAMPTZ, CITEXT). Engine sendiri merujuk artefak PG di ≥4 titik: `modul.parameter` **(JSONB)** (§2.1, §4.1, §8.1) dan "**PG** `CREATE SEQUENCE`" (§9.3, baris 393). |
| **Masalah** | Sisa basi dari versi dokumen pra-P0 yang tidak diperbarui Engine Specialist. Jika diikuti, implementasi memakai SqlClient ke SQL Server sementara DDL target adalah PG — skema dan DAL tidak akan pernah cocok (JSONB, enum, `FOR UPDATE` idiom, sequence). |
| **Rekomendasi** | `arch_engine.md`: baris 45 → "PostgreSQL 16/17 (container)"; baris 296 → "**Dapper** + `Npgsql`"; baris 297 → Database "**PostgreSQL 16/17 di container Linux**", alasan: target porting per `arch_db.md` (JSONB, enum native, TIMESTAMPTZ), alternatif ditolak: SQL Server (bobot container + lisensi; T-SQL→PL/pgSQL sudah dikerjakan DB Specialist); tulis ulang baris R7 (bukan lagi "PostgreSQL sebagai fase 2"). |

### K3 — (a) Engine tidak menyebut `kode_dompet` / `reseller_saldo` sama sekali, padahal multi-dompet = keputusan FINAL

| | |
|---|---|
| **Lokasi** | `arch_db.md` §4.1 (keputusan "MULTI-DOMPET PENUH — FINAL, 2026-10-17"): `mutasi.kode_dompet`, `transaksi.kode_dompet`, `reseller_saldo`, `dompet.refund_ke`. Hasil grep: `kode_dompet` = 17 hit di `arch_db.md`, **0 hit** di `arch_engine.md`; `reseller_saldo` = 9 hit di DB, **0** di engine. |
| **Bukti** | `arch_engine.md:384` — helper uang tunggal `UbahSaldo(kode_reseller, delta, jenis_mutasi, …)` **tanpa parameter dompet**; `arch_engine.md:382–383` — locking "atas `reseller.saldo`" saja. |
| **Masalah** | Di bawah multi-dompet penuh, semua path uang (potong transaksi, refund, settle deposit, transfer, tukar komisi) harus mengunci dan mengupdate baris `reseller_saldo(kode_reseller, kode_dompet)` yang benar. Spec §9 saat ini mengunci baris yang salah untuk dompet non-'1' → race condition + saldo tercatat di dompet yang salah. Ini menggerogoti P0 #6 (keamanan uang) tepat di titik yang diputuskan FINAL. Satu-satunya rujukan dompet di engine adalah `dompet.refund_ke` (§9.2) — tidak cukup. |
| **Rekomendasi** | `arch_engine.md` §9: (1) ubah signature menjadi `UbahSaldo(kode_reseller, kode_dompet, delta, jenis_mutasi, …)`; (2) nyatakan locking = `SELECT … FOR UPDATE` atas baris `reseller_saldo` (buat baris via `INSERT … ON CONFLICT DO NOTHING` bila belum ada); (3) `reseller.saldo` ikut diupdate atomik **hanya bila** `kode_dompet='1'` (kompatibilitas, sesuai §4.1 DB); (4) setiap baris `mutasi` wajib mengisi `kode_dompet`; `transaksi.kode_dompet` diisi saat pembuatan; (5) refund menghormati `dompet.refund_ke` ('.' = dompet asal) — sudah ada, pertahankan. |

### K4 — (b) Mitigasi R6 mengutip §2d secara terbalik (kontradiksi internal engine)

| | |
|---|---|
| **Lokasi** | `arch_engine.md:331` (tabel risiko R6): mitigasi "**Pakai Cloud API resmi sejak awal** (keputusan §2d)" |
| **Bukti** | `arch_engine.md` §2d: "**Cloud API resmi ditolak user**"; keputusan user 2026-10-17 = "OTOMASI UNOFFICIAL DIPERTAHANKAN … port setia pendekatan asli (Selenium + WA Web)". |
| **Masalah** | Mitigasi R6 memerintahkan kebalikan dari keputusan user yang eksplisit dan sudah di-hardcode di §2d. Jika diikuti, implementasi melanggar keputusan user. |
| **Rekomendasi** | Perbaiki baris 331 menjadi: "Worker terisolasi + session/profile persistence + monitoring "WA Center tidak bisa mengirim" + fallback notifikasi via Telegram (sesuai §2d); Cloud API resmi tetap ditolak user." |

### K5 — (a/b) Engine klaim [B] "kolom produk" untuk Qty Yang Diizinkan; kolomnya tidak ada di DB

| | |
|---|---|
| **Lokasi** | `arch_engine.md:266`: "**Qty Yang Diizinkan** (kolom produk, pengganti Min/Max Qty) [B]:" + aturan evaluasi baris 267. `arch_db.md` tabel `produk`: kolom ini **tidak ada** (grep `qty_diizinkan` = 0 di kedua file). |
| **Bukti** | `docs/FITUR_MENU.md` Area C: fitur "Qty Yang Diizinkan" [B] (Version.txt:847,:342) — tetapi **tidak pernah menamai kolomnya**; 14 kolom grid produk di SCHEMA_DEEPDIVE tidak memuatnya (kemungkinan salah satu dari 2 kolom hidden 13–14). Jadi klaim "[B]" atas keberadaan *kolom* tidak jujur — yang [B] hanya perilakunya. |
| **Masalah** | P0 #3 (mekanika harga) tidak bisa diimplementasikan sesuai spec: engine memerintahkan "qty dipaksa 1 bila produk TIDAK punya isian ini" tetapi tidak ada kolom yang bisa dibaca. |
| **Rekomendasi** | 1. `arch_db.md`: tambahkan `qty_diizinkan TEXT -- [R] daftar qty diizinkan (mis. '1,2,3'); NULL = bebas` ke DDL `produk` (atau dokumentasikan sebagai salah satu `kolom_hidden_13/14` bila spike porting UI mengonfirmasi). 2. `arch_engine.md:266`: perbaiki label menjadi "[B perilaku (Version.txt 4.2.9); R nama kolom]". |

---

## Temuan non-kritis

| ID | Dim | Lokasi | Deskripsi | Rekomendasi |
|---|---|---|---|---|
| N1 | (a) | `arch_engine.md:393` vs `arch_db.md` §2.1/§5.1 | Engine memutuskan "SEQUENCE database (PG `CREATE SEQUENCE`)" untuk angka unik tiket; DDL DB hanya punya `angka_unik INTEGER` tanpa `CREATE SEQUENCE` dan tanpa `DEFAULT nextval(...)`. Konsep konsisten (sequence + unique backstop `uq_tiket_deposit_angka_unik` = OK, bukan kontradiksi), tetapi artefak DDL-nya belum ada. | `arch_db.md`: tambahkan `CREATE SEQUENCE IF NOT EXISTS seq_tiket_angka_unik;` + `ALTER TABLE tiket_deposit ALTER COLUMN angka_unik SET DEFAULT nextval('seq_tiket_angka_unik');` di §2.1/§5.1; catat `setval()` pasca bulk-load bila seed manual. |
| N2 | (a) | `arch_engine.md:273` vs `arch_db.md` §2.1 | Engine mensyaratkan "constraint CHECK: `status=Terjadwal` ⇒ `tgl_jadwal NOT NULL`" [R]; DDL `transaksi` di DB tidak memuat CHECK tersebut. | `arch_db.md`: tambahkan `CONSTRAINT chk_transaksi_jadwal CHECK (status <> <ordinal_Terjadwal> OR tgl_jadwal IS NOT NULL)` — ordinal diambil dari enum ItemStatus (jangan hardcode tanpa komentar). |
| N3 | (c) | `arch_engine.md:376` (§8.3), `:433` (§12) | Catatan basi "[R tabel — sedang ditambah DB specialist]" — kedua tabel (`transaksi_histori`, `pengguna`/`peran`/`peran_menu`) **sudah ada** di `arch_db.md` §2.7/§2.8. | Hapus frasa "sedang ditambah DB specialist"; ganti rujukan menjadi "`arch_db.md` §2.7/§2.8". |
| N4 | (b) | `arch_db.md` §5.2 vs `arch_engine.md` §12 | Seed peran tidak sinkron: DB seed `ADMIN` + `OPERATOR` (uppercase, 2 peran); engine §12 menetapkan peran bawaan `admin`, `operator`, `teknisi`, `kasir-kiosk` (lowercase, 4 peran). | Samakan: pilih satu konvensi casing kode peran; seed keempat peran di `arch_db.md` §5.2 (dengan `peran_menu` minimal per peran). |
| N5 | (a) | `arch_db.md` §2.7/§2.8 vs `arch_engine.md` (nihil) | Tabel DB tanpa spec engine (nama tabel tak pernah muncul di engine): `grup_reseller`, `harga_grup`, `markup_produk` (resolusi harga §4.2 hanya menyebut "markup per-reseller" tanpa nama tabel), `voucher` (aturan alokasi FIFO + `FOR UPDATE` + state machine USED di DB §2.7 tanpa pasangan engine), `auto_respon` (+ urutan evaluasi `AutoResponFirst`), `data_bank_riwayat`, `mutasi_poin`, `mutasi_unit`, `hadiah_poin`, `monitor_saldo`, `saldo_snapshot_harian`. Sebagian memang P1 — tetapi engine harus menyebutnya agar tak hilang. | `arch_engine.md`: tambahkan rujukan eksplisit per tabel (satu paragraf di §4.2 untuk harga_grup/markup_produk/grup_reseller; sub-bab voucher di fase 2; catat auto_respon/AutoResponFirst di pipeline §4; daftarkan tabel P1 di roadmap §7). |
| N6 | (b/c) | `arch_db.md:105`, `:860`; `arch_engine.md:424` | `tiket_deposit.berita_bank` [R] ("berita yang cocok saat auto-claim") menduplikasi `reseller.berita_bank` [B] — kolom yang menurut bukti dipakai engine untuk pencocokan pola. Statusnya cache terdenormalisasi yang belum diputuskan. | Putuskan satu: (opsi A) pertahankan sebagai cache + dokumentasikan "ditulis engine saat auto-claim cocok, sumber kebenaran tetap `reseller.berita_bank`"; (opsi B) hapus kolom, engine baca via join `reseller` / catat pola di `data_bank.catatan`. |
| N7 | (c) | `arch_db.md` §2.8; `arch_engine.md:376`, `:347` | `transaksi_histori` = **P1** di FITUR_MENU (Area B: "Timeline/riwayat perubahan status … **P1**"), tetapi kedua file memperlakukannya sebagai MVP (DB: "Engine tulis tiap transisi status"; engine §7 langkah 3 memasukkannya di kriteria selesai). | Selaraskan: catat keputusan sadar (naikkan ke P0 karena audit operasional, atau turunkan ke P1 di kedua file). |
| N8 | (a) | `arch_db.md` §2.2/§2.3 vs `docs/FITUR_MENU.md` | Kolom berlabel [B] di FITUR_MENU yang belum ada di DDL: `reseller.suspend` (**P0** — "terpisah dari `aktif`"), `reseller.harus_ubah_pin` (**P0**), `produk.harga_awal` (P1, Version.txt:590), `produk.postpaid` (P1, Version.txt:1124), `reseller.total_markup` (P1), `reseller.kode_referral`/`alamat_ip`/`password_ip`/`report_url` (P1/P2). Engine pun belum merujuknya (konsisten antar-file, tetapi gap terhadap sumber kebenaran). | `arch_db.md`: tambahkan minimal kolom P0 (`suspend BOOLEAN DEFAULT FALSE`, `harus_ubah_pin BOOLEAN DEFAULT FALSE` di `reseller`); sisanya antre P1. `arch_engine.md`: tambahkan cek "suspend dulu, lalu aktif" di validasi + penolakan `HarusUbahPin`. |
| N9 | (a) | `arch_db.md` §2.1; `arch_engine.md` §11.2 | `tiket_deposit` tidak punya `kode_dompet`/`kode_reseller` (7 kolom asli [B] memang tidak punya) — di bawah multi-dompet FINAL, belum ditentukan dompet mana yang dikredit saat settle. | `arch_engine.md` §11.2: tentukan aturan (mis. default dompet `'1'` kecuali ada mapping bank→dompet di `setting`/worker config Internet Banking). |
| N10 | (a) | `arch_engine.md` §11.1 | Jalur komisi langsung (`SettingTundaKomisi=0`: "tambah `mutasi` … + `reseller.saldo`") tidak menyebut `kode_dompet` pada baris mutasi. | Liput dalam perbaikan K3 (semua path uang lewat `UbahSaldo` ber-dompet). |

### Hasil cek (d) duplikasi
Tidak ditemukan definisi ganda: setiap tabel (`transaksi`, `tiket_deposit`, `mutasi`, `data_bank`, `dompet`, `reseller_saldo`, `transaksi_histori`, `data_bank_riwayat`, dsb.) didefinisikan tepat sekali di `arch_db.md`; tidak ada section yang diduplikasi di `arch_engine.md`. Katalog placeholder §4.1 tidak memuat entri ganda (`[saldo]`/`[saldoawal]`/`[saldolama]` berbeda makna dan terdokumentasi).

---

## Catatan verifikasi khusus (dari tugas)

1. **Lokasi `approved`**: diputuskan **pindah ke `data_bank`** (K1). Bukti `FITUR_MENU.md:364–365` (Area G → Layar: Data Bank, [B] Version.txt:75,:77) tidak ambigu — konteksnya modul Data Bank, bukan Tiket Deposit. Penempatan di `tiket_deposit` adalah [R] yang salah tebak.
2. **SEQUENCE vs unique constraint**: **konsisten, bukan kontradiksi**. Engine §9.3 (SEQUENCE sebagai generator thread-safe) + DB `uq_tiket_deposit_angka_unik` (backstop: klaim paralel berangka sama DITOLAK → retry) adalah desain berlapis yang benar. Yang kurang hanya DDL `CREATE SEQUENCE`-nya di `arch_db.md` (N1).
