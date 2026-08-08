# [Quantum] The Coin That Won't Land — Oathbinding

## Deskripsi Soal

> After the realm cracked apart, nobody trusted a word anyone said, so the
> Registry built the Oathbinding to settle disputes for good. You swear a
> vow before the court, the Warden seals it in a box nobody can open, and
> later they call you back and ask what you swore. Lie about it and the box
> catches you... Or so they say. The court throws question after question
> at you, round after round, waiting for you to fold. Get through every
> round without a slip and the confession you drag out is your flag.

Target: `154.57.164.76:30469`

Soal ini membungkus sebuah **skema quantum bit commitment** dalam narasi
"pengadilan". Front-end web-nya (`/#hearing`) menampilkan 4 "move" yang
sebenarnya adalah alur protokol commitment:

1. **Commit the vow** — kirim vow ke Warden, simpan setengah untuk diri
   sendiri.
2. **The court's toss** — pengadilan membalas dengan sebuah *coin toss*
   (challenge bit) yang tidak bisa kita tebak/pilih sendiri.
3. **Read your half** — kita boleh membaca (mengukur) bagian yang kita
   simpan.
4. **Open and be believed** — vow kita dan bagian yang disegel Warden harus
   cocok, setiap ronde, atau pengadilan menganggap kita gagal.

Detail teknis lengkap diberikan di halaman **"Read the writ"**
(`GET /api/oath`):

| Method | Endpoint      | Fungsi |
|--------|---------------|--------|
| POST   | `/api/new`    | Membuka hearing baru. Mengembalikan `token`, jumlah `strands` (8), dan `rounds` (32). |
| POST   | `/api/commit` | `{token, slots:[<strand> x strands]}`. Setiap strand adalah sirkuit kuantum 2-qubit (`a`, `b`) yang diterapkan pada state awal `\|00⟩`. Qubit `a` kita simpan sendiri, qubit `b` disegel oleh Warden. Mengembalikan **challenge bit `c`**. |
| POST   | `/api/peek`   | `{token, basis}` (`basis` = `"Z"` atau `"X"`). Mengukur qubit `a` milik kita sendiri pada basis tersebut, mengembalikan hasil ukur. |
| POST   | `/api/open`   | `{token, values:[0/1 x strands]}`. Warden mengukur setiap qubit `b` yang disegel pada basis `c` (`c=0 → Z`, `c=1 → X`). Vow dianggap valid **hanya jika semua nilai cocok**. |
| GET    | `/api/oath`   | Menampilkan spesifikasi lengkap protokol (writ). |

Gate yang tersedia: gate 1-qubit `I X Y Z H S SDG` pada `a` atau `b`
(format `['H','a']`), dan gate 2-qubit `CX` dengan `control, target`
(format `['CX','a','b']`).

Aturan: **setiap ronde harus lolos** (32 ronde total) untuk dianggap
"believed" dan mendapatkan flag.

## Tools yang Dipakai

- Python 3.11
- Library `requests` (HTTP client)
- Sedikit pemahaman dasar **quantum computing**: qubit, superposisi, gate
  `H` (Hadamard) dan `CX` (CNOT), basis pengukuran Z vs X, dan konsep
  **Bell state** / EPR pair.

## Step-by-Step Penyelesaian

### 1. Memahami inti soal: kenapa protokolnya bisa "dibohongi"

Secara jujur, cara pakai protokol ini adalah: encode bit klasik (0 atau 1)
ke qubit `a`, salin ke `b` (misalnya lewat `CX`), lalu segel `b`. Dengan
begitu kita terikat pada satu nilai bit yang sudah pasti sejak awal.

Tapi ada celah teoretis yang cukup terkenal di dunia quantum cryptography:
**quantum bit commitment tidak bisa 100% mengikat (binding) secara tanpa
syarat** — ini dikenal sebagai hasil **Mayers–Lo–Chau**. Alih-alih
mengirim bit yang sudah pasti, kita bisa membuat qubit `a` dan `b`
**entangled** (terjerat) dalam bentuk **Bell state**:

```
H   a       # a menjadi superposisi (|0⟩ + |1⟩) / √2
CX  a b     # a dan b menjadi |Φ+⟩ = (|00⟩ + |11⟩) / √2
```

Sifat penting `|Φ+⟩`: state ini adalah eigenstate +1 dari **Z⊗Z** *dan*
juga eigenstate +1 dari **X⊗X**. Artinya:

- Kalau kedua qubit (`a` dan `b`) diukur di basis **Z**, hasilnya *selalu*
  sama (00 atau 11).
- Kalau kedua qubit diukur di basis **X**, hasilnya *juga selalu* sama.

Konsekuensinya: pada saat commit, **belum ada bit pasti yang "disumpahkan"**
— nilainya baru "ditentukan" setelah diukur. Karena korelasi ini berlaku
di *kedua* basis, kita bisa menunggu challenge bit `c` diumumkan lebih
dulu, baru mengukur qubit `a` kita di basis yang sama (`c=0→Z`,
`c=1→X`), lalu melaporkan hasil ukur itu sebagai "vow" kita. Hasilnya
akan selalu cocok dengan pengukuran Warden atas `b` — padahal kita
curang, karena tidak pernah benar-benar mengunci satu bit sejak awal.

Strategi ini harus diulang untuk **setiap strand** (8 buah) dan **setiap
ronde** (32 ronde).

### 2. Membuka hearing baru

```
POST /api/new
```

Contoh respons:

```json
{"rounds": 32, "strands": 8, "token": "NpTYZXfiPFH6Q2Fb"}
```

`token` ini dipakai di semua request berikutnya untuk melacak sesi kita.

### 3. Commit — kirim Bell pair, bukan bit biasa

Untuk tiap strand (ada 8), kirim sirkuit yang sama: `H` pada `a` lalu
`CX` dari `a` ke `b`.

```
POST /api/commit
{
  "token": "<token>",
  "slots": [
    [["H","a"], ["CX","a","b"]],
    [["H","a"], ["CX","a","b"]],
    ... (total 8 strand)
  ]
}
```

Server membalas dengan challenge bit untuk ronde ini, misalnya:

```json
{"challenge": 0, "round": 0}
```

**Kenapa langkah ini diambil:** ini adalah inti dari eksploitasi —
menggantikan "commit bit yang pasti" dengan "commit ke superposisi
terjerat", supaya bit sebenarnya baru "diputuskan" belakangan, setelah
tahu challenge apa yang diminta.

### 4. Peek — ukur qubit sendiri di basis yang sama dengan challenge

Petakan `challenge` ke basis: `0 → "Z"`, `1 → "X"`.

```
POST /api/peek
{"token": "<token>", "basis": "Z"}
```

Contoh respons:

```json
{"a_outcomes": [0, 1, 1, 1, 1, 1, 1, 1]}
```

**Kenapa langkah ini diambil:** kita memilih basis pengukuran yang sama
dengan basis yang nanti dipakai Warden untuk mengukur `b` (basis `c`).
Karena sifat Bell state di atas, hasil ukur `a` di basis ini dijamin akan
sama dengan hasil ukur `b` Warden di basis yang sama juga.

### 5. Open — laporkan hasil peek sebagai "vow"

```
POST /api/open
{"token": "<token>", "values": [0, 1, 1, 1, 1, 1, 1, 1]}
```

Server memverifikasi dengan mengukur seluruh qubit `b` yang disegel pada
basis `c`, dan membandingkan dengan `values` yang kita kirim. Karena
korelasi Bell state, seluruh nilai akan cocok.

### 6. Ulangi untuk seluruh 32 ronde

Langkah 3–5 (commit → peek → open) diulang sebanyak `rounds` (32 kali).
Setiap ronde memakai Bell pair baru (state lama sudah runtuh/collapsed
setelah diukur, jadi harus commit ulang). Skrip Python berikut
mengotomasi seluruh proses:

```python
import requests

base = "http://154.57.164.76:30469"
s = requests.Session()

data = s.post(f"{base}/api/new").json()
token = data["token"]
strands = data["strands"]
rounds = data["rounds"]

bell_strand = [["H", "a"], ["CX", "a", "b"]]

for rnd in range(rounds):
    slots = [bell_strand for _ in range(strands)]
    c = s.post(f"{base}/api/commit",
               json={"token": token, "slots": slots}).json()["challenge"]

    basis = "Z" if c == 0 else "X"
    outcomes = s.post(f"{base}/api/peek",
                       json={"token": token, "basis": basis}).json()["a_outcomes"]

    result = s.post(f"{base}/api/open",
                     json={"token": token, "values": outcomes}).json()
    print(rnd, result)

# Setelah 32 ronde lolos, ambil flag
print(s.get(f"{base}/api/oath", params={"token": token}).text)
```

### 7. Ambil flag

Setelah semua ronde dinyatakan valid ("held"), server mengembalikan
konfirmasi bahwa kita "believed" oleh pengadilan beserta flag-nya —
sesuai deskripsi soal: *"the confession you drag out is your flag."*

```
FLAG: <isi dengan flag yang kamu dapat, mis. flag{...}>
```

## Kesimpulan

Challenge ini adalah demonstrasi elegan dari sebuah hasil teoretis
penting dalam kriptografi kuantum: **skema quantum bit commitment tidak
bisa unconditionally binding**. Selama pihak yang commit boleh
menggunakan operasi kuantum sembarang (bukan cuma encode bit klasik),
mereka selalu bisa "menunda" keputusan bit dengan cara membuat pasangan
qubit yang saling terjerat (Bell state / EPR pair), lalu memutuskan nilai
bit *setelah* tahu tantangan apa yang diberikan lawan.

Pelajaran untuk pemula:

- Basis pengukuran `Z` vs `X` menentukan "pertanyaan apa" yang diajukan
  ke sebuah qubit — hasil pengukuran berbeda tergantung basisnya.
- Bell state `|Φ+⟩` unik karena korelasinya sempurna di **dua** basis
  sekaligus (Z dan X), bukan cuma satu — inilah yang membuatnya bisa
  dipakai untuk "curang" di kedua kemungkinan challenge.
- Protokol keamanan yang terlihat kokoh secara klasik (komit lalu buka)
  bisa runtuh total begitu penyerang punya akses ke superposisi dan
  entanglement — makanya commitment scheme yang aman secara kuantum
  butuh asumsi tambahan (misalnya batasan komputasi/kuantum penyerang),
  bukan cuma "disegel dalam kotak".

---

*Ditulis sebagai bagian dari CTF writeup pribadi.*
