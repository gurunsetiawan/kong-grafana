# Kong API Monitoring di Kubernetes

Repositori ini berisi konfigurasi Kubernetes (menggunakan Kustomize) untuk mendeploy Kong API Gateway, sebuah contoh API, dan stack pemantauan berbasis Prometheus dan Grafana untuk memantau lalu lintas API Kong.

## Struktur Direktori
```
.
├── kubernetes/               # Berisi semua konfigurasi Kubernetes
│   ├── base/                 # Konfigurasi dasar yang dapat digunakan kembali
│   │   ├── kong/             # Deployment dan Service Kong
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

## Langkah-langkah Deployment

Ikuti langkah-langkah di bawah ini untuk mendeploy semua komponen ke cluster Kubernetes Anda.

### 1. Terapkan Komponen Dasar

Ini akan mendeploy Kong API Gateway dan aplikasi `hello-world-api`.

```bash
kubectl apply -k kubernetes/base/
Verifikasi Pod dan Service berjalan:
```

```Bash

kubectl get pods -l app=kong
kubectl get svc kong-proxy kong-admin
kubectl get pods -l app=hello-world
kubectl get svc hello-world-service
```

2. Terapkan Kong Route untuk Lingkungan Dev
Ini akan mengkonfigurasi Kong untuk mengarahkan lalu lintas ke hello-world-api.

```Bash

kubectl apply -k kubernetes/overlays/dev/
```
Verifikasi KongRoute telah dibuat:

```Bash

kubectl get kongroute
```
3. Terapkan Komponen Pemantauan (Prometheus & Grafana)
Ini akan mendeploy Prometheus untuk mengumpulkan metrik dari Kong, dan Grafana untuk memvisualisasikannya.

```Bash

kubectl apply -k kubernetes/overlays/monitoring/
```

Verifikasi Pod dan Service monitoring berjalan:


```Bash

kubectl get pods -l app=prometheus
kubectl get svc prometheus
kubectl get pods -l app=grafana
kubectl get svc grafana
```

Pengujian API dan Pemantauan
1. Uji API melalui Kong
Dapatkan IP eksternal dari Service kong-proxy:

```Bash

kubectl get svc kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Atau untuk Minikube/Docker Desktop:
# minikube service kong-proxy --url
```
Gunakan IP tersebut untuk memanggil hello-world-api melalui Kong:


```Bash

curl http://<KONG_PROXY_EXTERNAL_IP>/hello/get
```
Anda akan melihat respons JSON dari httpbin yang menunjukkan Kong berhasil mengarahkan permintaan.


2. Akses Grafana
Dapatkan IP eksternal dari Service grafana:

```Bash

kubectl get svc grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Atau untuk Minikube/Docker Desktop:
# minikube service grafana --url
```
Buka browser Anda dan navigasi ke http://<GRAFANA_EXTERNAL_IP>.

```
Username default: admin
Password default: admin (Anda akan diminta untuk mengubahnya pada login pertama).
```
Setelah login, navigasikan ke Dashboards (biasanya ikon kotak di sidebar kiri). Anda akan melihat dashboard bernama "Kong Official Dashboard" (atau nama lain yang Anda gunakan di configmap.yaml Grafana). Dashboard ini akan menampilkan metrik lalu lintas API yang dikumpulkan oleh Prometheus dari Kong.

Pembersihan (Opsional)
Untuk menghapus semua sumber daya yang telah di-deploy:

```Bash

kubectl delete -k kubernetes/overlays/monitoring/
kubectl delete -k kubernetes/overlays/dev/
kubectl delete -k kubernetes/base/
```
---

### `kubernetes/README.md`

# Kubernetes Configurations

Direktori ini berisi semua konfigurasi Kubernetes YAML yang diperlukan untuk mendeploy Kong API Gateway, sebuah contoh aplikasi, dan stack pemantauan (Prometheus dan Grafana).

## Organisasi

Konfigurasi diorganisasikan menggunakan Kustomize, yang memungkinkan penggunaan kembali komponen dasar dan penyesuaian untuk lingkungan yang berbeda melalui *overlays*.

-   **`base/`**: Berisi definisi inti dari setiap aplikasi (Kong, Hello World API) yang tidak bergantung pada lingkungan spesifik.
-   **`overlays/`**: Berisi penyesuaian untuk lingkungan atau tujuan tertentu.
    -   **`dev/`**: Overlays untuk lingkungan pengembangan, termasuk konfigurasi KongRoute.
    -   **`monitoring/`**: Overlays untuk stack pemantauan Prometheus dan Grafana.

## Cara Menerapkan

Untuk menerapkan konfigurasi, navigasikan ke direktori `kubernetes/` dan gunakan `kubectl apply -k` pada direktori *overlay* yang diinginkan.

**Contoh:**

```bash
# Terapkan komponen dasar
kubectl apply -k base/
```

# Terapkan konfigurasi dev (termasuk KongRoute)
```
kubectl apply -k overlays/dev/
```

# Terapkan stack monitoring
```
kubectl apply -k overlays/monitoring/
```

Penjelasan Komponen

Kong API Gateway:
deployment.yaml: Mendefinisikan Pod Kong dengan plugin Prometheus diaktifkan.
service.yaml: Mendefinisikan Service untuk Kong Proxy (akses eksternal) dan Kong Admin API (akses internal).

Hello World API:
deployment.yaml: Contoh aplikasi API sederhana yang akan dipantau.
service.yaml: Service untuk aplikasi Hello World.

KongRoute: Mengkonfigurasi Kong untuk mengarahkan lalu lintas dari Kong Proxy ke hello-world-api berdasarkan path /hello.

Prometheus:
deployment.yaml: Mendefinisikan Pod Prometheus.
service.yaml: Service untuk Prometheus.
configmap.yaml: Mengkonfigurasi Prometheus untuk scrape metrik dari Kong Admin API (port 8001).

Grafana:
deployment.yaml: Mendefinisikan Pod Grafana.
service.yaml: Service untuk Grafana (akses eksternal).
datasource.yaml: Mengkonfigurasi Prometheus sebagai sumber data Grafana secara otomatis.
configmap.yaml: Berisi definisi JSON untuk dashboard Grafana, seperti "Kong Official Dashboard".
