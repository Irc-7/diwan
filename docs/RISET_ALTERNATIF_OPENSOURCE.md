# Riset: Alternatif Open-Source Server Pulsa/PPOB untuk Linux

Tanggal: 2026-10-16. Metode: 3 agen riset paralel + pencarian langsung. Read-only, tanpa install/download.

---

## 1. GitHub: proyek open-source server pulsa / PPOB

**Temuan kunci: TIDAK ADA open-source yang mereplikasi engine server pulsa klasik** (ala OtomaX — penerima SMS via modem GSM, pemroses transaksi mandiri). Yang ada jatuh ke 3 pola:

### A. Toko PPOB berbasis H2H (web app, bukan engine modem)
Meneruskan transaksi ke agregator (Digiflazz, Pulsa24Jam), tidak memproses SMS/modem sendiri:

| Repo | Bahasa | Status | Catatan |
|---|---|---|---|
| [kuotakita108-svg/FastPay](https://github.com/kuotakita108-svg/FastPay) | Go + React | Paling aktif (701 commits) | PPOB + kredit agent, H2H Pulsa24Jam, PostgreSQL 16, Docker → Linux OK |
| [dikiafriadi2004/monetrapay](https://github.com/dikiafriadi2004/monetrapay) | Go + Next.js + Flutter | Baru (1 commit) | REST API, MySQL+Redis, Docker, modul WA-bot |
| [Anonymous-Dust/ppob-super](https://github.com/Anonymous-Dust/ppob-super) | PHP 8.1 native, MIT | Baru (1 commit) | Katalog PPOB, role, deposit, provider pluggable (mock/Digiflazz), SQLite/MySQL |
| [shadiew/ppob-digiflazz](https://github.com/shadiew/ppob-digiflazz) | Laravel + Next.js | 1 commit | Terintegrasi Digiflazz |
| [senowahyu62/sebatasteman](https://github.com/senowahyu62/sebatasteman) | Python FastAPI | Melemah (~476 hari) | Kerangka API saja — logika eksekusi masih dummy |

### B. Bridge ke engine yang sudah ada
- [Matsumiko/auvyn-apies](https://github.com/Matsumiko/auvyn-apies) — Node.js/Express, aktif (12 commits, 2025). Bridge backend → "IP Center" (OtomaX/SW): REST API transaksi/saldo, signature SHA1, notifikasi Telegram, monitoring saldo. Dirancang untuk VPS Linux (setup.sh + PM2). **Bukan server mandiri** — butuh engine yang sudah jalan.

### C. Library / SDK (bukan server)
- [idmahbub/ppobcurl](https://github.com/idmahbub/ppobcurl) — PHP/Laravel multi-provider (MobilePulsa, Digiflazz, Tripay…)
- [ciptasolutindotech/ppoblaravel](https://github.com/ciptasolutindotech/ppoblaravel) — helper Laravel untuk request PPOB

### Terkait tapi bukan server pulsa
- [capcom6/android-sms-gateway](https://github.com/capcom6/android-sms-gateway) — SMS gateway Android, REST API, aktif
- [ushahidi/SMSSync](https://github.com/ushahidi/SMSSync) — HP Android jadi SMS gateway

---

## 2. Pasar software server pulsa Indonesia (di luar GitHub)

**Dominasi mutlak: Windows desktop berbayar.** Tidak ada yang gratis/freemium/open-source terverifikasi.

| Software | Harga | Platform |
|---|---|---|
| Tiger Engine ([tigerengine.co.id](https://tigerengine.co.id)) | Rp500rb – Rp8jt | Windows |
| Flash Machine | Rp5jt + Rp125rb/bln | Windows |
| IRS v7 | bervariasi | Windows (+web portal) |
| Tiga Putri ([tigaputri.asia](http://tigaputri.asia/)) | Rp1–5jt | Windows |
| C2 Metro, Roket Pulsa, MD Engine | Rp1–3,5jt | Windows |
| **Guava** (PT Limaguna Utama) | **Rp84–120jt** | **Native Linux (CentOS) — satu-satunya** |
| DigiSwitch ([digiswitch.id](https://digiswitch.id)) | ~Rp600rb/bln | Cloud/SaaS (OS-agnostic) |
| Payuni Cloud ([payuni.co.id](https://www.payuni.co.id)) | Rp4,5jt / sewa Rp1,15jt/bln | Cloud (panel web) |

Catatan:
- Yang berlabel "gratis" di marketplace = crack/keygen ilegal, bukan freemium resmi.
- "M+Pulsa" / "PulsaTronik" **tidak terverifikasi** sebagai software engine (hanya aplikasi agen Android).
- ASPINDO (asosiasi server pulsa) mensyaratkan anggotanya memakai software original buatan lokal — ekosistemnya memang berbayar.
- Komunitas: Kaskus (thread perbandingan FM vs Tiger dkk.), kanal Telegram AddON_OtomaX (addon API bank/e-wallet untuk semua engine).

---

## 3. Komponen open-source untuk membangun server pulsa di Linux

| Komponen | URL | Bahasa / Lisensi | Status | Relevansi |
|---|---|---|---|---|
| **Kannel** | [kannel.org](https://kannel.org/) | C / Kannel License | Stagnan (1.4.5, 2018; hanya distro rebuild) | Driver modem AT + SMPP + HTTP API satu paket |
| **Kamex** (fork Kannel) | [github.com/nickunix7/kamex](https://github.com/nickunix7/kamex) | MIT | Aktif (~47 hari) | Kannel modern: Prometheus, JSON API, Redis |
| **SMSTools3** | [smstools3.kekekasvi.com](http://smstools3.kekekasvi.com/) | C / GPL-2.0 | Maintenance (3.1.21; forum hidup s/d 2026) | Daemon SMS via spool file, s/d 64 modem, eventhandler script |
| **Gammu** | [wammu.eu/gammu](https://wammu.eu/gammu/) | C / GPL-2.0 | Aktif pelan | Library AT command paling matang |
| **python-gammu** | [github.com/gammu/python-gammu](https://github.com/gammu/python-gammu) | Python / GPL-2.0 | **Aktif (3.2.5, Feb 2026)** | Cara tercepat kendalikan modem dari Python (SMS, USSD) |
| **Jasmin** | [github.com/jookies/jasmin](https://github.com/jookies/jasmin) | Python/Twisted / ISC | Aktif pelan (0.11.0 Nov 2023) | SMPP client/server + HTTP API — tapi TANPA driver modem |
| **PlaySMS** | [github.com/playsms/playsms](https://github.com/playsms/playsms) | PHP / GPL-3.0 | Lambat (1.4.7 Des 2023) | Panel web SMS di atas SMSTools3/Kannel/Jasmin |
| **python-smpplib** | PyPI smpplib | Python | Aktif (2.2.4 Jan 2025) | Client SMPP ke aggregator |
| **python-gsmmodem** | asli mati ~2021; fork [python-gsmmodem-2025](https://github.com/jaydenpung/python-gsmmodem) | Python / LGPL-3.0 | Fork aktif (Des 2025) | Wrapper AT praktis: SMS PDU/text, **USSD** (cek pulsa), voice call |
| smstools-http-api | [github.com/lvasiliev/smstools-http-api](https://github.com/lvasiliev/smstools-http-api) | Python | — | REST API di atas spool SMSTools3 |

**PPOB/biller Indonesia: TIDAK ADA yang open-source.** Topik `ppob` di GitHub hanya berisi SDK tidak resmi untuk agregator swasta Digiflazz (butuh akun + deposit). Akses biller resmi (PLN, PDAM) wajib lewat kemitraan agregator.

---

## 4. Estimasi effort membangun dari nol

### Arsitektur yang masuk akal di Linux
```
[Modem GSM] → python-gammu / SMSTools3 → [Transaction Engine] → H2H API agregator
                                                    ↕
                                              PostgreSQL
                                                    ↕
                                            Web UI (ganti 66 WinForms)
```

### Matang (tinggal pakai) vs harus dibangun

| Lapisan | Status | Komponen |
|---|---|---|
| Komunikasi modem (AT/USSD/SMS) | **MATANG** | python-gammu 3.2.5, SMSTools3, python-gsmmodem-2025 |
| SMS gateway massal | **MATANG** | SMSTools3, Kamex |
| HTTP API / web framework | **MATANG** | FastAPI / Go — pekerjaan web standar |
| Database | **MATANG** | PostgreSQL (ganti SQL Server) |
| **Transaction engine** | **HARUS DIBANGUN** | State machine (cf. ItemStatus 49 status), validasi reseller/saldo/produk, routing produk, retry/timeout, anti-dobel — tidak ada padanannya di open-source |
| UI (66 form → web) | **HARUS DIBANGUN** | Dashboard + laporan |
| Koneksi biller | **KEMITRAAN KOMERSIAL** | API Digiflazz/Pulsa24Jam dkk. — bukan soal kode |

### Catatan strategis
- **Model bisnis sudah bergeser**: server pulsa modern Indonesia memakai agregator H2H via API, bukan modem. Membangun engine modem = mereplikasi model legacy. Varian H2H-only jauh lebih sederhana (tanpa lapisan modem sama sekali).
- Komponen open-source menutup ~40–50% pekerjaan (komunikasi, API, DB). **Engine transaksi + aturan bisnis adalah 50–60% sisanya** — dan di sinilah hasil RE OtomaX (skema DB 11 tabel, ItemStatus, TipeModul) menjadi aset utama: ia memberi blueprint domain yang tidak tersedia di tempat lain.
- Tidak ada jalan pintas open-source untuk engine-nya. Pilihan realistis: (a) bangun engine sendiri di atas komponen matang, (b) pakai SaaS (DigiSwitch ~Rp600rb/bln), atau (c) beli lisensi Windows engine yang sudah ada.
