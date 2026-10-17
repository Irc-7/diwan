# Inventaris CLI Metadata — OtomaX.exe (static analysis)

Tanggal: 2026-10-16 | Analis: subagent .NET metadata
Tool: `tools/cli_metadata.py` (parser Python murni, ~600 baris, tanpa eksekusi target)
Data: `tools/otomax-inventory.json` (lengkap), `tools/client-inventory.json` (validasi)

## 1. Validasi parser (Client.exe — pembanding bersih)

Parser divalidasi terhadap `rea inspect-managed-artifact` (REA v6.4.0,
`/tmp/rea-managed-client.yaml`). Hasil: **14/14 cocok**.

| Check | Parser | REA | Status |
|---|---|---|---|
| CLI header file offset | 520 | 520 | ✅ |
| metadata_rva / size | 8500 / 3188 | 8500 / 3188 | ✅ |
| entry_point token | 0x06000001 | 0x06000001 | ✅ |
| entry_point resolve | `Client.Program.Main` | (parent 0x06000001) | ✅ |
| runtime version | 2.5 | 2.5 | ✅ |
| flags | ILONLY | il-only | ✅ (beda kapitalisasi saja) |
| table counts (14 tabel) | identik | identik | ✅ |
| assembly | Client 1.0.0.0 | Client 1.0.0.0 | ✅ |
| AssemblyRef (3) | mscorlib, System, System.Windows.Forms 4.0.0.0 | sama | ✅ |
| CustomAttribute (22) | 22 nama tipe | 22 nama tipe | ✅ |
| ManifestResource | Client.Properties.Resources.resources, 180 B | sama | ✅ |

Satu perbedaan kosmetik: format GUID. REA mencetak `7977005c-a5a9-4959-…`
(hanya Data1 di-swap little-endian); parser memakai format GUID standar
Windows (`7977005c-a9a5-5949-…`, Data1/2/3 LE). Byte mentah identik.

Catatan: tabel `Constant` tidak muncul di kedua file, jadi lebar kolom
`Constant.Type` (diasumsikan u2 per ECMA-335 II.22.9) belum teruji empiris.

## 2. Hasil: OtomaX.exe BUKAN aplikasi — melainkan .NET packer stub

| Item | Nilai |
|---|---|
| File | 1.599.374 bytes, sha256 `256eb748…196032` |
| PE | e_lfanew=0x40, machine 0x14C (x86), 3 sections, PE32 (0x10B) |
| CLI header | file offset 520, metadata RVA 9828, size 1960, flags `ILONLY` |
| Entry point | token 0x06000004 → **`OtomaX._.Main`** |
| Metadata | v4.0.30319; streams `#~`(620B) `#Strings`(748B) `#US`(52B) `#GUID`(16B) `#Blob`(416B) |
| Assembly | `OtomaX` 4.0.0.0, culture netral, tanpa strong name |
| Module | `OtomaX.exe`, mvid `6b932d2d-0435-432e-85ef-10c5f838e8f1` |

### Tabel metadata (sangat kecil untuk file 1,6 MB)

| Tabel | Rows | Tabel | Rows |
|---|---|---|---|
| Module | 1 | MemberRef | 22 |
| TypeRef | 24 | CustomAttribute | 4 |
| TypeDef | **2** | DeclSecurity | 1 |
| Field | **1** | StandAloneSig | 4 |
| MethodDef | **5** | Assembly | 1 |
| Param | 9 | AssemblyRef | **2** |

### TypeDef (hanya 2)

1. `<Module>` (global)
2. **`OtomaX._`** — flags 0x00100000 (NotPublic + BeforeFieldInit), extends
   `System.Object`. Satu-satunya tipe aplikasi; nama sudah di-rename ke `_`.

Namespace (hanya 2, jadi "top 20" = seluruhnya): `<global>` 1, `OtomaX` 1.

### MethodDef (5) — inilah loader-nya

| # | Nama | IL size | Peran (dari MemberRef yang dipakai) |
|---|---|---|---|
| 1 | `lzmat` | 900 B | dekompresi LZMA — params `(pOs, pIs, cI)` |
| 2 | `lz` | 91 B | wrapper `(c, a, l)` |
| 3 | `lf` | 333 B | "load file": baca overlay via `FileStream`+`BinaryReader` (`ReadInt32/UInt32/UInt16`) — params `(fn, a)` |
| 4 | `Main` | 154 B | entry: `Application.get_ExecutablePath` → `lf` → `lz`/`lzmat` → `Assembly.Load(byte[])` → `get_EntryPoint` → `Invoke`; `MessageBox.Show` untuk error |
| 5 | `.ctor` | 7 B | konstruktor statis |

Tidak ada method P/Invoke (0), tidak ada method native (0).

### Field (1)

`mp` — private static (flags 0x0011). Satu-satunya field di stub.

### AssemblyRef (2) — SANGAT revealing

1. `mscorlib` 4.0.0.0 (pkt `b77a5c561934e089`)
2. `System.Windows.Forms` 4.0.0.0 (pkt sama)

**Tidak ada `System.Data`** — padahal aplikasi aslinya memakai SQL Server.
Referensi database/UI sesungguhnya tersembunyi di dalam payload terenkripsi.
Stub loader hanya butuh IO + Reflection + WinForms (untuk path exe & message box).

### CustomAttribute (4)

| Token | Parent | Tipe |
|---|---|---|
| 0x0C000001 | Module `OtomaX.exe` | `System.Security.UnverifiableCodeAttribute` |
| 0x0C000002 | Assembly `OtomaX` | `System.Runtime.CompilerServices.CompilationRelaxationsAttribute` |
| 0x0C000003 | Assembly `OtomaX` | `System.Runtime.CompilerServices.RuntimeCompatibilityAttribute` |
| 0x0C000004 | `OtomaX._.Main` | `System.STAThreadAttribute` |

`UnverifiableCodeAttribute` di level modul + `DeclSecurity` di bawah adalah
penanda khas stub protector (bukan kode aplikasi normal).

### DeclSecurity (1)

Action `0x0008` = `SecurityAction.RequestMinimum` pada Assembly, blob 158 B
berisi `SecurityPermissionAttribute, mscorlib, Version=4.0.0.0, …` — loader
meminta minimum permission set (full-trust-ish) agar bisa `Assembly.Load`
dari byte array.

### Resources / P/Invoke

- ManifestResource: **tidak ada**. MethodImpl: 0. ImplMap: **tidak ada**.
- Payload TIDAK disimpan sebagai managed resource — melainkan sebagai
  **overlay** (lihat §3).

### Statistik heap

- `#US`: **1 string** (16 char) — heap string user praktis dikosongkan;
  semua string aplikasi dipindah ke blob terenkripsi (konsisten dengan
  temuan string encryption `#=`+custom-base64).
- `#Strings`: 96 entri (nama tipe/metode stub + mscorlib typerefs).
- `#Blob`: 416 B (signature + DeclSecurity + CA blobs).

## 3. Overlay: aplikasi asli (1.589.134 bytes)

- Section terakhir (`.reloc`) berakhir di file offset `0x2800` (10240);
  ukuran file 1.599.374 → **overlay 1.589.134 bytes (99,4% file)**.
- Entropi 64 KB pertama overlay: **7,72 bits/byte** → terenkripsi/terkompresi.
- 258 kemunculan blob string terenkripsi prefix `#=` di dalam overlay
  (dua varian: biner high-entropy dan ASCII base64-variant, mis.
  `#=zCrAjTPE$5ooa$dzG5VlNd`).
- Alur loader (direkonstruksi dari MemberRef): baca file exe sendiri →
  parse header kecil via `BinaryReader` → ekstrak + dekripsi/dekompresi
  (`lzmat`) → `Assembly.Load(byte[])` → invoke entry point asli.

**Kesimpulan arsitektur:** OtomaX.exe = stub loader terproteksi
(±1,5 KB IL) + aplikasi VB.NET asli yang dienkripsi sebagai overlay.
Untuk melihat kode aplikasi (termasuk referensi `System.Data`,
ratusan tipe UI, dan string yang terdekripsi), langkah berikutnya adalah
**unpacking dinamis**: jalankan stub di sandbox Windows/.NET,
dump byte array yang di-`Assembly.Load` (atau hook `Assembly.Load`),
lalu analisis hasil dump dengan parser ini / dnSpy / de4dot
(de4dot mendukung banyak protector; identitas protector spesifik
belum teridentifikasi dari signature publik).

## 4. Mengapa REA melihat "dos-mz" padahal PE valid — HIPOTESIS TERKONFIRMASI DARI SOURCE

Bukan tebakan: dilacak langsung di source `rea-agents` yang terinstal.

**Lokasi:** `~/workspace/tools/rea/node_modules/rea-agents/dist/domain/dosMz.js`,
fungsi `mzWindowsHeaderOffset` (dipakai oleh `parseMz` di
`dist/domain/binaryTarget.js`):

```js
export const mzWindowsHeaderOffset = (bytes) => {
    if (bytes.length < 64)
        return null;
    const headerBytes = bytes.readUInt16LE(8) * 16;   // e_cparhdr * 16
    if (headerBytes !== 0 && headerBytes < 64)
        return null;                                  // <-- PENYEBAB
    ...
    const offset = bytes.readUInt32LE(60);            // e_lfanew @ 0x3C
    return offset === 0 ? null : offset;
};
```

Komentar di source menjelaskan maksudnya: *"without interpreting a short
DOS header's load-module or relocation-table bytes at 0x3c as e_lfanew"* —
heuristic fail-closed agar byte relocation table tidak disalahartikan
sebagai pointer PE header.

**Jejak eksekusi untuk OtomaX.exe:**
1. Stub DOS custom protector: `e_cparhdr = 2` → `headerBytes = 32 < 64`
   → fungsi return `null` (Client.exe: `e_cparhdr = 4` → 64 → lolos).
2. `parseMz` (binaryTarget.js): karena offset null → fallback ke
   `parseDosMzHeader`, yang **lolos validasi** sebagai image DOS murni
   (pages=1, headerBytes=32 ≥ 28, tanpa relokasi, entry point valid)
   → format diklasifikasikan **`"dos-mz"`**.
3. `ManagedStaticProvider` (dist/dotnet/ManagedStaticProvider.js:44):
   `if (this.target.format !== "pe") return err("Managed static triage
   requires a PE target; observed dos-mz")`.

Catatan: guard kedua di `parseMz` (`if (offset < 64 …) return err`)
tidak relevan di sini — `0x40 < 64` bernilai false; kegagalan murni
dari heuristic `dosMz.js`.

**Apakah bisa diatasi via GitHub? YA.** Ini bug/heuristic di repo
`morluto/rea`, dapat dilaporkan sebagai issue dengan materi:

- *Judul:* `managed provider rejects valid PE with minimal custom DOS stub (e_cparhdr*16 < 64) as dos-mz`
- *Isi:* `mzWindowsHeaderOffset` di `dist/domain/dosMz.js` me-return null
  untuk stub DOS valid yang pendek (`e_cparhdr=2`), sehingga PE dengan
  `e_lfanew=0x40` dan signature `PE\0\0` yang valid terklasifikasi
  `dos-mz`; akibatnya `inspect-managed-artifact` gagal dengan
  "requires a PE target; observed dos-mz". File pembanding dengan
  `e_cparhdr=4`/`e_lfanew=0x80` lolos normal.
- *Usulan fix:* sebelum return null untuk short header, coba validasi
  signature PE di offset 0x3C (`parsePeRecord`) — terima sebagai PE bila
  valid; hanya fallback ke dos-mz bila tidak. Atau: izinkan
  `headerBytes >= 28` (ukuran minimum DOS header) dikombinasikan dengan
  pemeriksaan signature.
- *Workaround sementara (tanpa ubah REA):* tidak diperlukan dari sisi
  kita — parser `tools/cli_metadata.py` sudah membaca metadata kedua
  file dengan benar; REA tetap bisa dipakai untuk file-file
  ber-stub standar.

## 5. File deliverable

- `~/workspace/rea-work/otomax/tools/cli_metadata.py` — parser (divalidasi 14/14 vs REA)
- `~/workspace/rea-work/otomax/tools/otomax-inventory.json` — inventaris OtomaX.exe
- `~/workspace/rea-work/otomax/tools/client-inventory.json` — inventaris Client.exe
- `~/workspace/rea-work/otomax/CLI_METADATA_INVENTORY.md` — laporan ini (juga di-save sebagai file)
