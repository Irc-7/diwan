# ItemStatus — Ordinal Terverifikasi dari Binary

Terverifikasi 2026-10-17 via CLI metadata `inner-assembly.bin` (parser `tools/cli_metadata.py`,
validasi 14/14 atas Client.exe): TypeDef row 1343 = `YAR.OtomaX.ItemStatus` — nama member
**lolos obfuscator sebagai plaintext** ("domain vocabulary" di BEDAH_LOGIKA_BISNIS.md §2).
Field rows 7525–7574 (50 fields = `value__` + 49 member), nilai dari tabel `Constant`
(ELEMENT_TYPE_I4). Tidak perlu decryptor string — ini metadata polos.

Resolves: KONSISTENSI_P0.md temuan N2 (placeholder `<TERJADWAL>` di `arch_db.md`) —
sekarang literal `5` di `chk_transaksi_jadwal`.

| Ordinal | Member | Grup UI (arch_engine.md §4.3) |
|---|---|---|
| 0 | BelumDiproses | Masuk |
| 1 | SedangDiproses | Masuk |
| 2 | MenungguJawaban | Masuk |
| 3 | GagalKirim | Operasional |
| 4 | TidakAdaParsing | Operasional |
| **5** | **Terjadwal** | **Masuk (dieksekusi saat tgl_jadwal tiba)** |
| 19 | Suspect | Operasional |
| 20 | Sukses | Sukses |
| 21 | SuksesMasukOutbox | Sukses |
| 22 | SuksesMasukTransaksi | Sukses |
| 23 | SuksesMasukCS | Sukses |
| 40 | Gagal | Gagal final |
| 41 | BukanReseller | Gagal validasi |
| 42 | FormatSalah | Gagal validasi |
| 43 | SaldoTidakCukup | Gagal validasi |
| 44 | ProdukSalah | Gagal validasi |
| 45 | StokKosong | Gagal validasi |
| 46 | TransaksiDobel | Gagal validasi |
| 47 | ProdukGangguan | Gagal validasi |
| 48 | ParameterSalah | Gagal validasi |
| 49 | PinSalah | Gagal validasi |
| 50 | Dibatalkan | Operasional |
| 51 | ResellerTidakAktif | Operasional |
| 52 | TujuanSalah | Gagal validasi |
| 53 | TujuanDiluarWilayah | Gagal validasi |
| 54 | KodeAreaTidakCocok | Gagal validasi |
| 55 | Timeout | Operasional |
| 56 | NomorBlacklist | Gagal validasi (fraud) |
| 57 | WrongSignature | Gagal validasi |
| 58 | NomorTidakAktif | Gagal validasi |
| 59 | HargaTidakSesuai | Gagal validasi |
| 60 | TidakAdaData | Operasional |
| 61 | QtyTidakSesuai | Gagal validasi |
| 62 | LimitHarian | Operasional |
| 63 | ResellerSuspend | Operasional |
| 64 | Diabaikan | Operasional |
| 65 | UnitTidakCukup | Gagal validasi |
| 66 | InvalidTerminal | Gagal validasi |
| 67 | GroupDissallowed | Gagal validasi |
| 68 | AccessDenied | Gagal validasi |
| 69 | Cutoff | Operasional |
| 70 | HarusUbahPin | Operasional |
| 71 | NomorHangus | Gagal validasi |
| 72 | NomorMasaTenggang | Gagal validasi |
| 73 | LimitKwhBulanan | Operasional |
| 74 | LimitTopupBulanan | Operasional |
| 200 | ProsesUlang | Operasional |
| 201 | Alihkan | Operasional |
| 255 | Unknown | Operasional |

Catatan: rentang 6–18, 24–39, 75–199, 202–254 tidak dipakai (gap di enum asli —
jangan dipakai untuk status baru tanpa cek tabrakan dengan rilis OtomaX lain).
