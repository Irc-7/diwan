# Laporan Gabungan: Riset Gap Hasil Reverse Engineering OtomaX

Tanggal: 2026-10-16. Empat agen spesialis, riset read-only (internet + GitHub, tanpa install).

---

## Gap 1 — Dekripsi string protector custom di Linux $0

**Status gap: TERBUKA tapi solvable.**

- Tidak ada tool di Linux yang mendekripsi string .NET tanpa runtime .NET/Mono. Semua parser (dotnetfile, dnfile-rs, ILens) hanya parsing, bukan dekripsi. (Sumber: riset agen 1)
- `de4dot --strtyp delegate --strtok <token>` adalah mekanisme generik untuk protector APAPUN (token decryptor sudah kita identifikasi), tapi butuh Mono ~100–200MB. (de4dot/de4dotEx)
- Teknik komunitas untuk StackTrace anti-tamper: patch check jadi `ret` (via dnlib), atau memory dump. (skill RE .NET)
- **Koreksi penting atas vonis saya sebelumnya:** ketergantungan `Thread.ManagedThreadId` BUKAN blocker dalam emulasi — di emulator, kita yang menentukan return value (cukup return konstanta). Yang tersisa: tambah handler `newobj`/`call`/`callvirt` untuk BCL yang dipakai decryptor (StackTrace, Thread, Assembly.GetManifestResourceStream, Dictionary, StringBuilder, CryptoStream, MemoryStream) sebagai mock. Pola BCL-nya repetitif; `il_emu.py` sudah 80% sampai.
- Rename permanen `#=...` tidak bisa dipulihkan tool apapun (nama asli hilang dari file).

**Rekomendasi:** Jalur A — perluas emulator Python + mock BCL. Satu-satunya jalur yang memenuhi semua constraint ($0, Linux, tanpa install berat).

## Gap 2 — WinForms → operasi Linux

**Status gap: ada jalur jelas, bukan dead-end.**

- Wine 10.x: cukup untuk WinForms standar, tapi packer custom + anti-tamper + WebView2 + P/Invoke app ini = rapuh. Hanya untuk eksperimen.
- Mono WinForms: mati suri (dihibahkan ke WineHQ 2024, setara WinForms 2.0). Tidak viable untuk 66 form.
- Migrasi WinForms→Avalonia: tidak ada konversi otomatis penuh; 66 form ≈ 3–6 bulan satu engineer.
- **Rekomendasi (Opsi B): split architecture** — engine transaksi ke .NET 8 (console/systemd service): SerialPort, ADO.NET, Selenium, Telegram.Bot, HttpClient semua native di Linux. UI 66 form → web (Blazor/SPA) bertahap, mulai dari dashboard monitoring. DB tetap SQL Server di kontainer Linux (nol perubahan query). WebView2 lenyap dari persamaan.
- SQL Server resmi jalan di Linux; migrasi PostgreSQL bisa fase 2 (jebakan: T-SQL→PL/pgSQL, case-sensitivity, datetime→timestamptz).

## Gap 3 — Ekosistem RE berbasis AI agent

**Status: ekosistem hidup, pola pemenang = hybrid.**

- Pola dominan: MCP server di depan IDA/Ghidra (re-mcp, GhidraMCP 7k+ stars, ida-pro-mcp) — didorong riset Cisco Talos.
- Roadmap morluto/rea: deobfuscasi/unpacking masuk radar tapi dibingkai "perbandingan terverifikasi", bukan deobfuscator otomatis. REA tetap orkestrator evidence, tidak akan jadi de4dot-ber-AI.
- Tidak ada pipeline polished "LLM mengemudikan de4dot/FLOSS" — kombinasinya masih manual/ad-hoc. Inilah gap yang diisi `il_emu.py`.
- Benchmark (SRE-Bench 2026): kemampuan source-code LLM tidak transfer ke binary analysis; tim hybrid (7,7) > otonom penuh (5,5). Ancaman baru: prompt injection di malware yang menarget pipeline RE berbasis LLM (Talos Okt 2026, 84 sampel).
- **Validasi pendekatan kita:** emulasi terarah per-target (pola `il_emu.py`) + evidence-attachment adalah pola yang benar; tidak ada produk yang mengemasnya.

## Gap 4 — Alternatif open-source server pulsa/PPOB

**Status: tidak ada yang setara engine modem klasik.**

- GitHub: tidak ada replikasi engine server pulsa ala OtomaX (modem GSM). Yang ada: toko PPOB H2H (web app → agregator: FastPay Go+React paling substansial, monetrapay, ppob-super PHP), bridge (auvyn-apies ke IP Center OtomaX), SDK tidak resmi (Digiflazz).
- Pasar Indonesia: semua Windows desktop berbayar (Tiger Engine, Flash Machine, IRS, dsb.). Native Linux cuma Guava (enterprise, Rp84–120jt). Yang gratis = crack.
- Komponen matang (~40–50% kebutuhan): python-gammu 3.2.5, SMSTools3, Kannel fork (Kamex), python-gsmmodem-2025 (USSD), python-smpplib, Jasmin, PlaySMS.
- **Yang harus dibangun:** transaction engine (state machine 49 status, validasi, routing, retry) + UI web + kemitraan agregator komersial untuk biller (PLN/PDAM tidak ada API open-source).
- Catatan strategis: model bisnis bergeser ke H2H via API — engine modem = model legacy; varian H2H-only jauh lebih sederhana. **Hasil RE kita (skema DB 11 tabel, ItemStatus, TipeModul) adalah blueprint domain yang tidak tersedia di tempat lain** — ini aset utama.

---

## Rekomendasi lintas-gap (jalur maju)

1. **Dekripsi string dulu (Jalur A Gap 1):** perluas `il_emu.py` + mock BCL → bulk decrypt semua `ldc.i4 <token>; call decryptor` → dapat SQL/URL/AT-command. Ini membuka kunci sisa logika bisnis. Estimasi: pekerjaan terarah, bukan riset lagi.
2. **Bangun di atas blueprint, bukan clone biner:** pakai skema DB + ItemStatus + TipeModul sebagai spesifikasi; engine baru di .NET 8/Python sebagai Linux service; komponen modem pakai python-gammu/SMSTools3.
3. **UI sebagai web dashboard** (bukan port 66 form): mulai dari monitoring + laporan.
4. **Keputusan bisnis:** H2H-only (tanpa modem) vs full engine modem — yang pertama 2–3x lebih sederhana dan selaras arah industri.

File detail per agen: `RISET_ALTERNATIF_OPENSOURCE.md` (agen 4). Laporan agen 1–3 diringkas di atas.
