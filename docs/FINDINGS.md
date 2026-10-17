# REA — OtomaX Free Edition 4.3.1

Tanggal: 2026-10-16 | Sumber: https://www.otomax-software.com/files/Free_Edition.zip
SHA256 zip: 0253faccee793173d30a7d03f5318ee085603dcf88c4e79049ba5b89aba9b31e

## Identitas aplikasi
- OtomaX 4.3.1 (17-Sep-2026), "OtomaX - Orisinil Topup Machine"
- FileDescription: "OtomaX - Fully automatic mobile prepaid recharge software"
- Copyright 2009–2026 Yusuf Arief Rahmanto, www.otomax-software.com
- VB.NET WinForms, .NET Framework 4.x (CLR v4.0.30319), PE32 32-bit, 3 sections (.text/.rsrc/.reloc)
- Assembly version 4.0.0.0; manifest MyApplication.app, asInvoker
- Changelog: 82 versi, 3.3.1 (Jun 2013) → 4.3.1 (Sep 2026)

## Isi paket (7 file, 5.2 MB uncompressed)
| File | Ukuran | Peran |
|---|---|---|
| OtomaX.exe | 1.6 MB | Aplikasi utama (ter-obfuscate) |
| Client.exe | 14.8 KB | Launcher → `OtomaX.exe -client` (tidak obfuscate) |
| SafeMode.exe | 14.8 KB | Launcher → `OtomaX.exe -safemode` (tidak obfuscate) |
| db_otomax.bak | 3.3 MB | Backup database SQL Server (sample) |
| Version.txt / Version_en.txt | 78/47 KB | Changelog ID/EN |
| NotaBill.prt | 338 B | Template struk "BUKTI TRANSFER DEPOSIT" |

## Proteksi → PACKER (koreksi 2026-10-16, temuan agent + verifikasi)
OtomaX.exe BUKAN sekadar obfuscator — ini **packer stub+overlay**:
- Outer = stub loader managed minimal (~10 KB section, sisanya overlay **1.52 MB**,
  entropi 7.72 bits/byte). Metadata outer hampir kosong: 2 TypeDef (`<Module>`,
  `OtomaX._`), 5 MethodDef (`Main` entry 0x06000004, `lzmat`, `lz`, `lf`, `.ctor`),
  2 string (`"File is invalid."`, `"OtomaX"`). Method `lz*` = dekompresi LZMA saat runtime.
- Inner assembly ter-embed di overlay: BSJB metadata root di offset **0x10ACCF**
  (v4.0.30319, stream `#~` terlihat) — tapi isi stream high-entropy (terenkripsi).
- DOS stub diganti protektor (pesan `!It's .NET EXE$`, e_cparhdr=2, header 32-byte).
  Inilah yang mengecoh triase format REA 6.4.0 ("observed dos-mz") — bug upstream
  morluto/rea#2172, SUDAH DIPERBAIKI di rea-agents **6.5.0** (upgrade + verifikasi
  2026-10-16: `inspect-managed-artifact` kini parse penuh → format pe, CLI valid).
- String encryption lapisan terpisah: 249 string format `#=` + custom-base64
  (`$`,`_`,`-`), panjang 2–195 char; symbol renaming jadi gibberish pendek.
- Protektor BELUM teridentifikasi pasti (bukan ConfuserEx menurut bukti empiris);
  tidak ada marker plaintext. Cara menamai: `de4dotEx -d` pada inner setelah unpack.
- Rantai analisis yang benar: unpack inner (dinamis: jalankan stub di Windows →
  dump modul dari memori via dnSpyEx) → de4dotEx (detect/deobfuscate) →
  ilspycmd / REA `inspect-managed-artifact|members` (decompile/inspeksi).

## Model data (dari db_otomax.bak — TIDAK terproteksi)
11 tabel: `cek_stok, data_bank, inbox, mutasi, oto_histori, oto_histori_update,
oto_user, produk, reseller, tiket_deposit, transaksi`

- `transaksi` (27 kolom): kode, tgl_entri, kode_produk, tujuan, kode_reseller,
  pengirim, tipe_pengirim, harga, kode_inbox, status, tgl_status, kode_modul,
  kode_terminal, ket_modul, harga_beli, kode_jawaban, saldo_awal, perintah,
  prioritas, counter, counter2, sn, modul_proses, kirim_ulang, penerima, qty
- `tiket_deposit` (7 kolom): kode, waktu, jumlah, status, tgl_status,
  kode_inbox, kode_data_bank
- `oto_user`: PK = `nomor` (no HP), kolom imsi, serial_number (identitas via SIM)
- Stored procedures: `DownloadFromMaster` (sync inkremental DB live → otomax_bak),
  `AddKomisiTunda(@trxid, @kodereseller, @jumlah)` (komisi tunda / MLM)
- Trigger: `tr_oto_user_data` (set tgl_data=GETDATE() tiap update oto_user)
- Semantik status transaksi: `status<20` = pending/diproses, `status>0` = terproses
- Contoh produk: "VSTID120 Steam Wallet Code", "VSTID250 Steam Wallet Code"
- Setting disimpan sebagai baris: Setting0, SettingMLM, Balasan0, Format0,
  PolaParam0, PolaParam20, ...

## Domain bisnis
Software server pulsa Indonesia: top-up otomatis via modem/SMS (tabel inbox,
kolom pengirim/tipe_pengirim), struktur reseller + downline (MLM, SettingMLM,
AddKomisiTunda), deposit via tiket + data bank (mutasi), voucher fisik.

## Artefak analisis
- /tmp/otomax-inspect.yaml — REA inspect-web-page halaman download (link href)
- /tmp/rea-analyze-otomax.txt — REA analyze (JS analyzer, tidak relevan utk .NET)
- /tmp/rea-artifact-otomax.yaml — REA inspect-artifact OtomaX.exe
- /tmp/rea-managed-otomax-650.yaml — REA 6.5.0 inspect-managed-artifact (parse penuh OK)
- /tmp/rea-members-650.yaml — REA 6.5.0 inspect-managed-members (CIL anchors stub)
- /tmp/otomax-strings.txt — strings ASCII (18.821 baris)
- /tmp/bak-strings.txt — strings db_otomax.bak (5.596 baris, berisi SP/trigger/SQL)
- tools/cli_metadata.py — parser PE/CLI metadata Python murni (~600 baris, tervalidasi 14/14 vs REA di Client.exe)
- tools/otomax-inventory.json — inventaris struktur OtomaX.exe (stub: 2 TypeDef, 5 MethodDef, AssemblyRef mscorlib+WinForms, UnverifiableCodeAttribute)
- tools/client-inventory.json — inventaris Client.exe (referensi validasi)
- CLI_METADATA_INVENTORY.md — laporan inventaris lengkap + root-cause REA (dosMz.js) + draft issue GitHub
- inner-assembly.bin (3.070.464 byte, sha256 a890e36e...) — **HASIL UNPACK SUKSES 2026-10-16**:
  diukir dari overlay via reimplementasi statis stub loader (tanpa eksekusi!):
  `lf` parse PE header sendiri → overlay @0x2610 → `lz` cek magic `M?Z` + baca
  outLen → `lzmat` (emulator IL Python, tools/il_emu.py) dekompresi custom LZ →
  PE valid (3 section, CLI il-only). Inner = aplikasi asli: 1.345 TypeDef,
  10.618 MethodDef, entry `Main`; nama type/method terenkripsi (`#=z...`),
  #US cuma 24 entri. Rantai penuh: parse → unpack → REA, semua $0 di Linux.
