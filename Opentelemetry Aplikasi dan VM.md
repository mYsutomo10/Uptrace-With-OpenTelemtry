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
### 2.3 Instal SDK OpenTelemetry
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
#Menerima data dari aplikasi (trace & metrics)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  #Menerima data dari VM (CPU, memori, disk, network)
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:

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

    metrics:
      receivers: [otlp, hostmetrics]
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
## 5. Buat Script Untuk Demo
```
import time
import requests
from prometheus_client import Counter, Gauge

from opentelemetry import trace, metrics
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Konfigurasi Endpoint Collector
OTLP_ENDPOINT = "http://localhost:4317"

resource = Resource.create({
    "service.name": "uptrace-demo-yoga",
    "seervice.version": "1.0.0"
})

def setup_tracing():
    """Menginisialisasi OpenTelemetry Tracer Provider dan Exporter."""

    # Inisialisasi Tracer Provider
    provider = TracerProvider(resource=resource)

    # Konfigurasi OTLP Exporter
    otlp_exporter = OTLPSpanExporter(endpoint=OTLP_ENDPOINT, insecure=True)

    # Batch Span Processor
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))

    # Tracer Provider global
    trace.set_tracer_provider(provider)

    # Instrumentasi requests
    RequestsInstrumentor().instrument()
    print("Tracing setup complete.")

REQUESTS_COUNTER = Counter(
    'app_requests_total',
    'Total hit ke aplikasi demo',
    ['endpoint', 'status']
)
ACTIVE_GAUGE = Gauge(
    'app_active_users',
    'Jumlah user yang aktif saat ini'
)

def setup_metrics():
    """Menginisialisasi OpenTelemetry Metrics Provider dan Exporter."""

    # Konfigurasi OTLP Exporter
    otlp_exporter = OTLPMetricExporter(endpoint=OTLP_ENDPOINT, insecure=True)

    # Metric Reader
    reader = PeriodicExportingMetricReader(otlp_exporter, export_interval_millis=5000)

    provider = MeterProvider(
        resource=resource,
        metric_readers=[reader]
    )

    # Meter Provider global
    metrics.set_meter_provider(provider)
    print("Metrics setup complete.")

tracer = trace.get_tracer(__name__)

def make_external_request():
    """Melakukan permintaan eksternal."""
    status = 0
    try:
        response = requests.get("https://www.google.com", timeout=5)
        status = response.status_code
        print(f"[Trace] Google request: success ({status})")
    except requests.exceptions.RequestException as e:
        status = 500
        print(f"[Trace] Google request: FAILED ({e})")
        trace.get_current_span().set_attribute("error", True)

    # Update metrik counter
    REQUESTS_COUNTER.labels('google', status).inc()

def run_simulation(iteration):
    """Menjalankan satu siklus simulasi traces dan metrics."""

    # Span utama
    with tracer.start_as_current_span(f"app-simulation-run-{iteration}") as span:
        print(f"--- [ITERATION {iteration}] Starting Simulation ---")

        # Aktivitas Internal
        with tracer.start_as_current_span("process-data"):
            time.sleep(0.2)
            span.set_attribute("data.size", 1024)

        # External Request
        make_external_request()

        # Update Metrik Gauge
        current_active = 10 + int(time.time() % 5)
        ACTIVE_GAUGE.set(current_active)
        print(f"[Metric] Set active users gauge to {current_active}")

        span.set_attribute("simulation.success", True)
        print(f"--- [ITERATION {iteration}] Telemetry Sent ---")


if __name__ == "__main__":
    print("Inisialisasi OpenTelemetry (Traces & Metrics)...")

    # Inisialisasi Tracing dan Metrics
    setup_tracing()
    setup_metrics()

    print("\nMemulai loop pengiriman telemetri. Tekan Ctrl+C untuk menghentikan.")

    iteration_count = 1
    while True:
        try:
            run_simulation(iteration_count)
            iteration_count += 1
            time.sleep(5)
        except KeyboardInterrupt:
            print("\nShutting down gracefully...")
            break
        except Exception as e:
            print(f"\nAn unexpected error occurred: {e}")
            time.sleep(10)

    # Waktu tunggu untuk BatchSpanProcessor dan MetricReader selesai mengirim data
    time.sleep(5)
    print("Application closed. Data flush complete.")
```
### 5.1 Jalankan Script
```
python3 app.py
```
