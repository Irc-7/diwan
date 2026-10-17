# DESIGN BRIEF — OtomaX Linux Web UI

**Tanggal:** 2026-10-17 | **Mode (impeccable):** Operate — pengunjung menyelesaikan tugas, bukan dikagumi.
**Referensi produk:** `PRD.md`, `docs/arch_ui.md`, `docs/mockup/dashboard.webp` (mockup v1, boleh jadi anti-referensi).

## Produk & pengguna

Web UI untuk operator server pulsa/PPOB (pengganti 66 WinForms → ±22 layar).
Pengguna: pemilik server, operator harian (shift malam), teknisi lapangan. Bahasa Indonesia.

## Layar kunci (MVP)

1. Dashboard + Monitor transaksi live (layar ini yang di-mockup duluan)
2. Transaksi (grid Excel-like, filter cepat)
3. Produk + Parsing, Reseller + Mutasi + Tiket Deposit + Data Bank
4. Monitor Modul & Terminal (status modem, AT log)

## Batasan desain

- Dark theme (operator kerja malam; konsisten dengan mockup v1).
- Real-time via SignalR: update status tanpa refresh; motion harus komunikatif, bukan dekoratif.
- Grid data rapat ala Excel (AG Grid): densitas tinggi, scanability > ekspresi.
- Responsif: desktop utama, HP operator untuk monitoring.
- $0 tooling: font gratis (Google Fonts / system stack), tanpa aset berbayar.
- Hindari default AI-slop (frontend-design): SaaS-card kit generik, aksen acid-green tunggal, eyebrow ALL-CAPS di mana-mana, fade-slide-up di tiap section.

## Keputusan arah (dikunci user 2026-10-17)

- **UX: Arah A "Terminal Operasi"** (`docs/mockup/dir-a/`) — dense ops-console, keyboard-first, log AT.
- **Token warna: dari ShipMate-improved** (`docs/mockup/shipmate-improved/dna.json`) — palet terukur, dark-first `#161311`, aksen `#ff8f52`, kontras teks terukur.
- **Tanpa impeccable** — kritik pakai ui-ux-pro-max ux-guidelines + verifikasi screenshot.
