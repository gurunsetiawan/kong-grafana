Tentu, ini dia semua skrip YAML yang sudah diperbaiki dan dikompilasi, lengkap dengan struktur file dan README.md untuk membantu Anda memulai pemantauan API Kong di Kubernetes.

Struktur Repositori
.
ÃÄÄ kubernetes/
³   ÃÄÄ base/
³   ³   ÃÄÄ kong/
³   ³   ³   ÃÄÄ deployment.yaml
³   ³   ³   ÀÄÄ service.yaml
³   ³   ÃÄÄ hello-world-api/
³   ³   ³   ÃÄÄ deployment.yaml
³   ³   ³   ÀÄÄ service.yaml
³   ³   ÀÄÄ kustomization.yaml
³   ÃÄÄ overlays/
³   ³   ÃÄÄ dev/
³   ³   ³   ÃÄÄ kustomization.yaml
³   ³   ³   ÀÄÄ kong-route.yaml
³   ³   ÃÄÄ monitoring/
³   ³   ³   ÃÄÄ kustomization.yaml
³   ³   ³   ÃÄÄ prometheus/
³   ³   ³   ³   ÃÄÄ deployment.yaml
³   ³   ³   ³   ÃÄÄ service.yaml
³   ³   ³   ³   ÀÄÄ configmap.yaml
³   ³   ³   ÀÄÄ grafana/
³   ³   ³       ÃÄÄ deployment.yaml
³   ³   ³       ÃÄÄ service.yaml
³   ³   ³       ÃÄÄ configmap.yaml
³   ³   ³       ÀÄÄ datasource.yaml
³   ÀÄÄ README.md
ÃÄÄ .gitignore
ÀÄÄ README.md
File-File Konfigurasi Kubernetes
1. Komponen Dasar (kubernetes/base/)
kubernetes/base/kong/deployment.yaml

YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong
  labels:
    app: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kong
  template:
    metadata:
      labels:
        app: kong
    spec:
      containers:
      - name: kong
        image: kong:3.7.0 # Gunakan versi spesifik yang stabil
        ports:
        - name: proxy-http
          containerPort: 8000
        - name: proxy-https
          containerPort: 8443
        - name: admin-http
          containerPort: 8001
        - name: admin-https
          containerPort: 8444
        env:
        - name: KONG_DATABASE
          value: "off" # Untuk pengujian cepat tanpa database. Gunakan 'postgres' atau 'cassandra' untuk produksi.
        - name: KONG_PROXY_LISTEN
          value: "0.0.0.0:8000, 0.0.0.0:8443 ssl"
        - name: KONG_ADMIN_LISTEN
          value: "0.0.0.0:8001, 0.0.0.0:8444 ssl"
        # --- Konfigurasi untuk Monitoring (Plugin Prometheus Kong) ---
        - name: KONG_PLUGINS
          value: "prometheus" # Mengaktifkan plugin Prometheus Kong
        - name: KONG_PROMETHEUS_LISTEN
          value: "0.0.0.0:8001" # Plugin Prometheus akan mengekspos metrik di port Admin API
        resources: # Sangat direkomendasikan untuk stabilitas
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
kubernetes/base/kong/service.yaml

YAML

apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
  labels:
    app: kong
spec:
  type: LoadBalancer # Gunakan LoadBalancer untuk akses eksternal di cloud, atau NodePort/ClusterIP
  selector:
    app: kong
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  - port: 443
    targetPort: 8443
    protocol: TCP
    name: https
---
apiVersion: v1
kind: Service
metadata:
  name: kong-admin
  labels:
    app: kong
spec:
  type: ClusterIP # Akses internal saja (Admin API tidak terekspos langsung ke publik)
  selector:
    app: kong
  ports:
  - port: 8001
    targetPort: 8001
    protocol: TCP
    name: http-admin # Penting: Prometheus akan scrape dari port ini
  - port: 8444
    targetPort: 8444
    protocol: TCP
    name: https-admin
kubernetes/base/hello-world-api/deployment.yaml

YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-api
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: kennethreitz/httpbin:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
kubernetes/base/hello-world-api/service.yaml

YAML

apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
kubernetes/base/kustomization.yaml

YAML

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- kong/deployment.yaml
- kong/service.yaml
- hello-world-api/deployment.yaml
- hello-world-api/service.yaml
2. Overlay Lingkungan Dev (kubernetes/overlays/dev/)
kubernetes/overlays/dev/kustomization.yaml

YAML

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base # Referensi ke direktori 'base'
resources:
- kong-route.yaml # Tambahkan route Kong khusus untuk dev
kubernetes/overlays/dev/kong-route.yaml

YAML

apiVersion: configuration.konghq.com/v1
kind: KongRoute
metadata:
  name: hello-world-route-dev
  namespace: default # Atau namespace lain yang Anda gunakan
spec:
  protocols:
  - http
  - https
  paths:
  - /hello # Kong akan mengarahkan semua permintaan ke /hello
  service:
    name: hello-world-service
    port: 80
3. Overlay Monitoring (kubernetes/overlays/monitoring/)
kubernetes/overlays/monitoring/kustomization.yaml

YAML

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- prometheus/deployment.yaml
- prometheus/service.yaml
- prometheus/configmap.yaml
- grafana/deployment.yaml
- grafana/service.yaml
- grafana/datasource.yaml
- grafana/configmap.yaml # Untuk dashboard Grafana
kubernetes/overlays/monitoring/prometheus/deployment.yaml

YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.52.0 # Gunakan versi stabil
        args:
          - "--config.file=/etc/prometheus/prometheus.yml"
          - "--storage.tsdb.retention.time=15d" # Contoh retensi data
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config-volume
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
      volumes:
      - name: prometheus-config-volume
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {} # Gunakan PersistentVolumeClaim untuk produksi
kubernetes/overlays/monitoring/prometheus/service.yaml

YAML

apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  type: ClusterIP # Prometheus biasanya diakses internal oleh Grafana
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
    name: http
kubernetes/overlays/monitoring/prometheus/configmap.yaml

YAML

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  labels:
    app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s # Atur interval scrape
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'kong'
        metrics_path: '/metrics' # Default path untuk plugin Prometheus Kong
        kubernetes_sd_configs:
        - role: endpoint
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: kong-admin;http-admin # Pastikan ini sesuai dengan nama service dan port Admin Kong Anda
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
kubernetes/overlays/monitoring/grafana/deployment.yaml

YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups: # Penting untuk Grafana menulis ke volume
          - 0
      containers:
      - name: grafana
        image: grafana/grafana:10.4.2 # Gunakan versi stabil
        ports:
        - containerPort: 3000
        env:
        - name: GF_PATHS_CONFIG
          value: /etc/grafana/grafana.ini # Pastikan ini sesuai dengan path configmap
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources # Untuk datasource otomatis
        - name: grafana-dashboards
          mountPath: /etc/grafana/provisioning/dashboards # Untuk dashboard otomatis
      volumes:
      - name: grafana-storage
        emptyDir: {} # Gunakan PersistentVolumeClaim untuk produksi
      - name: grafana-datasources
        configMap:
          name: grafana-datasources-config # Dari configmap di bawah
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboards-config # Dari configmap di bawah
kubernetes/overlays/monitoring/grafana/service.yaml

YAML

apiVersion: v1
kind: Service
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  type: LoadBalancer # Atau NodePort jika ingin diakses dari luar cluster Anda
  selector:
    app: grafana
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
kubernetes/overlays/monitoring/grafana/datasource.yaml

YAML

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources-config
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus:9090 # Ini adalah nama Service Prometheus di Kubernetes
        isDefault: true
        access: proxy
kubernetes/overlays/monitoring/grafana/configmap.yaml (untuk Dashboard Grafana)
Anda bisa mengunduh dashboard Kong yang sudah jadi dari Grafana Labs (cari "Kong" atau ID 7424) dan menempelkan JSON-nya di sini.

YAML

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-config
data:
  kong-official-dashboard.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": 7424,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "panels": [
        # Tempelkan seluruh JSON dashboard Grafana di sini
        # Anda bisa mengunduhnya dari https://grafana.com/grafana/dashboards/7424-kong-official-dashboard/
        # Pastikan formatnya benar setelah ditempel
      ],
      "schemaVersion": 26,
      "style": "dark",
      "tags": [
        "kong"
      ],
      "templating": {
        "list": [
          {
            "current": {
              "selected": true,
              "text": "All",
              "value": "$__all"
            },
            "datasource": "Prometheus",
            "definition": "label_values(kong_nginx_http_requests_total_sum, service)",
            "description": null,
            "hide": 0,
            "includeAll": true,
            "label": "Service",
            "multi": true,
            "name": "service",
            "options": [],
            "query": "label_values(kong_nginx_http_requests_total_sum, service)",
            "refresh": 1,
            "regex": "",
            "skipUrlSync": false,
            "sort": 0,
            "tagValuesQuery": "",
            "tags": [],
            "type": "query",
            "useAllValue": false
          }
        ]
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "",
      "title": "Kong Official Dashboard",
      "uid": "7424-kong-official-dashboard",
      "version": 1
    }
README.md (Root Repository)
Markdown

# Kong API Monitoring di Kubernetes

Repositori ini berisi konfigurasi Kubernetes (menggunakan Kustomize) untuk mendeploy Kong API Gateway, sebuah contoh API, dan stack pemantauan berbasis Prometheus dan Grafana untuk memantau lalu lintas API Kong.

## Struktur Direktori

.
ÃÄÄ kubernetes/               # Berisi semua konfigurasi Kubernetes
³   ÃÄÄ base/                 # Konfigurasi dasar yang dapat digunakan kembali
³   ³   ÃÄÄ kong/             # Deployment dan Service Kong
³   ³   ÃÄÄ hello-world-api/  # Deployment dan Service contoh API
³   ³   ÀÄÄ kustomization.yaml# Mengatur sumber daya dasar
³   ÃÄÄ overlays/             # Konfigurasi spesifik lingkungan
³   ³   ÃÄÄ dev/              # Overlays untuk lingkungan pengembangan
³   ³   ³   ÃÄÄ kustomization.yaml
³   ³   ³   ÀÄÄ kong-route.yaml # Route Kong khusus dev
³   ³   ÃÄÄ monitoring/       # Overlays untuk komponen pemantauan
³   ³   ³   ÃÄÄ kustomization.yaml
³   ³   ³   ÃÄÄ prometheus/   # Deployment, Service, dan ConfigMap Prometheus
³   ³   ³   ÀÄÄ grafana/      # Deployment, Service, ConfigMap, dan Datasource Grafana
³   ÀÄÄ README.md             # Penjelasan tentang struktur Kubernetes
ÃÄÄ .gitignore                # File yang akan diabaikan oleh Git
ÀÄÄ README.md                 # README utama repositori ini


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

Bash

kubectl get pods -l app=kong
kubectl get svc kong-proxy kong-admin
kubectl get pods -l app=hello-world
kubectl get svc hello-world-service
2. Terapkan Kong Route untuk Lingkungan Dev
Ini akan mengkonfigurasi Kong untuk mengarahkan lalu lintas ke hello-world-api.

Bash

kubectl apply -k kubernetes/overlays/dev/
Verifikasi KongRoute telah dibuat:

Bash

kubectl get kongroute
3. Terapkan Komponen Pemantauan (Prometheus & Grafana)
Ini akan mendeploy Prometheus untuk mengumpulkan metrik dari Kong, dan Grafana untuk memvisualisasikannya.

Bash

kubectl apply -k kubernetes/overlays/monitoring/
Verifikasi Pod dan Service monitoring berjalan:

Bash

kubectl get pods -l app=prometheus
kubectl get svc prometheus
kubectl get pods -l app=grafana
kubectl get svc grafana
Pengujian API dan Pemantauan
1. Uji API melalui Kong
Dapatkan IP eksternal dari Service kong-proxy:

Bash

kubectl get svc kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Atau untuk Minikube/Docker Desktop:
# minikube service kong-proxy --url
Gunakan IP tersebut untuk memanggil hello-world-api melalui Kong:

Bash

curl http://<KONG_PROXY_EXTERNAL_IP>/hello/get
Anda akan melihat respons JSON dari httpbin yang menunjukkan Kong berhasil mengarahkan permintaan.

2. Akses Grafana
Dapatkan IP eksternal dari Service grafana:

Bash

kubectl get svc grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Atau untuk Minikube/Docker Desktop:
# minikube service grafana --url
Buka browser Anda dan navigasi ke http://<GRAFANA_EXTERNAL_IP>.

Username default: admin
Password default: admin (Anda akan diminta untuk mengubahnya pada login pertama).
Setelah login, navigasikan ke Dashboards (biasanya ikon kotak di sidebar kiri). Anda akan melihat dashboard bernama "Kong Official Dashboard" (atau nama lain yang Anda gunakan di configmap.yaml Grafana). Dashboard ini akan menampilkan metrik lalu lintas API yang dikumpulkan oleh Prometheus dari Kong.

Pembersihan (Opsional)
Untuk menghapus semua sumber daya yang telah di-deploy:

Bash

kubectl delete -k kubernetes/overlays/monitoring/
kubectl delete -k kubernetes/overlays/dev/
kubectl delete -k kubernetes/base/

---

### `kubernetes/README.md`

```markdown
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

# Terapkan konfigurasi dev (termasuk KongRoute)
kubectl apply -k overlays/dev/

# Terapkan stack monitoring
kubectl apply -k overlays/monitoring/
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
<!-- end list -->


---

### `/.gitignore`

```gitignore
# Kubernetes specific files
*.kubeconfig
.envrc

# Log files
*.log
log/
logs/

# Build artifacts
target/
build/
dist/

# IDE specific files
.idea/
.vscode/
*.swp
*.bak
*.orig

# OS generated files
.DS_Store
Thumbs.db
