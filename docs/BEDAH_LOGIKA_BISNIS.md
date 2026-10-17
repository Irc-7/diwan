# Bedah Logika Bisnis OtomaX 4.x (dari metadata inner-assembly.bin)

Sumber: `inner-assembly.bin` (3.070.464 byte, hasil unpack statis 2026-10-16).
Metode: analisis metadata CLI murni (tanpa eksekusi, tanpa dekripsi string).

## 1. Identitas aplikasi

| Fakta | Nilai |
|---|---|
| Assembly | OtomaX, v4.0.0.0, il-only, x86 |
| Namespace asli | `YAR.OtomaX` (3 type lolos enkripsi nama) |
| UI | WinForms, **66 Form** subclass |
| Ukuran | 1.345 TypeDef, 10.618 MethodDef, 7.591 Field, 114 enum |
| Type terbesar | Form utama (509 method), engine inti (362 method), helper kripto (194 method) |

## 2. Domain vocabulary (plaintext, lolos obfuscator)

### `YAR.OtomaX.ItemStatus` — 49 status transaksi
Siklus hidup transaksi server pulsa/PPOB, Bahasa Indonesia:
- **Masuk**: `BelumDiproses` → `SedangDiproses` → `MenungguJawaban` → `Terjadwal`
- **Sukses**: `Sukses`, `SuksesMasukOutbox`, `SuksesMasukTransaksi`, `SuksesMasukCS`
- **Gagal validasi**: `BukanReseller`, `FormatSalah`, `SaldoTidakCukup`, `ProdukSalah`,
  `StokKosong`, `TransaksiDobel`, `ProdukGangguan`, `ParameterSalah`, `PinSalah`,
  `TujuanSalah`, `TujuanDiluarWilayah`, `KodeAreaTidakCocok`, `HargaTidakSesuai`,
  `QtyTidakSesuai`, `NomorBlacklist`, `NomorTidakAktif`, `NomorHangus`,
  `NomorMasaTenggang`, `WrongSignature`, `InvalidTerminal`, `AccessDenied`
- **Operasional**: `GagalKirim`, `TidakAdaParsing`, `Suspect`, `Dibatalkan`,
  `ResellerTidakAktif`, `ResellerSuspend`, `LimitHarian`, `LimitKwhBulanan`
  (token listrik PLN!), `LimitTopupBulanan`, `Timeout`, `Cutoff`, `Diabaikan`,
  `ProsesUlang`, `Alihkan`, `HarusUbahPin`, `TidakAdaData`, `UnitTidakCukup`,
  `GroupDissallowed`, `Unknown`, `Gagal`

### `YAR.OtomaX.TipeModul` — 16 kanal transaksi
Cara aplikasi menembak transaksi ke operator:
`Fisik`, `SMSSender`, `_GsmUssd`, `Dial`, `SMS`, `Keypad`, `IM`, `SMSCenter`,
`STK` (SIM Toolkit), `WebSMSSender`, `IP`, `SpecialHth`, `OH`, `ApiSiDompul`
(API aplikasi dealer Telkomsel!), `ApiAxis` (API operator Axis)

## 3. Tech stack (dari AssemblyRef + TypeRef)

| Lapisan | Teknologi |
|---|---|
| Database | SQL Server via ADO.NET (`SqlConnection/Command/DataAdapter`, `ExecuteReader`, parametrized) |
| Modem fisik | `System.IO.Ports.SerialPort` (kirim/baca AT command) |
| Web automation | **Selenium WebDriver 3.141 + ChromeDriver** (otomasi portal web operator) |
| Browser embedded | **WebView2** (Edge Chromium di dalam WinForms) |
| IM channel | **Telegram.Bot** API + **JabberNet** (XMPP) — modul `IM` |
| HTTP/API | `HttpClient`, `System.Net.Sockets`, JSON via **Newtonsoft.Json** (embedded source) |
| Sistem | WMI (`System.Management`), `GetUserGeoID` (P/Invoke) |
| Kripto | RijndaelManaged + PasswordDeriveBytes, CNG (`ncrypt.dll`), SSPI (`secur32`) |

## 4. P/Invoke (31 fungsi) — kapabilitas native

- **Otomasi jendela**: `FindWindow`, `SendMessage`, `SetForegroundWindow`,
  `ShowWindow`, `FlashWindowEx`, `ReleaseCapture` (kendali aplikasi eksternal)
- **Jaringan**: `DnsQuery_A` (resolusi DNS manual), SSPI `EncryptMessage`/`DecryptMessage`
- **Kripto**: CNG `NCryptEncrypt/ImportKey/OpenStorageProvider`
- **Lokasi**: `GetUserGeoID` — baca region Windows user

## 5. Proteksi string & anti-tamper

- String literal dienkripsi; `#US` heap cuma 24 entri (terenkripsi).
- **Decryptor string** ditemukan: method `#=zjwaRgdg=(int)` → `#=zYVg9mgl7OBbu9JnNCyM7pK0=(int,bool)`,
  baca dari **embedded resource stream**, XOR rolling-key, cache di `Dictionary<int,string>`.
- **Anti-tamper**: decryptor memakai `StackTrace.GetFrame().GetMethod().DeclaringType`
  dan membandingkan dengan TypeDef tertentu — hanya jalan bila dipanggil dari method yang diharapkan.
- 71 manifest resource terenkripsi (472 KB) + 2 blob misterius (`LQtlgGshXDA` 308 KB,
  `834651d190b148bfd9e8dc862f81facb` 27 KB).
- Modul kripto terpisah: `RijndaelManaged` + `PasswordDeriveBytes` (encrypt/decrypt byte[]),
  dipakai untuk payload sensitif (dipanggil dari `.ctor` form utama).

## 6. Arsitektur (inferensi struktural)

```
66 Form (UI: 509-method main form + 65 dialog/laporan)
        ↕
Engine inti (362 method, System.Object)
   ↙      ↓       ↘        ↘
TipeModul  DB layer   IM/Selenium  Kripto (194 method)
(16 kanal) (ADO.NET)  (Telegram/   (Rijndael +
                      XMPP/Chrome)  string decryptor)
```

Alur transaksi (dari ItemStatus): inbox → parsing → validasi
(reseller/saldo/produk/format) → tembak via modul (SMS/USSD/API/Selenium)
→ tunggu jawaban → update status → outbox/CS.

## 7. Batas analisis ini

- Nama type/method/field (kecuali 25 type + 2 enum + lib pihak ketiga) terenkripsi —
  pemetaan modul-ke-nama butuh dekripsi string (butuh emulasi StackTrace check atau eksekusi di Windows).
- SQL query, URL endpoint, AT command, dan config ada di resource/string terenkripsi.
- Langkah lanjut: `de4dotEx -d` di Windows, atau emulasi decryptor dengan mock StackFrame.
