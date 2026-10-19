# Praktikum Bab 4 - Reverse Proxy dan TLS

Praktikum menggunakan Nginx sebagai reverse proxy untuk Apache dan Flask. Perintah dijalankan di Ubuntu WSL.

## 1. Siapkan direktori kerja

![Direktori Bab 4](images/bab-04/01-direktori.png)

```bash
mkdir -p ~/docker-lab/bab-4/{apache/sites,nginx/conf,certs,logs/nginx,app}
cd ~/docker-lab/bab-4
ls
```

Direktori disiapkan untuk konfigurasi Nginx, halaman Apache, aplikasi Flask, sertifikat, dan log.

## 2. Buat sertifikat TLS self-signed

![Pembuatan sertifikat TLS](images/bab-04/02-sertifikat.png)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/lab.key \
  -out certs/lab.crt \
  -subj "/CN=localhost/O=DevSecOps Docker Lab" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

Sertifikat berlaku 365 hari, menggunakan RSA 2048-bit, dan mencantumkan `localhost` serta `127.0.0.1`.

## 3. Batasi akses private key

![Permission sertifikat dan gitignore](images/bab-04/03-permission-key.png)

```bash
chmod 600 certs/lab.key
chmod 644 certs/lab.crt
printf 'certs/lab.key\n' >> .gitignore
```

Private key dibatasi agar hanya pemilik file yang dapat membacanya, dan dikecualikan dari Git. Jangan unggah `certs/lab.key` ke repository.

## 4. Buat konfigurasi Compose

![File compose.yaml](images/bab-04/04-compose-yaml.png)

```bash
nano compose.yaml
```

Isi file mengatur tiga service: `proxy`, `apache-web`, dan `flask-app`. Hanya Nginx yang memiliki port host, yaitu `8080` untuk HTTP dan `8443` untuk HTTPS. Ketiganya terhubung melalui network `web-net`.

## 5. Konfigurasikan Nginx

![Konfigurasi reverse proxy Nginx](images/bab-04/05-nginx-config.png)

Buat `nginx/conf/default.conf`. Konfigurasi mengalihkan HTTP ke HTTPS, meneruskan halaman utama ke Apache, dan meneruskan `/api/` ke Flask.

```nginx
upstream apache_backend {
    server apache-web:80;
}

upstream flask_backend {
    server flask-app:5000;
}

server {
    listen 80;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;

    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://flask_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 6. Buat halaman Apache

![Halaman HTML Apache](images/bab-04/06-apache-html.png)

Buat `apache/sites/index.html` sebagai halaman uji. Tautan `/api/` digunakan untuk mengakses backend Flask melalui reverse proxy.

## 7. Tentukan dependensi Flask

![Dependensi Flask](images/bab-04/07-requirements.png)

Buat `app/requirements.txt`:

```text
Flask==3.1.2
gunicorn==23.0.0
```

## 8. Buat aplikasi Flask

![Aplikasi Flask](images/bab-04/08-flask-app.png)

Buat `app/app.py`. Aplikasi menyediakan endpoint `/` untuk mengembalikan informasi API dan `/health` untuk healthcheck.

## 9. Buat Dockerfile Flask

![Dockerfile Flask](images/bab-04/09-dockerfile.png)

Buat `app/Dockerfile`. Image memasang dependensi lalu menjalankan Gunicorn sebagai user non-root `appuser`.

## 10. Validasi konfigurasi Compose

![Validasi docker compose config](images/bab-04/10-compose-config.png)

```bash
docker compose config
```

Periksa hasil gabungan konfigurasi Compose. Pastikan port, volume, healthcheck, dan network sudah sesuai.

![Port dan mount hasil interpolasi](images/bab-04/11-compose-config-details.png)

Output menunjukkan port Nginx terikat pada `127.0.0.1`; konfigurasi dan sertifikat dipasang read-only.

![Daftar service Compose](images/bab-04/12-compose-services.png)

```bash
docker compose config --services
```

Pastikan service berikut tampil:

```text
apache-web
flask-app
proxy
```

## 11. Build dan jalankan stack

![Build dan startup service](images/bab-04/13-compose-up.png)

```bash
docker compose up -d --build
```

Compose membangun image Flask, mengunduh image Nginx dan Apache, lalu membuat network dan container.

![Status container](images/bab-04/14-compose-ps.png)

```bash
docker compose ps
```

Pastikan semua service berjalan dan `flask-app` berstatus `healthy`. Published port hanya terlihat pada service `proxy`.

## 12. Uji HTTP redirect

![HTTP mengalihkan ke HTTPS](images/bab-04/15-http-redirect.png)

```bash
curl -I http://localhost:8080/
```

Respons `301 Moved Permanently` dan header `Location` menunjukkan HTTP diarahkan ke HTTPS pada port `8443`.

## 13. Uji halaman Apache melalui HTTPS

![Halaman Apache melalui HTTPS](images/bab-04/16-https-apache.png)

```bash
curl -k -i https://localhost:8443/
```

Respons `200 OK` dan halaman HTML membuktikan Nginx menerima HTTPS lalu meneruskan request ke Apache. Opsi `-k` hanya digunakan karena sertifikat lab bersifat self-signed.

## 14. Uji Flask API dan health endpoint

![Flask API dan healthcheck](images/bab-04/17-flask-api.png)

```bash
curl -k https://localhost:8443/api/
curl -k -i https://localhost:8443/api/health
```

API mengembalikan JSON dengan status `ok`; health endpoint mengembalikan HTTP `200` dan status `healthy`. Nilai `forwarded_proto` menunjukkan request diteruskan melalui HTTPS.

## 15. Periksa koneksi TLS

![Handshake TLS](images/bab-04/18-tls-handshake.png)

```bash
openssl s_client \
  -connect localhost:8443 \
  -servername localhost \
  -brief </dev/null
```

Handshake berhasil menggunakan TLS 1.3. Pesan `self-signed certificate` wajar untuk sertifikat lab dan berarti sertifikat belum dipercaya oleh CA sistem.

## 16. Uji akses internal dari proxy

![Proxy mengakses Apache](images/bab-04/19-proxy-apache.png)

```bash
docker compose exec proxy wget -qO- http://apache-web/
```

Proxy dapat menjangkau Apache menggunakan nama service Compose `apache-web`.

![Proxy mengakses Flask](images/bab-04/20-proxy-flask.png)

```bash
docker compose exec proxy wget -qO- http://flask-app:5000/health
```

Proxy dapat menjangkau Flask melalui network internal. Apache dan Flask tidak memublikasikan port ke host.

## 17. Periksa log service

![Log Compose](images/bab-04/21-compose-logs.png)

```bash
docker compose logs --tail 100
```

Log menunjukkan startup Nginx, Apache, dan Gunicorn. Apache juga mencatat peringatan bahwa `ServerName` belum ditetapkan; pada praktikum ini halaman tetap berhasil dilayani.

## 18. Periksa access log Nginx

![Access log Nginx](images/bab-04/22-nginx-access-log.png)

```bash
tail -n 20 logs/nginx/access.log
```

Access log mencatat request yang diuji, termasuk redirect HTTP, halaman Apache, dan endpoint Flask.

## 19. Hentikan stack

![Cleanup Compose](images/bab-04/23-compose-down.png)

```bash
docker compose down
```

Container dan network Compose dihapus. File konfigurasi, sertifikat, dan log lokal tetap berada di direktori praktikum.

## Hasil

- HTTP pada port `8080` mengalihkan request ke HTTPS.
- Nginx meneruskan halaman web ke Apache dan `/api/` ke Flask.
- Flask berstatus `healthy`; endpoint API merespons dengan sukses.
- TLS 1.3 berhasil digunakan dengan sertifikat self-signed.
- Hanya Nginx yang memiliki published port.
- Access log Nginx mencatat request praktikum.