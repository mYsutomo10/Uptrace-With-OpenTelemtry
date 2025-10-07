## 1. Instalasi Docker di Virtual Machine (VM)
### 1.1 Perbarui Sistem dan Instal Prasyarat
```
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release -y
```
### 1.2 Tambahkan GPG Key dan Repositori Docker
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/debian](https://download.docker.com/linux/debian) $(lsb-release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### 1.3 Instal Docker Engine dan Verifikasi Instalasi
```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo docker run hello-world
```

## 2. Siapkan Virtual Env Python
### 2.1 Instal ```venv``` dan Buat Direktori
```
sudo apt update
sudo apt install python3-venv

mkdir otel-challenge && cd otel-challenge
```
### 2.2 Aktifkan Virtual Env
```
python3 -m venv venv
source venv/bin/activate
```
### 3. Instal Prasyarat
### 3.1 Instal OpenTelemetry
```
sudo apt update
pip install uptrace opentelemetry-instrumentation-psycopg2 psycopg2-binary
pip install opentelemetry-exporter-otlp
pip install opentelemetry-instrumentation-dbapi
```
### 3.2 Menambahkan Repositori PostgreSQL
```
sudo apt update
sudo apt install curl ca-certificates gnupg -y
sudo install -d /usr/share/postgresql-common/pgdg

curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
sudo gpg --dearmor -o /usr/share/postgresql-common/pgdg/apt.gpg

echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.gpg] \
https://apt.postgresql.org/pub/repos/apt bookworm-pgdg main" | \
sudo tee /etc/apt/sources.list.d/pgdg.list
```
### 3.3 Install PostgreSQL 16
```
sudo apt update
sudo apt install postgresql-16 -y
psql --version
```
## 4. Setup Database
### 4.1 Impor pagila
```
swget https://github.com/devrimgunduz/pagila/archive/refs/heads/master.zip
unzip master.zip
cd pagila-master/
```
### 4.2 Buat Database Baru
```
sudo -u postgres psql
CREATE DATABASE pagila;
\q
```
### 4.3 Impor Data Berkas Dump
```
sudo -u postgres psql -d pagila -f pagila-schema.sql
sudo -u postgres psql -d pagila -f pagila-data.sql
```
### 4.4 Buat User dan Beri Akses
```
CREATE USER yoga_intern WITH PASSWORD '<PASSWORD>';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO yoga_intern;
GRANT USAGE ON SCHEMA public TO yoga_intern;
```
## 5. Konfigurasi Env Variabel
```
export OTEL_EXPORTER_OTLP_ENDPOINT="<ENDPOINT>"
export UPTRACE_DSN="<UPTRACE_DSN>"
export OTEL_EXPORTER_OTLP_HEADERS="uptrace-dsn=<UPTRACE_DSN>"

export DB_PASSWORD="<PASSWORDS>"

export OTEL_EXPORTER_OTLP_COMPRESSION=gzip
export OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION=BASE2_EXPONENTIAL_BUCKET_HISTOGRAM
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=DELTA

export OTEL_BSP_EXPORT_TIMEOUT=10000
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=10000
export OTEL_BSP_MAX_QUEUE_SIZE=30000
export OTEL_BSP_MAX_CONCURRENT_EXPORTS=2
```

##6. Buat File Konfigurasi Collector di /etc/otelcol-contrib/config.yaml
```
extensions:
  health_check:
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

receivers:
#Menerima data dari aplikasi (trace & metrics)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
  resource/namespace:
    attributes:
      - key: service.namespace
        value: yoga
        action: insert

exporters:
  otlp/uptrace:
    endpoint: <UPTRACE_OTLP_ENDPOINT>
    headers:
      uptrace-dsn: '<UPTRACE_DSN>'
    sending_queue:
      enabled: true
      queue_size: 1000
    retry_on_failure:
      enabled: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource/namespace, batch]
      exporters: [otlp/uptrace]

  extensions: [health_check, pprof, zpages]
```
## 6.1. Jalankan OpenTelemetry Collector
```
sudo docker run -d \
  --name otel-collector-uptrace \
  --network host \
  --privileged \
  -v "$(pwd)/config.yaml":/etc/otelcol-contrib/config.yaml \
  -v /:/host:ro \
  otel/opentelemetry-collector-contrib:0.113.0
```
## 7. Buat Script Untuk Demo
```
import os
import uptrace
import psycopg2
import time
import random
from datetime import datetime
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry import trace
from psycopg2 import OperationalError

# Dapatkan Tracer global
tracer = trace.get_tracer("app.pagila")

def configure_telemetry():
    """Mengkonfigurasi OpenTelemetry dan Instrumentation."""
    print("Mengkonfigurasi OpenTelemetry...")

    # 1. Konfigurasi OpenTelemetry dengan Uptrace DSN dari variabel lingkungan
    uptrace.configure_opentelemetry(
        service_name="postgresql-yoga",
        service_version="1.0.0",
    )

    # 2. Instrumentasi Psycopg2
    Psycopg2Instrumentor().instrument(
        capture_parameters=True
    )
    print("OpenTelemetry siap.")

def run_db_operations():
    """Menjalankan operasi DB yang akan di-trace (dengan simulasi kegagalan koneksi)."""

    DB_HOST_REAL = "localhost"
    DB_HOST_FAIL = "nonexistent-host"

    # Tentukan apakah koneksi akan berhasil (50% kemungkinan gagal)
    is_success = random.random() < 0.5

    DB_CONFIG = {
        "dbname": "pagila",
        "user": "yoga_intern",
        "password": os.getenv("DB_PASSWORD"),
        "host": DB_HOST_FAIL if not is_success else DB_HOST_REAL
    }

    # Menentukan nama Span utama berdasarkan hasil yang diharapkan
    job_name = "pagila-job-success" if is_success else "pagila-job-fail"

    with tracer.start_as_current_span(job_name) as main_span:
        print(f"\n[JOB: {job_name}] Mencoba koneksi ke host: {DB_CONFIG['host']}...")
        conn = None

        try:
            # Span 'connect' akan dibuat secara otomatis
            conn = psycopg2.connect(**DB_CONFIG)
            cur = conn.cursor()
            print("Koneksi BERHASIL.")

            main_span.set_attribute("app.connection.status", "successful")

            # Kueri Cepat (Akan dicatat sebagai span 'SELECT')
            print("Menjalankan kueri cepat...")
            cur.execute("SELECT first_name, last_name FROM actor LIMIT 10")
            print(f"Hasil cepat: {len(cur.fetchall())} baris.")

            # Kueri Lambat (Real Slow Query)
						print("Menjalankan slow query...")
						start_time = time.time()

						cur.execute("""
						    SELECT COUNT(*) 
						    FROM payment 
						    WHERE amount > 3.99 AND staff_id IN (1, 2)
						    ORDER BY payment_date DESC;
						""")
						
						duration = time.time() - start_time
						main_span.set_attribute("db.operations.duration.seconds", duration)

            cur.close()
            conn.close()
            print("Trace sukses selesai.")

        except OperationalError as e:
                main_span.set_attribute("app.connection.status", "failed")
                main_span.record_exception(e)
                main_span.set_status(trace.Status(trace.StatusCode.ERROR, "Koneksi Database Gagal"))
            else:
                main_span.record_exception(e)
                main_span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            print(f"Terjadi Error: {e}")

        except Exception as e:
            main_span.record_exception(e)
            main_span.set_status(trace.Status(trace.StatusCode.ERROR, "Error Aplikasi/DB Lain"))
            print(f"Terjadi Error: {e}")

        finally:
            if conn:
                conn.close()

            print(f"Trace selesai. Cek di Uptrace.")


def main():
    configure_telemetry()

    # Lakukan loop setiap 5 detik
    DELAY_SECONDS = 5
    print(f"\nMemulai loop. Akan menjalankan operasi DB setiap {DELAY_SECONDS} detik.")

    try:
        while True:
            run_db_operations()
            time.sleep(DELAY_SECONDS)

    except KeyboardInterrupt:
        # Menangani penghentian skrip oleh pengguna
        print("\n\nLoop dihentikan oleh pengguna.")

    finally:
        # Pastikan traces terkirim sebelum keluar
        print("Mengirim traces dan mematikan OpenTelemetry...")
        uptrace.shutdown()
        print("Program Selesai.")

if __name__ == "__main__":
    main()
```
### 7.1 Jalankan Script
```
python3 db.py
```
