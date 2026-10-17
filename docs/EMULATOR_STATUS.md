# Status Perluasan Emulator (2026-10-16)

## Yang sudah dibangun

**`tools/il_emu2.py`** — emulator CIL extended (~900 baris):
- Model objek: `Obj` (type_name + fields dict), static fields, instance fields
- Opcode: newobj, call/callvirt (dispatch ke emulasi rekursif atau mock BCL),
  ldfld/stfld, ldsfld/stsfld, ldtoken, box, castclass, newarr, ldelem/stelem,
  ldind/stind, throw, leave/endfinally (simplified), switch, semua branch & aritmatika
- Mock BCL: Dictionary, StackTrace/StackFrame/MethodBase/Type/Assembly,
  StringBuilder, Monitor, Thread (get_ManagedThreadId → 1, konstan!),
  Stream (Read/ReadByte/set_Position), Encoding, Convert, List, Buffer.BlockCopy
- Virtual dispatch untuk interface method tanpa body
- TypeSpec dispatch untuk generic (Dictionary<int,string>, List<byte>, enumerator)

**`tools/test_decrypt.py`** — harness uji dekripsi satu token.

## Yang berhasil

1. ✅ Emulasi `.cctor` (method 9781): 3 static terinisialisasi
   - Dictionary cache: `{}`
   - `#=zNPYjARt3G29gqthx1Y9VBsx6rg2`: 16
   - `#=zyv1P4slaVd1K$7$mq8yq1I3GV1C`: 4143779927
2. ✅ Melewati StackTrace anti-tamper (mock mengembalikan frame valid)
3. ✅ Emulasi enumerator nested type (state machine MoveNext, GetEnumerator, Current)
4. ✅ Mencapai fase dekripsi (Stream positioning, length read)
5. ✅ Mengidentifikasi 11.421 token unik dari 23.949 call-site

## Blocker saat ini

Fase dekripsi akhir gagal karena nilai yang dihasilkan emulasi tidak tepat:
- Position: 2.463.195.425 (harusnya < 308.623)
- Length: kadang 1.8 miliar (harusnya kecil)
- Key derivation dari enumerator mungkin tidak tepat

Akar masalah: obfuscasi dirancang untuk mempersulit emulasi statis.
- Enumerator rekursif dengan state machine kompleks
- Aritmatika `not`/`neg` berlapis
- Ketergantungan antar nilai yang halus

## Opsi lanjut

A. **Lanjut debug emulator** — butuh analisis mendalam tiap nilai yang salah.
   Estimasi: jam kerja tambahan, hasil tidak pasti.

B. **Ekstrak formula manual** — baca IL untuk dapatkan rumus position/key
   yang tepat, lalu implementasi ulang bersih di Python (tanpa emulasi).
   Lebih terkontrol, tapi butuh reverse manual yang teliti.

C. **Targeted decryption** — fokus ke string bernilai tinggi saja
   (SQL query, URL, AT command) via analisis caller context.
   Mungkin tidak perlu semua 11.421 string.

D. **Jalur Wine + dynamic** — jalankan decryptor di Wine dengan .NET,
   dump string dari memory. Butuh setup Wine (berat, rapuh).
