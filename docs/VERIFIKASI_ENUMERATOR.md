# Verifikasi Enumerator: Laporan Analisis Independen

Tanggal: 2026-10-16 | Metode: Implementasi Python langsung dari IL (bukan emulasi), mengikuti forge debug.md Phase 1-2.
Script verifikasi: [tools/verify_enumerator.py](sandbox://workspace/rea-work/otomax/tools/verify_enumerator.py)

## (a) Pseudocode Enumerator

**TypeDef 615** — nested type `#=zOuB6d6qMa3hIaqs_wyLToDdCAEc8ptCK3w==+#=z3NnAeBM=`, dipakai decryptor (9783) di IL_0345-038F.

Fields: `state` (#=zMzMLh5U=), `thread_id` (#=z3NnAeBM=), `qj238ik` (#=zQJ238ik=),
`heVB3kg` (#=zheVB3kg=), `inner` (#=zd9dLGEU=), `Current` (#=zd3$qn2I=), `counter` (#=zSmpCn74=).

```python
# 4527 .ctor(int arg1)
def __init__(self, arg1, tid):
    self.state = arg1
    self.thread_id = tid          # Thread.CurrentThread.ManagedThreadId

# 4534 GetEnumerator()
def GetEnumerator(self, current_tid):
    if self.state != -2 or self.thread_id != current_tid:
        new = Nested(0, current_tid)
        new.qj238ik = self.heVB3kg
        return new
    self.state = 0
    return self                    # return this!

# 4529 MoveNext()
def MoveNext(self, tid):
    if self.state == 0:            # init
        self.state = -1
        self.counter = 7
        c = Nested(-2, tid)
        c.heVB3kg = self.qj238ik
        self.inner = c.GetEnumerator(tid)   # -> c (state=0)
        self.state = -3
    elif self.state == 1:          # resume
        self.state = -3
        if self.counter == 0: return False
    elif self.state != -3:
        return False
    while True:                    # loop IL_00B6
        if not self.inner.MoveNext(tid): return False
        loc2 = (self.inner.Current ^ self.qj238ik) & 0xFFFFFFFF
        if (loc2 & 3) == 0:
            loc2 ^= 2005660044
            self.counter -= 1
        if (loc2 & 15) == 0:
            loc2 ^= 0xFFFFFFFF & -1189707964
        self.Current = loc2
        self.state = 1
        return True

# 4531 get_Current() -> int:  return self.Current
# 4533 get_Current() -> object: return box(self.Current)
# 4528 Dispose(), 4530, 4532 Reset() -> throw, 4535 -> calls 4534
```

**Pemakaian di decryptor (IL_0345-038F):**
```python
A = Nested(loc50, tid); A.heVB3kg = loc50
E = A.GetEnumerator(tid)           # -> B = new Nested(0), B.qj238ik = loc50
while E.MoveNext(tid):
    cur = E.Current
    loc12 = loc12 ^ (cur - loc23)
    loc23 = loc23 - ((loc12 + 3) >> 8)
```

## (b) Sequence yang Seharusnya

**Hasil verifikasi independen: INFINITE RECURSION.**

Untuk input loc50=23225736, thread_id=1, implementasi langsung menghasilkan
`RecursionError` pada depth > 100. Tidak ada sequence valid yang dapat dihitung.

Alur yang menyebabkan rekursi tak berujung:
1. `B.MoveNext()` (state=0) → buat `C = new Nested(-2)` → `C.GetEnumerator()` return C → `C.MoveNext()`
2. `C.MoveNext()` (state=0) → buat `D = new Nested(-2)` → `D.GetEnumerator()` return D → `D.MoveNext()`
3. `D.MoveNext()` → buat E → ... **tidak ada base case.**

Setiap level membuat instance baru dan memanggil MoveNext-nya. Tidak ada kondisi
yang menghentikan rantai ini.

## (c) Bug dalam Pemahaman Kita — YA, FUNDAMENTAL

**Temuan kunci:** Klaim emulator "7 iterasi, loc12=3519711043" **tidak valid**.

Bukti:
1. Implementasi Python langsung dari IL yang *sama* menghasilkan infinite recursion, bukan 7 iterasi.
2. Agar emulator menghasilkan 7 iterasi, ia pasti tidak mengeksekusi rekursi MoveNext dengan benar.
3. Kemungkinan bug di emulator:
   - Virtual dispatch `callvirt` untuk inner MoveNext (interface method 8917 → impl 4529) mungkin tidak bekerja sebagaimana dikira, atau
   - `emulate_method` untuk GetEnumerator (4534) tidak mengembalikan `this` dengan benar, atau
   - Exception/step-limit ditangkap diam-diam sehingga rekursi terpotong.
4. Nilai `loc12 = 3519711043` yang ditangkap dari trace emulator **tidak dapat dipercaya** dan jangan dipakai untuk dekripsi.

**Kemungkinan penyebab kesalahpahaman:**
- Kita mungkin salah mengira target dari `callvirt <TypeSpec[33]>.GetEnumerator` — bisa jadi bukan 4534 melainkan method lain yang tidak rekursif.
- Atau ada kondisi thread-ID yang seharusnya gagal di level dalam (tapi kita mock selalu cocok).
- Atau enumerator ini adalah dead code / anti-analysis yang memang dirancang untuk menggantung analis.

**Rekomendasi:**
1. Jangan gunakan nilai loc12/key dari emulator untuk dekripsi.
2. Cari pola kode asli sebelum obfuscation (kemungkinan loop sederhana 7x dengan fungsi deterministik).
3. Pertimbangkan bahwa enumerator mungkin tidak perlu diemulasi sama sekali — cari jalur yang menghasilkan key tanpa enumerator.
4. Jika lanjut debug emulator: instrumentasikan setiap panggilan GetEnumerator/MoveNext dengan depth dan instance ID untuk membuktikan di mana rekursi terpotong.
