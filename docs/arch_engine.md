# Desain Engine Service .NET 8 — Clone Linux OtomaX 4.3.1

**Tanggal:** 2026-10-17 | **Penulis:** arsitek backend (subagent) | **Metode:** forge, evidence-based dari hasil teardown.

**Keputusan desain yang mengikat (dari EDISI_DAN_BATASAN.md):**
- TANPA batasan edisi — setara Ultimate, semua fitur terbuka, tanpa lisensi.
- Arsitektur Opsi B: engine = service .NET 8 (systemd), web UI terpisah via API.

**Sumber evidence:**
- `FINDINGS.md` — 1.345 TypeDef / 10.618 MethodDef, ItemStatus 49 status, TipeModul 15 channel, skema DB 11 tabel + SP + trigger.
- `TARGETED_DECRYPTION.md` — 28 token (AT command, URL, SQL); arsitektur modem: **1 method** `SerialPort.Write` [2458], 3 builder, expected-responses, regex.
- `LAPORAN_GAP_RISET.md` — Opsi B, komponen OSS (gammu/SMSTools3/Kannel), vonis functional clone feasible.
- `EDISI_DAN_BATASAN.md` — kuota terminal per edisi, fitur Ultimate-only, modul standar.

---

## 1. Struktur service

Satu binary .NET 8 Worker Service (`OtomaX.Engine`), dijalankan sebagai systemd unit. Di dalamnya, domain dipisahkan sebagai `BackgroundService` yang berkomunikasi via `Channel<T>` (in-memory, bounded) — bukan panggilan langsung antar-domain, supaya satu domain macet tidak menyeret yang lain.

```
┌─────────────────────────────────────────────────────────────┐
│                    OtomaX.Engine (systemd)                   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ IngressHub   │  │ TxnEngine    │  │  ModemManager     │   │
│  │ (pintu masuk │  │ (orkestrasi  │  │  (1 worker per    │   │
│  │  semua kanal)│─▶│  transaksi)  │─▶│   terminal/modem) │   │
│  └──────────────┘  └──────┬───────┘  └───────────────────┘   │
│         ▲                 │                                  │
│         │                 ▼                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ H2H / API    │  │ ReplyDispatch│  │ SchedulerService  │   │
│  │ Clients      │  │ (balasan ke  │  │ (jadwal, retry,   │   │
│  │ (HTTP, WA,   │◀─│  reseller)   │  │  housekeeping)    │   │
│  │  Telegram..) │  └──────────────┘  └───────────────────┘   │
│  └──────────────┘                                            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ApiHost (ASP.NET Core minimal API + SignalR)          │   │
│  │ → kontrak JSON untuk web UI (monitoring, CRUD)        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
        │                        │
   PostgreSQL 16/17 (container)   /dev/ttyUSB* (modem)
```

**Alasan pemisahan ini:** hasil teardown menunjukkan alur asli memang sudah terpisah secara logis — inbox → parsing → routing modul → eksekusi → balasan — dengan 485 method mencapai DAL secara transitif. Channel-based worker meniru pola "sistem antrian canggih, semua modul simultan dengan prioritas" dari aslinya, tanpa thread manual.

**Kontrak antar-domain (ringkas):**
- `IngressHub → TxnEngine`: `InboxMessage { Kode, Pengirim, TipePengirim, Isi, Waktu, Sumber }` (cermin tabel `inbox`).
- `TxnEngine → ModemManager / H2H`: `ModuleCommand { KodeTerminal, TipeModul, Perintah, Timeout, ExpectedResponses[] }`.
- `* → TxnEngine`: `ModuleResult { Sukses, Response, Sn, ErrorCode }`.
- `TxnEngine → ReplyDispatch`: `Reply { Tujuan, TipeTujuan, Isi, Prioritas }`.

---

## 2. Mapping 15 channel TipeModul → implementasi Linux

Enum asli (dari metadata, terverifikasi): `Fisik, SMSSender, _GsmUssd, Dial, SMS, Keypad, IM, SMSCenter, STK, WebSMSSender, IP, SpecialHth, OH, ApiSiDompul, ApiAxis`.

| # | TipeModul | Fungsi asli | Implementasi Linux | Catatan |
|---|---|---|---|---|
| 1 | SMSCenter | Terima SMS dari anggota via modem | `System.IO.Ports.SerialPort` + AT+CMGL/+CNMI → inbox | Inti; lihat §3 |
| 2 | SMSSender | Kirim SMS status/info ke anggota | SerialPort + AT+CMGS (PDU/text) | Bisa sharing 1 modem untuk beberapa sender (evidence: maxsi.id) |
| 3 | _GsmUssd | USSD (M-Kios dkk.) | SerialPort + AT+CUSD | Perlu session handling USSD (`+CUSD` unsolicited) |
| 4 | Dial | USSD/dial voice (FKios) | SerialPort + ATD / AT+CUSD | — |
| 5 | SMS | Isi pulsa via SMS (CDMA/GSM, H2H SMS) | SerialPort + AT+CMGS | Pola sama dengan SMSSender, beda routing |
| 6 | Keypad | Input via keypad (emulasi keypress modem) | SerialPort + AT+CKPD | Jarang dipakai; prioritas rendah |
| 7 | STK | SIM Tool Kit (M-Tronik/Axis) | SerialPort + AT+STK* (proaktif SIM) | Perlu parsing proactive command; moderate |
| 8 | Fisik | Voucher fisik | Bukan modem — modul stok/admin + cetak | Lihat "tidak diporting"/fase 2 |
| 9 | IM | Jabber/XMPP (GTalk) | **Perhatian khusus** — lihat bawah | Library .NET XMPP tidak matang |
| 10 | WebSMSSender | Kirim SMS via HTTP GET ke SMS gateway | `HttpClient` GET | Ultimate-only di aslinya; di clone selalu aktif |
| 11 | IP | IP Center/Client (H2H TCP) | `System.Net.Sockets` TcpListener/TcpClient | Protokol IP Center perlu di-sniff |
| 12 | SpecialHth | Host-to-Host khusus | `HttpClient` (per-supplier) | Pola: 3 API client teridentifikasi di teardown |
| 13 | OH | (kemungkinan "Other Host"/modul khusus) | `HttpClient` generik | Nama tidak jelas; buat generik-pluggable |
| 14 | ApiSiDompul | Otomasi API SiDompul | `HttpClient` | **Perhatian khusus** — lihat bawah |
| 15 | ApiAxis | Otomasi API Axis | `HttpClient` | **Perhatian khusus** — lihat bawah |

### 2.1 Pemetaan "modul menu UI" → TipeModul enum → worker engine

**Koreksi penting [B]:** "Internet Banking", "WA Center", "SMS Order/H2H", "IM Hybrid HTH" yang muncul di menu UI **BUKAN** nilai `TipeModul` — melainkan entri menu yang dipetakan ke worker engine. Enum `TipeModul` tetap 15 nilai persis (§2). Tabel pemetaan wajib ini dipakai wizard "Tambah Modul" agar tidak ambigu:

| Menu UI | TipeModul | Worker/Channel | Catatan |
|---|---|---|---|
| Internet Banking | — (bukan enum) | Worker Selenium per bank (Chrome for Testing headless, 1 browser context per bank) | [B] Kredensial per bank di vault; polling default 300 dtk; "Hanya cek jika ada tiket open" [B]; fallback input mutasi manual via web UI (R1). |
| WA Center | — (bukan enum) | Worker Selenium WA Web (unofficial) | [keputusan user 2026-10-17] Port setia pendekatan asli; risiko ban nomor + breakage selector; session/profile persistence; fallback notifikasi via Telegram. Lihat §2d. |
| SMS Order / H2H | — (bukan enum) | Worker `IP` (TCP H2H) atau `SpecialHth`/`OH` (HTTP), per konfigurasi | [I] H2H antar-server OtomaX = `OH` (OtomaX HTH); pemetaan pasti menunggu verifikasi binary saat spike protokol IP Center (R3). |
| IM Hybrid HTH | — (bukan enum) | Sidecar IM (§2c) + routing pesan H2H via `OH` | [I] Hybrid = kanal IM yang membawa pesan H2H; detail protokol menunggu verifikasi. |

Parameter per-tipe-modul disimpan di kolom `modul.parameter` (JSONB) — wizard "Tambah Modul" me-render field per tipe dari skema parameter ini [R skema; B kebutuhan dari FITUR_MENU.md gap #1].

### Yang butuh perhatian khusus

**a) Internet Banking (modul deposit otomatis).** Aslinya: Selenium + ChromeDriver + WebView2/IE11 mengikis mutasi BCA/BNI/BRI/Mandiri. Di Linux: **Selenium WebDriver + Chrome for Testing headless**. Ini komponen paling rapuh — bank aktif memblokir otomasi (bot detection, OTP, perubahan DOM). Mitigasi: isolasi sebagai worker terpisah (satu browser context per bank), retry + antrian fallback manual (operator input mutasi via web UI), dan pola polling interval sesuai setting asli (default 300 detik).

**b) ApiSiDompul / ApiAxis ("API APK").** Nama menipu: evidence ekosistem (channel Telegram addon) menunjukkan ini **HTTP API** (ada dashboard, "API MyTelkomsel", "API DIGIPOSAJA") — bukan otomasi UI Android. Implementasi = `HttpClient` + Polly. Fallback jika ternyata butuh UI Android asli: Appium + emulator — tandai risiko tinggi, tunda ke fase 2, dan JANGAN jadikan blocker fase 1.

**c) IM / XMPP (Jabber).** Library XMPP .NET yang maintained praktis tidak ada (SharpXMPP mati). Opsi: (1) definisikan `IImProvider` dan implementasikan via sidecar kecil (Python + slixmpp) yang bicara ke engine via localhost HTTP/gRPC; (2) evaluasi ulang kebutuhan — GTalk sudah mati, XMPP publik menyusut; Jabber di sini mostly untuk H2H antar-server. Rekomendasi: interface + sidecar, bukan port langsung.

**d) WA Center — OTOMASI UNOFFICIAL DIPERTAHANKAN (keputusan user 2026-10-17).** Aslinya mengotomasi WA Web (evidence: "Balasan WA salah kamar ke grup", "WA Center tidak bisa mengirim"). Di Linux: **Selenium + Chrome for Testing mengotomasi WhatsApp Web** — port setia dari pendekatan asli, satu stack dengan modul Internet Banking (§a). Risiko jujur: WhatsApp aktif mematahkan otomasi unofficial (selector berubah, anti-bot) dan ada risiko banned nomor; mitigasi: worker terisolasi, session/profile persistence, monitoring "WA Center tidak bisa mengirim", fallback notifikasi via Telegram. Cloud API resmi ditolak user.

**e) Telegram.** `Telegram.Bot` (native .NET, jalan di Linux) — straightforward, risiko rendah. Evidence: `ApiRequestException`, `ParseMode` di metadata.

**f) WebView2.** Windows-only — **tidak diporting**. Digantikan headless Chromium via Selenium (butir a). WebView2 lenyap dari persamaan, sesuai vonis Gap 2.

---

## 3. Lapisan AT command

Berdasarkan temuan teardown — arsitektur asli yang direkonstruksi:

- **Satu-satunya** pemanggil `SerialPort.Write` adalah method [2458] dengan signature:
  `bool Kirim(string cmd, ref string response, string[] expected, int timeout, bool flag)`
- Alur: `Write(cmd)` → baca via `ReadExisting()` (fast path) / `Read(buf,0,len)` (buffered) → cocokkan `expected[]`.
- 3 builder: [2450] `Concat(prefix, angka, suffix)` (pola `AT+CMGS="<no>"` + PDU), [2468] `Concat(prefix, variabel, suffix)`, [2473] `String.Format(template, arg)` (pola `AT+CMGF={0}`).
- Expected responses dari `.ctor` [2424]: kemungkinan `"OK"`, `"ERROR"`, `"> "`, `"+CMGS:"`, `"+CMTI:"` — standar GSM 07.05/07.07.
- 1 regex untuk parsing respons modem.

### Desain: `AtChannel` per terminal

Satu instance per modem (`/dev/ttyUSB*`), dengan command queue dan **satu command aktif dalam satu waktu** (modem GSM pada dasarnya half-duplex untuk AT):

```
┌──────────────────────────────────────────────────────────┐
│ AtChannel (per /dev/ttyUSBx)                              │
│                                                           │
│  Channel<AtRequest> ──▶  lock(port)                        │
│  AtRequest { Cmd, Expected[], Timeout,                     │
│              ResponseRegex?, AllowUnsolicited }            │
│                                                           │
│  Write(cmd + "\r")                                        │
│  loop sampai deadline:                                     │
│    chunk = ReadExisting() / Read()  (non-blocking poll)   │
│    buffer += chunk                                         │
│    if buffer contains any Expected[] → OK(buffer)          │
│    if ResponseRegex match → tangkap grup (mis. +CMGS: id)  │
│    route unsolicited (+CMTI, RING, +CUSD) → IngressHub     │
│    timeout → FAIL(buffer)                                  │
│  unlock(port)                                             │
└──────────────────────────────────────────────────────────┘
```

**Aturan penting (dari perilaku asli):**
1. **Serialisasi penuh per port** — jangan pernah interleave dua command; respons modem tidak ber-tag.
2. **Unsolicited result codes** (`+CMTI:` SMS masuk, `RING`, `+CUSD:`) TIDAK boleh dimakan oleh penunggu command — route ke `IngressHub` sebagai inbox.
3. **Mode prompt `"> "`** untuk SMS: `AT+CMGS="<tujuan>"` → tunggu `"> "` → kirim PDU/body → akhiri `0x1A` (Ctrl-Z) → tunggu `+CMGS:`/`OK`. Ini dua-tahap; implementasikan sebagai satu `AtRequest` dengan `PromptPhase`.
4. **Health check**: `AT` ping periodik; 3x gagal → tandai terminal down → trigger auto-reconnect (tutup/buka port, re-init `ATE0`, `AT+CMGF`, `AT+CNMI`).
5. **Linux-spesifik**: udev rule untuk nama stabil (`/dev/otomax-modem-N` via `ATTRS{idVendor/idProduct}` + serial), user service di grup `dialout`, reconnect otomatis saat USB dicabut (device hilang → `IOException` → backoff).

### Builder (cermin 3 builder asli)

- `SmsCommandBuilder`: `AT+CMGS="<tujuan>"` + body/PDU + Ctrl-Z; expected `["+CMGS:", "OK"]`, fail `["ERROR"]`.
- `UssdCommandBuilder`: `AT+CUSD=1,"<kode>",15`; expected `["+CUSD:", "OK"]`.
- `FormatCommandBuilder`: `string.Format(template, args)` generik (untuk `AT+CMGF={0}` dkk.).
- `SmsReader`: `AT+CMGL="REC UNREAD"` / `AT+CMGR=<idx>` → parse PDU → `IngressHub`; lalu `AT+CMGD=<idx>` hapus.

**Keputusan teknologi:** `System.IO.Ports.SerialPort` bawaan .NET (tidak perlu gammu/SMSTools3 — keduanya menambah proses eksternal; requirement kita $0 dan single-binary service). gammu/SMSTools3 tetap opsi fallback jika AT kustom bermasalah.

---

## 4. Transaction pipeline

Cermin alur asli (inbox → parsing → routing → eksekusi → status → balasan), dengan tabel yang sama dari `db_otomax.bak`:

```
 SMS/modem ─┐
 IM ────────┤
 IP/API ────┼─▶ IngressHub ──▶ inbox ──▶ TxnEngine ──▶ transaksi
 WA/Telegram ┘                     (tulis DB)      │
                                                   ▼
                          ┌────────────────────────────────┐
                          │ 1. Parsing (Format0, PolaParam, │
                          │    regex; → TidakAdaParsing jika│
                          │    gagal / FormatSalah)         │
                          │ 2. Validasi (reseller aktif?    │
                          │    blacklist? saldo cukup?      │
                          │    produk ada? anti-dobel?)     │
                          │ 3. Routing (pilih TipeModul +   │
                          │    terminal by prioritas, stok, │
                          │    failover → Alihkan)          │
                          │ 4. Eksekusi via ModemManager /  │
                          │    H2H client                   │
                          │ 5. Tangkap jawaban (SN, kode)   │
                          │ 6. Update status + tgl_status   │
                          │ 7. Komisi (AddKomisiTunda)      │
                          │ 8. Balasan → ReplyDispatch      │
                          └────────────────────────────────┘
```

### State machine status (49 ItemStatus — nilai int DIPERTAHANKAN)

Ordinal enum asli dipertahankan 1:1 sebagai `int` di DB dan kode (agar konsisten dengan semantik `status<20` = pending/diproses, `status>0` = terproses):

- **Pending** (`<20`): `BelumDiproses`, `SedangDiproses`, `MenungguJawaban`, `Terjadwal`, `Suspect`, `ProsesUlang` (+ `GagalKirim` sebagai retryable).
- **Sukses**: `Sukses`, `SuksesMasukOutbox`, `SuksesMasukTransaksi`, `SuksesMasukCS`.
- **Gagal final**: `Gagal`, `TidakAdaParsing`, `BukanReseller`, `FormatSalah`, `SaldoTidakCukup`, `ProdukSalah`, `StokKosong`, `TransaksiDobel`, `ProdukGangguan`, `ParameterSalah`, `PinSalah`, `Dibatalkan`, `ResellerTidakAktif`, `TujuanSalah`, `TujuanDiluarWilayah`, `KodeAreaTidakCocok`, `Timeout`, `NomorBlacklist`, `WrongSignature`, `NomorTidakAktif`, `HargaTidakSesuai`, `TidakAdaData`, `QtyTidakSesuai`, `LimitHarian`, `LimitTopupBulanan`, `ResellerSuspend`, `Diabaikan`, `UnitTidakCukup`, `InvalidTerminal`, `GroupDissallowed`, `AccessDenied`, `Cutoff`, `HarusUbahPin`, `NomorHangus`, `NomorMasaTenggang`, `Unknown`.
- **Khusus**: `Alihkan` → trigger failover ke modul/terminal lain (fitur "otomatis mengalihkan transaksi saat stok kosong/salah pin").

**Aturan transisi (evidence-based):**
- Hanya `TxnEngine` yang boleh menulis `status`/`tgl_status` (single writer → hindari race).
- `kirim_ulang`/`counter`/`counter2`: retry dengan backoff; `Alihkan` mereset counter untuk modul baru.
- Idempotensi: cek `TransaksiDobel` (per tanggal / jeda menit, sesuai setting) SEBELUM insert — pakai unique constraint DB sebagai backstop.
- Komisi: panggil setara `AddKomisiTunda(@trxid, @kodereseller, @jumlah)` hanya pada status sukses (atau sesuai SettingMLM).

### 4.1 Katalog placeholder jawaban + PolaParam

**Kapan substitusi dievaluasi:** placeholder disubstitusi oleh `ReplyDispatch` **tepat sebelum balasan dikirim** — bukan saat transaksi dibuat. Buktinya: `[saldo]` didefinisikan sebagai "saldo terakhir saat sistem membalas" [B] (Version.txt:417), sehingga konteks evaluasi = snapshot transaksi + data live pada saat dispatch.

#### Katalog resmi (grep langsung atas Version.txt 4.3.1 + NotaBill.prt)

| Placeholder | Makna | Label |
|---|---|---|
| `[trxcount]` | Total transaksi hari ini; **tidak** menghitung transaksi itu sendiri | [B] |
| `[sumofqty]` | Jumlah qty hari ini | [B] |
| `[qty]` | Qty transaksi (dipakai balasan `LaporanTransaksi`, `CekTransaksiAkhir`) | [B] |
| `[pin]` | PIN reseller; format UbahPin wajib memakainya agar PIN lama ikut dicek | [B] |
| `[pinbaru]` | PIN baru (format UbahPin/ResetPin) | [B] |
| `[saldo]` | Saldo terakhir **saat sistem membalas** | [B] |
| `[saldoawal]` | Saldo saat transaksi terinput (= kolom `transaksi.saldo_awal`) | [B] |
| `[saldolama]` | Saldo lama (renderer struk NotaBill.prt) | [B] |
| `[waktu]` | Tanggal hari ini | [B] |
| `[date]` / `[time]` | Tanggal / jam (renderer struk) | [B] |
| `[bank]` | Bank tiket deposit; tiket hanya di-settle modul bank yang sesuai | [B] |
| `[refid]` / `[trxid]` | ID transaksi (alias) | [B] |
| `[kodereseller]` | Kode reseller | [B] |
| `[namareseller]` | Nama reseller | [B] |
| `[kodeproduk]` | Kode produk | [B] |
| `[tujuan]` | Nomor tujuan | [B] |
| `[keterangan]` / `[description]` | Keterangan pada reply status (modul OH) | [B] |
| `[markup]` | Markup | [B] |
| `[markuptotal]` | Total markup = markup reseller + semua markup upline, pada balasan transaksi | [B] |
| `[selisih]` | Selisih harga | [B] |
| `[tagihan]` / `[bill]` | Tagihan postpaid yang tertangkap dari jawaban supplier | [B] |
| `[nomorseri]` | Nomor seri voucher | [B] |
| `[jumlah]` | Jumlah (uang) | [B] |
| `[kode]` / `[nama]` | Kode / nama generik (struk) | [B] |
| `[format]` | Format yang gagal (balasan `FormatSalah`) | [B] |
| `[pesan]` | Pesan (kiriman Forward CS) | [B] |
| `[pengirim]` | Pengirim (CS Monitor; untuk pesan non-member bisa kosong) | [B] |
| `to` / `to2` / `to3` | Varian nomor tujuan khusus WebSMSSender: `+628…` / `628…` / tanpa kode negara | [B] |

**Perlu verifikasi dari binary [R]** (terdeteksi di Version.txt, makna belum pasti): `[ver_combo]`, `[combo]`, `[aktif2]`, `[oldip]`, `[enduser]`, `[memberid]`, `[kodereferral]`, `[nomorhp]`, `[sign]` (dugaan: signature verifikasi IM Center — karena ada status `WrongSignature` [I]).

#### PolaParam + editor "uji"

- Setting key-value `PolaParam0`, `PolaParam20` (dan `PolaParam*` lain) di tabel `setting` = daftar regex bernama untuk menangkap parameter dari jawaban supplier [B].
- Parameter Parsing modul (mendukung s.d. 8000 karakter) + Parameter Parsing 2 (sama, tapi terenkripsi) [B] — regex per-modul tempat harga beli/saldo/SN ditangkap.
- **Spec editor UI** (Administrasi Jawaban) [B kebutuhan]: daftar regex bernama yang bisa ditambah/ubah + tombol **"uji"** — operator menempel contoh jawaban supplier → engine me-render daftar parameter yang tertangkap + preview balasan hasil substitusi placeholder. Tanpa ini, operator buta saat supplier mengubah format jawaban (R8).
- Dua titik evaluasi regex: (a) saat jawaban supplier tiba (tahap 5 pipeline → parsing parameter); (b) placeholder balasan saat `ReplyDispatch` (tahap 8).

### 4.2 Mekanika harga: Rumus Harga, Filter Tujuan, Qty Yang Diizinkan

Dievaluasi di `TxnEngine` pada tahap validasi/routing (langkah 2–3 pipeline), **sebelum** pemotongan saldo.

**Rumus Harga** (kolom `produk.rumus_harga`) [B definisi; I urutan evaluasi]:
- Formula: `(Qty * Nominal * 1000) + HrgMarkup` [B] (Version.txt:564). Nilai `0` = default (pakai `harga_jual`), `2` = bebas nominal/qty [B].
- Urutan evaluasi [I — jujur belum terverifikasi dari binary]: `harga_beli` (dari parsing/supplier) → terapkan `rumus_harga` → tambah markup level (`default_markup` + `selisih_harga` level + markup per-reseller) → `harga_jual` final yang dipotong dari saldo.
- **Peringatan [B]:** bug asli "perhitungan komisi tidak sesuai jika menggunakan rumus_harga selain 0" (Version.txt:555) — komisi HARUS dihitung dari harga hasil rumus, bukan `harga_jual` mentah. Jadikan acceptance test komisi × tiap varian rumus.

**Filter Tujuan** (kolom `produk.filter_tujuan`, RegEx) [B keberadaan; I titik evaluasi]:
- Dievaluasi saat validasi tujuan, setelah parsing format dan sebelum routing; gagal → status `TujuanSalah` / `TujuanDiluarWilayah` [I].
- Contoh pola (contoh buatan, bukan dari bukti): `^08\d{9,12}$` untuk nomor seluler 11–14 digit. UI wajib menyediakan editor + tombol "uji regex" [B kebutuhan].
- Berinteraksi dengan validasi provider: `prefix_tujuan`, `panjang_min`/`panjang_maks`, `kode_area` (gagal → `KodeAreaTidakCocok`) [B].

**Qty Yang Diizinkan** (pengganti Min/Max Qty) [B perilaku; R nama kolom `produk.qty_diizinkan`]:
- Aturan [B] (Version.txt 4.2.9): transaksi postpaid dengan qty > 1 sedangkan produk TIDAK punya isian ini → qty dipaksa menjadi 1 saat transaksi dibuat; qty masih bisa diubah kemudian melalui tangkapan jawaban.
- Evaluasi saat validasi qty; di luar daftar → `QtyTidakSesuai` [I pemetaan status].

### 4.3 Status `Terjadwal` + grouping 49 ItemStatus untuk UI

**Aturan eksekusi `Terjadwal`** [B perilaku; R kolom]:
- Ditulis dengan `transaksi.tgl_jadwal` terisi (kolom baru; constraint CHECK: `status=Terjadwal` ⇒ `tgl_jadwal NOT NULL`) [R].
- `SchedulerService` (Quartz) memindai periodik: `status=Terjadwal AND tgl_jadwal <= now()` → claim via `UPDATE … WHERE status=Terjadwal` (single-writer `TxnEngine`, idempoten) → ubah ke `BelumDiproses` → masuk pipeline normal [I mekanisme; B kebutuhan].
- **Dikecualikan dari hitungan "masih dalam proses"** [B] (Version.txt:1018): query pending operasional = `status < 20 AND status NOT IN (Terjadwal, Suspect)` [I konstruksi query].
- UI: filter khusus "Terjadwal" + kolom waktu eksekusi + aksi batalkan / jalankan-sekarang [B kebutuhan].

**Grouping untuk UI** (nilai int ordinal enum asli DIPERTAHANKAN 1:1 — jangan di-renumber):
- `Suspect`: disembunyikan dari daftar pending default; toggle "tampilkan suspect" [B].
- 21 status gagal validasi (`BukanReseller`, `FormatSalah`, `SaldoTidakCukup`, …) digrup ke filter "-- Gagal --" [B pola filter; I daftar grup].
- 4 varian Sukses dibedakan: `Sukses` vs `SuksesMasukOutbox` / `SuksesMasukTransaksi` / `SuksesMasukCS` (alur balasan berbeda) [B].
- `GagalKirim` = retryable (tombol kirim-ulang aktif); bedakan dari `Gagal` final [I].
- `Alihkan` = trigger failover otomatis; tampilkan modul asal→tujuan [B+R].

### Tabel yang dipakai (dari `db_otomax.bak`, 11 tabel — dipakai apa adanya)

`inbox`, `transaksi` (27 kolom — dipakai penuh termasuk `modul_proses`, `ket_modul`, `kode_terminal`), `tiket_deposit`, `mutasi`, `data_bank`, `cek_stok`, `oto_user` (PK=`nomor`), `reseller`, `produk`, `oto_histori`, `oto_histori_update`. Setting sebagai baris (`Setting0`, `SettingMLM`, `Balasan0`, `Format0`, `PolaParam*`) — pola key-value ini dipertahankan.

---

## 5. Keputusan teknologi per komponen

| Komponen | Pilihan | Alasan | Alternatif yang ditolak & kenapa |
|---|---|---|---|
| Runtime | **.NET 8 LTS** (Worker Service) | Sesuai Opsi B; `System.IO.Ports`, `HttpClient`, `System.Net.Sockets` native Linux; hiring/tooling .NET mature. **Catatan:** .NET 8 LTS berakhir Nov 2026 — rencanakan upgrade ke .NET 10 LTS saat tersedia stabil. | Python: dua runtime = dua deployment story; logika VB.NET lebih natural diport ke C#. |
| DAL | **Dapper** + `Npgsql` | Aslinya `SqlCommand(string)` mentah tanpa `set_CommandText` — Dapper paling dekat dengan pola ini, tanpa impedance ORM; `Npgsql` = driver resmi PostgreSQL. | EF Core: overkill untuk skema legacy + query rekonstruksi; NHibernate: mati. |
| Database | **PostgreSQL 16/17 di container Linux** | Target porting per `arch_db.md`: JSONB (`modul.parameter`), enum native (`jenis_mutasi`), TIMESTAMPTZ; `SELECT … FOR UPDATE` + `ON CONFLICT` idiomatik. | SQL Server: bobot container ~1GB+ + lisensi; T-SQL→PL/pgSQL sudah dikerjakan DB Specialist (bukan lagi fase 2). |
| Modem/serial | **`System.IO.Ports`** in-process | Single binary, tanpa proses eksternal; cukup untuk AT command standar. | gammu/SMSTools3: fallback jika AT kustom bermasalah; Kannel: terlalu berat untuk kebutuhan ini. |
| H2H/HTTP API | **`HttpClient`** + Polly (retry/timeout/circuit-breaker) | Pengganti `HttpClient`/`WebRequest` asli 1:1. | — |
| Internet Banking | **Selenium + Chrome for Testing (headless)** | Port paling setia dari stack Selenium asli; jalan di Linux. | Playwright: API lebih modern tapi berarti rewrite total pola Selenium asli; pertimbangkan jika Selenium terlalu rapuh. |
| Telegram | **Telegram.Bot** | Native .NET, Linux-ok, maintained. | — |
| XMPP/Jabber | **`IImProvider` + sidecar** (mis. Python slixmpp via HTTP lokal) | Tidak ada lib XMPP .NET yang maintained. | SharpXMPP: unmaintained, risiko keamanan. |
| WA Center | **Selenium + Chrome for Testing (WA Web, unofficial)** | Keputusan user 2026-10-17: port setia pendekatan asli (§2d). Risiko ban nomor + breakage; worker terisolasi, session persistence, fallback Telegram. | Cloud API resmi: ditolak user. |
| IP Center/Client | **`System.Net.Sockets`** | Protokol TCP sederhana; butuh sniff untuk format pesan. | — |
| Scheduler | **Quartz.NET** | Cron-like persistent untuk transaksi terjadwal, auto-delete log, dsb. | `PeriodicTimer` saja: cukup untuk interval sederhana; Quartz untuk yang butuh persistensi. |
| Logging | **Serilog** (file rolling + opsional DB) | Standar .NET. | — |
| API ke web UI | **ASP.NET Core minimal API + SignalR** | Satu proses dengan engine (port localhost saja); SignalR untuk monitoring realtime (inbox/transaksi live). | gRPC: overkill untuk web UI. |
| Cetak struk | **QuestPDF** (PDF) ganti `NotaBill.prt` | Raw printing di Linux rapuh; PDF → print via CUPS lebih portable. | Port template `.prt` mentah: ditunda. |

### Yang TIDAK diporting

1. **Sistem lisensi/edisi** — keputusan desain: dihapus total (setara Ultimate).
2. **WebView2** — Windows-only; diganti headless Chromium (Selenium).
3. **66 form WinForms** — diganti web UI bertahap (di luar scope dokumen ini; mulai dari dashboard monitoring + laporan).
4. **Client.exe `-client` mode** — diganti peran web UI + auth.
5. **Kiosk module** (toko/cetak struk) — fase 2; bukan inti transaksi.
6. **Dongle/SIM licensing** — dihapus bersama sistem lisensi.
7. **Proteksi packer/obfuscator** — tidak direplikasi (itu milik vendor).

---

## 6. Risiko utama & mitigasi

| # | Risiko | Dampak | Mitigasi |
|---|---|---|---|
| R1 | **Internet banking diblokir bank** (bot detection, OTP, DOM berubah) | Deposit otomatis mati | Isolasi worker per bank; antrian fallback input manual via web UI; pola asli memang sudah semi-manual (interval 300 dtk) |
| R2 | **ApiSiDompul/ApiAxis ternyata butuh UI Android** | 2 channel mati | Evidence kuat ini HTTP API; Appium+emulator sebagai rencana B; tidak jadi blocker fase 1 |
| R3 | **Protokol IP Center tidak diketahui** (string terenkripsi) | H2H TCP tidak jalan | Sniff traffic Windows app (Wireshark) — sudah direncanakan di TARGETED_DECRYPTION.md; buat parser pluggable |
| R4 | **Query SQL exact belum pulih** (string terenkripsi) | Logika DAL harus direkonstruksi | Skema 11 tabel lengkap → rekonstruksi dari kebutuhan bisnis; pola `SqlCommand(string)` sederhana |
| R5 | **Modem USB tidak stabil di Linux** (cabut, permission, nama device berubah) | Terminal down | udev rules nama stabil, grup `dialout`, health ping + auto-reconnect dengan backoff |
| R6 | **WA unofficial breakage** | Notifikasi WA mati | Worker terisolasi + session/profile persistence + monitoring "WA Center tidak bisa mengirim" + fallback notifikasi via Telegram (sesuai §2d); Cloud API resmi tetap DITOLAK user (keputusan 2026-10-17) |
| R7 | **Porting T-SQL legacy → PL/pgSQL** (query SQL exact belum pulih, SP `AddKomisiTunda`/`DownloadFromMaster` + trigger direkonstruksi) | Logika DAL harus direkonstruksi, bukan di-port mentah | Skema target lengkap di `arch_db.md`; Dapper + parameterized query; test parity T-SQL vs PL/pgSQL saat spike DAL (§7.1) |
| R8 | **Regex/parsing jawaban supplier berubah** | Transaksi gagal parsing | `PolaParam` + regex disimpan sebagai setting DB (seperti aslinya: baris setting) → bisa diubah tanpa deploy |
| R9 | **.NET 8 LTS habis Nov 2026** | Tanpa patch keamanan | Rencanakan upgrade .NET 10 LTS di awal roadmap; kode ditulis tanpa API yang deprecated |
| R10 | **Race condition saldo** jika satu path lupa `FOR UPDATE` | Saldo minus (bug asli Transfer Saldo) | Helper tunggal `UbahSaldo()` untuk semua path uang (§9); uji beban paralel |
| R11 | **False positive auto-nonaktif fraud** (nomor daur ulang operator) | Reseller sah dinonaktifkan | Kadaluarsa 30 hari + log + re-aktivasi manual mudah; notifikasi ke CS |
| R12 | **Badai transaksi Terjadwal** (ratusan fire bersamaan) | Lonjakan ke supplier, tembak duplikat | Claim batch + rate-limit per modul; idempotensi anti-dobel (§4) |
| R13 | **`TurnOffBlacklist` disalahgunakan** | Proteksi fraud mati diam-diam | Butuh hak admin + banner peringatan + audit setiap perubahan (§10) |
| R14 | **Komisi salah hitung dengan `rumus_harga ≠ 0`** (bug asli) | Uang komisi salah | Acceptance test komisi × tiap varian rumus; hitung dari harga hasil rumus (§4.2) |

---

## 7. Urutan implementasi yang disarankan

1. **Fondasi:** Worker Service + Dapper/Npgsql + PostgreSQL 16/17 container + Serilog + ApiHost (health/monitoring) + **RBAC sejak hari pertama** (tabel `pengguna`/`peran`/`peran_menu`, proteksi API, seed admin — tidak bisa disusulkan, §12). Kriteria: service hidup, baca/tulis DB, endpoint tanpa token ditolak.
2. **AtChannel + ModemManager:** 1 modem kirim/terima SMS (AT+CMGS/CMGL), health check, reconnect. Kriteria: SMS loopback jalan.
3. **TxnEngine inti:** inbox → parsing → validasi → routing → status → balasan, dengan 1 channel SMS. Termasuk sejak awal: **keamanan uang** (§9: `FOR UPDATE`, `saldo_awal`), **mekanika harga** (§4.2: Rumus Harga, Filter Tujuan, Qty Yang Diizinkan), **katalog placeholder** (§4.1), dan tulis `Terjadwal` + `transaksi_histori`. Kriteria: transaksi pulsa SMS end-to-end via 1 modem; uji race paralel hijau.
4. **Multi-terminal + failover (Alihkan)** + prioritas + stok. Kriteria: 2 modem, failover terbukti.
5. **H2H HTTP clients** (SpecialHth/OH/WebSMSSender) + Polly. Kriteria: transaksi via API agregator.
6. **IP Center/Client** setelah sniff protokol.
7. **Internet Banking** (Selenium) + Telegram + WA Web unofficial + IM sidecar.
8. **Scheduler** (eksekusi `Terjadwal`, auto kirim-ulang, housekeeping) + **komisi MLM + auto-claim bank** (§11) + **aksi operasional via command queue** (§8: kirim-ulang bulk, proses-ulang, ubah-status + audit) + **fraud auto-nonaktif** (§10).
9. **Web UI** bertahap (monitoring dulu), di atas ApiHost yang sudah ada sejak langkah 1.

Setiap langkah punya kriteria selesai yang bisa diuji — sesuai prinsip forge: bukti, bukan klaim.

---

## 8. Aksi operasional (operator) — via command queue

**Guard keras:** hanya `TxnEngine` yang boleh menulis `transaksi.status` / `tgl_status` (single writer, §4). Aksi operator dari API/web UI **tidak** melakukan UPDATE langsung — melainkan masuk **command queue** (`Channel<OperatorCommand>`, in-memory + persist bila perlu restart-safety) yang dieksekusi `TxnEngine` secara berurutan. Ini mencegah race antara klik operator dan worker pipeline [I desain].

### 8.1 Kirim-ulang
- Per-baris, per-modul pending, dan bulk ("kirim ulang semua transaksi modul OH yang pending") [B].
- Re-send **tanpa** revalidasi penuh; `counter` / `kirim_ulang` di-increment [B] (Version.txt:305: "Counter tembak berubah saat transaksi dikirim ulang").
- Batas bulk [I — usulan operasional, bukan dari bukti]: batch ≤ 100 per eksekusi + jeda antar-batch agar tidak membanjiri supplier.
- Guard [B]: tolak kirim-ulang bila transaksi sedang diproses (bug yang pernah di-fix, Version.txt:566) → API mengembalikan 409 + pesan jelas.
- Auto kirim-ulang per modul: interval minimum 10 detik [B]; perilaku asli — modul IM/IP tiap 30 detik dalam rentang 10 menit dihitung dari `tgl_entri` [B]; tersimpan di `modul.parameter` JSONB (`auto_resend_interval`).

### 8.2 Proses-ulang (modul SAMA)
- Menu "Proses Ulang (modul sama)" [B]: status → `ProsesUlang`; kolom `perintah` di-RESET [B] (Version.txt 3.5.3: "Reset perintah saat pengalihan atau proses ulang transaksi"); `kirim_ulang`/`counter` di-increment [B].
- Cache REFID [B]: transaksi ber-REFID yang berstatus `Kosong` / `Gangguan` / `ProdukSalah` **mempertahankan** status itu walaupun produk sudah dibuka (proteksi terhadap pesan IM/SMS/OH yang tertunda). Tidak berlaku: transaksi tanpa REFID, atau yang melalui jalur IP.

### 8.3 Ubah-status manual + audit
- Multi-select di grid; ubah ke `Gagal` → dialog wajib isi keterangan [B].
- Setiap perubahan menulis baris ke `transaksi_histori` (siapa, kapan, status lama→baru, keterangan) [R skema, lihat `arch_db.md` §2.7/§2.8; B kebutuhan audit].
- "Batalkan Semua Transaksi Yang Belum Diproses" [B]: bulk + konfirmasi + daftar status yang dicakup eksplisit (pernah bug untuk `TidakAdaParsing` — jadikan test case).
- "Hapus Data tidak menghapus Log Sistem" [B] — whitelist tabel audit dipertahankan.

## 9. Keamanan uang

### 9.1 Locking saldo (multi-dompet — keputusan FINAL, sinkron `arch_db.md` §4.1)

**Keputusan: MULTI-DOMPET PENUH.** Semua path uang (potong transaksi, refund, settle deposit, transfer, tukar komisi) mengunci dan mengupdate baris `reseller_saldo(kode_reseller, kode_dompet)` yang benar — bukan `reseller.saldo` saja.

- Satu helper tunggal `UbahSaldo(kode_reseller, kode_dompet, delta, jenis_mutasi, …)` — **semua** path uang wajib lewat sini; tidak ada UPDATE saldo ad-hoc [I desain].
- Locking: `SELECT … FOR UPDATE` atas **baris `reseller_saldo`** dalam satu transaksi DB [B kebutuhan; bug asli: race condition format Transfer Saldo → saldo reseller minus, Version.txt 4.2.6].
- Pastikan baris ada dulu (R mekanisme): `INSERT INTO reseller_saldo (kode_reseller, kode_dompet, saldo) VALUES (…, …, 0) ON CONFLICT DO NOTHING` sebelum locking — lalu `SELECT … FOR UPDATE` selalu dapat baris.
- `reseller.saldo` ikut diupdate atomik **HANYA bila `kode_dompet='1'`** (dompet utama) dalam transaksi yang sama — kompatibilitas dengan path potong-saldo existing, per `arch_db.md` §4.1 [R].
- Setiap baris `mutasi` yang ditulis helper **WAJIB mengisi `kode_dompet`** [R]; `transaksi.kode_dompet` diisi saat pembuatan transaksi [R].
- `transaksi.saldo_awal` ditulis engine **sebelum** pemotongan [B] (Version.txt:417-418); `mutasi.saldo_awal` + `saldo_akhir` = snapshot per baris untuk rekonsiliasi [I].

### 9.2 Refund idempoten
- Auto-refund jawaban [B]: syarat status `Dibatalkan` + trxid cocok.
- Sebelum insert mutasi refund: cek belum ada mutasi refund untuk `kode_transaksi` yang sama (filter `jenis` = refund pada enum `jenis_mutasi`) → skip bila sudah ada [I mekanisme; B kebutuhan anti-dobel].
- `dompet.refund_ke` mendukung notasi `.` [B] = dompet asal. Refund dilakukan via `UbahSaldo(kode_reseller, kode_dompet_tujuan, …)` dengan `kode_dompet_tujuan` hasil resolusi `refund_ke` ('.' → dompet asal transaksi); baris `mutasi` refund menulis `kode_dompet` dompet tujuan [R].

### 9.3 Generator angka unik tiket deposit
- **Keputusan: SEQUENCE database** (PG `CREATE SEQUENCE`, satu sequence khusus angka unik tiket) — bukan advisory lock.
- **Alasan:** thread-safe lintas proses/instans engine; tahan restart (nilai persisten di DB, tidak hilang seperti counter in-memory); tanpa sesi-kunci manual yang bisa lepas saat koneksi putus. Advisory lock ditolak: bergantung pada sesi koneksi dan dua worker di mesin berbeda tetap butuh koordinasi eksternal — sequence menyelesaikannya di satu tempat [I keputusan].
- Backstop [B]: unique constraint + "cek data dobel" mutasi bank (Version.txt:112-113); optimasi pembuatan angka unik pernah di-fix (Version.txt:259) → uji beban paralel wajib.
- Idempotensi deposit bank: unique constraint `(bank, tgl_bank, jumlah, keterangan)` pada `data_bank` [B kebutuhan] + auto-deposit hanya untuk data tanggal hari ini/PEND [B].

## 10. Proteksi fraud

### 10.1 Blacklist (`daftar_hitam`) + kolom Fraud
- Tabel `daftar_hitam` (nomor, fraud, kadaluarsa, keterangan) [B kebutuhan].
- **Kolom Fraud → auto-nonaktif reseller:** bila `fraud=1` dan nomor kena hit, engine menonaktifkan reseller otomatis (`reseller.aktif=0` + tulis log) [B] (Version.txt:598). Ini **aturan engine di tahap validasi** — bukan sekadar kolom [B].
- Setting jawaban `NomorBlacklist` → nomor otomatis masuk daftar hitam dengan kadaluarsa 30 hari [B] (Version.txt:117).
- Job startup: hapus entri yang sudah kadaluarsa > 7 hari [B] (Version.txt:1102).
- `TurnOffBlacklist` (setting optimasi) [B]: mematikan pengecekan — perubahan setting ini wajib hak admin + banner peringatan + tulis audit [I guard].

### 10.2 Validasi terpusat di 4 titik
Fungsi tunggal `CekNomor(nomor)` dipakai di: (1) halaman Pengirim, (2) format tambah nomor, (3) daftar downline, (4) daftar member baru [B] (Version.txt:11). Cakupannya: cek `daftar_hitam` + cek duplikat nomor WA/SMS [B].
- **Normalisasi sebelum bandingkan** [B]: `+62…` / `62…` / `08…` dinormalisasi ke bentuk kanonis (`62…`) dulu — bug "duplikat awalan 62" pernah di-fix (Version.txt:21).
- Titik tambahan [I]: transaksi masuk (→ status `NomorBlacklist`) dan klaim tiket deposit — engine memanggil `CekNomor` yang sama.
- Reseller: penambahan **hanya** via Pendaftaran (POST /reseller langsung ditolak; edit kode butuh hak hapus) [B]; pola ID `OR####` auto-number [B].

## 11. Komisi & auto-claim bank

### 11.1 Pemicu komisi
- `SettingTundaKomisi=1` → saat transaksi sukses, engine memanggil setara `AddKomisiTunda(trxid, kodereseller, jumlah)` = upsert ke tabel `komisi` (tunda, belum masuk saldo) [B].
- `SettingTundaKomisi=0` → komisi langsung: via `UbahSaldo(kode_reseller, kode_dompet, jumlah, jenis_mutasi.Komisi, …)` — `kode_dompet` = dompet komisi yang dikonfigurasi (default `'1'` = dompet utama) [R default]; baris `mutasi` wajib mengisi `kode_dompet` (§9.1); `reseller.saldo` ikut diupdate atomik hanya bila `kode_dompet='1'` [I — perilaku "langsung" tidak eksplisit di bukti; butuh verifikasi binary].
- `SettingKomisiFull=1` ("bagikan komisi penuh walaupun mengurangi margin/laba server") [B] → cabang kalkulasi: komisi penuh ke upline vs dipotong margin server [B setting; I rumus cabang].
- Komisi dihitung dari harga **hasil** `rumus_harga`, bukan `harga_jual` mentah — bug 555 [B].
- Penukaran: `TUKAR.[jumlah].[pin]` via SMS / menu Alat Bantu → validasi PIN → kurangi `komisi`, tambah `mutasi` + saldo via `UbahSaldo(…, kode_dompet='1', …)` — baris `mutasi` mengisi `kode_dompet`, atomik [B].

### 11.2 Pipeline auto-claim mutasi bank (worker Internet Banking)
Urutan **berurutan** [B] (Version.txt:68):
1. **Tiket deposit (angka unik)** — cocokkan jumlah unik ke `tiket_deposit` status `'O'`; `data_bank.approved` menandai persetujuan (aksi di modul Data Bank, tercatat di `data_bank_riwayat`); `tiket_deposit.berita_bank` untuk pencocokan pola berita.
2. **PolaAutoDeposit** — pola berita auto deposit (Setting lain-lain, atau klik-kanan modul Internet Banking → Filter Mutasi) [B].
3. **Berita bank** — dua kolom dengan peran berbeda (opsi A, diselesaikan per N6):
   - `reseller.berita_bank` [B] = **master template pola berita** per reseller (multi-keyword dipisah `;`, dukung prefix `regex:`) — dipakai engine untuk pencocokan pola auto-claim;
   - `tiket_deposit.berita_bank` [R] = **snapshot cache per tiket** — berita bank yang cocok saat auto-claim, ditulis engine saat auto-claim cocok; sumber kebenaran tetap `reseller.berita_bank` (sinkron dengan komentar DDL `arch_db.md:106`).
- Tiap tahap yang dicoba/dicocokkan **tercatat** di `data_bank.catatan`; `data_bank.kode_tiket` = link balik; flag `terklaim` [B].
- Yang tak cocok → antrean klaim manual (UI input/approve manual; MVP: input mutasi manual dulu) [B kebutuhan].
- Guard: hanya data bank tanggal hari ini/PEND yang diproses [B]; cek data dobel via unique constraint [B]; mutasi debit/keluar mencurigakan → Log Sistem saja, **tanpa** auto-correction [B].

## 12. RBAC (sejak MVP — tidak bisa disusulkan)

- Tabel `pengguna`, `peran`, `peran_menu` [R skema, lihat `arch_db.md` §2.7/§2.8; B kebutuhan] (hak akses menu Administrasi–Dompet pernah bug, Version.txt:44 → menu Dompet = test case RBAC).
- Proteksi API: authorization policy per peran di ApiHost (mis. `RequirePeran("admin")` / `RequireMenu("dompet")`); klaim peran + daftar menu disematkan di JWT saat login [I desain].
- UI: sembunyikan menu tanpa hak — dan **jangan** hanya andalkan hidden: API tetap menolak request tanpa hak [B kebutuhan].
- Peran bawaan minimal [R]: `admin` (semua), `operator` (transaksi + monitor + aksi operasional), `teknisi` (modul/terminal/setting), `kasir-kiosk` (layar kasir saja). Seed 1 admin + peran-peran ini saat instalasi [I].
- Aturan keras: tambah-saldo, ubah markup saat ada transaksi pending, mematikan blacklist, hapus data → butuh peran `admin` + tulis audit [B kebutuhan dari FITUR_MENU.md].
