# OtomaX 4.3.1 — Deep-dive Skema Database

Tanggal: 2026-10-16 | Sumber: `db_otomax.bak` (dari Free_Edition.zip, sha256 `0253faccee…`)
Metode: analisis strings atas file .bak + source lengkap 2 stored procedure, 2 view,
1 trigger, dan definisi kolom DataGrid UI yang ikut ter-embed di backup.
**Tidak ada satu pun CREATE TABLE di backup** — tipe data & constraint tidak bisa dipastikan;
daftar kolom di bawah berasal dari SELECT/INSERT eksplisit di SP dan definisi grid UI,
jadi urutan/nama kolomnya valid, tipenya tidak.

Koreksi atas FINDINGS.md: bukan 11 tabel — **18 objek** teridentifikasi
(14 tabel + 2 view + 2 tabel yang dirujuk kode tapi tidak ada di backup).

---

## 1. Inventaris objek

| # | Objek | Jenis | Kolom (dari bukti) |
|---|-------|-------|--------------------|
| 1 | `transaksi` | tabel, `kode` = IDENTITY | kode, tgl_entri, kode_produk, tujuan, kode_reseller, pengirim, tipe_pengirim, harga, kode_inbox, status, tgl_status, kode_modul, kode_terminal, ket_modul, harga_beli, kode_jawaban, saldo_awal, perintah, prioritas, counter, counter2, sn, modul_proses, kirim_ulang, penerima, qty (27) |
| 2 | `tiket_deposit` | tabel, `kode` = IDENTITY | kode, waktu, jumlah, status, tgl_status, kode_inbox, kode_data_bank (7) |
| 3 | `mutasi` | tabel, `kode` = IDENTITY | kode, kode_reseller, tanggal, jumlah, keterangan, kode_reseller_2, jenis, kode_transaksi, kode_outbox, saldo_akhir (10) |
| 4 | `data_bank` | tabel, `kode` = IDENTITY | kode, bank, tgl_proses, tgl_bank, jumlah, tipe, saldo, keterangan, kode_tiket, terklaim, catatan (11) |
| 5 | `oto_user` | tabel, PK = `nomor` | nomor, kode_produk, nama_produk, imsi, serial_number, tgl_status, tgl_data (+kolom lain tak terpetakan) |
| 6 | `reseller` | tabel | kode, saldo, tgl_aktivitas (+kolom lain tak terpetakan) |
| 7 | `produk` | tabel (definisi grid `admindgProduk`) | aktif, kode, nama, kode_operator, fisik, tanpa_kode, nominal, harga_beli, harga_jual, kosong, gangguan, stok, qty, prefix_tujuan (+2 kolom hidden di index 13–14) |
| 8 | `inbox` | tabel | kode, pesan, kode_reseller, pengirim, tipe_pengirim (+kolom lain tak terpetakan) |
| 9 | `oto_histori` | tabel | tgl_proses (+kolom lain tak terpetakan) |
| 10 | `oto_histori_update` | tabel | tgl_update (+kolom lain tak terpetakan) |
| 11 | `hlr` | tabel (grid `admindgHLR`) | kode, prefix_tujuan |
| 12 | `level` | tabel (grid `admindgLevel`) | kode, nama, kode_upline, selisih_harga, default_markup, no_ubah_markup, deposit_minimal, deposit_maksimal, jumlah_ym, jumlah_sms, blok_produk, keterangan |
| 13 | `parsing` | tabel (grid `admindgParsing`) | aktif, kode_modul, kode_produk, prioritas, harga_beli, _harga_jual, perintah, keterangan |
| 14 | `provider` | tabel (grid `admindgProvider`) | kode, nama, kosong, gangguan, prioritas, prefix_tujuan, kode_area, kode_hlr, panjang_min, panjang_maks, kode_level, struk_auto_cetak |
| 15 | `cek_stok` | **VIEW** (bukan tabel) | `t.kode, t.tgl_entri, t.kode_produk, t.tujuan, i.pesan FROM transaksi t LEFT JOIN inbox i ON i.kode=t.kode_jawaban JOIN produk p ON p.kode=t.kode_produk WHERE t.kode BETWEEN 445863 AND 448361 AND t.kode_modul=81 AND t.status=20 AND p.nominal=50` |
| 16 | `harga_agen` | **VIEW** (bukan tabel) | `o.nama, p.kode, p.harga_jual+200 FROM produk p JOIN operator o ON o.kode=p.kode_operator WHERE p.aktif=1 …` |
| 17 | `komisi` | tabel — **dirujuk `AddKomisiTunda`, tidak ada di backup** | kode_transaksi, kode_reseller, jumlah |
| 18 | `operator` | tabel — **dirujuk view `harga_agen`, tidak ada di backup** | kode, nama |

Tabel yang *diimplikasikan* tapi tak muncul sama sekali: `modul` (dirujuk `parsing.kode_modul`,
`transaksi.kode_modul`), `terminal` (`transaksi.kode_terminal`), `outbox`
(`mutasi.kode_outbox`).

---

## 2. Peta relasi

```
reseller ──1:N── transaksi      (transaksi.kode_reseller)
reseller ──1:N── mutasi         (mutasi.kode_reseller; mutasi.kode_reseller_2 = reseller tujuan transfer)
reseller ──1:N── inbox          (inbox.kode_reseller; ada cleanup: delete inbox yg reseller-nya tak ada)
reseller ──1:N── komisi         (komisi.kode_reseller)
reseller ──1:N── tiket_deposit  (via inbox: tiket_deposit.kode_inbox → inbox.kode)

produk ──1:N── transaksi        (transaksi.kode_produk)
produk ──N:1── operator         (produk.kode_operator)
produk ──1:N── parsing          (parsing.kode_produk; parsing = aturan routing modul→produk)

inbox ──1:N── transaksi        (transaksi.kode_inbox = SMS asal; transaksi.kode_jawaban → inbox.kode = SMS balasan)
inbox ──1:N── tiket_deposit    (tiket_deposit.kode_inbox)

data_bank ──1:N── tiket_deposit (tiket_deposit.kode_data_bank)
tiket_deposit ──1:1── data_bank (data_bank.kode_tiket = link balik; terklaim = flag klaim)

transaksi ──1:N── mutasi        (mutasi.kode_transaksi)
transaksi ──1:N── komisi        (komisi.kode_transaksi)

provider ──N:1── hlr            (provider.kode_hlr)
provider ──N:1── level          (provider.kode_level)
level ──1:N── level             (level.kode_upline = self-reference, hierarki MLM)

oto_user                        (standalone; identitas via nomor HP + IMSI + serial_number)
oto_histori / oto_histori_update (log; kolom waktu tgl_proses / tgl_update)
```

---

## 3. Logika bisnis dari kode

**`DownloadFromMaster`** — sync inkremental DB live → `otomax_bak`:
- `reseller`: update saldo + tgl_aktivitas yang berubah, insert kode baru
- `oto_user`: update (kode_produk, nama_produk, imsi, serial_number, tgl_status) by `nomor`, insert nomor baru
- `oto_histori` / `oto_histori_update`: insert by watermark waktu (max tgl_proses / tgl_update)
- `transaksi`: update status/kode_modul/tgl_status/kode_jawaban/sn untuk kode yang di backup masih `status<20` tapi di live sudah `status>0`; insert kode > max(kode) — **`kode` IDENTITY**
- `tiket_deposit`: update status untuk tiket `waktu > GETDATE()-4` yang di backup `status='O'` tapi di live sudah berubah; insert baru — status `'O'` = Open
- `mutasi`, `data_bank`: insert by max(kode)

**`AddKomisiTunda(@trxid, @kodereseller, @jumlah)`** — upsert ke `komisi` (MLM komisi tunda).
Setting `SettingTundaKomisi=1` → komisi tidak langsung masuk deposit.
Format SMS: `TUKAR.[jumlah].[pin]` / `TUKAR` = tukar komisi tunda jadi deposit.

**Trigger `tr_oto_user_data`** — tiap update `oto_user`, set `tgl_data=GETDATE()` where nomor cocok.

**View `cek_stok`** — query operasional stok: transaksi sukses (`status=20`) produk nominal 50 via modul 81, join pesan balasan inbox. Hardcoded range kode 445863–448361 (contoh di sample DB).

**View `harga_agen`** — daftar harga agen: `harga_jual+200` per produk aktif, exclude pola kode tertentu (S50P, %H, %F, FT%, FK%, HF%, Z%, M%, IN%, A[^X]%).

**Cleanup** (di akhir SP): `delete from mutasi where kode_reseller in (select kode from #rsmati)` dan `delete from inbox where kode_reseller not in (select kode from reseller)` — ada konsep reseller mati (`#rsmati` temp table).

---

## 4. Sampel data (isi sample DB)

**Produk** (kode → nama): `VSTID45/60/90/120/250/400/600` Steam Wallet Code ·
`XD3/6/8/800`, `XD15` XL HotRod 24Jam · `XDC6/12/18/30/42` XTRA Combo ·
`SBU3/4/5/25/30/40/50` by.U · `VT1` tri web · `X5/10/15/25/50/100` Axis ·
`PLN20/50/100` token listrik · `SM5/10/25`, `I5/10/25`, `T5/10/20` pulsa.

**Reseller**: ID pola `OR####` (OR0001 OriPulsa, OR0230 DANI, OR0301 KURNIA, OR0574, OR0665, OR0744, OR0759, OR0772 RAFJAYACELL, OR0965, OR1046, OR1150…).

**SMS inbox** (lalu lintas asli): pola `S10.0812… SUKSES. SN… @waktu`, info gangguan
("BPJS Gangguan", "XL Gangguan.. alihkan ke center lain", "PLN OFF 23.30 s/d 00.20"),
update saldo ("-1.849.025", "Saldo -48.875"), "Ketik TUKAR utk ambil komisi".

**Setting rows**: Setting0, SettingMLM, Balasan0, Format0, PolaParam0, PolaParam20,
`BalasanFormatPolaParam2Setting` — konfigurasi disimpan sebagai baris, bukan tabel khusus.

---

## 5. Catatan metode & batas

- Semua di atas dari `/tmp/bak-strings.txt` (strings ASCII atas .bak);
  nomor baris acuan: SP `DownloadFromMaster` ~140–231 & 4498–4545,
  `AddKomisiTunda` 4564–4566, view 4568–4577, trigger 4419+, grid defs ~5246,
  kolom transaksi 218–220, mutasi 4531–4535, data_bank 4538–4542.
- Tanpa CREATE TABLE: tipe data, PK (selain yang disebut di kode), FK constraint,
  dan index tidak diketahui. Relasi di §2 adalah inferensi dari JOIN/kolom —
  valid secara semantik, bukan constraint terverifikasi.
- `komisi` & `operator` dirujuk kode tapi tak ada sebagai objek `dbo.*` di backup:
  kemungkinan tabel kosong di sample DB (jejak strings minim) atau hanya ada di DB live.
- **Jalur deobfuscation** (opsi B kemarin): obfuscator tak teridentifikasi
  (tanpa marker ConfuserEx/d sejenis; string encryption custom `#=` + base64
  alfabet `$ _ -`). Deobfuscation otomatis model de4dot butuh kenali dulu
  protektornya — praktisnya butuh dnSpy + analisis dinamis di Windows.
  Jalur ini tidak dikerjakan di sesi ini.

## 6. File terkait
- `FINDINGS.md` — laporan RE awal (identitas app, proteksi exe, ringkasan 11 tabel)
- `extracted/` — isi Free_Edition.zip (exe + db_otomax.bak)
- Artefak mentah: `/tmp/bak-strings.txt`, `/tmp/otomax-strings.txt`,
  `/tmp/otomax-inspect.yaml`, `/tmp/rea-artifact-otomax.yaml`
