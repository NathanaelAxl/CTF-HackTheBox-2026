# [Cloud] Wrong Stamp — CTF Writeup

## Deskripsi Soal

**Kategori:** Cloud (AWS CloudTrail Forensics)
**Nama Challenge:** Wrong Stamp

> Elric Ashspar finds a seizure stamp in a dead clerk's bag near Stonepass. Vaultrune uses copies of that stamp to take supplies from Sythra's border guards, leaving a road Stormbound needs open without them. The stamp looks old, but Elric spots fresh tool marks on it. He must find the flaw, prove the stamp is fake, and give the guards a quick way to reject future copies.
>
> Stonepass's surviving CloudTrail history is the only record of activity surrounding the copied stamp. You hold read-only investigator access. Reconstruct the final recorded actions and determine which identity stopped the trail from logging.

Secara singkat, ini adalah challenge **investigasi forensik cloud**: kita diberi kredensial AWS read-only dan diminta merekonstruksi kronologi serangan dari log **AWS CloudTrail**, untuk menemukan siapa yang menyusup dan bagaimana mereka mematikan audit trail.

### Flag yang harus dijawab

1. What was the last CloudTrail API action performed by the compromised user from the internal IP immediately before the attacker session began?
2. What was the first CloudTrail API action called from the attacker IP?
3. Which API action did the attacker attempt that was explicitly denied before the trail was stopped?
4. Which S3 bucket did the attacker enumerate before stopping the trail?
5. What is the name of the CloudTrail trail that was stopped?
6. Which IAM username's credentials were used to execute the trail disable?
7. From which IP address was the trail disabled?
8. Which API action was used to disable the audit trail?

## Tools yang Dipakai

- **AWS CLI v2** — untuk berinteraksi dengan endpoint AWS API yang disediakan panitia
- **PowerShell** — shell environment (Windows) untuk menjalankan command dan parsing JSON
- **Browser** — untuk membaca briefing soal dan kredensial awal

## Persiapan

Dari packet soal, disediakan dua endpoint:

| Port | Fungsi |
|---|---|
| `:30797` | Halaman briefing/cerita (bukan endpoint API) |
| `:30686` | **AWS API endpoint** — ini yang dipakai untuk `AWS_ENDPOINT_URL` |

Cara memastikan mana yang endpoint API: saat diakses langsung lewat browser, port `30686` mengembalikan response JSON khas AWS:

```json
{
  "__type": "AccessDeniedException",
  "message": "User is not authorized to perform: MissingAuthentication"
}
```

Response inilah yang menjadi penanda bahwa port tersebut adalah AWS API endpoint (bukan halaman web statis), karena hanya AWS API yang membalas dengan format error seperti ini ketika diakses tanpa signature/autentikasi yang valid.

Kredensial awal (read-only investigator) diberikan di halaman briefing:

```
IAM_USER            = stonepass-investigator
ACCESS_KEY_ID        = AKIAUMY861XLXFUW6KV8
SECRET_ACCESS_KEY    = w9T8XcGEgc3kNCJVpcih2cBHGnBI304j9rA7VdSc
REGION               = us-east-1
```

## Step-by-Step Penyelesaian

### 1. Install AWS CLI v2

Download dan install AWS CLI v2 dari situs resmi AWS. Pastikan proses instalasi berjalan mulus (kalau sempat *interrupted*, jalankan ulang installer dengan hak akses Administrator).

Cek instalasi berhasil:
```powershell
aws --version
```

### 2. Setup environment variable

Karena challenge menggunakan endpoint AWS API custom (bukan AWS resmi), kita perlu override `AWS_ENDPOINT_URL` agar CLI mengarah ke server soal, bukan ke `amazonaws.com`.

```powershell
$env:AWS_ENDPOINT_URL="http://154.57.164.65:30686"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_ACCESS_KEY_ID="AKIAUMY861XLXFUW6KV8"
$env:AWS_SECRET_ACCESS_KEY="w9T8XcGEgc3kNCJVpcih2cBHGnBI304j9rA7VdSc"
Remove-Item Env:\AWS_SESSION_TOKEN -ErrorAction SilentlyContinue
```

`AWS_SESSION_TOKEN` di-unset karena kredensial yang diberikan adalah *long-term access key* (IAM User), bukan kredensial sementara (STS), jadi token sesi tidak diperlukan dan bisa menyebabkan konflik jika tersisa dari konfigurasi lama.

### 3. Verifikasi identitas

```powershell
aws sts get-caller-identity
```

Output:
```json
{
    "UserId": "AIDAEE1O11J7IK08X0WV",
    "Account": "491827305948",
    "Arn": "arn:aws:iam::491827305948:user/stonepass-investigator"
}
```

Ini mengonfirmasi endpoint dan kredensial sudah benar — kita berhasil autentikasi sebagai `stonepass-investigator`.

### 4. Tarik seluruh CloudTrail event history

```powershell
aws cloudtrail lookup-events --max-results 50 > events.json
```

`lookup-events` mengambil riwayat event dari **CloudTrail Event History** (API `LookupEvents`), yaitu source data utama yang bisa diakses investigator read-only tanpa perlu akses langsung ke S3 bucket log mentah.

### 5. Parsing JSON menjadi timeline yang mudah dibaca

Data mentah `events.json` berbentuk JSON bertumpuk dan tidak berurutan waktu secara natural saat dibaca manual, jadi perlu di-parse ke tabel/CSV agar bisa dianalisis secara kronologis.

```powershell
Get-Content events.json -Raw | ConvertFrom-Json |
  Select-Object -ExpandProperty Events |
  ForEach-Object {
    $ct = $_.CloudTrailEvent | ConvertFrom-Json
    [PSCustomObject]@{
      Time     = $_.EventTime
      Event    = $_.EventName
      User     = $ct.userIdentity.userName
      SourceIP = $ct.sourceIPAddress
      Error    = $ct.errorCode
      Bucket   = $ct.requestParameters.bucketName
    }
  } | Sort-Object Time | Export-Csv -Path timeline.csv -NoTypeInformation

Get-Content timeline.csv
```

Field yang diekstrak (`Time`, `Event`, `User`, `SourceIP`, `Error`, `Bucket`) dipilih karena field-field inilah yang relevan untuk menjawab pertanyaan investigasi: siapa melakukan apa, dari IP mana, apakah berhasil/ditolak, dan resource apa yang disentuh.

### 6. Analisis timeline

Setelah timeline diurutkan berdasarkan waktu (`Sort-Object Time`), terlihat tiga fase aktivitas yang jelas:

**Fase 1 — Aktivitas normal user (IP internal `10.30.41.118`)**
User `stonepass-warden` melakukan aktivitas rutin (DescribeTrails, GetTrailStatus, ListUsers, dsb) dari IP internal, berlangsung dari `22:08:38` hingga `03:08:17.750` (event terakhir: `ListAccessKeys`).

**Fase 2 — Setup environment (`127.0.0.1`)**
Terdapat burst event tanpa username dari `127.0.0.1` (StopLogging/DeleteTrail gagal dengan `InternalFailure`, lalu CreateTrail, StartLogging, CreateUser, PutUserPolicy, CreateAccessKey). Fase ini diidentifikasi sebagai *noise* provisioning environment challenge, bukan bagian dari narasi serangan, sehingga diabaikan dalam analisis.

**Fase 3 — Sesi penyerang (IP eksternal `192.0.2.55`, memakai kredensial `stonepass-warden` yang dicuri)**
Dimulai `03:08:38.781` (`GetTrailStatus`) hingga `03:08:48.734` (`StopLogging` berhasil).

Cuplikan bagian kritis dari `timeline.csv`:

```
"2026-07-26T03:08:17.750000+07:00","ListAccessKeys","stonepass-warden","10.30.41.118",,
"2026-07-26T03:08:38.781000+07:00","GetTrailStatus","stonepass-warden","192.0.2.55",,
"2026-07-26T03:08:41.313000+07:00","DeleteTrail","stonepass-warden","192.0.2.55","AccessDeniedException",
"2026-07-26T03:08:43.266000+07:00","ListBucket","stonepass-warden","192.0.2.55",,"stonepass-audit-trail-logs"
"2026-07-26T03:08:45.281000+07:00","ListObjectsV2","stonepass-warden","192.0.2.55",,"stonepass-audit-trail-logs"
"2026-07-26T03:08:47.100000+07:00","GetObject","stonepass-warden","192.0.2.55","NoSuchKey","stonepass-audit-trail-logs"
"2026-07-26T03:08:48.734000+07:00","StopLogging","stonepass-warden","192.0.2.55",,
```

Detail JSON event `StopLogging` juga mengonfirmasi nama trail dan `accessKeyId` yang digunakan:

```json
{
  "eventName": "StopLogging",
  "userIdentity": {
    "userName": "stonepass-warden",
    "accessKeyId": "AKIA0QCBY6E7OWGLPAAP"
  },
  "sourceIPAddress": "192.0.2.55",
  "requestParameters": {
    "name": "stonepass-audit-trail"
  }
}
```

## flag

| No | Pertanyaan | Jawaban |
|---|---|---|
| 1 | Last action dari internal IP sebelum attacker session | **`ListAccessKeys`** (IP `10.30.41.118`, `03:08:17.750`) |
| 2 | First action dari attacker IP | **`GetTrailStatus`** (IP `192.0.2.55`, `03:08:38.781`) |
| 3 | Action yang ditolak sebelum trail dihentikan | **`DeleteTrail`** (`AccessDeniedException`, `03:08:41.313`) |
| 4 | S3 bucket yang di-enumerate | **`stonepass-audit-trail-logs`** |
| 5 | Nama trail yang dihentikan | **`stonepass-audit-trail`** |
| 6 | IAM username yang kredensialnya dipakai | **`stonepass-warden`** |
| 7 | IP asal trail dinonaktifkan | **`192.0.2.55`** |
| 8 | API action untuk disable trail | **`StopLogging`** |

## Kesimpulan

Investigasi ini menunjukkan pola khas **credential compromise** pada lingkungan AWS: kredensial IAM user legit (`stonepass-warden`) dicuri dan digunakan dari IP eksternal (`192.0.2.55`), bukan dari IP internal biasa (`10.30.41.118`) tempat user tersebut biasa beraktivitas.

Alur serangan yang terekam:
1. Penyerang mengecek status trail (`GetTrailStatus`)
2. Mencoba menghapus trail sepenuhnya (`DeleteTrail`) — **ditolak** karena permission IAM user tidak mengizinkan aksi ini
3. Sebagai gantinya, penyerang mengenumerasi bucket log (`ListBucket`, `ListObjectsV2`) untuk mencari file audit log
4. Gagal mengambil satu file spesifik (`GetObject` → `NoSuchKey`)
5. Akhirnya berhasil menghentikan logging dengan `StopLogging` — cukup untuk membungkam pencatatan aktivitas berikutnya tanpa perlu menghapus trail sepenuhnya

**Pelajaran forensik:** Ketika `DeleteTrail` gagal karena kurang privilege, penyerang beralih ke `StopLogging` yang punya efek serupa (trail berhenti mencatat) namun butuh permission lebih rendah. Ini menunjukkan pentingnya membatasi permission `cloudtrail:StopLogging` secara ketat, bukan hanya `cloudtrail:DeleteTrail`, serta mengaktifkan alerting real-time (misalnya via EventBridge + SNS) untuk event `StopLogging` agar tim keamanan bisa merespons sebelum jejak forensik lain ikut hilang.
