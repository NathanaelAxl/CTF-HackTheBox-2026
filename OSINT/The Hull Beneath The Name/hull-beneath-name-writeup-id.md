# [OSINT] The Hull Beneath The Name

> **Kategori:** OSINT
> **Flag:** `HTB{BRINEWALKER_9384728_E06}`

---

## Deskripsi Soal

Masih di environment "Court-Eaves Console" (menu: Mission Briefing, Registry Search, Harbor Ledger, Tideglass Browser, Evidence Satchel, Oath Submission), kali ini investigasinya berpusat pada satu catatan tangan yang tergesa-gesa.

**Cerita singkat:**

> Dermaga Eastreach tidak pernah tidur. Kargo bergerak di bawah segel resmi, harbor clerk mengesahkan manifest yang jarang benar-benar mereka baca, dan rumah dagang Lord Damas Marrowcairn bersikeras setiap muatan itu biasa saja: minyak lampu, tali pemakaman, pigmen mineral, garam pengawet. Tapi seorang signal clerk yang ketakutan sempat mencatat sebuah nomor sebelum fajar — MMSI dari sebuah kapal kargo abu-abu yang oleh kantor kargo disebut **BRINE WALKER**, meski huruf di buritan kapalnya terlihat lebih pendek. Segel kargo Eastreach-nya bertuliskan **EC-4418**, dan kapal itu sudah pergi sebelum tengah hari.

**Dock Scribe's Note (bukti awal, dari Mission Briefing):**
```
"Grey general cargo vessel. Cargo office called her BRINE WALKER,
but the stern letters looked shorter. Signal clerk copied MMSI
257771420. Entered before dawn. Eastreach cargo seal EC-4418."
```

**Investigation Targets (4 hal yang harus dibuktikan):**
1. Identify the vessel's current registered name.
2. Recover the vessel's IMO number.
3. Determine the Eastreach berth where cargo was discharged.
4. Confirm the vessel's declared previous port.

**Flag format:** `HTB{VESSELNAME_IMO_BERTH}`
**Contoh:** `HTB{EXAMPLEVESSEL_1234567_E01}`

Jadi target akhirnya jelas: hanya 3 dari 4 data itu (nama, IMO, berth) yang perlu dirangkai jadi flag — previous port dipakai untuk validasi silang saja, bukan bagian dari flag.

---

## Tools yang Dipakai

| Tool | Kegunaan |
|---|---|
| **Mission Briefing** | Membaca bukti awal (dock scribe's note) dan 4 investigation target |
| **Registry Search (Stormcoast Maritime Register)** | Mencari identitas kapal berdasarkan MMSI, nama, IMO, atau callsign |
| **Harbor Ledger** | Mencari data kedatangan/keberangkatan kapal berdasarkan cargo seal, termasuk berth dan previous port |

---

## Step-by-Step Penyelesaian

### 1. Baca Mission Briefing dan catat identifier yang tersedia

Dari dock scribe's note, ada dua identifier konkret yang bisa langsung dipakai untuk pencarian:
- **MMSI:** `257771420`
- **Cargo seal (customs reference):** `EC-4418`

**Kenapa ini penting:** MMSI jauh lebih reliable dibanding nama kapal yang disebut si scribe ("BRINE WALKER") — nama itu sendiri sudah diberi catatan kecurigaan ("stern letters looked shorter"), jadi jangan langsung dipakai sebagai kata kunci pencarian utama.

### 2. Cari kapal di Registry Search pakai MMSI

Buka **Registry Search**, ganti field ke **MMSI**, masukkan `257771420`.

**Hasil (1 record found):**
```
CURRENT NAME:   BRINEWALKER   (ex BRINE WALKER)
MMSI:           257771420
IMO NUMBER:     9384728
CALLSIGN:       LAVQ7
FLAG STATE:     Stormcoast Maritime Register
VESSEL TYPE:    General Cargo Vessel
YEAR BUILT:     2007
GROSS TONNAGE:  6,842 GT
DEADWEIGHT:     9,114 DWT
```

Ini langsung menjawab **investigation target #1 dan #2**:
- **Current registered name:** `BRINEWALKER`
- **IMO Number:** `9384728`

**Catatan soal "stern letters looked shorter":** ini ternyata bukan petunjuk untuk mencari kapal *lain* dengan nama berbeda — itu cuma detail naratif yang menjelaskan kenapa nama resmi di registry (`BRINEWALKER`, tanpa spasi) sedikit beda dari yang didengar/dicatat scribe secara lisan (`BRINE WALKER`, dengan spasi). Field **"ex BRINE WALKER"** di record ini mengonfirmasi itu cuma variasi penulisan nama yang sama, bukan identitas ganda.

### 3. Cari data pelabuhan lewat Harbor Ledger, pakai cargo seal

Buka **Harbor Ledger**, cari baris dengan **Customs reference `EC-4418`** (langsung terlihat di kolom paling kanan tabel).

**Hasil (baris yang cocok):**
```
PORT CALL ID:    EPA-2026-0717-044
VESSEL:          BRINEWALKER
IMO:             9384728
BERTHED UTC:     2026-07-17 03:44 UTC
BERTH:           E-06
PREVIOUS PORT:   Saltmere Roads
STATUS:          CLEARED
CUSTOMS:         EC-4418
```

Baris ini menjawab **investigation target #3 dan #4** sekaligus, dan IMO-nya (`9384728`) cocok persis dengan hasil Registry Search di langkah sebelumnya — konfirmasi silang bahwa kita mencocokkan kapal yang benar.

- **Berth:** `E-06`
- **Previous port:** `Saltmere Roads`

**Kenapa cross-check ini penting:** dengan dua sumber data independen (Registry Search via MMSI, Harbor Ledger via cargo seal) yang sama-sama mengarah ke IMO `9384728`, kita punya kepastian bahwa identitas kapal ini konsisten — bukan cuma tebakan dari satu sumber saja.

### 4. (Opsional) Buka detail cargo declaration untuk konteks tambahan

Klik link `EC-4418` di Harbor Ledger untuk masuk ke halaman **Eastreach Customs — Cargo Declaration**. Isinya konsisten dengan cerita di Mission Briefing (lamp oil, bell cord, mineral compound — "funeral rites, municipal bell maintenance, and winter preservation"), dengan **Notify Party: Eastreach Maritime Coordination PLC**. Bagian ini tidak menambah data baru untuk flag, tapi membantu memastikan tidak ada informasi lain yang terlewat sebelum submit jawaban.

### 5. Susun jawaban ke Oath Submission

Empat jawaban investigation target dimasukkan:

| Target | Jawaban |
|---|---|
| Current registered name | BRINEWALKER |
| IMO number | 9384728 |
| Berth | E-06 |
| Previous port | Saltmere Roads |

### 6. Rangkai flag sesuai format

Format flag: `HTB{VESSELNAME_IMO_BERTH}`

- **VESSELNAME** → `BRINEWALKER`
- **IMO** → `9384728`
- **BERTH** → `E06` (tanda hubung pada "E-06" dihilangkan, mengikuti pola di contoh resmi `E01` bukan `E-01`)

**Flag akhir:**
```
HTB{BRINEWALKER_9384728_E06}
```

Submission berhasil dengan status **"Scenario Pwned"**.

---

## Kesimpulan

Challenge ini jauh lebih straightforward dibanding kesan pertamanya. Narasinya ("stern letters looked shorter", "cargo office called her BRINE WALKER") sengaja ditulis untuk terdengar seperti soal tentang **penyamaran identitas kapal** (AIS spoofing, rename history, dsb.) — dan itu sempat membuat proses investigasi awal berputar-putar mencari "nama asli yang tersembunyi" lewat name history, MMSI lain, dan spesifikasi fisik kapal yang mirip. Semua jalur itu ternyata jalan buntu, karena target sebenarnya sudah dijabarkan eksplisit di Mission Briefing: cukup **nama terdaftar saat ini, nomor IMO, berth, dan previous port** — bukan mengungkap identitas ganda.

**Alur investigasi yang benar:**
1. **Mission Briefing** → ambil identifier paling reliable dari bukti awal (MMSI, bukan nama)
2. **Registry Search by MMSI** → nama resmi kapal + IMO
3. **Harbor Ledger by cargo seal** → berth + previous port, sekaligus verifikasi silang IMO

**Pelajaran penting** dari challenge ini:

- **Selalu baca instruksi/objective secara lengkap sebelum mulai menebak arah investigasi.** Detail naratif yang dramatis (seperti "letters looked shorter") bisa jadi cuma bumbu cerita, bukan clue teknis — sementara jawaban sesungguhnya sering sudah tertulis eksplisit di bagian objective/briefing.
- **Gunakan identifier paling unik dan sulit dipalsukan sebagai kata kunci pencarian pertama** (MMSI/IMO), bukan nama yang bisa bervariasi penulisannya (spasi, kapitalisasi) atau memang sengaja disorot sebagai "meragukan" di narasi.
- **Selalu cross-check lewat dua sumber data independen** (di sini: Registry Search dan Harbor Ledger) sebelum menyimpulkan jawaban — kalau keduanya mengarah ke IMO yang sama, itu konfirmasi kuat.
- **Perhatikan detail kecil di format flag**, seperti tanda hubung yang dihilangkan pada kode berth — bandingkan selalu dengan contoh yang diberikan sebelum submit.
