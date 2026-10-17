# Improvements — ShipMate → OtomaX (improved)

Mockup: `index.html` · DNA: `dna.json` · 17 Okt 2026 · Bahasa Indonesia

---

## Design plan (two-pass, ringkas)

**Pass 1 — plan:**
- **Warna:** kertas hangat light (`#fbfbfa`/`#ffffff`) / arang hangat dark (`#161311`/`#201c18`); satu oranye `#f66e27` → dark dikalibrasi `#ff8f52`; status dipisah per makna — hijau sukses, amber pending, merah gagal.
- **Tipe:** Plus Jakarta Sans untuk semua UI; JetBrains Mono untuk kode, waktu, nomor HP, nominal, port, log AT — "suara buku log".
- **Layout (1 kalimat):** Meja NOC jam 2 pagi — strip alarm berteriak sekali di atas, lalu operator memindai log mono yang rapat, diapit kesehatan hardware di rel kanan.
- **Prinsip keunikan:** motion = sinyal, bukan dekorasi; kepadatan tabel adalah fitur; Bahasa Indonesia aktif, sentence case.

**Pass 2 — review vs brief (yang direvisi):**
1. Risiko generik: "4 KPI card + chart + tabel" = kit SaaS klise. Revisi → KPI dibuat borderless dengan aksen 3px warna-makna + angka mono (rasa ledger, bukan kartu); tabel dikompresi ke 31px/baris seperti log Excel; strip krisis full-bleed solid merah **memecah** ritme kartu — grid yang rusak di satu titik disengaja.
2. Satu elemen berani saja → strip krisis. Donat referensi yang "cantik" (4 shade oranye) dikorbankan: makna > estetika.
3. Tanpa eyebrow ALL-CAPS, tanpa fade-slide-up per section — keduanya adalah tell AI-slop yang disebut brief.

---

## 7 improvements terpenting vs referensi

1. **Donat memakai warna semantik, bukan 4 shade oranye.** Referensi memakai skala oranye (hero `#f28650` → peach pucat) untuk semua segmen — "Delayed" terbaca sebagai "sukses yang lebih muda". Di sini: hijau `#2e9e5b` sukses, amber `#cf8617` pending, merah `#cf3d3d` gagal, di donat, badge, bar sinyal, dan log sekaligus. Alasan: di layar operator malam, salah baca status = salah tindakan; makna harus dikodekan hue, bukan lightness.
2. **Dark mode dirancang ulang, bukan invert.** Oranye `#f66e27` di atas hitam menyilaukan → dikalibrasi ke `#ff8f52` (lightness naik, saturasi turun). Permukaan memakai arang hangat `#161311`, bukan hitam netral — kehangatan ShipMate dipertahankan. Shadow diperdalam, teks sekunder diganti total. Alasan: operator shift malam memakai dark mode berjam-jam; invert murahan melelahkan mata dan membunuh personality.
3. **Strip krisis yang tegas, bukan sopan.** Referensi tidak punya bahasa alarm ("Pending" ditulis pink lembut). Di sini: banner solid `#b3261e` full-width dengan pulse 2,6 detik: "Modem 3 (Tri) down 12 menit — 47 transaksi pending menumpuk di antrean. Pindahkan beban ke modem cadangan." + tombol "Buka terminal". Alasan: satu-satunya elemen berani di layar; krisis infrastruktur harus mustahil diabaikan.
4. **Kontras teks kecil diperbaiki (≥ 4,5:1).** `#767169` referensi hanya ~4,1:1 di atas `#fbfbfa` → diganti `#5b564e` (~6,8:1) untuk light; dark memakai `#b3aa9c` (~7,3:1). Badge status memakai teks gelap di atas tint terang (bukan teks berwarna di atas putih). Alasan: operator membaca angka kecil berjam-jam; kontras lemah = kelelahan + salah baca.
5. **Kepadatan tabel operasional, bukan "airy".** Referensi: baris ~40px, tabel wilayah paling renggang. Di sini: 31px/baris, 12,5px, ~20 baris terlihat, header sticky, baris baru masuk dengan flash oranye 1,6 detik (bukan fade-slide). Alasan: brief menuntut "grid data rapat ala Excel, scanability > ekspresi" — operator memindai pola, bukan mengagumi whitespace.
6. **JetBrains Mono sebagai suara data.** Referensi tidak punya mono. Kode transaksi, timestamp, nomor HP, nominal, port `/dev/ttyUSBx`, dan log AT semuanya mono tabular. Alasan: membedakan "data mesin" dari "bahasa manusia" dalam satu pandangan; sekaligus menghilangkan rasa SaaS generik.
7. **Bahasa Indonesia aktif, sentence case.** "Jeda live" (bukan "PAUSE LIVE"), "Coba sambungkan ulang", "Tidak ada transaksi yang cocok dengan filter." Tombol memakai verba aksi yang menjelaskan akibatnya. Alasan: brief menetapkan Bahasa Indonesia; kalimat aktif mempercepat keputusan di bawah tekanan shift malam.

## Perubahan pendukung

- **Toggle tema berfungsi penuh** — persist `localStorage`, label berubah ("Mode gelap/terang"), ikon matahari/bulan, knob beranimasi; default light (mengikuti referensi ShipMate), dark sebagai pilihan operator malam.
- **Badge Live berdenyut + jam WIB ticking** (`Asia/Jakarta`, tiap detik) + tombol "Jeda live" yang benar-benar menghentikan feed (badge jadi "Dijeda", tombol jadi "Lanjutkan").
- **Filter chips dengan hitungan live** (Semua/Sukses/Pending/Gagal) + pencarian yang memfilter semua kolom + donat/KPI ikut bergerak saat data baru masuk.
- **Panel Modem & Terminal** sebagai terjemahan "Call the courier": 4 modem dengan port, bar sinyal 4 segmen, badge Terhubung/Lemah/Down, CSQ mono, log AT bergulir, tombol "Coba sambungkan ulang" dengan respons simulasi.
- **Grafik 7 hari digambar sebagai SVG murni** (tanpa dependensi) dengan tooltip hover per titik; tidak ada library chart.
- **Aksesibilitas lantai:** `prefers-reduced-motion` mematikan semua animasi; `:focus-visible` oranye; `role="alert"` pada strip krisis; `aria-pressed`/`aria-selected` pada kontrol.

---

## Catatan trade-off desain

1. **Satu hero vs kekayaan visual.** Strip krisis memonopoli keberanian visual — donat dan grafik sengaja dibuat "membosankan" agar alarm selalu menang. Di hari tanpa krisis, layar terasa datar; itu disengaja (ruang kontrol yang tenang = sistem sehat).
2. **Kepadatan vs keterbacaan mobile.** 20 baris × 8 kolom tidak muat nyaman di HP; mockup ini desktop-first (operator memakai HP hanya untuk monitoring — kolom Tujuan/Reseller bisa disembunyikan di breakpoint kecil pada implementasi nyata).
3. **Simulasi vs data nyata.** Feed, jam, log AT, dan donat disimulasikan via `setInterval` sebagai pengganti SignalR. Angka KPI/donat konsisten satu sama lain, tetapi tidak ada reconnect/backoff sungguhan — itu pekerjaan lapisan SignalR nanti.
4. **Oranye tetap satu-satunya aksen.** Disiplin referensi dipertahankan (tidak menambah aksen kedua), sehingga oranye hanya untuk: logo, nav aktif, grafik garis, tombol Jeda (saat dijeda), dan flash baris. Risikonya: tombol primer lain harus memakai ink netral agar oranye tidak kehilangan makna.
5. **Warm-dark bukan tren.** Kebanyakan dark mode memakai abu netral/kebiruan; arang hangat `#161311` dipilih untuk kontinuitas dengan light mode. Di monitor murah, warm-dark bisa terlihat "kotor" — perlu uji di layar operator sungguhan.
