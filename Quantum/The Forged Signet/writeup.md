# The Forged Signet — Writeup

**Kategori:** Quantum
**Judul Challenge:** The Forged Signet
**Tagline:** *"More than a seal. Pure resonance."*

---

## 1. Deskripsi Soal

Kita diberi akses ke sebuah web app bernama **SIGNET** yang mensimulasikan verifikasi "seal" kuno. Ceritanya:

> Setiap seal diverifikasi menggunakan sebuah fungsi rahasia `f`. Fungsi ini punya cacat: `f(x) = f(x ⊕ s)` untuk suatu string rahasia non-zero `s` (disebut **First Mark**) sepanjang `n` bit. Artinya, `f` tidak bisa membedakan input `x` dari `x ⊕ s`. Tugas kita: temukan `s`, lalu submit untuk "forge" seal-nya.

Detail teknis dari halaman **Oracle Datasheet**:

| Field | Value |
|---|---|
| Serial | `dbaa48720490999a` |
| Register | 64 qubits |
| Promise | `f(x) = f(x ⊕ s)` untuk First Mark `s` non-zero, `n` bit |
| Circuit | `H^n . U_f . measure&discard(output) . L . measure(input)` — kita pilih layer single-qubit `L` |

Ini adalah rumusan klasik dari **Simon's Problem**, dan cara menyelesaikannya adalah **Simon's Algorithm** — salah satu algoritma kuantum pertama yang menunjukkan speedup eksponensial dibanding algoritma klasik.

### Endpoint API (dari tab Console → Raw API)

```
GET  /api/oracle            -> baca datasheet
POST /api/run   {layer, shots}  -> jalankan sirkuit, dapat bitstring hasil pengukuran input register
POST /api/forge {s}             -> submit First Mark s (64-bit biner) untuk forge seal
```

`layer` bisa berupa nama gate tunggal (diterapkan ke semua 64 qubit) atau list 64 nama gate, dengan opsi: `I, X, Y, Z, H, S, SDG`.

---

## 2. Konsep: Kenapa Ini Simon's Algorithm

Simon's Problem: diberikan fungsi $f:\{0,1\}^n \to \{0,1\}^n$ yang bersifat 2-to-1 dengan properti $f(x) = f(x \oplus s)$ untuk suatu $s \neq 0$ rahasia. Tujuannya menemukan $s$ dengan jumlah query jauh lebih sedikit dibanding cara klasik (klasik butuh $O(2^{n/2})$ query, kuantum cukup $O(n)$).

**Alur sirkuit per run:**

1. Siapkan register input $n$ qubit dan register output $n$ qubit, semua $|0\rangle$.
2. Terapkan **Hadamard** ke semua qubit input → superposisi merata atas semua $x$.
3. Terapkan oracle $U_f$: $|x\rangle|0\rangle \to |x\rangle|f(x)\rangle$ (entangle input dengan output).
4. **Ukur & buang** register output. Ini membuat register input collapse ke superposisi $\frac{1}{\sqrt2}\big(|x\rangle + |x \oplus s\rangle\big)$ untuk suatu $x$ acak (tidak diketahui).
5. Terapkan layer $L$ ke setiap qubit input.
6. Ukur register input → didapat string $y$.

**Kenapa harus $L = H$?**
Bukti standar Simon's algorithm menunjukkan bahwa jika $L = H^{\otimes n}$ diterapkan pada state $\frac{1}{\sqrt2}(|x\rangle + |x\oplus s\rangle)$, interferensi hanya menyisakan amplitudo non-nol pada $y$ yang memenuhi:

$$y \cdot s \equiv 0 \pmod 2$$

Gate lain (`X, Y, Z, S, SDG, I`) yang ditawarkan di UI **tidak** memberi struktur interferensi yang sama — pilihan itu cuma jebakan/distraktor. Karena itu di UI, layer `H` sudah default terpilih, dan itulah yang kita pakai.

Dengan mengumpulkan cukup banyak `y` yang independen secara linear (butuh **63 vektor independen** untuk $n=64$, karena ruang solusi non-trivial dari sistem homogen berdimensi 1), kita bisa menyelesaikan sistem persamaan linear atas $\mathbb{F}_2$ (GF(2)) dan mendapatkan `s` secara unik (sampai dengan kelipatan non-zero, tapi karena biner cuma ada satu solusi non-zero).

---

## 3. Tools yang Dipakai

- **Python 3** (built-in `python`/`py` launcher di Windows)
- **`requests`** — library HTTP untuk memanggil REST API challenge
- Gaussian elimination manual di GF(2) (ditulis sendiri, tanpa library tambahan)
- Browser (untuk membaca datasheet & console API awal)

---

## 4. Step-by-Step Penyelesaian

### Step 1 — Eksplorasi Web App

Buka `http://<IP>:<PORT>/` di browser. Ada 4 tab navigasi: **Oracle**, **Resonance**, **Forge**, dan ikon `</>` (Console).

- Tab **Oracle** → menampilkan datasheet: promise `f(x)=f(x⊕s)`, register 64 qubit, dan deskripsi sirkuit.
- Tab **Resonance** → form untuk memanggil `/api/run` secara manual dari UI (pilih layer `L` dan jumlah shots).
- Tab **Forge** → form untuk submit `s` hasil temuan ke `/api/forge`.
- Tab **Console (`</>`)** → menunjukkan **raw API spec** beserta contoh `curl`:

```bash
# baca datasheet
curl -s http://154.57.164.73:32740/api/oracle

# query oracle dengan single-qubit layer L
curl -s -XPOST http://154.57.164.73:32740/api/run \
  -d '{"layer":"<gate>","shots":128}' -H 'content-type: application/json'

# forge setelah punya First Mark
curl -s -XPOST http://154.57.164.73:32740/api/forge \
  -d '{"s":"<64-bit First Mark>"}' -H 'content-type: application/json'
```

**Kenapa langkah ini penting:** tanpa membaca dokumentasi API, kita tidak tahu format request/response yang harus dipanggil dari script. Nama field JSON tidak standar (`layer`, `shots`, `s`), jadi harus dibaca langsung dari UI.

### Step 2 — Kenali Pola Soal

Dari kalimat kunci "*the verifier f obeys f(x) = f(x XOR s)*" dan sirkuit `H^n . U_f . measure&discard . L . measure`, langsung terlihat ini adalah **Simon's Algorithm** — bukan Grover, bukan Deutsch-Jozsa. Ini penting supaya tidak salah pilih pendekatan.

### Step 3 — Tulis Script Solver

Script Python melakukan:

1. **Panggil `/api/run` berulang** dengan `layer="H"` (Hadamard di semua qubit) dan `shots=256` per panggilan.
2. **Parsing response** — cek dulu struktur JSON mentah karena nama field tidak didokumentasikan persis:

```python
resp = call_run(layer="H", shots=256)
print(resp)  # debug: lihat struktur JSON aslinya
```

Ternyata field-nya bernama `"samples"`, berisi list string biner 64-karakter, contoh:

```json
{"layer": "H", "samples": ["1000000001100111...", "0011101110001111...", ...]}
```

3. **Bangun basis GF(2)** dari setiap `y` (skip yang bernilai nol, karena tidak memberi informasi):

```python
def gf2_insert(basis, v):
    cur = v
    for pivot_bit in sorted(basis.keys(), reverse=True):
        if (cur >> pivot_bit) & 1:
            cur ^= basis[pivot_bit]
    if cur != 0:
        pivot_bit = cur.bit_length() - 1
        basis[pivot_bit] = cur
        return True
    return False
```

Setiap `y` baru direduksi terhadap basis yang sudah ada (mirip row-reduction). Kalau hasilnya non-zero, berarti `y` menambah rank baru → disimpan sebagai pivot baru.

4. **Berhenti saat rank = 63** (dari total 64 bit, satu dimensi tersisa untuk `s`):

```python
while len(basis) < N - 1:
    ...
```

5. **Ubah ke RREF (Reduced Row Echelon Form)** lalu cari **kolom bebas** (bit yang bukan pivot manapun) — itu adalah arah `s`:

```python
def recover_s(rref, n=N):
    pivot_bits = {r.bit_length() - 1 for r in rref}
    free_bits = [b for b in range(n) if b not in pivot_bits]
    # harus tepat 1 free bit
    free = free_bits[0]
    s = 1 << free
    for r in rref:
        pivot_i = r.bit_length() - 1
        if (r >> free) & 1:
            s |= (1 << pivot_i)
    return s
```

6. **Submit ke `/api/forge`**:

```python
s_bin = format(s, f"0{N}b")
r = session.post(f"{BASE}/api/forge", json={"s": s_bin})
print(r.status_code, r.text)
```

### Step 4 — Jalankan Script

Di Windows, `python3` sering tidak dikenali (alias tidak terdaftar). Solusinya pakai `python` atau launcher `py`:

```bash
python solve.py
```

**Output:**

```
[debug] example /api/run response: {'layer': 'H', 'samples': ['1000000001100111...', ...]}
[call 1] +256 samples (total 256), rank = 63/63
Recovered First Mark s = 1001011011010110111100001100011111111011111000000011110000000101
Submitting to /api/forge ...
200 {"flag":"HTB{th3_f1rst_m4rk_1s_4_h1dd3n_x0r_p3r10d_58291f09a8ebb91f9eb11506cf9b15ff}","forged":true}
```

**Menarik:** hanya butuh **1 kali panggilan API** (256 shots sekaligus) untuk langsung mencapai rank penuh 63/63 — karena 256 sampel jauh lebih banyak dari minimum 63 yang dibutuhkan, kemungkinan besar sudah cukup independen dari sekali jalan.

### Step 5 — Verifikasi Flag

Response `/api/forge` mengembalikan:

```json
{"flag":"HTB{th3_f1rst_m4rk_1s_4_h1dd3n_x0r_p3r10d_58291f09a8ebb91f9eb11506cf9b15ff}","forged":true}
```

`"forged": true` mengonfirmasi bahwa `s` yang ditemukan benar.

---

## 5. Kesimpulan

Challenge ini adalah implementasi praktis dari **Simon's Algorithm**, salah satu contoh klasik quantum advantage: menemukan periode tersembunyi `s` dari fungsi 2-to-1 dengan hanya $O(n)$ query kuantum, dibanding $O(2^{n/2})$ query klasik.

Poin pembelajaran utama:

1. **Kenali pola soal dari deskripsi matematis** — kalimat `f(x) = f(x ⊕ s)` adalah signature khas Simon's Problem.
2. **Pilihan layer `L` di akhir sirkuit sangat menentukan** — hanya Hadamard (`H`) yang menghasilkan interferensi konstruktif sesuai bukti algoritma; gate lain di menu (`X, Y, Z, S, SDG, I`) adalah distraktor.
3. **Setiap hasil pengukuran `y` memberi satu persamaan linear** $y \cdot s \equiv 0 \pmod 2$. Untuk menyelesaikan `s` sepanjang `n` bit secara unik (hingga faktor non-zero), butuh $n-1$ persamaan yang independen secara linear.
4. **Gaussian elimination di GF(2)** adalah alat sederhana namun cukup untuk menyelesaikan sistem ini — tidak perlu library aljabar linear berat.
5. **Selalu baca dokumentasi API mentah** (di sini lewat tab Console) sebelum menulis script — nama field JSON (`layer`, `shots`, `samples`, `s`) tidak bisa ditebak sembarangan.

Total waktu eksekusi sangat cepat karena server mengizinkan hingga 256 shots per request, sehingga rank penuh bisa dicapai hanya dalam satu kali panggilan API.

**Flag:** `HTB{th3_f1rst_m4rk_1s_4_h1dd3n_x0r_p3r10d_58291f09a8ebb91f9eb11506cf9b15ff}`
