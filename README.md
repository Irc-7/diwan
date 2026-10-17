# OtomaX Linux Clone

Clone fungsional **OtomaX 4.3.1** (software server pulsa/PPOB) untuk Linux — setara edisi Ultimate, **tanpa batasan edisi, tanpa lisensi**.

## Arsitektur

**Opsi B:** engine `.NET 8` Worker Service (systemd) + web UI terpisah (React + AG Grid), komunikasi via REST + SignalR.

```
IngressHub → TxnEngine → ModemManager / H2H Clients → ReplyDispatch
                ↓
        ApiHost (REST /api/v1 + SignalR /hub/monitor) → Web UI (browser)
```

## Status

🚧 **Fase desain** — teardown & arsitektur selesai, implementasi belum dimulai.

## Dokumen

| Dokumen | Isi |
|---|---|
| `PRD.md` | Product Requirements Document |
| `ROADMAP.md` | Tahapan implementasi |
| `docs/ARSITEKTUR_LINUX.md` | Sketsa arsitektur lengkap |
| `docs/arch_engine.md` | Desain engine service (.NET 8) |
| `docs/arch_db.md` | Rencana porting DB → PostgreSQL (DDL lengkap) |
| `docs/arch_ui.md` | Peta 22 layar web + API contract |
| `docs/SCHEMA_DEEPDIVE.md` | Skema database hasil reverse engineering |
| `docs/EDISI_DAN_BATASAN.md` | Struktur edisi asli + keputusan tanpa-batasan |
| `docs/TARGETED_DECRYPTION.md` | 28 token bernilai tinggi (AT command, URL, SQL) |
| `docs/FINDINGS.md` | Hasil awal teardown binary |
| `docs/LAPORAN_GAP_RISET.md` | Riset gap: porting, tooling, OSS |

## Keputusan desain kunci

1. Tanpa batasan edisi — semua modul terbuka, terminal unlimited.
2. Tanpa sistem lisensi/kunci/masa berlaku.
3. 49 `ItemStatus` dipertahankan 1:1 sebagai integer.
4. DB fase 1: SQL Server 2022 container (nol ubah query); fase 2: PostgreSQL.
5. Otomasi WA unofficial dipertahankan (Selenium + WA Web).
6. Deployment: Linux desktop **dan** server (service + browser UI).

## Mulai cepat (rencana)

```bash
# Fase 1 (di masa depan)
dotnet publish -c Release -r linux-x64 --self-contained  # engine
docker compose up -d                                      # SQL Server + engine
# buka http://localhost:5000
```

## Lisensi

Dokumen arsitektur ini adalah hasil reverse engineering untuk interoperabilitas.
Implementasi clone harus ditulis ulang bersih (clean-room) — bukan salinan kode asli.
