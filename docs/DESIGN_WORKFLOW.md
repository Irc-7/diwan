# DESIGN WORKFLOW — "OtomaX Design Loop"

Workflow khusus desain frontend proyek ini: adaptasi metodologi **forge**
(brainstorm → design → spec → plan → execute → verify → review → finish)
dengan skill **frontend-design, design-dna, ui-ux-pro-max-skill, motion-design**
dan agen spesialis per tahap. **impeccable TIDAK dipakai untuk pembuatan dashboard**
(keputusan user 2026-10-17) — kritik kualitas diganti checklist ui-ux-pro-max-skill
(ux-guidelines.csv) + verifikasi screenshot. Berlaku untuk semua 22 layar.

> Catatan: shadcn-ui/ui belum terinstall (bukan skill, tapi library komponen —
> dipasang per-proyek via CLI saat tahap Build React).

## Tahap

| # | Tahap | Agen | Skill | Output | Gate |
|---|---|---|---|---|---|
| 0 | Brief | — | — | `docs/DESIGN_BRIEF.md` | brief dikunci |
| 1 | Research | Researcher | ui-ux-pro-max, image-search | referensi + prinsip + 3 arah konseptual | — |
| 2 | Directions | Visual Designer ×3 (paralel) | frontend-design, design-dna | 3 HTML mockup + DNA JSON tiap arah | — |
| 3 | Critique | Critic | ui-ux-pro-max ux-guidelines + screenshot verify (tanpa impeccable) | skor + temuan per arah | — |
| 4 | **User pick** | — | — | 1 arah terpilih | ⛔ user wajib pilih dulu |
| 5 | System | System Designer | design-dna, impeccable (`init`/`document`) | `DESIGN.md` + tokens + komponen | — |
| 6 | Build (22 layar) | Implementer | frontend-design | React + AG Grid | — |
| 7 | Motion | Motion Designer | motion-design (philosophy-first) | motion spec: intent → timing → easing | — |
| 8 | Polish | Reviewer | ui-ux-pro-max guidelines, `harden` manual (tanpa impeccable) | temuan diperbaiki batch | — |
| 9 | Verify | — | screenshot review | screenshot desktop+mobile | ≤2 ronde, lalu stop |

## Aturan mengikat

1. **Brief wins** — brief tertulis mengalahkan selera agen; jangan redirect.
2. **User memilih arah** sebelum tahap 5. Tidak ada implementasi atas rekomendasi saja.
3. **Satu elemen bold** per layar; sisanya disiplin dan tenang (frontend-design).
4. Setiap arah visual = Design DNA 3 dimensi (tokens terukur, style, effects).
5. Motion menjawab aksi user (buka/tutup/konfirmasi); ambient motion hemat.
6. QA bounded: bangun penuh → inspeksi 1 batch → perbaiki → konfirmasi 1 ronde → stop.
7. Copy Indonesia: kalimat aktif, sentence case, error menjelaskan cara perbaiki.
