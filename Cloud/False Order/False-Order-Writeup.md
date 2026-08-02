# False Order — Cloud CTF Writeup

**Kategori:** Cloud (AWS / CloudTrail Forensics)
**Difficulty:** Medium
**Skills:** IAM, AssumeRole, S3 Versioning, CloudTrail Log Analysis

---

## Deskripsi Soal

> Caldrin Vowmark reaches an Ashguard checkpoint with a sealed order that tells
> Stormbound's soldiers to leave the east gate and report to Crownspire. The
> officer in charge believes the order came from Garran Voss, and he will move
> his soldiers as soon as the seal is checked. If they leave, Vaultrune can
> take the gate before help arrives. Caldrin knows Garran's orders carry small
> marks that copyists miss. He has to inspect the order, show the officer that
> it is false, and stop the unit from leaving before Vaultrune's soldiers
> reach the gate.

Secara teknis, kita diminta menginvestigasi sebuah **AWS environment** untuk
membuktikan bahwa sebuah objek S3 (`east-gate-order.json`) telah dipalsukan,
dan menelusuri jejaknya lewat **CloudTrail**: siapa pelakunya, IP apa yang
dipakai, role apa yang di-*assume*, dan aksi S3 apa saja yang terjadi.

Kita diberi kredensial read-only investigator untuk mengakses:
- CloudTrail trail: `coalition-gate-audit-trail`
- S3 bucket (versioned): `ashguard-order-custody`
- Target object: `custody/east-gate-order.json`

---

## Tools yang Dipakai

| Tool | Kegunaan |
|---|---|
| **AWS CLI v2** | Berinteraksi dengan endpoint AWS tiruan (mock API) milik challenge |
| **PowerShell** | Shell environment (Windows) untuk parsing & filtering JSON |
| **Browser** | Membaca briefing soal & mengambil starting credentials |

> Challenge ini menggunakan endpoint AWS API **tiruan** (kemungkinan
> LocalStack/Moto-style mock server), bukan akun AWS asli — jadi semua
> request harus diarahkan ke IP:port instance, bukan ke AWS beneran.

---

## Step-by-Step Penyelesaian

### 1. Membaca briefing & mengambil kredensial awal

Saat membuka instance, ada dua alamat penting yang diberikan:

- **Docker instance (halaman cerita / briefing):** `154.57.164.77:30362`
- **AWS API endpoint (Docker spawned):** `154.57.164.77:32392`

Dari halaman briefing, didapat kredensial IAM awal:

```
IAM User          : gate-investigator
Access Key ID     : AKIATOOLDGPE9OZ8T0O3
Secret Access Key : E0XbjuYExL3/0YY1/SOm+cHEnp2o/c0q0TYUll+n
Region            : us-east-1
```

**Catatan penting:** halaman briefing secara eksplisit mengingatkan untuk
mengarahkan `AWS_ENDPOINT_URL` ke **port AWS API**, bukan port halaman cerita
di address bar. Ini jebakan umum di soal cloud CTF — mudah salah pakai port.

### 2. Setup environment (PowerShell)

```powershell
$env:AWS_ENDPOINT_URL = "http://154.57.164.77:32392"
$env:AWS_DEFAULT_REGION = "us-east-1"
$env:AWS_ACCESS_KEY_ID = "AKIATOOLDGPE9OZ8T0O3"
$env:AWS_SECRET_ACCESS_KEY = "E0XbjuYExL3/0YY1/SOm+cHEnp2o/c0q0TYUll+n"
Remove-Item Env:\AWS_SESSION_TOKEN -ErrorAction SilentlyContinue

aws sts get-caller-identity --endpoint-url http://154.57.164.77:32392
```

Verifikasi berhasil dengan output:

```json
{
    "UserId": "AIDARTVREFCT1MFL0HAJ",
    "Account": "638291047582",
    "Arn": "arn:aws:iam::638291047582:user/gate-investigator"
}
```

> **Troubleshooting yang dialami:** sempat muncul error
> `InvalidClientTokenId`. Setelah dicek ulang huruf demi huruf, penyebabnya
> adalah **salah baca Access Key ID** — tertukar antara angka `0` (nol) dan
> huruf `O` di bagian `...GPE9**0**Z8...` vs `...GPE9**O**Z8...`. Ini
> mengingatkan bahwa saat menyalin credential dari screenshot, karakter
> ambigu seperti `0/O`, `1/l/I` wajib dicek ulang secara teliti.

Supaya tidak perlu mengetik `--endpoint-url` di setiap command, bisa di-set
permanen untuk sesi:

```powershell
aws configure set default.endpoint_url http://154.57.164.77:32392
```

### 3. Menarik seluruh log CloudTrail

Karena jumlah event lebih dari batas satu halaman (`--max-results 50`),
AWS CLI mengembalikan `NextToken` untuk pagination. Solusi paling praktis
adalah **tidak membatasi `--max-results`**, sehingga AWS CLI otomatis
melakukan auto-pagination:

```powershell
aws cloudtrail lookup-events --endpoint-url http://154.57.164.77:32392 --output json > events_all.json

# Pastikan tidak ada NextToken tersisa
Get-Content events_all.json | Select-String "NextToken"
```

Hasil: total **610 events** berhasil ditarik dalam satu file.

### 4. Parsing & mengurutkan event berdasarkan waktu

Field `CloudTrailEvent` di dalam response adalah string JSON bersarang,
sehingga perlu di-*decode* ulang:

```powershell
$check  = Get-Content events_all.json -Raw | ConvertFrom-Json
$events = $check.Events | ForEach-Object { $_.CloudTrailEvent | ConvertFrom-Json }
$sorted = $events | Sort-Object eventTime

$sorted | Select-Object eventTime, eventName, sourceIPAddress, `
    @{N='userName';E={$_.userIdentity.userName}}, `
    @{N='userArn';E={$_.userIdentity.arn}}, `
    errorCode | Format-Table -AutoSize
```

Dari tabel timeline ini terlihat pola aktivitas:
- IP internal **`10.41.53.22`** (`coalition-gate-clerk`) melakukan aktivitas
  rutin/normal (`ListObjectsV2`, `GetObject`, `HeadObject`, dll) dari
  **21 Juli** hingga **25 Juli jam 17:34 UTC**.
- Setelah itu muncul IP asing **`198.18.44.91`** dengan identitas
  **`seal-copyist-contractor`** yang mulai melakukan reconnaissance.

### 5. Investigasi detail aktivitas mencurigakan

Untuk melihat detail penuh tiap event (termasuk `requestParameters` dan
`responseElements`) dari IP mencurigakan dan sesi *assumed-role*:

```powershell
$sorted | Where-Object {
    $_.sourceIPAddress -eq "198.18.44.91" -or
    ($_.userIdentity.arn -like "*assumed-role*")
} | ForEach-Object {
    "=== $($_.eventTime) | $($_.eventName) | errorCode=$($_.errorCode) ==="
    "userIdentity.arn: $($_.userIdentity.arn)"
    "requestParameters: $($_.requestParameters | ConvertTo-Json -Compress)"
    "responseElements: $($_.responseElements | ConvertTo-Json -Compress)"
    "---"
} | Out-File attacker_detail.txt

Get-Content attacker_detail.txt
```

Dari sini terungkap **rangkaian serangan lengkap**:

1. `seal-copyist-contractor` login pertama kali → `GetCallerIdentity`
   (`17:35:00Z`)
2. Recon bucket & object → `ListBuckets`, `ListObjectsV2`
3. Coba baca objek target langsung → **`GetObject` ditolak**
   (`errorCode: AccessDenied`)
4. Coba `AssumeRole` ke role `ashguard-order-auditor` → **ditolak**
   (`errorCode: AccessDenied`)
5. Coba lagi kombinasi *probe* `GetObject` + `AssumeRole` ke role yang sama →
   **ditolak lagi**
6. Berhasil `AssumeRole` ke role **`ashguard-order-scanner`** dengan
   `roleSessionName: coalition-gate-clerk` — menyamar seolah sesi ini milik
   user internal yang sah (teknik *log spoofing/atribusi palsu*)
7. Dengan kredensial *assumed-role* tsb:
   - `ListBucketVersions` pada object target
   - **`DeleteObject`** → memasang *delete marker* (karena bucket
     versioning aktif, bukan hard delete)
   - **`PutObject`** → mengunggah versi objek baru (order palsu)

### 6. Verifikasi lewat S3 Object Versioning

Karena bucket menggunakan **versioning**, kita bisa membuktikan manipulasi
dengan melihat riwayat versi objek:

```powershell
aws s3api list-object-versions `
    --endpoint-url http://154.57.164.77:32392 `
    --bucket ashguard-order-custody `
    --prefix custody/east-gate-order.json
```

Version ID yang ditemukan di log CloudTrail cocok dengan hasil di sini:
- `54df2487-8a99-4843-85a7-71afb1f1b9b6` → *delete marker* (dari `DeleteObject`)
- `561957bd-e5cc-468d-884d-0e1c7c16b68d` → versi baru/palsu (dari `PutObject`)

Ini membuktikan bahwa order asli tidak dihapus permanen — cukup diberi
delete-marker — lalu digantikan versi palsu, persis seperti "tanda kecil
yang dilewatkan penyalin" yang disebutkan di narasi soal.

---

## Flag

| # | Pertanyaan | Jawaban |
|---|---|---|
| 1 | Aksi CloudTrail terakhir dari IP gatehouse internal sebelum sesi attacker mulai | `ListObjectsV2` (`2026-07-25T17:34:01.001Z`, IP `10.41.53.22`) |
| 2 | Aksi pertama dari IP attacker | `GetCallerIdentity` (`2026-07-25T17:35:00.624Z`, IP `198.18.44.91`) |
| 3 | Aksi S3 yang ditolak sebelum assume role | `GetObject` |
| 4 | Path S3 objek yang ditampering | `s3://ashguard-order-custody/custody/east-gate-order.json` |
| 5 | ARN role yang di-assume (sesi destruktif) | `arn:aws:iam::638291047582:role/ashguard-order-scanner` |
| 6 | ARN principal STS pada `DeleteObject` | `arn:aws:sts::638291047582:assumed-role/ashguard-order-scanner/coalition-gate-clerk` |
| 7 | IP untuk `AssumeRole` & aksi S3 destruktif | `198.18.44.91` |
| 8 | Username IAM pemilik kredensial long-lived pemanggil `AssumeRole` | `seal-copyist-contractor` |
| 9 | Nama role yang gagal di-assume sebelum sukses | `ashguard-order-auditor` |
| 10 | `roleSessionName` pada `AssumeRole` sukses | `coalition-gate-clerk` |
| 11 | `errorCode` pada probe `GetObject` yang ditolak | `AccessDenied` |
| 12 | Aksi S3 yang menandai upload ledger palsu setelah `DeleteObject` | `PutObject` |

---

## Kesimpulan

Challenge **False Order** adalah simulasi kasus **credential misuse & role
chaining** di lingkungan cloud:

1. **Root cause**: sebuah IAM user (`seal-copyist-contractor`) dengan
   kredensial long-lived berhasil melakukan *lateral movement* lewat
   `AssumeRole` — mencoba beberapa role sampai menemukan satu yang
   memiliki kebijakan izin terlalu longgar (`ashguard-order-scanner`).
2. **Teknik penyamaran**: `roleSessionName` diset menyerupai identitas user
   sah (`coalition-gate-clerk`) untuk mengaburkan atribusi di log.
3. **Dampak**: objek kritikal (order digital) berhasil diganti dengan versi
   palsu, namun tetap terdeteksi karena bucket S3 menggunakan **versioning**
   — sehingga delete tidak permanen dan histori versi bisa dijadikan bukti
   forensik.
4. **Pelajaran utama**:
   - Selalu aktifkan **S3 Versioning** pada bucket yang menyimpan data
     kritikal — ini menyelamatkan investigasi di soal ini.
   - **Least privilege** pada IAM Role sangat penting — role
     `ashguard-order-scanner` seharusnya tidak punya izin `DeleteObject`
     dan `PutObject` jika fungsinya hanya "scanner".
   - CloudTrail adalah sumber kebenaran (*source of truth*) untuk
     merekonstruksi timeline serangan — kombinasi `sourceIPAddress`,
     `userIdentity.arn`, `errorCode`, dan `requestParameters` cukup untuk
     menyusun cerita lengkap dari awal login sampai eksekusi akhir.

---
