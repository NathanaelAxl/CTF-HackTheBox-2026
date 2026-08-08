# [OSINT] The Ledger Beneath the Hull

> **Kategori:** OSINT
> **Difficulty:** Easy
> **Estimasi waktu:** 15–25 menit
> **Flag:** `HTB{THE_INK_WAS_DIVIDED_AMONG_FIVE_NAMES}`

---

## Deskripsi Soal

Environment challenge berupa "Court-Eaves Console", sebuah simulasi meja kerja investigator maritim lengkap dengan beberapa tool internal: **Tideglass Browser**, **Maritime Registry**, **P&I Directory**, **Companies Register**, dan **Oath Submission**.

Ceritanya:

> Lord Damas Marrowcairn tidak mengendalikan armada kapal secara langsung — ia memilikinya lewat labirin dokumen perusahaan. Satu kapal kargo, **ASHEN MERCY**, berada di tengah jaringan lima perusahaan: sebuah shell company yang memegang kepemilikan hull, sebuah management firm yang menjalankan mesinnya, sebuah coordination house yang mengatur pelayarannya, dan sebuah commodities trader yang mengisi muatannya. Setiap lapisan terlihat bersih, setiap lapisan atas nama orang lain — tapi semua benang, kalau ditarik cukup keras, berujung ke tangan yang sama.

**Objective:** merekonstruksi rantai kepemilikan kapal **ASHEN MERCY (IMO 9724418)** secara lengkap, lalu membuktikan bahwa pengendali utama di baliknya adalah Lord Damas Marrowcairn.

**Intelligence Objectives (5 pertanyaan yang harus dijawab di form Oath Submission):**
1. Identify the vessel's registered owner
2. Identify the ISM manager
3. Identify the commercial operator
4. Identify the time charterer
5. Identify the ultimate controlling company behind the operator

**Bukti awal yang diberikan (Cargo Release CR-EA-71904):**
```
VESSEL IMO:       9724418
P&I ENTRY:        PI-VAL-88291
RELEASE APPROVER: Eastreach Maritime Coordination PLC
CARGO:            Preservation minerals and treated cord
NOTE:             "The approving company is not listed as the vessel owner."
```

Catatan terakhir itu jadi petunjuk paling penting: perusahaan yang muncul di dokumen cargo bukan berarti dia pemilik kapalnya — jadi identitas kepemilikan asli harus ditelusuri lewat sumber lain.

---

## Tools yang Dipakai

| Tool | Kegunaan |
|---|---|
| **Mission Briefing** | Membaca objective soal dan bukti awal (IMO, P&I entry) |
| **Maritime Registry** | Mencari data registrasi kapal berdasarkan nomor IMO |
| **P&I Directory** | Mencari detail entri asuransi kapal (registered owner, operator, charterer) |
| **Companies Register** | Menelusuri profil perusahaan: direktur, pemegang saham, parent/subsidiary |
| **Oath Submission** | Form untuk mengirim jawaban lima objective dan memicu penilaian |

---

## Step-by-Step Penyelesaian

### 1. Baca Mission Briefing

Halaman awal berisi konteks cerita dan bukti starting evidence: **IMO 9724418** dan **P&I Entry PI-VAL-88291**. Ini dua identifier utama yang jadi titik masuk untuk seluruh investigasi.

**Kenapa langkah ini diambil:** selalu mulai dari briefing untuk tahu persis apa yang perlu dibuktikan (di sini: 5 pertanyaan spesifik), bukan sekadar menebak-nebak arah investigasi.

### 2. Cari kapal di Maritime Registry pakai nomor IMO

Buka **Maritime Registry**, masukkan `9724418` di kolom pencarian.

**Hasil:**
```
VESSEL:       ASHEN MERCY
IMO:          9724418
MMSI:         636019772
Flag State:   Outer Isles Register
Vessel Type:  General Cargo
Year Built:   2014
```

Scroll ke bawah, ditemukan bagian **"Registered Ownership & Management"**:

```
REGISTERED OWNER:   Thirteenth Tide Shipping Ltd
ISM MANAGER:        Morrow Fleet Management SA
TECHNICAL MANAGER:  Morrow Fleet Management SA
```

Ini langsung menjawab **objective #1 dan #2**.

**Catatan di halaman itu:** *"The registered owner is the legal owner of the vessel asset. This record does not indicate commercial operation or charter arrangements. Consult the P&I Directory for operational details."* — artinya untuk operator dan charterer, harus dicari di sumber lain (P&I Directory), bukan Maritime Registry.

### 3. Cari di P&I Directory — perhatikan formatnya

Percobaan pertama search dengan `PI-VAL-88291` di **P&I Directory** menghasilkan **"NO ENTRY FOUND"**.

**Kenapa gagal:** format kode P&I yang tertulis di dokumen awal ternyata tidak persis sama dengan yang dipakai sistem pencarian internal.

**Solusi:** ikuti saran alternatif yang ditampilkan sistem sendiri ("Try: PI-VAL-88291 · IMO 9724418") — search ulang pakai **nomor IMO** (`9724418`) alih-alih kode P&I.

**Hasil:**
```
P&I ENTRY: PI-VAL-88291
STATUS:    Active
VESSEL:    ASHEN MERCY (IMO 9724418)

PARTIES — INSURANCE RECORD
MEMBER / REGISTERED OWNER:  Thirteenth Tide Shipping Ltd
CO-ASSURED:                 Morrow Fleet Management SA (ISM Manager)
COMMERCIAL OPERATOR:        Eastreach Maritime Coordination PLC
TIME CHARTERER:             Gilded Knife Commodities Ltd
```

Ini menjawab **objective #3 dan #4** sekaligus mengonfirmasi ulang jawaban #1 dan #2.

Ada juga **Analyst Note** di bagian bawah entri ini:
> *"The commercial operator and charterer listed in this P&I entry differ from the registered owner. This is standard for time-chartered vessels operating under third-party commercial management. Investigate the corporate relationships between these parties."*

Ini jadi petunjuk eksplisit untuk lanjut ke langkah berikutnya: menelusuri hubungan korporat antar-perusahaan, bukan berhenti di sini.

### 4. Telusuri hubungan korporat di Companies Register

Klik nama **"Eastreach Maritime Coordination PLC"** dari P&I Directory, yang membawa ke **Companies Register**.

**Data yang ditemukan:**
```
COMPANY NUMBER:     ER-220014
REGISTERED OFFICE:  3 Saltmarsh Lane, Eryndal
BUSINESS:           Port agency, voyage planning, cargo coordination, bunker procurement

DIRECTORS:
  01. Damas Marrowcairn
  02. Leven Orr
  03. Sera Dain

SHAREHOLDERS:
  Marrowcairn Strategic Holdings PLC — 100%

CORPORATE STRUCTURE:
  PARENT COMPANY: Marrowcairn Strategic Holdings PLC
```

Nama **Damas Marrowcairn** sudah langsung muncul sebagai direktur, dan perusahaan ini 100% dimiliki oleh **Marrowcairn Strategic Holdings PLC**.

**Kenapa langkah ini penting:** ini titik pertama di mana nama Marrowcairn muncul secara eksplisit, mengonfirmasi arah investigasi sudah benar.

### 5. Verifikasi di level holding company

Klik masuk ke **"Marrowcairn Strategic Holdings PLC"** (parent company yang baru ditemukan).

**Data yang ditemukan:**
```
DIRECTOR 01:      Damas Marrowcairn

SHAREHOLDERS:
  Damas Marrowcairn — 72%
  Unnamed institutional investors — 28%

CORPORATE STRUCTURE (Subsidiaries):
  01. Eastreach Maritime Coordination PLC   ← commercial operator
  02. Gilded Knife Commodities Ltd          ← time charterer
  03. Eastreach Trade Credit Ltd
```

**Analyst Note** di halaman ini menyatakan secara eksplisit:
> *"Controls both the commercial operator and the time charterer of the vessel."*

Ini menjawab **objective #5** sekaligus jadi bukti "smoking gun" investigasi: Marrowcairn Strategic Holdings PLC bukan cuma nyambung ke operator, tapi juga jadi parent company dari **operator sekaligus charterer** kapal ASHEN MERCY — sesuai persis dengan yang ditanyakan.

### 6. Submit jawaban ke Oath Submission — dan perbaiki kesalahan di Q05

Kelima jawaban dimasukkan ke form **Oath Submission**:

| No | Pertanyaan | Jawaban |
|---|---|---|
| Q01 | Registered Owner | Thirteenth Tide Shipping Ltd |
| Q02 | ISM Manager | Morrow Fleet Management SA |
| Q03 | Commercial Operator | Eastreach Maritime Coordination PLC |
| Q04 | Time Charterer | Gilded Knife Commodities Ltd |
| Q05 | Ultimate Controller | ~~Gilded Knife Commodities Ltd~~ → **Marrowcairn Strategic Holdings PLC** |

Percobaan pertama di **Q05** salah, karena secara tidak sengaja diisi dengan jawaban Q04 ("Gilded Knife Commodities Ltd" — itu jawaban time charterer, bukan parent company). Setelah dikoreksi jadi **"Marrowcairn Strategic Holdings PLC"** (sesuai temuan di langkah 5, yang mengontrol *both* operator dan charterer), submission berhasil dengan skor **5/5**.

**Pelajaran:** saat form meminta jawaban berbeda-beda untuk pertanyaan yang mirip (di sini: charterer vs. ultimate controller), selalu cek ulang jawaban sebelum submit — gampang salah tempel kalau jawabannya berasal dari catatan yang sama.

### 7. Susun flag dari kalimat penutup case

Setelah submission berhasil, halaman menampilkan status **"Scenario Pwned"** beserta narasi penutup:

> *"...clean because the ink has been divided among **five names**."*

Format flag yang diminta: `HTB{ITEM_WAS_SHARED_AMONG_QUANTITY_TARGETS}` (contoh: `HTB{THE_WATER_WAS_RATIONED_AMONG_SIX_VILLAGES}`).

Memetakan kalimat penutup ke format itu:
- **ITEM** → THE INK
- **VERB** → WAS DIVIDED
- **QUANTITY** → FIVE
- **TARGETS** → NAMES

**Flag akhir:**
```
HTB{THE_INK_WAS_DIVIDED_AMONG_FIVE_NAMES}
```

"Five names" ini konsisten dengan narasi awal soal: lima perusahaan berlapis (shell company, management firm, coordination house, commodities trader, dan holding company) yang semuanya berujung ke tangan yang sama.

---

## Kesimpulan

Challenge ini adalah latihan OSINT klasik tentang **menelusuri Ultimate Beneficial Owner (UBO)** di balik struktur korporat berlapis — sebuah teknik yang benar-benar dipakai di dunia nyata untuk investigasi kejahatan finansial, sanksi, dan pencucian uang lewat shell company.

**Alur investigasinya:**
1. **Maritime Registry** (by IMO) → registered owner + ISM manager
2. **P&I Directory** (by IMO, bukan kode P&I secara langsung) → commercial operator + time charterer, plus petunjuk untuk lanjut ke corporate structure
3. **Companies Register** → menelusuri direktur dan shareholder dari operator, naik satu level ke parent company, sampai ketemu nama yang sama muncul sebagai direktur/pemegang saham mayoritas *dan* pengendali kedua entitas kunci (operator + charterer) sekaligus

**Pelajaran penting** dari challenge ini:

- **Satu sumber data jarang cukup.** Registry kapal, direktori asuransi, dan company register masing-masing menyimpan sebagian kecil informasi; harus digabung untuk dapat gambaran penuh.
- **Perhatikan format identifier yang tersedia.** Ketika satu kode (di sini: kode P&I) tidak ditemukan, coba identifier alternatif yang lebih standar (nomor IMO) — sering kali sistem sendiri memberi petunjuk lewat placeholder atau pesan error.
- **Ikuti "analyst notes" atau catatan di setiap halaman.** Catatan-catatan itu biasanya bukan sekadar flavor text — mereka secara eksplisit mengarahkan langkah investigasi berikutnya (di sini: "Investigate the corporate relationships between these parties").
- **Kepemilikan yang sah secara hukum tidak sama dengan kendali sebenarnya.** Registered owner (Thirteenth Tide Shipping Ltd) legal, tapi kendali sesungguhnya atas commercial operator *dan* time charterer ada di tangan satu orang lewat holding company — pola umum dalam struktur ownership yang sengaja dibuat berlapis untuk kabur.
- **Cek ulang sebelum submit**, terutama di form dengan banyak field yang jawabannya mirip — kesalahan copy-paste antar-field adalah human error paling umum di tahap akhir.
