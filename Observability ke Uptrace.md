## Dokumen ini berisi panduan langkah demi langkah untuk menginstal Docker, menyiapkan lingkungan Python, dan mengimplementasikan instrumentasi OpenTelemetry pada sebuah aplikasi demo. Data telemetri (Traces, Metrics, Logs) akan dikumpulkan oleh OpenTelemetry Collector dan diekspor Uptrace.

#1. Instalasi Docker di Virtual Machine (VM)
## Perbarui Sistem dan Instal Prasyarat
'''
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release -y
'''
