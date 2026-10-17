# ROADMAP — OtomaX Linux Clone

**Estimasi keseluruhan:** multi-bulan (functional clone, bukan pixel-perfect).
Setiap fase punya kriteria selesai yang bisa diuji — bukti, bukan klaim.

## Fase 0 — Validasi (paralel, sebelum komitmen besar)

| # | Spike | Kriteria selesai |
|---|---|---|
| 0.1 | Restore `db_otomax.bak` ke SQL Server Express (container Linux) | DDL eksak 11 tabel + SP + trigger terekstrak |
| 0.2 | `AtChannel` + 1 modem USB di Linux (`AT`, `AT+CMGS`, `AT+CMGL`) | SMS kirim/terima loopback jalan |
| 0.3 | Sniff protokol IP Center (Wireshark vs app Windows) | Format pesan IP Center terdokumentasi |

## Fase 1 — Inti operasional (server bisa jalan)

1. Fondasi: Worker Service + Dapper + SQL Server container + Serilog + ApiHost (health).
2. `AtChannel` + `ModemManager`: 1 modem, health check, auto-reconnect.
3. `TxnEngine` inti: inbox → parsing → validasi → routing → status → balasan (1 channel SMS).
4. Multi-terminal + failover (`Alihkan`) + prioritas + stok.
5. H2H HTTP clients (SpecialHth/OH/WebSMSSender) + Polly.
6. Web UI MVP: Dashboard + Monitor live → Transaksi → Produk + Parsing → Reseller + Mutasi + Tiket Deposit → Monitor Modul.

**Exit criteria Fase 1:** transaksi pulsa SMS end-to-end via 2 modem dengan failover terbukti; dashboard live dari browser.

## Fase 2 — Kelengkapan operasional

- IP Center/Client (hasil sniff 0.3).
- Inbox/Outbox + Auto Respon; Komisi/Mutasi Komisi/Tukar; Poin/Unit.
- Provider/HLR/Level/Cek Harga; Voucher Fisik + Kiosk.
- Internet Banking (Selenium, per bank, + fallback manual).
- Telegram; WA Web unofficial (Selenium); IM sidecar.
- Laporan (komisi, saldo awal, transaksi).
- Migrasi DB → PostgreSQL (DDL siap di `docs/arch_db.md`).

## Fase 3 — Administrasi & packaging

- Pengaturan, Hak Akses (RBAC penuh), Alat Bantu, Log Sistem, Dompet.
- Packaging: `.deb`, Docker Compose, tarball (+ AppImage opsional).
- CI: GitHub Actions → build .deb + docker image per tag.
- PWA / aplikasi Expo (opsional, jalur UI alternatif).

## Ketergantungan antar fase

```
0.1 ─┐
0.2 ─┼─▶ Fase 1 ─▶ Fase 2 ─▶ Fase 3
0.3 ─┘
```

Fase 1 tidak boleh mulai sebelum 0.1 dan 0.2 hijau. 0.3 boleh menyusul (IP Center di Fase 2).
