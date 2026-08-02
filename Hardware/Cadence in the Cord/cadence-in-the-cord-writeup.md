# [Hardware] Cadence in the Cord

## Deskripsi Soal

> With the Brine Signet shattered, every house hunts whatever might make its story law. Lady Seralyne, the Velvet Spider of Suncourt, sells what she claims is the dragon's true note: not the lost thing itself, only a counterfeit cadence arranged to be believed, and a wavering house is ready to buy it as proof its claim rings true. We cut one of her sendings from the wire first. Read the pleasant words; then attend to the silences between them, and expose the forgery she is truly selling.

**Kategori:** Hardware
**File yang diberikan:** `capture.sr` (hasil capture logic analyzer, format sigrok)

Dari narasi soal ada dua petunjuk kunci:
1. Ada "pesan yang enak dibaca" (*pleasant words*) yang sengaja dibuat meyakinkan — ini adalah **umpan (forgery)**.
2. Flag sebenarnya bersembunyi di **jeda/keheningan** (*silences*) di antara pesan tersebut, bukan pada isi datanya.

## Tools yang Digunakan

- **Python** dengan `numpy` — untuk parsing binary dan analisis sinyal
- Pemahaman dasar format **sigrok `.sr`** (arsip ZIP berisi metadata + raw logic samples)
- Konsep dasar protokol **UART** (baud rate, framing bit, start/stop bit)
- (Opsional) **PulseView / sigrok-cli** bisa dipakai sebagai alternatif visual, tapi di sini seluruhnya diselesaikan lewat scripting Python

## Step-by-Step Penyelesaian

### 1. Membongkar Struktur File `.sr`

File `.sr` dari sigrok sebenarnya adalah arsip ZIP biasa. Langkah pertama adalah mengekstraknya untuk melihat isi `metadata` dan channel logic yang tersedia.

```python
import zipfile

z = zipfile.ZipFile('capture.sr')
print(z.namelist())          # daftar file di dalam .sr
print(z.read('metadata').decode())
```

Hasil `metadata` menunjukkan:

```ini
[global]
sigrok version=0.5.2

[device 1]
capturefile=logic-1
total probes=8
samplerate=2 MHz
total analog=0
probe2=D1
unitsize=1
```

**Kenapa langkah ini diambil:** tanpa membongkar struktur file terlebih dahulu, kita tidak tahu berapa channel yang direkam, sample rate-nya, maupun cara data logic disimpan (per-chunk). Info `samplerate=2 MHz` dan `total probes=8` ini krusial untuk tahap decoding berikutnya.

### 2. Menggabungkan Chunk Data Logic

Data logic disimpan sigrok dalam banyak file kecil (`logic-1-1`, `logic-1-2`, ..., `logic-1-782`). Semua chunk ini perlu digabung berurutan sesuai nomor urutnya menjadi satu stream byte.

```python
names = [n for n in z.namelist() if n.startswith('logic-1-')]
names.sort(key=lambda n: int(n.split('-')[-1]))   # urutkan numerik, bukan string
data = b''.join(z.read(n) for n in names)

open('raw.bin', 'wb').write(data)
print(len(data))   # 16.000.000 sample -> 8 detik @ 2 MHz
```

**Kenapa diurutkan manual:** jika hanya di-sort sebagai string, `logic-1-10` akan muncul sebelum `logic-1-2`, sehingga stream data jadi rusak/acak.

Karena `unitsize=1` dan `total probes=8`, setiap 1 byte sample berisi status 8 channel logic sekaligus (1 bit per channel).

### 3. Mencari Channel yang Aktif

Dengan 8 channel dalam 1 byte, langkah berikutnya adalah memeriksa bit mana saja yang benar-benar "hidup" (banyak transisi), karena channel yang idle terus (constant) hampir pasti tidak relevan.

```python
import numpy as np

data = np.fromfile('raw.bin', dtype=np.uint8)

for bit in range(8):
    ch = (data >> bit) & 1
    transitions = np.sum(np.diff(ch.astype(np.int8)) != 0)
    print(f'bit {bit}: transitions={transitions}')
```

Hasilnya:

| Bit | Transisi | Keterangan |
|-----|----------|------------|
| 0   | 222      | Sedikit aktivitas |
| **1** | **3562** | **Paling aktif — ini channel `D1` sesuai metadata** |
| 2–7 | 0        | Konstan (idle/tidak terpakai) |

**Kenapa langkah ini penting:** ini menghemat waktu — daripada mencoba decode 8 channel sekaligus, kita fokus hanya pada channel yang benar-benar membawa data, yaitu bit index 1 (`D1`).

### 4. Mengidentifikasi Baud Rate dari Lebar Pulsa

Untuk decode UART, kita perlu tahu baud rate-nya. Caranya dengan mengukur jarak antar-edge (transisi) sinyal dan mencari pola satuan waktu terkecil (unit dasar).

```python
ch1 = (data >> 1) & 1
edges = np.where(np.diff(ch1.astype(np.int8)) != 0)[0]
diffs = np.diff(edges)

print(sorted(set(diffs.tolist()))[:10])
# -> [208, 209, 416, 417, 625, 626, ...] -> kelipatan dari ~208 sample
```

Sample rate 2 MHz, dan pulsa terkecil ≈ 208 sample:

```
208 sample / 2.000.000 sample/detik ≈ 104 µs  ->  1 / 104µs ≈ 9600 baud
```

**Kenapa dicek begini:** pola "kelipatan dari satu angka dasar" adalah ciri khas sinyal UART NRZ, di mana setiap bit punya durasi tetap. Menemukan unit dasar = menemukan baud rate.

### 5. Decode UART (8N1, LSB-first)

Dengan baud rate 9600 diketahui, saya menulis UART decoder manual: mendeteksi *falling edge* sebagai start bit, lalu sampling di tengah tiap periode bit.

```python
samplerate = 2_000_000
baud = 9600
bit_samples = samplerate / baud   # ≈ 208.33 sample per bit

def decode_uart(ch1, bit_samples, databits=8, lsb_first=True):
    n = len(ch1)
    pos = 0
    result, starts, ends = [], [], []
    while pos < n - 1:
        # start bit: transisi high -> low
        if ch1[pos] == 1 and ch1[pos+1] == 0:
            start = pos + 1
            mid_start = start + bit_samples / 2
            bits = [ch1[int(round(mid_start + b*bit_samples))] for b in range(databits)]
            stop_idx = int(round(mid_start + databits*bit_samples))
            if stop_idx < n and ch1[stop_idx] == 1:   # validasi stop bit
                byte = 0
                for i, bv in enumerate(bits):
                    byte |= (bv << i) if lsb_first else (bv << (databits-1-i))
                result.append(byte)
                starts.append(start)
                ends.append(stop_idx)
                pos = stop_idx + 1
                continue
        pos += 1
    return result, starts, ends

r, starts, ends = decode_uart(ch1, bit_samples, databits=8, lsb_first=True)
print(len(r))   # 593 byte berhasil didecode
```

Hasil decode: **593 byte** data, tapi isinya berupa karakter acak/tidak terbaca (bukan flag). Ini konsisten dengan petunjuk soal — inilah "**pleasant words**" alias **umpan (forgery)** yang sengaja dibuat menyerupai transmisi valid, padahal bukan flag sebenarnya.

### 6. Fokus ke "Silences" — Jeda Antar Byte

Sesuai petunjuk narasi ("*attend to the silences between them*"), saya ukur jeda waktu (kondisi *idle high*) di antara setiap byte UART yang berhasil didecode — bukan isi byte-nya.

```python
gaps = [starts[i+1] - ends[i] for i in range(len(starts) - 1)]
units = [g / 208.33 for g in gaps]   # normalisasi ke satuan bit-period

import collections
print(collections.Counter([round(u) for u in units]).most_common(10))
```

Distribusi jeda ternyata **bimodal** — mengelompok jelas ke dua nilai:

- **Jeda pendek** ≈ 16–26 unit bit-period (~4.000–5.400 sample)
- **Jeda panjang** ≈ 93–103 unit bit-period (~19.300–21.100 sample)

Pola dua-kelompok yang jelas ini adalah indikasi kuat bahwa **durasi jeda sedang dipakai untuk encode bit biner** (short = `0`, long = `1`) — sebuah bentuk *timing-based steganography*, mirip prinsip Morse code tapi diterapkan pada jeda antar frame UART.

### 7. Klasifikasi Bit dan Rekonstruksi Flag

```python
bits = [0 if u < 50 else 1 for u in units]   # threshold di tengah dua cluster
print(len(bits))   # 592 bit dari 592 jeda (593 byte -> 592 gap)

def bits_to_bytes(bits, msb_first=True):
    out = []
    for i in range(0, len(bits) - 7, 8):
        chunk = bits[i:i+8]
        val = 0
        for b in chunk:
            val = (val << 1) | b if msb_first else val | (b << chunk.index(b))
        out.append(val)
    return bytes(out)

flag_bytes = bits_to_bytes(bits, msb_first=True)
print(flag_bytes)
```

**Output:**

```
you read the silence well HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```

**Kenapa langkah ini yang membuka flag:** 592 bit tersembunyi setara dengan 74 byte data ASCII — cukup untuk sebuah kalimat konfirmasi plus flag. Encoding MSB-first pada percobaan pertama langsung menghasilkan teks yang bersih dan terbaca, menandakan pilihan bit-order dan threshold sudah tepat.

## Flag

```
HTB{th3_f1rst_m4rk_r1ngs_tru3_b3n34th_th3_w0rds}
```

## Kesimpulan

Challenge ini adalah studi kasus *timing-based covert channel* di atas protokol UART biasa. Poin pembelajaran utama:

1. **Isi payload UART bukan selalu tempat data sebenarnya berada.** Di sini, 593 byte hasil decode langsung adalah *decoy* yang sengaja dibuat terlihat seperti transmisi valid ("*counterfeit cadence*" — cocok dengan judul challenge).
2. **Metadata timing (jeda/gap) bisa membawa informasi sendiri**, terlepas dari isi data di "permukaan". Analisis distribusi durasi jeda (bimodal: pendek vs panjang) adalah kunci untuk menyadari adanya *hidden channel*.
3. **Pendekatan sistematis** — dari membongkar format file, mencari channel aktif, menentukan baud rate, decode protokol standar, baru kemudian menganalisis anomali di luar decode standar — sangat membantu memecah masalah yang awalnya terlihat rumit menjadi langkah-langkah kecil yang terukur.
4. Semua proses di atas bisa dikerjakan murni dengan **Python + numpy**, tanpa harus bergantung pada tool GUI seperti PulseView, yang berguna terutama saat bekerja di environment tanpa akses grafis.

---
