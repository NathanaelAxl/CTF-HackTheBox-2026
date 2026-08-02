# [Cloud] Empty Chairs — Writeup

## Deskripsi Soal

> Elowen Ashglass reaches one of Sythra's road watch posts after its morning signal fails. The fire pit is warm, spears lean by the door, and the scouts are gone. Red ash under the doorways and paired tracks on the road show that the Quiet Helpers took them alive. Without that post, Stormbound loses sight of the road ahead. Elowen must follow the tracks, find the missing scouts, and learn where the Helpers are taking them before frost covers the trail.
>
> The watch post's dispatch systems recorded activity before the scouts disappeared. You hold read-only investigator access to CloudTrail and the manifest bucket's S3 server access logs. Reconstruct the external session, then follow each service transition until the trail ends.

Singkatnya: kita diberi kredensial IAM **read-only** ke sebuah akun AWS (dijalankan di atas emulator ala LocalStack), dan diminta merekonstruksi *attack chain* seorang penyerang — mulai dari kredensial mana yang dipakai, role apa yang diambil-alih (privilege escalation via `AssumeRole`), sampai ke aksi terakhir yang menghapus jejak (`PurgeQueue`). Total ada **18 pertanyaan** yang harus dijawab berdasarkan bukti di **AWS CloudTrail** dan **S3 Server Access Logs**.

## Tools yang Dipakai

- **AWS CLI v2** (di Windows PowerShell)
- **PowerShell** (`ConvertFrom-Json`, `Where-Object`, `Group-Object`, dll) untuk parsing & filtering log — sebagai pengganti `jq` di Linux
- Browser (untuk membaca halaman briefing/instance challenge)

## Step-by-Step Penyelesaian

### 1. Setup Koneksi ke AWS API Endpoint

Challenge memberi dua port berbeda:
- Port **briefing/web** (halaman cerita & kredensial) — jangan dipakai untuk API
- Port **AWS API** — inilah yang harus dipakai untuk `AWS_ENDPOINT_URL`

Cara membedakannya: saat port yang salah diakses langsung di browser, responsnya berupa JSON mentah ala AWS (`AccessDeniedException` / `MissingAuthentication`) — ini menandakan itu adalah endpoint API AWS (LocalStack-style), bukan halaman web biasa.

```powershell
$env:AWS_ENDPOINT_URL = "http://<challenge-ip>:<aws-port>"
$env:AWS_DEFAULT_REGION = "us-east-1"
$env:AWS_ACCESS_KEY_ID = "<access-key-dari-briefing>"
$env:AWS_SECRET_ACCESS_KEY = "<secret-key-dari-briefing>"
Remove-Item Env:\AWS_SESSION_TOKEN -ErrorAction SilentlyContinue

aws sts get-caller-identity
```

Kalau berhasil, akan muncul identitas IAM user (`eastreach-investigator`) — ini konfirmasi koneksi sudah benar.

### 2. Coba Enumerasi Resource (Gagal — dan Ini Petunjuk Penting)

Percobaan awal untuk `list` semua resource (`s3 ls`, `cloudtrail describe-trails`, `sqs list-queues`, `iam list-roles`, dll) semuanya mengembalikan **AccessDenied**. Ini konsisten dengan cerita: user investigator sengaja dibatasi hanya untuk operasi tertentu, bukan untuk mendiskoveri seluruh akun.

Solusinya: gunakan API yang **tidak butuh permission list**, yaitu `cloudtrail lookup-events` — API ini query langsung ke Event History berdasarkan akun, bukan berdasarkan nama trail.

### 3. Menarik Seluruh CloudTrail Event History

```powershell
aws cloudtrail lookup-events --max-results 50 --no-cli-pager --output json > events_page1.json
```

Karena hasilnya dipaginasi (`NextToken`), semua halaman ditarik sekaligus dengan loop PowerShell:

```powershell
$allEvents = @()
$nextToken = $null

do {
    if ($nextToken) {
        $result = aws cloudtrail lookup-events --max-results 50 --next-token $nextToken --no-paginate --output json | ConvertFrom-Json
    } else {
        $result = aws cloudtrail lookup-events --max-results 50 --no-paginate --output json | ConvertFrom-Json
    }
    $allEvents += $result.Events
    $nextToken = $result.NextToken
    Write-Host "Total events so far: $($allEvents.Count)"
} while ($nextToken)

$allEvents | ConvertTo-Json -Depth 10 | Out-File -Encoding utf8 all_events.json
```

> **Catatan:** `--starting-token` dan `--no-cli-pager` tidak boleh dipakai bersamaan untuk pagination manual — yang benar adalah `--next-token` + `--no-paginate`.

Total didapat **3309 event**, rentang waktu 21–26 Juli 2026.

### 4. Parsing & Profiling Awal

```powershell
$events = Get-Content all_events.json | ConvertFrom-Json
$parsed = $events | ForEach-Object { $_.CloudTrailEvent | ConvertFrom-Json }

# Distribusi jenis event
$parsed | Group-Object eventName | Sort-Object Count -Descending | Select-Object Name, Count

# Distribusi source IP
$parsed | Group-Object sourceIPAddress | Sort-Object Count -Descending | Select-Object Name, Count
```

Dari sini terlihat pola penting:
- IP **10.23.x.x** dan **127.0.0.1** → traffic internal sistem watchpost (normal)
- IP **203.0.113.41, 198.51.100.22, 203.0.113.17** → banyak melakukan `AssumeRole` yang **selalu gagal (AccessDenied)** — kemungkinan percobaan brute-force/recon yang gagal
- IP **203.0.113.88** → satu-satunya IP yang memiliki event langka seperti `PurgeQueue`, `SendMessage`, `Decrypt`, `Publish`, `PutItem` — **ini kandidat kuat penyerang yang berhasil**

### 5. Membedah Full Attack Chain dari IP 203.0.113.88

```powershell
$parsed | Where-Object { $_.sourceIPAddress -eq "203.0.113.88" } |
  Sort-Object eventTime |
  Select-Object eventTime, eventName, errorCode,
    @{N='userType';E={$_.userIdentity.type}},
    @{N='userArn';E={$_.userIdentity.arn}},
    @{N='accessKey';E={$_.userIdentity.accessKeyId}},
    requestParameters, responseElements |
  Format-List
```

Output ini memberikan **seluruh timeline serangan** secara berurutan:

| Waktu | Event | Detail |
|---|---|---|
| 21:09:46 | `GetCallerIdentity` | Identitas awal: `eastreach-contractor` (access key `AKIAJP962S90PXOYMFG0`) |
| 21:09:46 | `ListSecrets` | Recon secret yang tersedia |
| 21:09:47 | `GetSecretValue` (x3, AccessDenied) | Gagal baca `dispatch-signing-draft`, `patrol-ledger-token`, `depot-routing/manifest` |
| 21:09:48 | `GetSecretValue` (sukses) | Berhasil baca `dispatch-signing` |
| 21:09:49 | `AssumeRole` (AccessDenied) | Gagal assume role `eastreach-archive-reader` |
| 21:09:50 | `AssumeRole` (sukses) | Berhasil assume role `eastreach-dispatch-role`, session name `dispatch-runner` |
| 21:09:51 | `GetCallerIdentity` | Identitas berubah jadi `AssumedRole` |
| 21:09:52 | `ReceiveMessage` | Baca pesan di queue `eastreach-scout-recall-pending` |
| 21:09:53 | `SendMessage` | Kirim pesan palsu berisi `order_id: ER-DIV-9941`, `route_override: quietmarch-depot-11/bypass-C` |
| 21:09:53 | `Decrypt` | Dekripsi pakai KMS key alias `eastreach/watchpost-routing-key` |
| 21:09:54 | `Publish` | Kirim notifikasi ke SNS topic `eastreach-diversion-alerts` |
| 21:09:54 | `PutItem` | Tulis data palsu ke DynamoDB table `eastreach-watchpost-ledger` |
| 21:09:55–56 | `ListObjectsV2` / `GetObject` | Akses bucket `eastreach-watchpost-manifests`, ambil `signed-routing-bundle.json` |
| 21:09:58 | `GetTrailStatus` | Cek status trail `eastreach-watchpost-trail` sebelum menghapus jejak |
| 21:09:58 | `PurgeQueue` | **Hapus semua pesan** di queue `eastreach-scout-recall-pending` — menghancurkan bukti |

Semua data ini langsung menjawab hampir separuh dari 18 pertanyaan.

### 6. Mencari ReceiveMessage Terakhir Sebelum IP Eksternal Masuk

Untuk pertanyaan soal source IP internal terakhir sebelum diambil alih attacker:

```powershell
$parsed | Where-Object { 
    $_.eventName -eq "ReceiveMessage" -and 
    $_.requestParameters.queueUrl -like "*eastreach-scout-recall-pending*" -and
    $_.eventTime -lt "2026-07-25T21:09:52.507Z"
} | Sort-Object eventTime | Select-Object -Last 1
```

Hasil: `10.23.87.41` — IP internal legit terakhir sebelum queue "diambil alih" attacker.

### 7. Mencari Nama Bucket Access Log (Tanpa Permission ListBuckets)

Karena `ListAllMyBuckets` diblokir, nama bucket log tidak bisa ditebak langsung. Triknya: cari di CloudTrail event mana saja yang menyebut `bucketName` — termasuk event `PutObject` dari `s3.amazonaws.com` (event internal S3 saat menulis access log):

```powershell
$parsed | Where-Object { $_.requestParameters.bucketName -ne $null } |
  Group-Object { $_.requestParameters.bucketName } | Select-Object Name, Count
```

Ditemukan bucket **`eastreach-manifest-access-logs`** — inilah bucket S3 Server Access Logs yang dimaksud di soal.

### 8. Mengambil & Membaca S3 Access Log

```powershell
aws s3 ls s3://eastreach-manifest-access-logs/ --recursive
```

Cari file log dengan timestamp yang cocok dengan waktu `GetObject` di CloudTrail (21:09:56), lalu download dan baca:

```powershell
aws s3 cp s3://eastreach-manifest-access-logs/eastreach-watchpost-manifests/2026-07-25-21-09-56-f8c24ed1f4974c2a .
Get-Content .\2026-07-25-21-09-56-f8c24ed1f4974c2a | Select-String "signed-routing-bundle"
```

Isi log (format S3 Server Access Log standar):

```
... 203.0.113.88 arn:aws:sts::719384620571:assumed-role/eastreach-dispatch-role/dispatch-runner
B810FF15A4004968 REST.GET.OBJECT signed-routing-bundle.json "GET /eastreach-watchpost-manifests/signed-routing-bundle.json HTTP/1.1" 200 ...
```

Field **Operation** pada log ini adalah `REST.GET.OBJECT` — inilah jawaban untuk pertanyaan soal operation field GetObject.

## Flag (18/18)

| # | Pertanyaan | Jawaban |
|---|---|---|
| 1 | Source IP pada `ReceiveMessage` internal terakhir sebelum diambil alih | `10.23.87.41` |
| 2 | External IP yang melakukan full attack chain (termasuk `PurgeQueue`) | `203.0.113.88` |
| 3 | Secret name yang AccessDenied pertama kali | `eastreach/dispatch-signing-draft` |
| 4 | Secret name yang berhasil dibaca sebelum role assumption | `eastreach/dispatch-signing` |
| 5 | IAM role yang gagal di-assume sebelum pivot berhasil | `eastreach-archive-reader` |
| 6 | IAM role ARN yang berhasil di-assume | `arn:aws:iam::719384620571:role/eastreach-dispatch-role` |
| 7 | `roleSessionName` pada AssumeRole yang berhasil | `dispatch-runner` |
| 8 | IAM identity type setelah pivot berhasil | `AssumedRole` |
| 9 | Nama SQS queue target fraudulent order | `eastreach-scout-recall-pending` |
| 10 | `order_id` yang diinjeksi | `ER-DIV-9941` |
| 11 | `route_override` destination | `quietmarch-depot-11/bypass-C` |
| 12 | KMS `keyId` yang dipakai untuk Decrypt | `arn:aws:kms:us-east-1:719384620571:alias/eastreach/watchpost-routing-key` |
| 13 | Nama SNS topic penerima diversion alert | `eastreach-diversion-alerts` |
| 14 | Nama DynamoDB table penerima forged ledger `PutItem` | `eastreach-watchpost-ledger` |
| 15 | S3 access log Operation field untuk `GetObject` | `REST.GET.OBJECT` |
| 16 | Nama CloudTrail trail yang diverifikasi sebelum purge | `eastreach-watchpost-trail` |
| 17 | API action yang menghancurkan bukti | `PurgeQueue` |
| 18 | IAM user pemilik long-lived key di awal chain | `eastreach-contractor` |

## Kesimpulan

Challenge ini mensimulasikan skenario **cloud incident response** yang realistis: seorang penyerang (`eastreach-contractor`) menggunakan kredensial IAM user *long-lived* untuk melakukan reconnaissance (`ListSecrets`, percobaan `AssumeRole` yang gagal berkali-kali dari beberapa IP berbeda), kemudian berhasil membaca secret yang tepat, melakukan **privilege escalation** lewat `AssumeRole` ke role dengan hak akses lebih tinggi (`eastreach-dispatch-role`), menyalahgunakan akses tersebut untuk mengirim pesan palsu (fraud order) ke sistem dispatch, mengubah data di DynamoDB, mengirim notifikasi palsu ke SNS, mengambil dokumen sensitif dari S3, lalu **menutup jejak** dengan menghapus isi queue lewat `PurgeQueue`.

Poin pembelajaran utama:

1. **`cloudtrail lookup-events` tidak butuh permission `List*`/`Describe*`** — sangat berguna saat akses investigator dibatasi ketat.
2. **Pola AccessDenied berulang pada `AssumeRole` dari banyak IP** adalah sinyal recon/brute-force sebelum serangan sukses — bandingkan dengan IP yang benar-benar berhasil melanjutkan ke aksi merusak.
3. **Nama resource (bucket, table, topic) yang tidak bisa di-*list* langsung** kadang bisa ditemukan lewat `requestParameters` pada event CloudTrail lain yang menyentuh resource tersebut secara tidak langsung.
4. **S3 Server Access Logs** memberi detail forensik tambahan (format request HTTP, response code, operation field) yang tidak selalu tercatat lengkap di CloudTrail.
5. Selalu bedakan **IP internal (dalam VPC/private range)** vs **IP publik** saat melakukan threat hunting — pola ini sering jadi kunci pertama menemukan "siapa yang bukan seharusnya di sana".

---
*Kategori: Cloud (AWS) — CloudTrail & S3 Access Log Forensics*
