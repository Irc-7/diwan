# PRD — OtomaX Linux Clone ("Orisinil Topup Machine" untuk Linux)

**Versi:** 1.0 | **Tanggal:** 2026-10-17 | **Status:** Draft untuk persetujuan
**Sumber referensi:** OtomaX 4.3.1 Free Edition (hasil reverse engineering, lihat `docs/`)

## 1. Ringkasan produk

Clone fungsional OtomaX — software server pulsa/PPOB otomatis — yang berjalan di **Linux** (desktop maupun server). Transaksi pulsa, paket data, dan PPOB diproses otomatis melalui modem GSM (AT command), SMS, USSD/STK, HTTP API (H2H), dan TCP/IP, dengan manajemen reseller berjenjang (MLM), komisi, tiket deposit, dan laporan.

**Keputusan produk yang mengikat:**
- Setara edisi **Ultimate**: semua modul terbuka, terminal/modem unlimited, **tanpa batasan edisi, tanpa lisensi, tanpa masa berlaku**.
- Functional clone, bukan pixel-perfect: UI desktop 66 WinForms diganti web UI modern (±22 layar).
- Otomasi WA unofficial **dipertahankan** (port setia dari aslinya).

## 2. Tujuan & non-tujuan

**Tujuan:**
- Server pulsa dapat beroperasi penuh di Linux: terima transaksi → proses via modem/API → balas otomatis.
- Monitoring real-time dan administrasi via browser (desktop maupun HP operator).
- Skema data kompatibel dengan database OtomaX asli (migrasi mudah).

**Non-tujuan (tidak dibangun):**
- Sistem lisensi/kunci edisi/masa berlaku — dihapus total.
- Proteksi packer/obfuscator — tidak direplikasi.
- Modul Kiosk (kasir struk fisik) — fase 2.
- Mode `-client` / SafeMode — digantikan web UI + auth.
- WebView2 — digantikan headless Chromium.

## 3. Persona pengguna

| Persona | Kebutuhan utama |
|---|---|
| Pemilik server | Setup awal, pantau kesehatan bisnis, laporan, atur produk & harga |
| Operator harian | Monitor transaksi live, proses ulang transaksi gagal, kelola inbox/outbox, tambah modem |
| Reseller (via API/H2H) | Transaksi via SMS/Jabber/IP — tidak pakai UI langsung |
| Teknisi lapangan | Tambah/pindahkan modem, cek sinyal via AT log |

## 4. Kebutuhan fungsional

### 4.1 Transaction engine (inti)
- Pipeline: inbox → parsing format → validasi (reseller aktif, blacklist, saldo, anti-dobel) → routing modul (prioritas, stok, failover `Alihkan`) → eksekusi → tangkap jawaban (SN) → update status → komisi → balasan otomatis.
- **49 status transaksi** (`ItemStatus`) dipertahankan 1:1 sebagai integer; hanya TxnEngine yang menulis status.
- Idempotensi: cek transaksi dobel per tanggal/jeda menit + unique constraint DB.
- Retry dengan backoff (`counter`/`counter2`/`kirim_ulang`); failover otomatis antar modul/terminal.

### 4.2 Modul/channel (15 tipe `TipeModul`)
| Channel | Mekanisme Linux |
|---|---|
| SMSCenter, SMSSender, SMS, _GsmUssd, Dial, Keypad, STK | SerialPort + AT command (`AtChannel` per modem: 1 command aktif/port, expected-responses, regex parsing, unsolicited routing) |
| WebSMSSender, SpecialHth, OH, ApiSiDompul, ApiAxis | HttpClient + Polly (retry/timeout/circuit-breaker) |
| IP Center/Client | TCP sockets (protokol di-sniff dari app Windows) |
| Fisik (voucher) | Stok/admin + cetak via PDF (fase 2) |
| IM (XMPP/Jabber) | Sidecar (lib XMPP .NET tidak maintained) |
| WA Center | Otomasi WA Web unofficial via Selenium (dipertahankan sesuai keputusan) |
| Telegram | Telegram.Bot (native) |
| Internet Banking | Selenium + Chrome headless, worker per bank + fallback input manual |

### 4.3 Keuangan
- Mutasi saldo reseller (masuk/keluar/transfer), snapshot `saldo_akhir`.
- Komisi MLM tunda → tukar jadi deposit (`AddKomisiTunda`).
- Tiket deposit (kode unik/berita bank) + klaim otomatis via mutasi bank (Internet Banking) atau manual.
- Poin/unit reseller; dompet; voucher fisik.

### 4.4 Master data & pengaturan
- CRUD: Produk (+Rumus Harga, Filter Tujuan RegEx — fitur Ultimate), Provider/HLR/Operator, Parsing (routing), Level/MLM, Reseller, Pengirim + blacklist.
- Administrasi jawaban: template format & balasan (`[trxcount]`, `[pin]`, dsb.).
- Setting key-value; hak akses pemakai (RBAC); night mode; log sistem.

### 4.5 Monitoring & laporan real-time
- Dashboard operasional, monitor transaksi live, monitor modul & terminal (termasuk raw AT log untuk debug).
- Laporan: rekap komisi, saldo awal reseller, laporan transaksi.
- Event real-time via SignalR: `transaksi.baru`, `transaksi.status`, `inbox.baru`, `modul.status`, `saldo.berubah`.

## 5. Kebutuhan non-fungsional

| Aspek | Target |
|---|---|
| Platform | Linux x86_64 (desktop & server); ARM64 best-effort |
| Runtime | .NET 8 self-contained single binary (tidak butuh runtime di target) |
| Database | Fase 1: SQL Server 2022 container; Fase 2: PostgreSQL |
| Packaging | .deb (desktop/umum), Docker Compose (server), tarball, AppImage (opsional) |
| Keamanan | JWT + RBAC sejak MVP; tidak ada kredensial di repo; secret via env/vault |
| Observabilitas | Serilog (file rolling); health endpoint; metrik dasar |
| Biaya tooling | $0 (open-source / bawaan .NET) |

## 6. Kriteria sukses (definisi "jadi")

- [ ] 1 modem: SMS kirim/terima loopback via AT command di Linux.
- [ ] Transaksi pulsa SMS end-to-end: inbox → parsing → routing → eksekusi → status sukses → balasan terkirim.
- [ ] Failover 2 modem terbukti (cabut 1 modem, transaksi tetap jalan via `Alihkan`).
- [ ] Tiket deposit → tambah saldo reseller tercatat benar di mutasi.
- [ ] Komisi MLM terhitung dan bisa ditukar jadi deposit.
- [ ] Dashboard + monitor live berfungsi dari browser (desktop & HP).
- [ ] Build `.deb` terinstall bersih di Ubuntu LTS + Docker Compose jalan di server.

## 7. Risiko produk utama

1. Internet banking diblokir bank → fallback input manual (sudah di arsitektur).
2. WA Web unofficial dipatahkan WhatsApp → fallback notifikasi Telegram.
3. Protokol IP Center belum diketahui → butuh sniffing app Windows.
4. Query SQL exact belum pulih dari binary → rekonstruksi dari skema (11 tabel lengkap).

## 8. Referensi

- `docs/ARSITEKTUR_LINUX.md` — sketsa arsitektur lengkap
- `docs/arch_engine.md`, `docs/arch_db.md`, `docs/arch_ui.md` — detail per domain
- `docs/SCHEMA_DEEPDIVE.md` — skema database hasil RE
- `docs/EDISI_DAN_BATASAN.md` — keputusan tanpa-batasan + struktur edisi asli
- `docs/TARGETED_DECRYPTION.md` — 28 token bernilai tinggi + konteks
