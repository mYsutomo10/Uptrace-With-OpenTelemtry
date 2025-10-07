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
### 2.3 Instal OpenTelemetry
```
pip install opentelemetry-sdk opentelemetry-api \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-requests \
            opentelemetry-instrumentation-logging \
            psutil requests prometheus_client
```
## 3. Konfigurasi Env Variabel
### 3.1 Konfigurasi Env Variabel OTLP
```
export OTEL_EXPORTER_OTLP_ENDPOINT="<ENDPOINT>"
export UPTRACE_DSN="<UPTRACE_DSN>"
export OTEL_EXPORTER_OTLP_HEADERS="uptrace-dsn=<UPTRACE_DSN>"

export OTEL_EXPORTER_OTLP_COMPRESSION=gzip
export OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION=BASE2_EXPONENTIAL_BUCKET_HISTOGRAM
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=DELTA

export OTEL_BSP_EXPORT_TIMEOUT=10000
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=10000
export OTEL_BSP_MAX_QUEUE_SIZE=30000
export OTEL_BSP_MAX_CONCURRENT_EXPORTS=2
```
### 3.2 Buat File Konfigurasi Collector di /etc/otelcol-contrib/config.yaml
```
extensions:
  health_check:
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  opencensus:
    endpoint: 0.0.0.0:55678

  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:

  prometheus:
    config:
      scrape_configs:
      - job_name: 'otel-collector'
        scrape_interval: 10s
        static_configs:
        - targets: ['0.0.0.0:8888']

  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250
      thrift_binary:
        endpoint: 0.0.0.0:6832
      thrift_compact:
        endpoint: 0.0.0.0:6831
      thrift_http:
        endpoint: 0.0.0.0:14268

  zipkin:
    endpoint: 0.0.0.0:9411

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
      receivers: [otlp, opencensus, jaeger, zipkin]
      processors: [resource/namespace, batch]
      exporters: [otlp/uptrace]

    metrics:
      receivers: [otlp, opencensus, prometheus, hostmetrics]
      processors: [resource/namespace, batch]
      exporters: [otlp/uptrace]

    logs:
      receivers: [otlp]
      processors: [resource/namespace, batch]
      exporters: [otlp/uptrace]

  extensions: [health_check, pprof, zpages]
```
## 4. Jalankan OpenTelemetry Collector
```
sudo docker run -d \
  --name otel-collector-uptrace \
  --network host \
  --privileged \
  -v "$(pwd)/config.yaml":/etc/otelcol-contrib/config.yaml \
  -v /:/host:ro \
  otel/opentelemetry-collector-contrib:0.113.0
```
