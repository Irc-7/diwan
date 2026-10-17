# Sketsa Arsitektur Linux — Clone OtomaX 4.3.1

**Tanggal:** 2026-10-17 | **Metode:** forge — 3 agen arsitek spesialis (DB, Engine, UI), evidence dari teardown.
**Dokumen detail:** `arch_db.md` (DDL PostgreSQL 544 baris) · `arch_engine.md` (desain service) · `arch_ui.md` (peta 22 layar web).

## 0. Keputusan desain yang mengikat

1. **Tanpa batasan edisi** — setara Ultimate: semua modul terbuka, terminal unlimited, tanpa lisensi/kunci edisi, tanpa masa berlaku (disetujui user 2026-10-17, lihat `EDISI_DAN_BATASAN.md`).
2. **Opsi B:** engine = service .NET 8 (systemd) + web UI terpisah via API. Wine hanya eksperimen; Mono WinForms mati suri (vonis `LAPORAN_GAP_RISET.md`).
3. **Functional clone, bukan pixel-perfect** — 66 WinForms tidak diport 1:1.

## 1. Gambaran arsitektur

```
┌──────────────────────────────────────────────────────────────────┐
│  OtomaX.Engine — .NET 8 Worker Service (systemd, single binary)   │
│                                                                  │
│  IngressHub ──▶ TxnEngine ──▶ ModemManager (1 worker/terminal)    │
│       ▲              │  ▲              H2H/API Clients (HttpClient)│
│       │              ▼  │              SchedulerService (Quartz)   │
│  (SMS/IM/IP/WA)  ReplyDispatch                                   │
│                                                                  │
│  ApiHost: ASP.NET Core minimal API + SignalR (localhost)          │
└──────────────┬───────────────────────────────┬───────────────────┘
               │                               │
        SQL Server 2022                  /dev/ttyUSB* (modem,
        (container, fase 1)               udev, grup dialout)
        PostgreSQL (fase 2,
        DDL siap di arch_db.md)
               │
┌──────────────▼───────────────────┐
│  Web UI — React+TS SPA + AG Grid  │  (atau Blazor Server bila tim murni C#)
│  REST /api/v1 + SignalR /hub/monitor (JWT + RBAC sejak MVP)       │
└──────────────────────────────────┘
```

Komunikasi antar-domain engine via `Channel<T>` bounded (satu domain macet tidak menyeret yang lain) — meniru pola asli "antrian canggih, modul simultan berprioritas".

## 2. Engine service (.NET 8)

**Pipeline transaksi** (cermin aslinya): `inbox` → parsing (Format0/PolaParam/regex) → validasi (reseller, blacklist, saldo, anti-dobel) → routing (prioritas, stok, failover `Alihkan`) → eksekusi → tangkap jawaban (SN) → update status → komisi (`AddKomisiTunda`) → balasan via ReplyDispatch.

**49 nilai `ItemStatus` dipertahankan 1:1 sebagai int** (semantik `status<20`=pending, `status>0`=terproses); hanya TxnEngine yang menulis status (single writer); idempotensi via cek dobel + unique constraint.

**15 channel `TipeModul` → Linux:**

| Channel | Implementasi |
|---|---|
| SMSCenter, SMSSender, _GsmUssd, Dial, SMS, Keypad, STK | `System.IO.Ports.SerialPort` + AT command (lihat §4) |
| WebSMSSender, SpecialHth, OH, ApiSiDompul, ApiAxis | `HttpClient` + Polly (retry/timeout/circuit-breaker) |
| IP (Center/Client) | `System.Net.Sockets` — protokol perlu di-sniff (Wireshark thd app Windows) |
| Fisik (voucher) | Modul stok/admin, fase 2 |
| IM (XMPP/Jabber) | `IImProvider` + sidecar (tidak ada lib XMPP .NET yang maintained) |
| WA Center | **Otomasi WA Web unofficial dipertahankan** (keputusan user 2026-10-17) — Selenium + Chrome for Testing mengotomasi WhatsApp Web, port setia dari pendekatan aslinya | Risiko jujur: WhatsApp aktif mematahkan otomasi unofficial (selector berubah, anti-bot) + risiko banned akun; mitigasi: isolasi worker, session persistence, fallback notifikasi via Telegram |
| Telegram | `Telegram.Bot` (native, maintained) |
| Internet Banking | Selenium + Chrome for Testing headless, worker terpisah per bank + fallback input manual |

**Tidak diporting:** sistem lisensi/edisi, WebView2 (Windows-only), 66 form WinForms, mode `-client`, Kiosk (fase 2), proteksi packer.

**Catatan:** .NET 8 LTS berakhir Nov 2026 → rencanakan upgrade .NET 10 LTS di roadmap.

## 3. Lapisan AT command (hasil teardown)

Satu-satunya pemanggil `SerialPort.Write` di binary asli adalah 1 method dengan pola `tulis → tunggu expected[] → timeout`. Desain `AtChannel` per modem:
- **Satu command aktif per port** (half-duplex); tulis `cmd\r`, baca sampai salah satu expected cocok / regex match / timeout.
- **Unsolicited** (`+CMTI`, `RING`, `+CUSD`) di-route ke IngressHub, tidak dimakan penunggu command.
- **Dua-tahap SMS:** `AT+CMGS="<no>"` → tunggu `"> "` → PDU → Ctrl-Z → tunggu `+CMGS:`.
- Health ping `AT` periodik; 3x gagal → tandai down → auto-reconnect (re-init `ATE0`, `AT+CMGF`, `AT+CNMI`).
- Linux: udev rules untuk nama stabil, grup `dialout`, reconnect saat USB dicabut.
- Builder cermin 3 builder asli: SMS (`AT+CMGS`), USSD (`AT+CUSD`), format generik (`String.Format`).
- Expected responses standar GSM 07.05/07.07: `OK`, `ERROR`, `"> "`, `+CMGS:`, `+CMTI:`.

## 4. Database

**Fase 1: SQL Server 2022 di container Linux** — nol perubahan query/T-SQL; SP `DownloadFromMaster`/`AddKomisiTunda` + trigger `tr_oto_user_data` jalan apa adanya. (Rekomendasi agen engine; menghindari jebakan translasi di awal.)

**Fase 2: PostgreSQL** — DDL lengkap siap di `arch_db.md` (18+5 objek, tiap kolom dilabel **[B]** terbukti / **[I]** inferensi / **[R]** rekonstruksi, 138 label). Prinsip: identifier lowercase, `IDENTITY → GENERATED BY DEFAULT AS IDENTITY`, `DATETIME → TIMESTAMPTZ`, `MONEY → NUMERIC(19,2)`, `BIT → BOOLEAN`; pertimbangkan `CITEXT` untuk kunci string (collation SQL Server case-insensitive). SP `AddKomisiTunda` → PL/pgSQL upsert; `DownloadFromMaster` sebaiknya diganti pg_dump terjadwal.

**Gap jujur:** tidak ada `CREATE TABLE` di backup — tipe data best-effort. Mitigasi termurah: restore `.bak` ke SQL Server Express (gratis, jalan di container Linux) lalu generate DDL eksak.

## 5. Web UI

**66 WinForms → ±22 layar web** dalam 9 area: Dashboard & Monitoring (live), Transaksi, Master Data (Produk/Provider/Parsing/Level/Cek Harga), Reseller & Keanggotaan, Modul & Terminal, Inbox/Outbox, Keuangan (Mutasi/Komisi/Poin/Tiket Deposit/Data Bank/Dompet/Voucher), Laporan, Pengaturan & Sistem (RBAC, log).

**Stack:** React + TypeScript SPA + AG Grid Community (ekspektasi "gaya Excel": copy-paste, sortir — disebut eksplisit di changelog) di atas REST + SignalR. Alternatif tim murni C#: Blazor Server.

**API contract:** REST `/api/v1` (transaksi, produk, reseller, inbox/outbox, mutasi, komisi, tiket-deposit, modul, terminal, laporan, setting, auth JWT) + SignalR `/hub/monitor` (event `transaksi.baru`, `transaksi.status`, `inbox.baru`, `modul.status`, `saldo.berubah`; subscribe berfilter per layar). Payload memakai nama kolom DB hasil RE apa adanya.

**MVP (Fase 1):** Dashboard + Monitor live → Transaksi → Produk + Parsing → Reseller + Mutasi + Tiket Deposit + Data Bank → Monitor Modul & Terminal. Fase 2: Inbox/Outbox, Komisi/Poin, Provider/HLR/Level, Voucher+Kiosk, Laporan. Fase 3: Pengaturan, Hak Akses, Alat Bantu, Log, Dompet.

## 6. Urutan implementasi engine (9 langkah, tiap langkah ada kriteria uji)

1. Fondasi: Worker Service + Dapper + SQL Server container + Serilog + ApiHost (health).
2. AtChannel + ModemManager: 1 modem SMS loopback jalan.
3. TxnEngine inti: transaksi pulsa SMS end-to-end via 1 modem.
4. Multi-terminal + failover (`Alihkan`) + prioritas + stok.
5. H2H HTTP clients + Polly.
6. IP Center/Client (setelah sniff protokol).
7. Internet Banking (Selenium) + Telegram + WA Cloud API + IM sidecar.
8. Scheduler + komisi MLM + laporan.
9. Web UI bertahap (monitoring dulu) di atas ApiHost.

## 7. Risiko utama

| # | Risiko | Mitigasi |
|---|---|---|
| R1 | Internet banking diblokir bank (bot detection/OTP/DOM berubah) | Worker terpisah per bank; fallback input manual via web UI |
| R2 | ApiSiDompul/Axis ternyata butuh UI Android | Evidence kuat ini HTTP API; rencana B Appium, bukan blocker fase 1 |
| R3 | Protokol IP Center tak diketahui (string terenkripsi) | Sniff Wireshark thd app Windows; parser pluggable |
| R4 | Query SQL exact belum pulih | Skema 11 tabel lengkap → rekonstruksi dari kebutuhan bisnis |
| R5 | Modem USB tak stabil di Linux | udev + grup dialout + health ping + auto-reconnect |
| R6 | State machine 49 status salah dimodelkan | Model eksplisit di engine + uji transisi; UI hanya menampilkan |
| R7 | Skema DB tak lengkap (tanpa CREATE TABLE) | DDL inferensi + migrasi sejak awal; mitigasi termurah: restore .bak ke Express |
| R8 | WA Web unofficial dipatahkan WhatsApp (keputusan: pertahankan otomasi unofficial) | Worker terisolasi + session persistence + monitoring; fallback notifikasi via Telegram |

## 8. Estimasi

Vonis gap research tetap berlaku: **functional clone feasible, multi-bulan** — bukan minggu. Sketsa ini adalah fondasi desainnya; langkah berikut yang masuk akal: (a) validasi DDL via restore `.bak` ke SQL Server Express, (b) spike AtChannel + 1 modem di Linux, (c) sniff protokol IP Center — ketiganya bisa paralel dan masing-masing memberi bukti sebelum komitmen besar.
