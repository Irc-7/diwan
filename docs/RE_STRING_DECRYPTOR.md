# RE Notes: String Decryptor (inner-assembly.bin)

## Skema enkripsi string
- Encrypted strings di resource `LQtlgGshXDA` (308.623 byte, high-entropy).
- Akses via `ldc.i4 <token>; call #=zx5aF$nyk_UNkW9Lv5JFFqBOPV1uu=.#=zjwaRgdg=(int)`.
- Wrapper (9782): cache `Dictionary<int,string>`, delegasi ke real decryptor (9783).

## Real decryptor (9783, 1111 instr)
1. Nama resource dibangun dari konstanta: `LQtlgGshXDA`
   - loc0=-2116882021, loc1=loc0+(-1870588739)=307496536
   - round1: loc0+(-1203414655)+loc1 → "LQtl"
   - round2: loc0^(-524185429)^loc1 → "gGsh"
   - round3: 1875061915-loc0+loc1 → "XDA"
2. `GetManifestResourceStream("LQtlgGshXDA")`
3. **Anti-tamper StackTrace**: `GetFrame(2).GetMethod().DeclaringType` dibandingkan
   dengan `System.RuntimeMethodHandle` (TypeRef#483). Hanya seleksi branch, tidak
   mengalirkan nilai.
4. Position: `pos = (token ^ ((-2061138166 ^ loc0) + loc1) ^ loc2) ^ ((loc0 + (-1049114581)) - loc1)`
   - loc2 = static field `#=zyv1P4slaVd1K$7$mq8yq1I3GV1Cf` (= -465104809, dari .cctor)
     XOR `(uint32)second_type_check()` lalu transformasi lanjutan.
5. XOR: `data[i] ^= (byte)(key >> (((i & 3) << 3) & 31))` — rotating 4-byte key.
   Key di static field `#=zRZwOk$gFffisYvpjMzsev_zwzLW_`.

## .cctor (9781): komputasi static field = -465104809
- loc1=1470439984, loc3=loc1+543665095=2014105079
- Chain1 (IL_004D-0074): loc0 = (648740973^loc1^loc3 | not/neg chain) XOR ((-1124625853-loc1-loc3 | not/neg chain))
- Chain2 (branch DeclaringType != null, IL_0140-0156): loc0 ^= ...
- Hasil: -465104809 (0xE4471057)
- StackTrace di .cctor: frame(0) = .cctor sendiri, DeclaringType = own type != RuntimeMethodHandle
  → branch "not null" (IL_0140).

## Second type check (#=zqiHjKWotfBaAR9WASnYkdf2J_d96=, method 8713)
- Integrity check: `GetCallingAssembly() == TypeDef#1136.Assembly`?
- Strong-name/public-key-token check.
- Return int64 (kemungkinan 0 bila lolos).
- Perlu analisis lanjutan untuk nilai pasti.

## Status
- [x] Resource name & extraction
- [x] .cctor static field (-465104809)
- [x] XOR scheme (rotating 4-byte)
- [x] Position formula (butuh loc2 final)
- [x] Second type check result (= 0, integrity check lolos)
- [!] ~~BLOCKER: Key derivation iterasi collection yang menyimpan Thread.ManagedThreadId~~ —
  **KOREKSI 2026-10-16 (lihat LAPORAN_GAP_RISET.md Gap 1):** `Thread.ManagedThreadId`
  BUKAN blocker. Satu-satunya kode yang menyentuh thread ID adalah holder .ctor
  (menyimpan ke field) dan satu pembaca (method 4534) yang hanya equality-compare
  (thread-affinity guard klasik) — nilai tidak pernah mengalir ke kunci/plaintext.
  Emulasi dengan fake tid konstan lolos deterministik. Key derivation = aritmetika
  integer murni (xor/not/neg/add), sudah ditangani il_emu.py.
- [!] Nama type/method (#=...) adalah rename permanen — original names HILANG,
  tidak bisa di-recover oleh tool apapun (termasuk de4dotEx).

## Kesimpulan
Dekripsi string via emulasi Python + mock BCL adalah jalur viable ($0, Linux, tanpa
install berat) — user memilih jalur ini 2026-10-16 ("oke perluas emulator", "lanjut
debug emulator" dengan metodologi forge). Opsi: (1) perluas il_emu.py + mock BCL
(StackTrace, Thread, Assembly.GetManifestResourceStream, Dictionary, StringBuilder,
CryptoStream, MemoryStream), (2) dynamic analysis di Windows, (3) terima state saat ini.
