# Praktikum Bab 7 - Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana

Praktikum dijalankan pada Windows dengan Docker Desktop dan WSL 2, dari direktori `~/docker-lab/bab-7`. Prometheus mengumpulkan metrics dari Node Exporter dan cAdvisor, lalu Grafana menampilkannya dalam dashboard.

```
Node Exporter --(metrics host)----+
                                  v
                              Prometheus ---> Grafana
                                  ^
cAdvisor ------(metrics container)+
```

## 1. Siapkan direktori

![Direktori praktikum Bab 7](images/bab-07/01-direktori.png)

```bash
mkdir -p ~/docker-lab/bab-7/{prometheus/rules,prometheus/tests,grafana/provisioning/datasources,grafana/provisioning/dashboards,grafana/dashboards,scripts,reports}
cd ~/docker-lab/bab-7
ls -la
```

`mkdir -p` membuat direktori yang diperlukan, termasuk folder induk yang belum ada. `cd` berpindah ke direktori praktikum dan `ls -la` memeriksa struktur yang sudah dibuat.

## 2. Buat template environment

![File .env.example](images/bab-07/02-env-example.png)

File `.env.example` berisi variabel akun Grafana dan integrasi PostgreSQL Bab 6 yang bersifat opsional:

```dotenv
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ganti-password-lab
GRAFANA_DB_USER=grafana_reader
GRAFANA_DB_PASSWORD=ganti-password-reader
POSTGRES_DB=labdb
```

Nilai di atas merupakan contoh. Ganti password dengan credential lab sendiri. Jangan gunakan password yang tampil di screenshot terdahulu karena sudah terpapar.

## 3. Buat konfigurasi lokal `.env`

![Salin template ke .env](images/bab-07/03-copy-env.png)

```bash
cp .env.example .env
chmod 600 .env
```

`cp` menyalin template menjadi konfigurasi lokal yang dibaca Docker Compose. `chmod 600` membatasi akses file kepada pemiliknya. Jangan unggah atau membagikan `.env`.

## 4. Atur `.gitignore`

![File .gitignore](images/bab-07/04-gitignore.png)

```gitignore
.env
reports/*
!reports/.gitkeep
*.log
```

Aturan ini mencegah file environment, laporan lokal, dan log masuk ke Git. Baris pengecualian mempertahankan `.gitkeep` jika digunakan untuk menandai folder laporan kosong.

## 5. Tulis `compose.yaml`

![compose.yaml bagian Prometheus dan Grafana](images/bab-07/05-compose-1.png)

![compose.yaml bagian exporter dan load generator](images/bab-07/06-compose-2.png)

![compose.yaml bagian network, volume, dan penutup](images/bab-07/07-compose-3.png)

Compose mendefinisikan lima service:

- `prometheus` melakukan scrape dan menyimpan time-series. Port host `9090` hanya terikat pada `127.0.0.1`.
- `grafana` menyediakan dashboard pada `127.0.0.1:3000` dan membaca credential dari `.env`.
- `node-exporter` mengekspos metrics host pada jaringan internal Compose, tanpa port host yang dipublikasikan.
- `cadvisor` mengekspos metrics container. Konfigurasi memakai `privileged: true`, sehingga hanya layak untuk lingkungan lab terisolasi.
- `load-generator` adalah service opsional di profile `load`, dibatasi hingga 0,50 CPU dan 64 MB memori.

Volume `prom-data` dan `grafana-data` mempertahankan data masing-masing service. Ketiga komponen monitoring berkomunikasi melalui network bridge `bab7-monitoring-net`.

## 6. Konfigurasikan Prometheus

![prometheus.yml](images/bab-07/08-prometheus-config.png)

File `prometheus/prometheus.yml` mengatur interval scrape dan evaluasi rule 15 detik, timeout scrape 10 detik, serta target Prometheus, Node Exporter, dan cAdvisor. Nama service digunakan sebagai alamat karena semuanya berada pada network Compose yang sama.

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    environment: lab
    chapter: "7"

rule_files:
  - /etc/prometheus/rules/recording-rules.yml
  - /etc/prometheus/rules/alert-rules.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: [prometheus:9090]
  - job_name: node-exporter
    static_configs:
      - targets: [node-exporter:9100]
  - job_name: cadvisor
    static_configs:
      - targets: [cadvisor:8080]
```

## 7. Buat recording rules

![Recording rules](images/bab-07/09-recording-rules.png)

`recording-rules.yml` menyimpan hasil query turunan untuk CPU host, penggunaan memori, ruang filesystem, dan penggunaan CPU serta memori container. Dashboard menggunakan nama rule tersebut agar ekspresi tidak perlu ditulis ulang pada setiap panel.

## 8. Buat alert rules

![Alert rules bagian pertama](images/bab-07/10-alert-rules-1.png)

![Alert rules bagian kedua](images/bab-07/11-alert-rules-2.png)

`alert-rules.yml` mendefinisikan kondisi `TargetDown`, CPU host tinggi, memori host tinggi, CPU container tinggi, dan kegagalan evaluasi rule Prometheus. Field `for` mengharuskan kondisi bertahan selama durasi tertentu sebelum alert aktif. Threshold pada praktikum adalah nilai eksperimen, bukan standar universal produksi.

## 9. Buat unit test alert

![Unit test TargetDown](images/bab-07/12-rules-test.png)

File `prometheus/tests/rules.test.yml` memberikan data simulasi ketika target Node Exporter berubah dari `up=1` menjadi `up=0`. `promtool test rules` memakai data ini untuk menguji apakah rule `TargetDown` menghasilkan alert seperti yang diharapkan. Tes ini memeriksa logika rule, bukan konektivitas target yang sedang berjalan.

## 10. Siapkan data source Grafana

![Konfigurasi data source Grafana](images/bab-07/13-datasources.png)

`datasources.yml` menambahkan Prometheus sebagai data source utama. Data source PostgreSQL Bab 6 disediakan sebagai integrasi opsional dan baru berfungsi setelah database serta akses jaringannya disiapkan. Gunakan role baca khusus, bukan akun administrator database.

## 11. Siapkan provisioning dashboard

![Konfigurasi dashboard provider](images/bab-07/14-dashboard-provider.png)

`provider.yml` mengarahkan Grafana untuk membaca file dashboard dari direktori yang ditentukan. Cara ini membuat dashboard dapat dimuat secara otomatis saat Grafana berjalan.

## 12. Buat dashboard JSON

![Dashboard JSON bagian 1 dari 6](images/bab-07/15-dashboard-json-1.png)

![Dashboard JSON bagian 2 dari 6](images/bab-07/16-dashboard-json-2.png)

![Dashboard JSON bagian 3 dari 6](images/bab-07/17-dashboard-json-3.png)

![Dashboard JSON bagian 4 dari 6](images/bab-07/18-dashboard-json-4.png)

![Dashboard JSON bagian 5 dari 6](images/bab-07/19-dashboard-json-5.png)

![Dashboard JSON bagian 6 dari 6](images/bab-07/20-dashboard-json-6.png)

File `bab-7-overview.json` membuat panel jumlah target aktif, CPU dan memori host, serta Top 5 container berdasarkan CPU dan memori. Panel memakai PromQL dan data source Prometheus. Panel container dapat kosong jika cAdvisor pada lingkungan Docker Desktop/WSL tidak menyediakan metric atau label yang diperlukan.

## 13. Buat script verifikasi stack

![Script verify-stack.sh bagian 1](images/bab-07/21-verify-stack-script-1.png)

![Script verify-stack.sh bagian 2](images/bab-07/22-verify-stack-script-2.png)

Script menunggu endpoint Prometheus dan Grafana siap, memeriksa jumlah target, menjalankan beberapa query PromQL, lalu memeriksa konfigurasi dan rules menggunakan `promtool`.

## 14. Validasi konfigurasi

![Pemeriksaan sintaks script](images/bab-07/23-verify-script-syntax.png)

```bash
chmod +x scripts/verify-stack.sh
bash -n scripts/verify-stack.sh
```

`chmod +x` memberi izin untuk menjalankan script secara langsung. `bash -n` memeriksa sintaks Bash tanpa menjalankan isi script.

![Validasi Compose dan JSON dashboard](images/bab-07/24-compose-json-validation.png)

```bash
docker compose config --quiet
python3 -m json.tool grafana/dashboards/bab-7-overview.json >/dev/null
```

Perintah tersebut memvalidasi konfigurasi Compose dan sintaks JSON dashboard. Keduanya selesai tanpa error pada screenshot.

![Menjalankan pemeriksaan konfigurasi Prometheus](images/bab-07/25-promtool-check-config-1.png)

```bash
docker compose run --rm --no-deps \
  --entrypoint /bin/promtool \
  prometheus \
  check config /etc/prometheus/prometheus.yml
```

Hasil menunjukkan konfigurasi Prometheus valid dan dua file rules ditemukan.

![Hasil pemeriksaan konfigurasi Prometheus dan rules](images/bab-07/26-promtool-check-config-2.png)

![Hasil unit test Prometheus rules](images/bab-07/27-promtool-test-rules.png)

```bash
docker compose run --rm --no-deps \
  --entrypoint /bin/promtool \
  prometheus \
  test rules /etc/prometheus/tests/rules.test.yml
```

Hasil `SUCCESS` menandakan unit test untuk rule `TargetDown` lulus. Validasi ini belum membuktikan seluruh service runtime sehat.

## 15. Jalankan stack

```bash
docker compose up -d
docker compose ps
```

![Status awal container Bab 7](images/bab-07/28-compose-ps-starting.png)

Screenshot menunjukkan Prometheus, Grafana, Node Exporter, dan cAdvisor mulai berjalan. Sebagian container masih berstatus `Starting` atau baru aktif beberapa detik. Tunggu hingga service siap sebelum memeriksa target.

Antarmuka lokal:

- Prometheus: `http://127.0.0.1:9090`
- Grafana: `http://127.0.0.1:3000`

## 16. Jalankan script verifikasi dan buka Grafana

```bash
./scripts/verify-stack.sh
```

![Hasil verify-stack.sh](images/bab-07/29-verify-stack-result.png)

Script melaporkan Prometheus dan Grafana siap, tetapi kemudian menghasilkan `[FAIL] Target up=1, down=0`. Artinya endpoint monitoring bisa diakses, tetapi script hanya menghitung satu target `up`, bukan tiga target yang diharapkan. Ini bukan hasil verifikasi stack penuh. Periksa halaman Targets dan status/log exporter, lalu jalankan verifikasi kembali.

![Halaman login Grafana](images/bab-07/30-grafana-login.png)

![Halaman awal Grafana](images/bab-07/31-grafana-home.png)

Halaman tersebut membuktikan antarmuka Grafana dapat dibuka, tetapi belum membuktikan dashboard berisi data.
## 17. Jalankan query PromQL

Query dapat dijalankan dari Prometheus Graph atau Grafana Explore:

```promql
up
```

```promql
instance:node_cpu_utilisation:ratio_rate5m * 100
```

```promql
instance:node_memory_utilisation:ratio * 100
```

```promql
topk(5, container:cpu_usage_cores:rate1m)
```

```promql
topk(5, container:memory_working_set_bytes:sum)
```

`up = 1` berarti Prometheus berhasil mengambil metric dari target pada saat scrape. Itu tidak membuktikan aplikasi memenuhi fungsi bisnis. Query sukses tetapi tidak menghasilkan series juga bukan berarti nilainya nol.

Pada screenshot query `up`, tiga series muncul untuk Node Exporter, Prometheus, dan cAdvisor dengan nilai `1`. Ini menunjukkan ketiga target berhasil discrape pada waktu query tersebut. Hasil itu berbeda dengan pemeriksaan script yang sebelumnya hanya menghitung satu target; kedua screenshot dapat berasal dari waktu yang berbeda atau hasil parsing script yang tidak menghitung semua target sebagaimana dimaksud. Gunakan halaman Targets/API serta query `up` pada waktu yang sama untuk memastikan status aktual.

![Query up pada Prometheus](images/bab-07/32-query-up-prometheus.png)

Query CPU host `instance:node_cpu_utilisation:ratio_rate5m * 100` menampilkan series Node Exporter dengan nilai sekitar `0.56` persen pada snapshot yang terlihat. Query memori `instance:node_memory_utilisation:ratio * 100` menampilkan sekitar `8.98` persen. Nilai tersebut adalah pengamatan pada waktu screenshot, bukan ringkasan performa sepanjang praktikum.

![Query CPU host](images/bab-07/33-query-host-cpu.png)

![Query CPU host di Grafana Explore](images/bab-07/34-grafana-host-cpu.png)

![Query memori host](images/bab-07/35-query-host-memory.png)

![Query memori host di Grafana Explore](images/bab-07/36-grafana-host-memory.png)

Pada screenshot query `topk` untuk CPU dan memori container, Prometheus menampilkan `Empty query result`. Artinya query tidak menemukan series yang cocok pada saat itu. Hal ini tidak boleh ditafsirkan sebagai penggunaan CPU atau memori container nol. cAdvisor di Docker Desktop/WSL 2 mungkin tidak mengekspos metric atau label `name` yang dibutuhkan recording rule. Karena query `up` untuk cAdvisor terlihat bernilai 1, endpoint cAdvisor dapat discrape, tetapi itu sendiri belum menjamin metric container yang diinginkan tersedia.

![Query CPU container kosong](images/bab-07/37-query-container-cpu-empty.png)

![Query memori container kosong](images/bab-07/38-query-container-memory-empty.png)

## 18. Uji beban container

Jalankan load generator:

```bash
docker compose --profile load up -d load-generator
```

![Menjalankan load generator](images/bab-07/39-load-generator-start.png)

Amati query berikut selama sedikitnya dua menit:

```promql
container:cpu_usage_cores:rate1m{name=~".*load-generator.*"}
```

Pada pemeriksaan pertama, query belum menemukan series untuk `load-generator`. Pemeriksaan dilakukan tidak lama setelah container dimulai. Query memakai `rate(...[1m])`, sehingga Prometheus memerlukan sampel yang cukup dalam jendela satu menit untuk menghitung laju CPU. Hasil kosong saat itu kemungkinan karena pemeriksaan terlalu cepat, bukan bukti bahwa cAdvisor tidak menyediakan metrics. Tunggu setidaknya satu menit setelah load generator berjalan, lalu jalankan query kembali. Hasil kosong tetap tidak boleh ditafsirkan sebagai CPU nol.

![Query CPU load generator tidak menghasilkan data](images/bab-07/40-load-generator-query-empty.png)

```promql
ALERTS{alertname="ContainerHighCPU"}
```

Setelah mengambil evidence, hentikan dan hapus container generator:

```bash
docker compose --profile load stop load-generator
docker compose --profile load rm -f load-generator
```

![Menghentikan dan menghapus load generator](images/bab-07/41-load-generator-cleanup.png)

Jika metrics container kosong di Docker Desktop/WSL 2, catat keterbatasan tersebut. Jangan menyimpulkan CPU container bernilai nol hanya dari query kosong.

## 19. Uji alert `TargetDown`

Hentikan Node Exporter:

```bash
docker compose stop node-exporter
```

![Menghentikan Node Exporter](images/bab-07/42-stop-node-exporter.png)

Di Prometheus Graph, jalankan:

```promql
up{job="node-exporter"}
```

Hasil query menunjukkan nilai `0`, yang berarti scrape Prometheus ke Node Exporter gagal setelah service dihentikan.

![Target Node Exporter berstatus down](images/bab-07/43-node-exporter-up-zero.png)

Setelah menunggu durasi `for` pada alert rule, periksa:

```promql
ALERTS{alertname="TargetDown",alertstate="firing"}
```

Setelah kondisi berlangsung cukup lama, hasil query menampilkan alert `TargetDown` dengan `alertstate="firing"`. Ini membuktikan rule mendeteksi target yang tidak tersedia. Pengujian ini tidak menguji pengiriman notifikasi ke email atau sistem pesan.

![Alert TargetDown berstatus firing](images/bab-07/44-targetdown-firing.png)

Rule pada konfigurasi ini memiliki `for: 1m`; evaluasi dilakukan setiap 15 detik. Tunggu cukup lama agar target ter-scrape sebagai down dan durasi rule terpenuhi. Pulihkan service:

```bash
docker compose start node-exporter
```

![Menyalakan kembali Node Exporter](images/bab-07/45-start-node-exporter.png)

Pastikan query `up{job="node-exporter"}` kembali bernilai `1`. Tes ini membuktikan Prometheus mengubah status alert, bukan bahwa notifikasi telah dikirim ke operator.

## 20. Integrasi opsional dengan PostgreSQL Bab 6

Integrasi ini bersifat opsional. Tujuannya menampilkan log PostgreSQL Bab 6 di Grafana Bab 7. Script `connect-bab6.sh` memeriksa environment serta database Bab 6, menghubungkan database ke network monitoring Bab 7, menyiapkan role baca khusus, memberi privilege yang diperlukan, lalu merestart Grafana.

### 20.1 Buat script `connect-bab6.sh`

![Script connect-bab6.sh bagian 1](images/bab-07/46-connect-bab6-script-1.png)

![Script connect-bab6.sh bagian 2](images/bab-07/47-connect-bab6-script-2.png)

![Script connect-bab6.sh bagian 3](images/bab-07/48-connect-bab6-script-3.png)

Script memakai `BAB6_DIR` jika lokasi Bab 6 berbeda dari default `../bab-6`.
### 20.2 Beri izin dan validasi sintaks

![Permission dan pemeriksaan sintaks connect-bab6.sh](images/bab-07/49-connect-bab6-script-validation.png)

```bash
chmod +x scripts/connect-bab6.sh
bash -n scripts/connect-bab6.sh
```

`chmod +x` memberi izin agar script dapat dijalankan langsung. `bash -n` memeriksa sintaks tanpa menjalankan perubahan pada database atau network.

### 20.3 Periksa prasyarat integrasi

![Pemeriksaan prasyarat integrasi Bab 6](images/bab-07/50-check-bab6-prerequisites.png)

Screenshot menunjukkan file `.env` Bab 7 dan Bab 6 ditemukan, serta container `postgres-db` Bab 6 berstatus `healthy`. Ini memenuhi prasyarat dasar untuk menjalankan script integrasi.

### 20.4 Jalankan integrasi

```bash
./scripts/connect-bab6.sh
```

![Menjalankan script integrasi Bab 6](images/bab-07/51-connect-bab6-run.png)

Output menunjukkan role baca dibuat atau diperbarui, privilege diberikan, dan Grafana direstart. Script menggunakan role `grafana_reader`, bukan akun admin PostgreSQL, agar akses Grafana dibatasi.

### 20.5 Jalankan query log di Grafana

Di Grafana Explore, pilih data source **PostgreSQL Logs Bab 6**, lalu jalankan query:

```sql
SELECT
    time AS "time",
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    data ->> 'log' AS log_message
FROM logs.fluentbit
WHERE $__timeFilter(time)
ORDER BY time DESC
LIMIT 200;
```

![Query PostgreSQL Bab 6 di Grafana Explore](images/bab-07/52-grafana-postgres-query.png)

![Hasil log Bab 6 di Grafana](images/bab-07/53-grafana-postgres-results.png)

Tabel hasil menampilkan waktu, tag, nama container, sumber log, dan pesan. Ini menjadi bukti bahwa Grafana dapat membaca record dari database Bab 6.

### 20.6 Putuskan koneksi network tambahan

Setelah pengujian, lepaskan container PostgreSQL Bab 6 dari network monitoring Bab 7 jika integrasi tidak lagi diperlukan:

```bash
  docker network disconnect bab7-monitoring-net \
  "$(docker compose -f ../bab-6/compose.yaml ps -q postgres-db)"
```

![Memutus network integrasi Bab 6](images/bab-07/54-disconnect-bab6-network.png)

## 21. Hasil dan batasan

- Prometheus mengumpulkan metrics dari dirinya sendiri, Node Exporter, dan cAdvisor melalui jaringan Compose.
- Grafana memakai Prometheus sebagai data source untuk dashboard resource.
- Alert `TargetDown` dapat diuji dengan menghentikan Node Exporter dan memeriksa perubahan status alert.
- cAdvisor menggunakan mode privileged pada konfigurasi lab, sehingga tidak sebaiknya dipakai tanpa pertimbangan keamanan.
- Pada Docker Desktop/WSL 2, metrics host dan container terutama menggambarkan lingkungan VM Linux Docker, bukan seluruh Windows.
- Status target `UP` hanya membuktikan scrape berhasil, bukan kesehatan fungsi bisnis.