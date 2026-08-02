# Fractured Seal — Crypto Writeup

**Kategori:** Crypto
**Flag:** `HTB{r3c0v3r1ng_RSA_k3ys___l1k3___Me0w___me0o00o0o0w___Me0w}`

## Deskripsi Soal

> One of the Registry's oldest key-scrolls survived the fall of Crownspire, though time and fire spared only fragments of its writing, and most in the vault dismissed it as useless. Caldrin didn't. She always said a seal doesn't have to be whole to still remember the door it once opened.

Kita diberikan 3 file:

- `encrypt.py` — script yang menunjukkan proses enkripsinya (RSA standar 1024-bit `p`, `q`, `e = 0x10001`).
- `fractured_seal.pem` — private key RSA, tapi **sebagian besar isinya sudah "dihancurkan"** (diganti karakter `*`), plus ada hiasan ASCII art kucing yang ditumpuk di area yang sudah rusak.
- `flag.enc` — flag yang sudah dienkripsi dengan public key dari `seal.pem`.

Karena banyak bagian private key hilang, kita tidak bisa langsung memakainya untuk decrypt. Tantangannya: rekonstruksi cukup informasi dari key yang "patah" ini untuk memfaktorkan `N` dan mendapatkan private exponent `d`.

## Tools yang Dipakai

- **Python**
- **sympy** — untuk operasi polinomial (GCD, solve) dan cek bilangan prima
- **mpmath** — untuk aritmatika presisi tinggi (dipakai di implementasi LLL sendiri)
- Implementasi **LLL (Lenstra–Lenstra–Lovász) buatan sendiri**, karena `fpylll` tidak tersedia dan environment tidak ada akses internet untuk instalasi
- Konsep matematika: **RSA**, format **ASN.1/DER** pada file PEM, dan **Coppersmith's Attack** (factoring dengan sebagian bit high-order diketahui)

## Step-by-Step

### 1. Memahami skema enkripsi

`encrypt.py` menunjukkan RSA klasik:

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 0x10001
d = pow(e, -1, (p-1)*(q-1))

m = bytes_to_long(open('flag.txt', 'rb').read())
open('seal.pem', 'wb').write(RSA.construct((n, e, d)).export_key())
open('flag.enc', 'wb').write(long_to_bytes(pow(m, e, n)))
```

Jadi flag dienkripsi dengan `c = m^e mod n`, dan untuk decrypt kita butuh `d` (atau minimal `p` dan `q` supaya bisa hitung `d` sendiri). File `fractured_seal.pem` seharusnya berisi `n, e, d, p, q, ...` tapi datanya rusak — di sinilah tantangannya dimulai.

### 2. Membedah file PEM yang rusak

File `.pem` itu isinya cuma **base64 dari struktur DER (ASN.1)**. Karena sebagian karakternya diganti `*` (dan sebagian lagi ditumpuki ASCII art kucing), kita tidak bisa langsung `base64.decode()` begitu saja.

Trik yang dipakai: proses base64-nya per 4 karakter (1 "quartet" = 3 byte). Kalau ke-4 karakter dalam satu quartet semuanya valid (bukan `*`/karakter aneh), quartet itu bisa didecode sempurna. Kalau ada `*` di dalamnya, byte-byte di posisi itu kita tandai "tidak diketahui".

```python
b64alpha = string.ascii_letters + string.digits + '+/='
# ganti semua karakter yang bukan base64 valid (termasuk ASCII art kucing) jadi '*'
norm = ''.join(c if c in b64alpha else '*' for c in full_pem_body)
```

Kenapa langkah ini penting: dengan begini kita tahu **persis** byte mana dari `n`, `e`, `d`, `p`, `q` yang masih utuh, dan mana yang hilang — tanpa langkah ini kita cuma menebak-nebak.

Hasil pembedahan:

- **`N` (modulus, 2048-bit):** 255 byte pertama diketahui penuh, byte terakhir hanya 6 bit atas yang diketahui (2 bit bawah hilang) → cuma ada **4 kemungkinan nilai `N`**.
- **`q` (salah satu faktor prima, 1024-bit):** 73 byte pertama (dari sisi paling signifikan/atas) diketahui penuh + 4 bit tambahan → total **588 dari 1024 bit** diketahui (~57.4%).
- **`e`, `d`, `p`, dan komponen CRT lainnya:** hilang total (di area yang penuh `*` / tertutup ASCII art).

Cara memverifikasi bahwa area yang dianggap "diketahui" itu benar: decode potongan byte yang utuh, lalu cocokkan dengan struktur DER standar RSA private key (header `SEQUENCE`, `INTEGER` untuk tiap komponen). Setelah dicocokkan, header ASN.1-nya pas persis — ini konfirmasi bahwa parsing sudah benar.

### 3. Kenapa 57.4% bit `q` itu cukup?

Ini kunci dari soal ini: kalau kita tahu **sekitar setengah bit teratas** dari salah satu faktor prima `q` pada `N = p·q`, kita bisa memfaktorkan `N` dalam waktu polinomial — ini adalah hasil klasik dari **Coppersmith's Attack** ("factoring with high bits known", Coppersmith 1996).

Batas teorinya: kalau `p, q` berukuran seimbang (`p ≈ q ≈ √N`), dan bagian yang tidak diketahui dari `q` lebih kecil dari `N^(1/4)`, maka `q` bisa direkonstruksi. Di soal ini, bagian tak diketahui `q` adalah 436 bit (~42.6%), sedangkan `N^(1/4)` untuk `N` 2048-bit adalah 512 bit — jadi **secara teori pasti bisa diselesaikan**, dengan margin yang cukup nyaman.

### 4. Menyusun Coppersmith Attack

Idenya: tulis `q = q0 + x`, di mana `q0` adalah bagian bit atas yang sudah diketahui (bit bawah dianggap 0), dan `x` adalah bagian tidak diketahui (`0 <= x < 2^436`). Kita cari `x` sedemikian sehingga `f(x) = x + q0 ≡ 0 (mod q)` — meskipun `q` sendiri tidak diketahui, ini bisa diselesaikan lewat teknik **lattice reduction (LLL)**.

Percobaan pertama pakai konstruksi lattice paling dasar:

```
g_i(x) = N^(m-i) * (x + q0)^i,   untuk i = 0..m
```

**Ternyata ini bug/kesalahan konsep** — setelah divalidasi dengan test sintetis kecil (RSA 256-bit buatan sendiri), konstruksi dasar ini **gagal total**, bahkan untuk kasus yang seharusnya sangat mudah. Setelah diturunkan ulang secara matematis, ternyata konstruksi dasar ini punya batas asimtotik `X < N^(2β-1)` — untuk `β = 0.5` (kasus `p ≈ q`), ini berarti hampir tidak bisa merecover bit apa pun, berapa pun besar `m`-nya.

**Perbaikannya:** tambahkan polinomial "shift" tambahan:

```
h_j(x) = x^j * (x + q0)^m,   untuk j = 1..t
```

dengan `t = m` (rasio optimal untuk `β = 0.5`, diturunkan dari optimasi rumus batas `X < N^(β²/δ)`). Setelah perbaikan ini, test sintetis pada fraksi bit yang sama (57.4%) berhasil menemukan root yang benar dengan `m = t = 3`.

### 5. Implementasi LLL sendiri

Karena `fpylll` tidak ada dan tidak bisa install (no internet), LLL diimplementasikan manual:

1. **Versi awal:** pakai `fractions.Fraction` untuk Gram-Schmidt yang presisinya eksak — benar, tapi **terlalu lambat** untuk dimensi lattice ~9-13 dengan bilangan ribuan bit.
2. **Versi final:** pakai `mpmath` (floating point presisi tinggi) untuk Gram-Schmidt, jauh lebih cepat, dengan presisi yang di-set otomatis berdasarkan ukuran bit terbesar di matriks.

Validasi: dicocokkan dengan contoh klasik LLL dari Wikipedia — hasilnya identik dengan yang seharusnya, jadi implementasi dasarnya benar.

### 6. Menjalankan attack pada data asli

Setelah root `x` ditemukan lewat LLL + polynomial GCD (ambil beberapa vektor hasil reduksi, cari akar bersama lewat GCD polinomial, bukan cari akar numerik — supaya tidak ada masalah presisi floating point), langkah terakhir:

```python
q = q0 + x
assert N % q == 0          # verifikasi q benar-benar faktor dari N
p = N // q

e = 0x10001
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

c = int.from_bytes(open('flag.enc', 'rb').read(), 'big')
m = pow(c, d, N)
flag = m.to_bytes((m.bit_length() + 7) // 8, 'big')
print(flag)
```

Karena `N` punya 4 kandidat (akibat 2 bit tak diketahui di byte terakhir), attack dicoba untuk tiap kandidat satu per satu. Dengan `m = t = 3`, **kandidat `N` ke-2** berhasil menemukan `q` yang benar-benar habis membagi `N`. Dari situ tinggal hitung `p`, `d`, lalu decrypt.

Hasilnya:

```
FACTOR FOUND! q = 1353144083788427907516058780509312090660676352497...
b'HTB{r3c0v3r1ng_RSA_k3ys___l1k3___Me0w___me0o00o0o0w___Me0w}'
```

## Kesimpulan

Challenge ini pada dasarnya adalah latihan **Coppersmith's Attack** (factoring RSA dengan sebagian bit salah satu faktor prima diketahui), dibungkus dengan cerita PEM yang "dirusak". Poin-poin pentingnya:

- Selalu **verifikasi struktur data sendiri** (di sini: parsing ASN.1/DER manual) daripada percaya begitu saja pada asumsi awal — ini yang membantu menemukan angka pasti "588 dari 1024 bit diketahui" secara presisi, bukan kira-kira.
- Konstruksi lattice Coppersmith **tidak satu ukuran untuk semua kasus** — untuk kasus faktorisasi RSA seimbang (`β = 0.5`), wajib pakai polinomial shift tambahan (`t = m`), kalau tidak, seberapa besar pun `m` dinaikkan tetap tidak akan berhasil.
- Ketika tool "standar" (`fpylll`) tidak tersedia, LLL bisa diimplementasikan sendiri — tapi pilih pendekatan yang **cepat sekaligus cukup akurat** (mpmath presisi tinggi, bukan `Fraction` yang eksak tapi lambat).
- ASCII art kucing di file PEM cuma dekorasi/red herring yang kebetulan jadi petunjuk tema flag ("meow") — tidak mempengaruhi solusi teknisnya sama sekali.

**Flag:** `HTB{r3c0v3r1ng_RSA_k3ys___l1k3___Me0w___me0o00o0o0w___Me0w}`
