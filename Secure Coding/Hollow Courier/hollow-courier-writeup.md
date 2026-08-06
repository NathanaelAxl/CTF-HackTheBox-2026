# [Secure Coding] Hollow Courier

> **Kategori:** Secure Coding
> **Platform:** Hack The Box
> **Flag:** `HTB{thr33_kn0ck5_0n_s34led_g4t3s_7e4983d57bdd1dfe632b10b9c47b856e}`

---

## Deskripsi Soal

Repositori berisi source code aplikasi bernama **The Hollow Courier**, sebuah aplikasi Flask ("Salt Gate") yang berdiri di belakang reverse proxy Caddy ("perimeter"). Cerita soal ini adalah kiasan dari sebuah kerentanan teknis nyata:

> Lysa Harrowmere mencegat paket rute Vaultrune yang memerintahkan Ashguard menjauh dari jalur suplai Crownspire. Garran Voss harus menemukan kurir yang telah disusupi, mencari tahu bagaimana ia lolos dari pemeriksaan gerbang, dan menghentikannya tanpa memblokir kurir asli dari halaman dalam.

**Objective (tujuan teknis):**
> Restore trustworthy forwarded-client provenance across the Caddy and Flask boundary while preserving the legitimate internal route.

Singkatnya: ada endpoint internal (`/gate/decree`) yang seharusnya **hanya** bisa diakses oleh service internal ("watch relay"), tapi ternyata bisa diakses dari luar dengan cara memalsukan identitas asal request (IP spoofing). Tugasnya adalah menemukan bug ini, memperbaikinya, tanpa merusak akses milik service internal yang sah.

**Kriteria scoring:**
- **Hard score**: 60/60 (build sukses, fungsionalitas normal tetap jalan, security check lolos)
- **Soft score**: minimal 28/40 (code quality, security reasoning, patch correctness)

---

## Tools yang Dipakai

| Tool | Kegunaan |
|---|---|
| **Git** | Clone repository, buat branch, commit & push patch |
| **Visual Studio Code** | Membaca dan mengedit source code |
| **Docker Desktop (Windows)** | Build & jalankan aplikasi secara lokal untuk testing |
| **PowerShell** | Menjalankan semua command (git, docker, curl) |
| **curl** | Mengirim HTTP request untuk menguji endpoint (dari host maupun dari dalam container) |
| **Python 3 (di dalam container)** | Alternatif pengganti curl untuk testing dari dalam container, karena image tidak menyertakan curl |

---

## Step-by-Step Penyelesaian

### 1. Clone repository

Info akses (username, password, clone URL) didapat dari halaman repository di platform SecureCoding.

```bash
git clone http://htb_developer:HTBDeveloperPassword@[IP]:[PORT]/git/core_application.git
cd core_application
```

### 2. Buat branch kerja

README menyebutkan bahwa push langsung ke branch `main` diblokir, dan semua pekerjaan harus melalui branch `developer`.

```bash
git checkout -b developer
```

### 3. Baca konfigurasi reverse proxy (Caddy)

Karena soal menyebutkan "boundary Caddy dan Flask", langkah pertama adalah memeriksa konfigurasi proxy-nya.

```bash
cat checkpoint/conf/Caddyfile
```

Isinya:

```caddyfile
# Application perimeter.
:8000 {
        reverse_proxy 127.0.0.1:5000 {
                # Accept forwarding metadata from the deployment's private proxy tier.
                trusted_proxies private_ranges
                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-Proto {scheme}
        }
}
```

Poin penting: ada directive `trusted_proxies private_ranges`. Ini artinya Caddy akan **mempercayai** header forwarding (seperti `X-Forwarded-For`) yang dikirim oleh siapa pun, **selama koneksi datang dari IP dalam rentang private** (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).

### 4. Telusuri struktur aplikasi Flask

```bash
ls -R checkpoint/app
```

Ditemukan file `gate.py` — namanya relevan dengan cerita soal ("Salt Gate"), jadi kemungkinan besar berisi logic pengecekan asal request.

```bash
cat checkpoint/app/gate.py
```

Isinya:

```python
INTERNAL_NETWORKS = (
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.2/32"),  # alias loopback "watch relay"
)

def is_internal_request() -> bool:
    try:
        origin = ipaddress.ip_address(request.remote_addr or "")
    except ValueError:
        return False
    return any(origin in network for network in INTERNAL_NETWORKS)
```

Fungsi ini memutuskan apakah sebuah request berasal dari jaringan internal, berdasarkan `request.remote_addr`. Nilai `127.0.0.2` sengaja di-whitelist sebagai alias khusus untuk "watch relay" (service internal yang sah).

### 5. Cek middleware yang mengisi `request.remote_addr`

```bash
cat checkpoint/app/__init__.py
```

Ditemukan:

```python
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=2, x_proto=1, x_host=1)
```

`ProxyFix` adalah middleware Werkzeug yang membaca header `X-Forwarded-For` untuk mengisi `request.remote_addr` dengan benar ketika aplikasi berjalan di belakang reverse proxy. Parameter `x_for=2` artinya Flask **mengasumsikan ada 2 lapis proxy** di depannya, dan akan mengambil IP dari 2 posisi terakhir pada header tersebut.

### 6. Cek endpoint yang dilindungi

```bash
cat checkpoint/app/routes.py
```

Ditemukan endpoint sensitif:

```python
@gate_bp.route("/gate/decree", methods=["POST"])
def decree():
    if not gate.require_internal():
        abort(403)
    ...
```

Endpoint ini menyegel sebuah "crown decree" (dalam cerita: order yang punya kekuatan hukum) dan **hanya boleh** diakses dari jaringan internal.

### 7. Cek test suite yang sudah ada

```bash
cat checkpoint/tests/test_gate.py
```

Ada catatan penting di docstring file test:

> *"The inner desk's provenance rules belong to the perimeter and are exercised in deployment, not in this in-process test harness."*

Artinya tidak ada automated test yang menguji logic IP internal ini secara langsung — jadi perbaikan tidak akan bertabrakan dengan test yang sudah ada, tapi juga berarti perlu divalidasi secara manual.

### 8. Menyusun teori kerentanan

Dari semua temuan di atas, tersusun alur serangan sebagai berikut:

1. Attacker (dari luar) mengirim request ke Caddy dengan header palsu: `X-Forwarded-For: 127.0.0.2`.
2. Karena koneksi attacker ke Caddy (lewat Docker/NAT/infrastruktur apa pun) berasal dari IP yang termasuk private range, directive `trusted_proxies private_ranges` membuat Caddy **mempercayai** header itu apa adanya, lalu meneruskannya ke Flask.
3. Flask dengan `ProxyFix(x_for=2)` mengambil IP dari posisi ke-2 dari kanan pada `X-Forwarded-For` — namun karena hanya ada 1 proxy sungguhan (Caddy) di antara attacker dan Flask, hasil hitungan ini justru mengembalikan nilai yang **dikontrol attacker**, yaitu `127.0.0.2`.
4. `127.0.0.2` cocok dengan whitelist `INTERNAL_NETWORKS`, sehingga attacker dianggap sebagai "watch relay" yang sah dan bisa memanggil `/gate/decree`.

### 9. Uji coba (proof of concept) sebelum patch

```powershell
curl.exe -X POST http://154.57.164.82:32748/app/gate/decree `
  -H "X-Forwarded-For: 127.0.0.2" `
  -d "order=test&serial=fake"
```

**Hasil:**
```json
{"authority":"CROWN","decree":1,"order":"test","sealed":true}
```

Terbukti — hanya dengan mengirim satu header palsu, endpoint internal berhasil diakses dari luar.

### 10. Patch #1 — Perbaiki jumlah hop di `ProxyFix`

Karena topologi sebenarnya hanya punya **satu** proxy (Caddy) di depan Flask, nilai `x_for` harus disesuaikan menjadi `1`, agar Flask hanya mempercayai IP yang benar-benar ditambahkan oleh Caddy, bukan yang disisipkan client.

`checkpoint/app/__init__.py`:
```python
# Sebelum
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=2, x_proto=1, x_host=1)

# Sesudah
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)
```

### 11. Setup Docker untuk testing lokal

Karena docker belum tersedia, terlebih dahulu meng-install dan menjalankan **Docker Desktop**, memastikan status **"Engine running"**, baru kemudian menjalankan:

```powershell
docker build -t hollow-courier checkpoint
docker run --rm -p 8000:8000 hollow-courier
```

### 12. Re-test setelah patch #1 — Ternyata masih bisa ditembus

```powershell
curl.exe -X POST http://127.0.0.1:8000/app/gate/decree `
  -H "X-Forwarded-For: 127.0.0.2" `
  -d "order=test&serial=fake"
```

**Hasil:** tetap `{"sealed": true, ...}`.

Bahkan tanpa header spoofing sama sekali pun tetap tembus:
```powershell
curl.exe -X POST http://127.0.0.1:8000/app/gate/decree -d "order=test&serial=nospoof"
```
**Hasil:** tetap `{"sealed": true, ...}`.

**Analisis:** ini terjadi karena Docker Desktop di Windows menggunakan NAT untuk koneksi dari host ke container. Akibatnya, "IP pengirim langsung" yang dilihat oleh Caddy bukan IP asli, melainkan alamat gateway NAT Docker — yang kebetulan juga termasuk private range. Karena directive `trusted_proxies private_ranges` masih ada di Caddyfile, Caddy tetap mempercayai header dari client apa adanya.

### 13. Patch #2 — Hapus `trusted_proxies private_ranges`

Directive ini terlalu longgar: ia mempercayai *siapa pun* yang koneksinya berasal dari IP private, padahal tidak ada "private proxy tier" sungguhan di topologi aplikasi ini.

`checkpoint/conf/Caddyfile`:
```caddyfile
# Sebelum
:8000 {
        reverse_proxy 127.0.0.1:5000 {
                trusted_proxies private_ranges
                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-Proto {scheme}
        }
}

# Sesudah
:8000 {
        reverse_proxy 127.0.0.1:5000 {
                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-Proto {scheme}
        }
}
```

Dengan ini, Caddy **selalu** menambahkan (append) IP pengirim asli ke `X-Forwarded-For`, tidak peduli apakah IP tersebut private atau public — sehingga tidak lagi mempercayai isi header yang dikirim client secara mentah.

### 14. Validasi patch dari dalam container (menghindari noise NAT)

Karena testing dari host Windows terpengaruh NAT Docker, pengujian dilakukan dari **dalam container itu sendiri**, agar koneksi benar-benar datang dari `127.0.0.1` (loopback container) — bukan `127.0.0.2` (alias khusus watch relay) dan bukan pula IP NAT.

```powershell
docker build -t hollow-courier checkpoint
docker run --rm -p 8000:8000 hollow-courier
```

Di terminal lain:
```powershell
docker ps
docker exec -it <container_name> sh
```

Karena image tidak menyertakan `curl`, digunakan Python untuk mengirim request dari dalam container:

```sh
python3 -c "
import urllib.request
req = urllib.request.Request(
    'http://localhost:8000/app/gate/decree',
    data=b'order=test&serial=fake',
    headers={'X-Forwarded-For': '127.0.0.2'},
    method='POST'
)
try:
    r = urllib.request.urlopen(req)
    print(r.status, r.read())
except urllib.error.HTTPError as e:
    print(e.code, e.read())
"
```

**Hasil:**
```
403 ... "No business here" ...
```

Kedua patch berhasil: percobaan spoofing sekarang ditolak dengan status **403**.

### 15. Commit dan push patch

```powershell
git add checkpoint/app/__init__.py checkpoint/conf/Caddyfile
git commit -m "fix: correct client IP provenance across Caddy and Flask boundary

- Caddyfile: remove 'trusted_proxies private_ranges', which caused Caddy
  to blindly trust client-supplied X-Forwarded-For whenever the direct
  connection came from any private-range address (e.g. Docker NAT),
  letting an external caller spoof the internal watch-relay IP.
- __init__.py: fix ProxyFix x_for from 2 to 1, matching the actual
  single-hop topology (Caddy is the only proxy in front of Flask).
  With x_for=2 Flask trusted a header value the client itself supplied.

Together these ensure request.remote_addr in gate.py reflects the real
caller, so /gate/decree can no longer be reached by spoofing
X-Forwarded-For: 127.0.0.2 from outside. The legitimate internal watch
relay (which connects directly to port 5000, bypassing Caddy, as
127.0.0.2) is unaffected."
```

Saat push pertama kali sempat ditolak karena branch remote sudah punya commit lain:

```powershell
git push -u origin developer
# ! [rejected] developer -> developer (fetch first)
```

Solusinya, sinkronkan dulu dengan `rebase` sebelum push ulang:

```powershell
git pull --rebase origin developer
git push -u origin developer
```

Push berhasil, dan sistem otomatis mendaftarkan PR baru:
```
remote: [*] Push on developer branch detected, registering PR event...
remote: [post-receive] PR event registered: f716021e..808d0eb6 on developer
```

### 16. Cek status Pull Request

Dibuka melalui browser:
```
http://[IP]:[PORT]/pulls
```

PR terbaru berstatus **`accepted`**.

### 17. Ambil flag

```powershell
curl.exe -s http://[IP]:[PORT]/flag
```

**Hasil:**
```json
{
  "STATUS": "SOLVED",
  "FLAG": "HTB{thr33_kn0ck5_0n_s34led_g4t3s_7e4983d57bdd1dfe632b10b9c47b856e}",
  "MESSAGE": "Congratulations on getting the flag!",
  "HARD_SCORE": 60,
  "SOFT_SCORE": {
    "code_quality": 12,
    "security_reasoning": 14,
    "patch_correctness": 7
  }
}
```

**Skor akhir:**
| Objective | Nilai | Maksimal | Required |
|---|---|---|---|
| Hard score | 60 | 60 | 60 |
| Code quality | 12 | 15 | 12 |
| Security reasoning | 14 | 15 | 12 |
| Patch correctness | 7 | 10 | 6 |
| **Total soft score** | **33** | 40 | 28 |

---

## Kesimpulan

Kerentanan pada challenge ini adalah **spoofing IP klien melalui header `X-Forwarded-For`**, akibat dua kesalahan konfigurasi yang saling memperparah satu sama lain di batas antara reverse proxy (Caddy) dan aplikasi backend (Flask):

1. **Caddy** dikonfigurasi dengan `trusted_proxies private_ranges`, yang membuatnya mempercayai header forwarding dari client mana pun selama koneksi datang dari rentang IP private — padahal tidak ada proxy tepercaya sungguhan di jaringan private tersebut.
2. **Flask** dikonfigurasi dengan `ProxyFix(x_for=2)`, mengasumsikan ada dua lapis proxy di depannya, padahal pada kenyataannya hanya ada satu (Caddy). Akibatnya, Flask salah mengambil posisi IP dari header yang sebagian isinya masih dikendalikan oleh attacker.

Kombinasi keduanya membuat siapa pun dari luar bisa **menyamar sebagai service internal** ("watch relay") hanya dengan mengirim satu header HTTP palsu, lalu mengakses endpoint privileged `/gate/decree`.

**Pelajaran penting** dari challenge ini:
- **Jumlah hop proxy pada `ProxyFix` (atau middleware sejenis) harus selalu sama persis dengan topologi jaringan yang sebenarnya** — jangan menebak atau melebih-lebihkan, karena setiap hop tambahan yang tidak nyata membuka celah bagi attacker untuk menyisipkan nilai palsu.
- **Kepercayaan terhadap header forwarding tidak boleh didasarkan hanya pada rentang IP (seperti "private range")**, karena banyak infrastruktur (Docker NAT, load balancer, Kubernetes) secara alami membuat IP pengirim langsung terlihat seperti IP private — bukan karena benar-benar merupakan proxy tepercaya.
- **Reverse proxy harus menjadi satu-satunya sumber kebenaran** untuk identitas IP client — ia harus selalu menimpa/menambahkan header tersebut berdasarkan koneksi TCP yang sebenarnya, bukan sekadar meneruskan apa yang dikirim client.
- **Testing lokal via Docker Desktop di Windows bisa menyesatkan** untuk kasus seperti ini karena lapisan NAT tambahan; melakukan pengujian dari dalam container sendiri memberi hasil yang lebih representatif terhadap kondisi produksi.

