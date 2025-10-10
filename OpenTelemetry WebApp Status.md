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

mkdir web-status && cd web-status
```
### 2.2 Aktifkan Virtual Env
```
python3 -m venv venv
source venv/bin/activate
```
### 2.3 Instal SDK OpenTelemetry
```
pip install flask opentelemetry-api opentelemetry-sdk \
    opentelemetry-instrumentation-flask \
    opentelemetry-exporter-otlp-proto-grpc
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
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 10000
    send_batch_max_size: 10000

  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  otlp:
    endpoint: <UPTRACE_OTLP_ENDPOINT>
    headers:
      uptrace-dsn: "<UPTRACE_DSN>"
    compression: gzip
    tls:
      insecure: false

  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
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
## 5. Setup Nginx
```
sudo nano /etc/nginx/sites-available/healthcheck
```
```
server {
    listen 80;
    server_name healthcheck.local;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        proxy_pass http://127.0.0.1:5000/health;
        proxy_set_header Host $host;
        access_log /var/log/nginx/health_access.log;
    }
}
```
### 5.1 Enable konfigurasi
```
sudo ln -s /etc/nginx/sites-available/healthcheck /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```
### 5.2 Setup host
```
sudo nano /etc/hosts
```
Tambahkan ```127.0.0.1 healthcheck.local```
## 6. Buat Web App
```
from flask import Flask, jsonify
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.flask import FlaskInstrumentor
import os
import time

# Setup OpenTelemetry
resource = Resource.create({
    "service.name": "nginx-yoga",
    "service.version": "1.0.0"
})

trace.set_tracer_provider(TracerProvider(resource=resource))
tracer_provider = trace.get_tracer_provider()

# OTLP Exporter ke Collector
otlp_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",
)

span_processor = BatchSpanProcessor(otlp_exporter)
tracer_provider.add_span_processor(span_processor)

# Flask App
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

@app.route('/health')
def health():
    """Internal health check endpoint"""
    return jsonify({
        "status": "healthy",
        "service": "healthcheck-service",
        "timestamp": time.time()
    }), 200

@app.route('/')
def index():
    """Main endpoint"""
    return jsonify({
        "message": "Service is running",
        "endpoints": ["/health", "/"]
    }), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
### 6.1 Health Checker
```
import requests
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# Setup OpenTelemetry
resource = Resource.create({
    "service.name": "health-checker-yoga",
    "check.type": "internal"
})

trace.set_tracer_provider(TracerProvider(resource=resource))
tracer_provider = trace.get_tracer_provider()

otlp_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",
    insecure=True
)

span_processor = BatchSpanProcessor(otlp_exporter)
tracer_provider.add_span_processor(span_processor)

tracer = trace.get_tracer(__name__)

def check_health():
    """Check internal health via Nginx"""
    with tracer.start_as_current_span("health.check") as span:
        try:
            response = requests.get("http://127.0.0.1:5000/health", timeout=5)

            span.set_attribute("status_code", response.status_code)
            span.set_attribute("health.status", "ok" if response.status_code == 200 else "down")
            span.set_attribute("check.type", "internal")

            print(f"[Internal] Status: {response.status_code} - {response.json()}")
            return response.status_code == 200
        except Exception as e:
            span.set_attribute("health.status", "down")
            span.set_attribute("error", str(e))
            print(f"[Internal] Error: {e}")
            return False

if __name__ == "__main__":
    print("Starting health checker...")
    while True:
        check_health()
        time.sleep(30)
```
### 6.3 HTTP Checker
```
import requests
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# Setup OpenTelemetry
resource = Resource.create({
    "service.name": "http-checker-yoga",
    "check.type": "external"
})

trace.set_tracer_provider(TracerProvider(resource=resource))
tracer_provider = trace.get_tracer_provider()

otlp_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",
    insecure=True
)

span_processor = BatchSpanProcessor(otlp_exporter)
tracer_provider.add_span_processor(span_processor)

tracer = trace.get_tracer(__name__)

def check_http():
    with tracer.start_as_current_span("http.check") as span:
        try:
            response = requests.get("http://webyoga.ina/", timeout=5)

            span.set_attribute("http.status_code", response.status_code)
            span.set_attribute("http.url", "http://webyoga.ina/")
            span.set_attribute("web.status", "online" if response.status_code == 200 else "offline")
            span.set_attribute("check.type", "external")
            span.set_attribute("response.size", len(response.content))

            try:
                data = response.json()
                print(f"[External] Status: {response.status_code} - {data}")
            except:
                print(f"[External] Status: {response.status_code} - {response.text[:100]}")

            return response.status_code == 200
        except requests.exceptions.RequestException as e:
            span.set_attribute("web.status", "offline")
            span.set_attribute("error", str(e))
            span.set_attribute("error.type", type(e).__name__)
            print(f"[External] Error: {e}")
            return False
        except Exception as e:
            span.set_attribute("web.status", "offline")
            span.set_attribute("error", str(e))
            print(f"[External] Unexpected error: {e}")
            return False

if __name__ == "__main__":
    print("Starting HTTP checker...")
    while True:
        check_http()
        time.sleep(30)
```
## 7. Jalankan Komponen
```
python health_checker.py & 
python http_checker.py & 
python app.py
```
