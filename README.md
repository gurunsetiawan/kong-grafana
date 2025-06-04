# Kong API Monitoring di Kubernetes

Repositori ini berisi konfigurasi Kubernetes (menggunakan Kustomize) untuk mendeploy Kong API Gateway, sebuah contoh API, dan stack pemantauan berbasis Prometheus dan Grafana untuk memantau lalu lintas API Kong.

## Struktur Direktori
```
.
├── kubernetes/               # Berisi semua konfigurasi Kubernetes
│   ├── base/                 # Konfigurasi dasar yang dapat digunakan kembali
│   │   ├── kong/             # Deployment, Service, dan Secret Kong
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── database-secret.yaml
│   │   ├── hello-world-api/  # Deployment dan Service contoh API
│   │   └── kustomization.yaml# Mengatur sumber daya dasar
│   ├── overlays/             # Konfigurasi spesifik lingkungan
│   │   ├── dev/              # Overlays untuk lingkungan pengembangan
│   │   │   ├── kustomization.yaml
│   │   │   └── kong-route.yaml # Route Kong khusus dev
│   │   ├── monitoring/       # Overlays untuk komponen pemantauan
│   │   │   ├── kustomization.yaml
│   │   │   ├── prometheus/   # Deployment, Service, dan ConfigMap Prometheus
│   │   │   └── grafana/      # Deployment, Service, ConfigMap, dan Datasource Grafana
│   └── README.md             # Penjelasan tentang struktur Kubernetes
├── .gitignore                # File yang akan diabaikan oleh Git
└── README.md                 # README utama repositori ini
```

## Prasyarat

Pastikan Anda telah menginstal dan mengkonfigurasi yang berikut:

* **kubectl**: Command-line tool untuk berinteraksi dengan cluster Kubernetes Anda.
* **Kustomize**: Sudah terintegrasi dengan `kubectl` versi 1.14 ke atas, cukup gunakan `kubectl apply -k`.
* **Cluster Kubernetes**: Minikube, Docker Desktop (dengan Kubernetes diaktifkan), Kind, atau cluster cloud (GKE, AKS, EKS, dll.).
* **PostgreSQL**: Database yang diperlukan untuk Kong. Anda dapat menggunakan:
  * Helm chart: `helm install kong-database bitnami/postgresql`
  * Atau database PostgreSQL yang sudah ada

## Langkah-langkah Deployment

### 1. Siapkan Database PostgreSQL

Jika menggunakan Helm:
```bash
# Tambahkan repo Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Install PostgreSQL
helm install kong-database bitnami/postgresql \
  --set postgresqlUsername=kong \
  --set postgresqlPassword=kongpassword \
  --set postgresqlDatabase=kong
```

### 2. Terapkan Komponen Dasar

Ini akan mendeploy Kong API Gateway dan aplikasi `hello-world-api`.

```bash
kubectl apply -k kubernetes/base/
```

Verifikasi Pod dan Service berjalan:
```bash
kubectl get pods -l app=kong
kubectl get svc kong-proxy kong-admin
kubectl get pods -l app=hello-world
kubectl get svc hello-world-service
```

### 3. Terapkan Kong Route untuk Lingkungan Dev

Ini akan mengkonfigurasi Kong untuk mengarahkan lalu lintas ke hello-world-api.

```bash
kubectl apply -k kubernetes/overlays/dev/
```

Verifikasi KongRoute telah dibuat:
```bash
kubectl get kongroute
```

### 4. Terapkan Komponen Pemantauan (Prometheus & Grafana)

Ini akan mendeploy Prometheus untuk mengumpulkan metrik dari Kong, dan Grafana untuk memvisualisasikannya.

```bash
kubectl apply -k kubernetes/overlays/monitoring/
```

Verifikasi Pod dan Service monitoring berjalan:
```bash
kubectl get pods -l app=prometheus
kubectl get svc prometheus
kubectl get pods -l app=grafana
kubectl get svc grafana
```

## Fitur Utama

### 1. API Traffic Monitoring
- Metrik dasar seperti request rate, latency, dan error rate melalui Prometheus
- Visualisasi metrik melalui Grafana dashboard

### 2. Audit Trail & Logging
- Request/Response logging melalui plugin `http-log`
- Informasi audit tambahan melalui plugin `request-transformer`:
  - Request ID
  - User ID (jika menggunakan authentication)
  - Service Name
  - Route Name
- File logging untuk menyimpan log ke file lokal
- Log dapat diakses melalui volume mount di pod Kong

### 3. Security Features
- Security context untuk Kong container
- Health checks (liveness dan readiness probes)
- Secret untuk kredensial database
- Akses terbatas ke Admin API

## Pengujian API dan Pemantauan

### 1. Uji API melalui Kong

Dapatkan IP eksternal dari Service kong-proxy:
```bash
kubectl get svc kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Atau untuk Minikube/Docker Desktop:
# minikube service kong-proxy --url
```

Gunakan IP tersebut untuk memanggil hello-world-api melalui Kong:
```bash
curl http://<KONG_PROXY_EXTERNAL_IP>/hello/get
```

### 2. Akses Grafana

Untuk Kind cluster, gunakan port-forward:
```bash
kubectl port-forward svc/grafana 3000:80
```

Buka browser Anda dan navigasi ke http://localhost:3000

```
Username default: admin
Password default: admin (Anda akan diminta untuk mengubahnya pada login pertama).
```

### 3. Akses Prometheus

Untuk Kind cluster, gunakan port-forward:
```bash
kubectl port-forward svc/prometheus 9090:80
```

Buka browser Anda dan navigasi ke http://localhost:9090

## Mengakses Log dan Audit Trail

### 1. Melihat Log Langsung dari Pod Kong

```bash
# Mendapatkan nama pod Kong
kubectl get pods -l app=kong

# Melihat log file
kubectl exec -it <kong-pod-name> -- cat /usr/local/kong/logs/access.log

# Melihat log secara real-time (seperti tail -f)
kubectl exec -it <kong-pod-name> -- tail -f /usr/local/kong/logs/access.log
```

### 2. Menggunakan kubectl logs

```bash
# Melihat log Kong secara real-time
kubectl logs -f -l app=kong

# Melihat log dengan filter
kubectl logs -l app=kong | grep "error"
```

### 3. Mengakses Log dari Volume

Log disimpan dalam volume `kong-logs`. Anda bisa mengaksesnya dengan:

```bash
# Mendapatkan nama pod Kong
kubectl get pods -l app=kong

# Masuk ke pod Kong
kubectl exec -it <kong-pod-name> -- /bin/sh

# Di dalam pod, Anda bisa melihat log
cd /usr/local/kong/logs
cat access.log
```

### Format Log

Setiap entri log akan mencakup informasi berikut:
- Request ID
- Timestamp
- Client IP
- HTTP Method
- Path
- Response Status
- Latency
- User ID (jika menggunakan authentication)
- Service Name
- Route Name

Contoh format log:
```
2025-06-04T08:00:00.000Z | 192.168.1.1 | GET | /api/v1/users | 200 | 50ms | user123 | user-service | user-route
```

## Pembersihan (Opsional)

Untuk menghapus semua sumber daya yang telah di-deploy:

```bash
kubectl delete -k kubernetes/overlays/monitoring/
kubectl delete -k kubernetes/overlays/dev/
kubectl delete -k kubernetes/base/
# Jika menggunakan Helm untuk PostgreSQL:
helm uninstall kong-database
```

## Perubahan Terbaru

### Perbaikan Konfigurasi Prometheus

Konfigurasi Prometheus telah diperbaiki untuk mengatasi error "unknown Kubernetes SD role \"endpoint\"". Perubahan yang dilakukan:
- Mengganti `role: endpoint` menjadi `role: endpoints` di dalam file `prometheus.yml` pada ConfigMap Prometheus.

Langkah-langkah untuk memastikan Prometheus berjalan normal:
1. Terapkan ulang konfigurasi monitoring:
   ```bash
   kubectl apply -k kubernetes/overlays/monitoring/
   ```
2. Periksa status pod Prometheus:
   ```bash
   kubectl get pods -l app=prometheus
   ```
3. Jika pod Prometheus mengalami CrashLoopBackOff, hapus pod tersebut agar Kubernetes membuat ulang dengan konfigurasi terbaru:
   ```bash
   kubectl delete pod -l app=prometheus
   ```
4. Verifikasi kembali status pod Prometheus untuk memastikan sudah berjalan normal.

## Catatan Khusus untuk Kind Cluster

Jika Anda menggunakan Kind cluster, perhatikan hal-hal berikut:
1. Service tipe LoadBalancer tidak didukung secara native. Gunakan NodePort atau port-forward untuk mengakses service dari luar cluster.
2. Untuk mengakses Grafana, gunakan port-forward:
   ```bash
   kubectl port-forward svc/grafana 3000:80
   ```
   Kemudian akses Grafana di browser melalui `http://localhost:3000`

3. Untuk mengakses Prometheus, gunakan port-forward:
   ```bash
   kubectl port-forward svc/prometheus 9090:80
   ```
   Kemudian akses Prometheus di browser melalui `http://localhost:9090`
