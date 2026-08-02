# [Mobile] Overstrike — CTF Writeup

## Deskripsi Soal

> **Kategori:** Mobile
> **Nama Challenge:** Overstrike
> **File:** `mobile_overstrike.zip` (berisi `Overstrike.apk`)

**Deskripsi Soal:**

> The Signet is shattered, and its silence has a shape now: House Vaultrune's quiet trade in re-cut marks, genuine authority lifted and struck anew, each forgery slipped into the Registry so the lie reads as record. Eliric goes down under Crownspire, toward the sealed archive Chancellor Veylen Marr keeps, and finds the old vow-stones of the Ash-Vault will not bridge the fire-rift for any mark he carries. They answer only to the true seal. To reach the truth the forgers have filed as fact, Eliric must do the very thing that condemns them, and cross on a seal of his own making, one the world itself cannot tell from genuine.

Singkatnya: kita diberi sebuah APK game buatan **Godot Engine (C#/Mono)**. Cerita di dalam soal sebenarnya adalah petunjuk tersamar untuk mekanisme teknisnya — kita harus **memalsukan (forge) sebuah "seal"** (nilai numerik) agar dunia game menganggapnya asli (`WorldIsAligned == true`), lalu membaca "Registry" yang ter-enkripsi di dalamnya untuk mendapatkan flag.

---

## Tools yang Dipakai

| Tool | Fungsi |
|---|---|
| `unzip` | Ekstrak file APK (APK pada dasarnya adalah file ZIP) |
| `mono-utils` (`monodis`) | Disassembler IL untuk file `.dll` (.NET/Mono) |
| Python 3 + `dnfile` | Parsing metadata `.NET` assembly, terutama untuk membaca **raw bytes** dari field statis yang tidak ikut ter-dump oleh `monodis` |
| Python 3 (`hashlib`, `struct`) | Reimplementasi ulang algoritma kriptografi (SplitMix64 & skema XOR-keystream berbasis SHA-256) untuk membalikkan (invert) dan mendekripsi data |

> Catatan: Awalnya saya ingin memakai **ILSpy/`ilspycmd`** agar hasil decompile langsung berupa C# yang rapi, tapi tidak tersedia di environment. Solusinya, saya pakai `monodis` untuk melihat IL (bytecode .NET) secara manual — sedikit lebih effort, tapi masih cukup terbaca untuk kasus ini.

---

## Step-by-Step Penyelesaian

### 1. Ekstrak APK

APK adalah file ZIP biasa, jadi bisa langsung diekstrak:

```bash
mkdir overstrike && cd overstrike
unzip -o Overstrike.apk -d extracted
```

Hasil eksplorasi struktur folder menunjukkan ini adalah **game Godot** (ada folder `assets/.godot`, `assets/scenes`, `assets/scripts`, dan `project.binary`), bukan APK Android native biasa.

### 2. Cari source code / logic utama

Di dalam `assets/scripts/` ada file-file `.cs` seperti `GameState.cs`, `Archive.cs`, `MarkPickup.cs`, dll — tapi ternyata isinya **kosong (1 baris saja)**. Ini adalah file *remap* placeholder, bukan source code asli.

```bash
cd extracted/assets/scripts
for f in *.cs; do echo "=== $f ==="; wc -l $f; done
```

Source code C# yang sebenarnya sudah **dikompilasi** menjadi assembly `.dll` untuk runtime Mono. Assembly ini ditemukan di:

```
assets/.godot/mono/publish/arm64/Overstrike.dll
```

Inilah target utama untuk di-reverse.

### 3. Decompile / Disassemble `Overstrike.dll`

Karena `ilspycmd`/`dotnet` tidak tersedia, saya install `mono-utils` (menyediakan `monodis`) via `apt`:

```bash
apt-get update
apt-get install -y mono-utils
```

Lalu disassemble ke bentuk IL (CIL) yang bisa dibaca manusia:

```bash
monodis --output=Overstrike.il Overstrike.dll
```

Dari sini saya grep class-class penting dan menemukan struktur class `GameState` — nama yang sangat mencurigakan sebagai pusat logic "seal/mark" di cerita soal.

### 4. Analisis logic `GameState`

Dari hasil IL, `GameState` punya field dan method kunci berikut (saya tuliskan ulang dalam bentuk C# semu agar mudah dibaca):

```csharp
public class GameState : Node
{
    public ulong CarriedMark;
    public ulong WorldSeal;
    public const ulong TrueSeal = 0xd9a1bb0cabb52586;
    private static readonly byte[] SealedRecord; // 56 byte, data biner statis

    public override void _Process(double delta)
    {
        WorldSeal = Mix(CarriedMark);
    }

    public bool WorldIsAligned => WorldSeal == TrueSeal;

    public static ulong Mix(ulong x)
    {
        x += 0x9E3779B97F4A7C15;
        x = (x ^ (x >> 30)) * 0xBF58476D1CE4E5B9;
        x = (x ^ (x >> 27)) * 0x94D049BB133111EB;
        x = x ^ (x >> 31);
        return x;
    }

    public string UnsealRegistry()
    {
        // key = SHA256(CarriedMark sebagai 8 byte little-endian)
        // Loop: block_i = SHA256(key + counter_i sebagai 4 byte little-endian)
        // XOR-kan byte SealedRecord dengan keystream block_i secara berurutan
        // Ubah hasil byte menjadi karakter (byte non-printable diganti simbol placeholder)
    }
}
```

**Poin penting yang saya sadari:**

1. `Mix()` **persis** merupakan fungsi *finalizer/mixer* dari algoritma **SplitMix64** (konstanta `0x9E3779B97F4A7C15`, `0xBF58476D1CE4E5B9`, `0x94D049BB133111EB`, beserta pola shift 30/27/31 adalah signature khasnya).
2. Fungsi ini bersifat **bijektif** (satu-ke-satu) — artinya bisa **dibalik (invert)**. Ini relevan dengan narasi soal: *"cross on a seal of his own making, one the world itself cannot tell from genuine"* → kita perlu **mencari nilai `CarriedMark`** sedemikian rupa sehingga `Mix(CarriedMark) == TrueSeal`, alih-alih mencoba menebak/brute-force.
3. `UnsealRegistry()` mendekripsi array `SealedRecord` (56 byte) menggunakan `CarriedMark` yang sama sebagai basis kunci — jadi begitu kita berhasil forge `CarriedMark` yang benar, kita otomatis bisa mendekripsi flag-nya juga.

### 5. Ekstrak raw bytes `SealedRecord` dari DLL

`monodis` hanya menampilkan **referensi token** ke data biner (`ldtoken field ... at D_00010aa0`), bukan isi bytes-nya secara langsung. Untuk mengambil raw bytes tersebut, saya pakai library Python `dnfile` yang bisa membaca metadata `.NET` assembly secara terprogram, termasuk tabel `FieldRva` (pemetaan field statis ke lokasi data mentahnya di file):

```python
import dnfile

pe = dnfile.dnPE("Overstrike.dll")
mdt = pe.net.mdtables

# Tabel FieldRva memetakan field -> RVA (alamat) data mentahnya
for row in mdt.FieldRva.rows:
    print(hex(row.Rva), row.Field.row.Name)
```

Output menunjukkan field `SealedRecord` berada di RVA `0x10aa0` dengan panjang 56 byte (`0x38` di IL). Byte-nya diambil dengan:

```python
data = pe.get_data(0x10aa0, 56)
print(data.hex())
# -> 0d563344126e440f363dec5e87cad5b60401b6b596e4b87e79e0ecdc075299fbb36800572022033ca6607c32fd1f7cb3dc9d7873132f600b
```

### 6. Invert `Mix()` (SplitMix64) untuk mencari `CarriedMark`

Karena setiap operasi di `Mix()` bisa dibalik (`xorshift` bisa di-*unshift*, perkalian dengan bilangan ganjil mod 2⁶⁴ punya invers modular), kita bisa mencari `CarriedMark` dari `TrueSeal` tanpa brute-force:

```python
MASK = (1 << 64) - 1

def unshift_xor(z, shift):
    """Membalikkan operasi x ^ (x >> shift)"""
    x = z
    for _ in range(70):        # cukup iterasi agar konvergen
        x = z ^ (x >> shift)
    return x & MASK

C1 = 0x9E3779B97F4A7C15
C2 = 0xBF58476D1CE4E5B9
C3 = 0x94D049BB133111EB

inv_C2 = pow(C2, -1, 1 << 64)   # invers modular perkalian mod 2^64
inv_C3 = pow(C3, -1, 1 << 64)

TrueSeal = 0xd9a1bb0cabb52586

# Membalik Mix() langkah demi langkah, dari belakang ke depan
z2b = unshift_xor(TrueSeal, 31)
z2  = (z2b * inv_C3) & MASK
z1b = unshift_xor(z2, 27)
z1  = (z1b * inv_C2) & MASK
x0  = unshift_xor(z1, 30)
CarriedMark = (x0 - C1) & MASK

print(hex(CarriedMark))   # -> 0xd7caad24dd98b676
```

**Verifikasi** dengan menjalankan `Mix()` versi forward terhadap hasil ini harus menghasilkan kembali `TrueSeal` — dan benar, cocok. Inilah "seal buatan sendiri" (*forged mark*) yang dimaksud di narasi soal.

### 7. Dekripsi `SealedRecord` (Registry) memakai `CarriedMark` hasil forge

Setelah `CarriedMark` didapat, tinggal reimplementasi ulang skema enkripsi `UnsealRegistry()` di Python:

```python
import hashlib, struct

CarriedMark = 0xd7caad24dd98b676
SealedRecord = bytes.fromhex(
    "0d563344126e440f363dec5e87cad5b60401b6b596e4b87e79e0ecdc075299f"
    "bb36800572022033ca6607c32fd1f7cb3dc9d7873132f600b"
)

# key = SHA256(CarriedMark sebagai 8 byte little-endian)
key = hashlib.sha256(struct.pack('<Q', CarriedMark)).digest()

pad = bytearray(len(SealedRecord))
pos, counter = 0, 0
while pos < len(SealedRecord):
    block = hashlib.sha256(key + struct.pack('<i', counter)).digest()
    i = 0
    while i < len(block) and pos < len(SealedRecord):
        pad[pos] = SealedRecord[pos] ^ block[i]
        i += 1
        pos += 1
    counter += 1

print(pad.decode())
```

**Output:**

```
HTB{0v3rstr1k3_r3cut_th3_w0rld_s34l_by_f0rg1ng_th3_mark}
```

Flag berhasil didapat! 

---

## Kesimpulan

Challenge **Overstrike** ini secara garis besar menguji tiga skill sekaligus:

1. **Reverse engineering APK berbasis Godot (Mono/.NET)** — mengenali bahwa logic sebenarnya bukan di file `.cs` mentah (yang cuma placeholder), melainkan sudah dikompilasi ke assembly `.dll`, sehingga perlu tool disassembler .NET (`monodis`) untuk membacanya.
2. **Mengenali algoritma kriptografi dari pola konstanta** — mengenali `Mix()` sebagai **SplitMix64 mixer** dari konstanta khasnya, lalu memanfaatkan sifat **bijektif**-nya untuk membalik (invert) fungsi tersebut secara matematis, bukan brute-force. Ini juga sejalan langsung dengan hint naratif soal soal tentang "membuat seal palsu yang tidak bisa dibedakan dari yang asli".
3. **Memahami skema keystream berbasis hash (SHA-256 counter mode)** — mengenali pola `SHA256(key || counter)` sebagai pembangkit keystream untuk XOR, lalu mereplikasi skema tersebut di Python untuk mendekripsi data yang tersegel (`SealedRecord`).

**Pelajaran untuk pemula:**
- Saat menghadapi APK yang "aneh" (ukurannya besar, ada folder `.godot`), cek dulu apakah itu game engine (Godot/Unity/Unreal) — pendekatan reverse-nya beda dari APK Android native biasa.
- Kalau tool decompiler favorit (`ilspycmd`, `jadx`, dll) tidak tersedia, disassembler tingkat rendah seperti `monodis` tetap bisa dipakai — cuma perlu effort lebih untuk "menerjemahkan" IL ke logic tingkat tinggi.
- Konstanta-konstanta "aneh" dalam kode (seperti `0x9E3779B97F4A7C15`) seringkali adalah tanda algoritma kriptografi/hashing yang sudah dikenal (di sini: SplitMix64) — googling konstanta tersebut sangat membantu mempercepat analisis.
- Kalau sebuah fungsi transformasi bersifat bijektif/reversible, jangan buru-buru brute-force — coba analisis apakah fungsinya bisa dibalik secara matematis, jauh lebih cepat dan elegan.

---

**Flag:** `HTB{0v3rstr1k3_r3cut_th3_w0rld_s34l_by_f0rg1ng_th3_mark}`
