# KubePulse: Production Observability & Self-Healing Setup Guide

This document is a comprehensive, step-by-step implementation guide detailing the KubePulse observability setup. It covers local tool installation, directory layout, complete configurations, build steps, and cluster deployment validations.

---

## 1. Prerequisites and Tool Installation

To set up the KubePulse environment, ensure the following tools are installed on your host system:

### Step 1: Install Docker Desktop
- **Windows**: Download and install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/).
- **Validation**: Open a terminal and verify Docker is running:
  ```bash
  docker --version
  docker ps
  ```

### Step 2: Install Winget (Windows Package Manager)
- Winget comes pre-installed on Windows 10/11. If missing, install it from the Microsoft Store.

### Step 3: Install Kubectl CLI
- Run the following winget command in PowerShell to install the Kubernetes CLI:
  ```powershell
  winget install Kubernetes.kubectl
  ```
- **Validation**: Verify kubectl version:
  ```bash
  kubectl version --client
  ```

### Step 4: Install Kind (Kubernetes-in-Docker)
- Run the following winget command in PowerShell to install Kind:
  ```powershell
  winget install Kubernetes.kind
  ```
- **Validation**: Verify Kind version:
  ```bash
  kind --version
  ```

---

## 2. Directory Layout & Architecture

The workspace is organized into a modular, containerized structure to simplify deployment:

```text
KubePulse/
├── kind-config.yaml          # Kind cluster port-mapping configuration
├── implementation.md         # Master implementation plan
├── k8s/                      # Kubernetes manifests
│   ├── namespace.yaml        # Namespace definition (kubepulse)
│   ├── rbac.yaml             # ServiceAccount, ClusterRole, and Bindings
│   ├── ingress.yaml          # Global Ingress routes for frontend & backend
│   ├── backend.yaml          # Backend API Deployment & Service
│   ├── frontend.yaml         # Streamlit frontend UI Deployment & Service
│   └── monitoring/           # Prometheus and Grafana manifests
│       ├── prometheus.yaml   # Prometheus deployment & scraping rules
│       ├── grafana.yaml      # Grafana datasources, dashboards provider, and dashboard ConfigMaps
│       └── alertmanager.yaml # Alertmanager webhook configuration
├── backend/                  # FastAPI Backend Source
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py           # API gateway entrypoint
│   │   ├── k8s_client.py     # Kubernetes API client operations
│   │   └── alerts.py         # Alertmanager webhook handling & notifications
│   ├── Dockerfile            # Container build specification
│   └── requirements.txt      # Python dependencies
└── frontend/                 # Web Dashboard Source
    ├── app.py                # Streamlit dashboard layout & metrics logic
    ├── requirements.txt      # Frontend Python dependencies
    └── Dockerfile            # Streamlit frontend server build
```

---

## 3. Provisioning the Kind Cluster

We configure a multi-port mapped Kind cluster to route HTTP/HTTPS traffic directly from the host (`localhost`) into the cluster's ingress controller.

### Create Cluster
Create the cluster using the Kind configuration file:
```powershell
kind create cluster --config kind-config.yaml --name kubepulse
```

### Install Ingress Controller
Deploy the NGINX Ingress controller, optimized for Kind:
```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Wait for Ingress Controller to be ready
```powershell
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## 4. Complete Codebase & Configurations

### kind-config.yaml
*Kind cluster port-mapping configuration*

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP

```

---

### k8s/namespace.yaml
*Namespace definition (kubepulse)*

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubepulse

```

---

### k8s/rbac.yaml
*Kubernetes service account and cluster role permissions*

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubepulse-backend
  namespace: kubepulse
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubepulse-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "nodes", "namespaces", "services", "events", "persistentvolumes", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubepulse-backend-binding
subjects:
- kind: ServiceAccount
  name: kubepulse-backend
  namespace: kubepulse
roleRef:
  kind: ClusterRole
  name: kubepulse-reader
  apiGroup: rbac.authorization.k8s.io

```

---

### k8s/ingress.yaml
*Global Ingress routing rule specifications*

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubepulse-ingress
  namespace: kubepulse
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /grafana
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
      - path: /prometheus
        pathType: Prefix
        backend:
          service:
            name: prometheus
            port:
              number: 9090
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: kubepulse-backend
            port:
              number: 8000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubepulse-frontend
            port:
              number: 8501

```

---

### k8s/backend.yaml
*Backend API service deployment manifests*

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubepulse-backend
  namespace: kubepulse
  labels:
    app: kubepulse-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubepulse-backend
  template:
    metadata:
      labels:
        app: kubepulse-backend
    spec:
      serviceAccountName: kubepulse-backend
      containers:
      - name: backend
        image: kubepulse-backend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8000
        env:
        - name: IN_CLUSTER
          value: "true"
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kubepulse-backend
  namespace: kubepulse
  labels:
    app: kubepulse-backend
spec:
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
  selector:
    app: kubepulse-backend

```

---

### k8s/frontend.yaml
*Streamlit UI service deployment manifests*

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubepulse-frontend
  namespace: kubepulse
  labels:
    app: kubepulse-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubepulse-frontend
  template:
    metadata:
      labels:
        app: kubepulse-frontend
    spec:
      containers:
      - name: frontend
        image: kubepulse-frontend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8501
        resources:
          limits:
            cpu: "200m"
            memory: "256Mi"
          requests:
            cpu: "50m"
            memory: "64Mi"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8501
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8501
          initialDelaySeconds: 2
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kubepulse-frontend
  namespace: kubepulse
  labels:
    app: kubepulse-frontend
spec:
  ports:
  - port: 8501
    targetPort: 8501
    protocol: TCP
  selector:
    app: kubepulse-frontend

```

---

### k8s/monitoring/prometheus.yaml
*Prometheus server configuration and node scrapers*

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kubepulse
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kubepulse
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  namespace: kubepulse
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        authorization:
          credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        authorization:
          credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        authorization:
          credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      - job_name: 'kubernetes-service-endpoints'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: kubernetes_name
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: kubepulse
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
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.52.0
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        - "--web.external-url=http://localhost/prometheus/"
        - "--web.route-prefix=/prometheus"
        ports:
        - containerPort: 9090
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
          requests:
            cpu: "100m"
            memory: "256Mi"
        volumeMounts:
        - name: prometheus-config-volume
          mountPath: /etc/prometheus/
        - name: prometheus-storage-volume
          mountPath: /prometheus/
      volumes:
      - name: prometheus-config-volume
        configMap:
          defaultMode: 420
          name: prometheus-server-conf
      - name: prometheus-storage-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kubepulse
  labels:
    app: prometheus
spec:
  ports:
  - port: 9090
    targetPort: 9090
    protocol: TCP
  selector:
    app: prometheus

```

---

### k8s/monitoring/grafana.yaml
*Grafana datasources, dashboards provisioning, and dashboards*

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: kubepulse
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus.kubepulse.svc.cluster.local:9090/prometheus
      isDefault: true
    - name: Alertmanager
      type: alertmanager
      access: proxy
      url: http://alertmanager.kubepulse.svc.cluster.local:9093
      jsonData:
        implementation: prometheus
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-provider
  namespace: kubepulse
data:
  provider.yaml: |
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-k8s-dashboard
  namespace: kubepulse
data:
  k8s-dashboard.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "datasource",
              "uid": "grafana"
            },
            "enable": true,
            "hide": true,
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 4,
            "w": 6,
            "x": 0,
            "y": 0
          },
          "id": 1,
          "title": "Active Pods",
          "type": "stat",
          "datasource": "Prometheus",
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(kubelet_active_pods)",
              "instant": true,
              "range": false,
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "thresholds"
              },
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  }
                ]
              }
            }
          },
          "options": {
            "colorMode": "value",
            "graphMode": "none",
            "justifyMode": "center",
            "textMode": "value"
          }
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 4,
            "w": 9,
            "x": 6,
            "y": 0
          },
          "id": 2,
          "title": "Cluster CPU Usage (Cores)",
          "type": "gauge",
          "datasource": "Prometheus",
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(rate(container_cpu_usage_seconds_total{container!=\"\",container!=\"POD\"}[2m]))",
              "instant": true,
              "range": false,
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "none",
              "min": 0,
              "max": 4,
              "color": {
                "mode": "palette-classic"
              }
            }
          }
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 4,
            "w": 9,
            "x": 15,
            "y": 0
          },
          "id": 3,
          "title": "Cluster Memory Usage",
          "type": "gauge",
          "datasource": "Prometheus",
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(container_memory_working_set_bytes{container!=\"\",container!=\"POD\"})",
              "instant": true,
              "range": false,
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "bytes",
              "color": {
                "mode": "palette-classic"
              }
            }
          }
        },
        {
          "title": "CPU Usage per Pod (Top 10)",
          "type": "timeseries",
          "datasource": "Prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 4
          },
          "id": 4,
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(rate(container_cpu_usage_seconds_total{container!=\"\",container!=\"POD\"}[2m])) by (pod)",
              "legendFormat": "{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "drawStyle": "line",
                "lineInterpolation": "smooth"
              },
              "unit": "none"
            }
          }
        },
        {
          "title": "Memory Usage per Pod (Top 10)",
          "type": "timeseries",
          "datasource": "Prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 4
          },
          "id": 5,
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(container_memory_working_set_bytes{container!=\"\",container!=\"POD\"}) by (pod)",
              "legendFormat": "{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "drawStyle": "line",
                "lineInterpolation": "smooth"
              },
              "unit": "bytes"
            }
          }
        },
        {
          "title": "Network RX Rate per Pod",
          "type": "timeseries",
          "datasource": "Prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 12
          },
          "id": 6,
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(rate(container_network_receive_bytes_total[2m])) by (pod)",
              "legendFormat": "{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "drawStyle": "line",
                "lineInterpolation": "smooth"
              },
              "unit": "Bps"
            }
          }
        },
        {
          "title": "Network TX Rate per Pod",
          "type": "timeseries",
          "datasource": "Prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 12
          },
          "id": 7,
          "targets": [
            {
              "datasource": "Prometheus",
              "editorMode": "code",
              "expr": "sum(rate(container_network_transmit_bytes_total[2m])) by (pod)",
              "legendFormat": "{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "drawStyle": "line",
                "lineInterpolation": "smooth"
              },
              "unit": "Bps"
            }
          }
        }
      ],
      "refresh": "5s",
      "schemaVersion": 39,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-15m",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "browser",
      "title": "KubePulse Kubernetes Monitoring",
      "uid": "kubepulse-k8s",
      "version": 1,
      "weekStart": ""
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: kubepulse
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
      containers:
      - name: grafana
        image: grafana/grafana:10.4.2
        ports:
        - containerPort: 3000
        env:
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: "Viewer"
        - name: GF_SECURITY_ALLOW_EMBEDDING
          value: "true"
        - name: GF_SERVER_ROOT_URL
          value: "http://localhost/grafana/"
        - name: GF_SERVER_SERVE_FROM_SUB_PATH
          value: "true"
        - name: GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH
          value: "/var/lib/grafana/dashboards/k8s-dashboard.json"
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "256Mi"
        volumeMounts:
        - name: grafana-datasources-volume
          mountPath: /etc/grafana/provisioning/datasources/
        - name: grafana-dashboards-provider-volume
          mountPath: /etc/grafana/provisioning/dashboards/
        - name: grafana-dashboards-volume
          mountPath: /var/lib/grafana/dashboards/
      volumes:
      - name: grafana-datasources-volume
        configMap:
          name: grafana-datasources
      - name: grafana-dashboards-provider-volume
        configMap:
          name: grafana-dashboards-provider
      - name: grafana-dashboards-volume
        configMap:
          name: grafana-k8s-dashboard
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kubepulse
  labels:
    app: grafana
spec:
  ports:
  - port: 3000
    targetPort: 3000
    protocol: TCP
  selector:
    app: grafana

```

---

### k8s/monitoring/alertmanager.yaml
*Alertmanager routing to Backend webhook*

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: kubepulse
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'kubepulse-webhook'

    receivers:
    - name: 'kubepulse-webhook'
      webhook_configs:
      - url: 'http://kubepulse-backend.kubepulse.svc.cluster.local:8000/api/alerts/webhook'
        send_resolved: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: kubepulse
  labels:
    app: alertmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.27.0
        args:
        - "--config.file=/etc/alertmanager/alertmanager.yml"
        - "--storage.path=/alertmanager/"
        ports:
        - containerPort: 9093
        resources:
          limits:
            cpu: "200m"
            memory: "256Mi"
          requests:
            cpu: "50m"
            memory: "64Mi"
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: storage-volume
          mountPath: /alertmanager
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager-config
      - name: storage-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: kubepulse
  labels:
    app: alertmanager
spec:
  ports:
  - port: 9093
    targetPort: 9093
    protocol: TCP
  selector:
    app: alertmanager

```

---

### backend/Dockerfile
*FastAPI Backend container build configuration*

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY ./app ./app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

```

---

### backend/requirements.txt
*FastAPI Python packages requirements*

``` text
fastapi==0.111.0
uvicorn==0.30.1
kubernetes==30.1.0

```

---

### backend/app/main.py
*FastAPI main endpoint definitions and webhook handler*

``` python
from fastapi import FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from app.k8s_client import K8sClient
from app.alerts import alert_store

app = FastAPI(title="KubePulse Backend API", version="1.0.0")

# Configure CORS for local development
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize K8s Client lazily
k8s_client = None

def get_k8s_client():
    global k8s_client
    if k8s_client is None:
        try:
            k8s_client = K8sClient()
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Failed to connect to Kubernetes API: {str(e)}")
    return k8s_client

@app.get("/health")
def health_check():
    return {"status": "healthy", "service": "kubepulse-backend"}

@app.post("/api/alerts/webhook")
async def alert_webhook(request: Request):
    try:
        payload = await request.json()
        alert_store.add_alerts(payload)
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Invalid payload: {str(e)}")

@app.get("/api/cluster/alerts")
def get_alerts(history: bool = False):
    if history:
        return alert_store.get_history()
    return alert_store.get_active()

@app.get("/api/cluster/summary")
def get_summary():
    client = get_k8s_client()
    try:
        summary = client.get_summary()
        # Enrich summary with active alerts count
        summary["active_alerts"] = len(alert_store.get_active())
        return summary
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/cluster/nodes")
def get_nodes():
    client = get_k8s_client()
    try:
        return client.get_nodes()
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/cluster/namespaces")
def get_namespaces():
    client = get_k8s_client()
    try:
        return client.get_namespaces()
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/cluster/pods")
def get_pods(namespace: str = None):
    client = get_k8s_client()
    try:
        return client.get_pods(namespace)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/cluster/deployments")
def get_deployments(namespace: str = None):
    client = get_k8s_client()
    try:
        return client.get_deployments(namespace)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/cluster/events")
def get_events():
    client = get_k8s_client()
    try:
        return client.get_events()
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

```

---

### backend/app/k8s_client.py
*Kubernetes python API wrapper client*

``` python
import os
from kubernetes import client, config
from kubernetes.client.rest import ApiException

class K8sClient:
    def __init__(self):
        # Determine whether to load in-cluster config or local kubeconfig
        if os.getenv("IN_CLUSTER") == "true":
            try:
                config.load_incluster_config()
            except config.config_exception.ConfigException:
                # Fallback to local config if loading fails (e.g. running locally for tests)
                config.load_kube_config()
        else:
            config.load_kube_config()
            
        self.v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()

    def get_summary(self):
        try:
            nodes = self.v1.list_node().items
            namespaces = self.v1.list_namespace().items
            pods = self.v1.list_pod_for_all_namespaces().items
            deployments = self.apps_v1.list_deployment_for_all_namespaces().items
            
            # Count statuses
            running_pods = sum(1 for p in pods if p.status.phase == "Running")
            failed_pods = sum(1 for p in pods if p.status.phase in ["Failed", "Unknown"])
            pending_pods = sum(1 for p in pods if p.status.phase == "Pending")
            
            ready_nodes = sum(1 for n in nodes if any(c.type == "Ready" and c.status == "True" for c in n.status.conditions))
            
            return {
                "nodes": {
                    "total": len(nodes),
                    "ready": ready_nodes,
                    "unready": len(nodes) - ready_nodes
                },
                "namespaces": len(namespaces),
                "pods": {
                    "total": len(pods),
                    "running": running_pods,
                    "failed": failed_pods,
                    "pending": pending_pods
                },
                "deployments": len(deployments)
            }
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

    def get_nodes(self):
        try:
            nodes = self.v1.list_node().items
            node_list = []
            for node in nodes:
                # Get conditions
                ready_status = "Unknown"
                for condition in node.status.conditions:
                    if condition.type == "Ready":
                        ready_status = "Ready" if condition.status == "True" else "NotReady"
                        break
                
                # Get resource capacity
                cpu = node.status.capacity.get("cpu", "N/A")
                memory = node.status.capacity.get("memory", "N/A")
                
                # Get labels/roles
                roles = []
                for label in node.metadata.labels:
                    if label.startswith("node-role.kubernetes.io/"):
                        roles.append(label.split("/")[-1])
                if not roles:
                    roles = ["worker"]
                
                node_list.append({
                    "name": node.metadata.name,
                    "status": ready_status,
                    "roles": roles,
                    "cpu_capacity": cpu,
                    "memory_capacity": memory,
                    "kubelet_version": node.status.node_info.kubelet_version,
                    "os_image": node.status.node_info.os_image,
                    "kernel_version": node.status.node_info.kernel_version
                })
            return node_list
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

    def get_namespaces(self):
        try:
            ns_list = self.v1.list_namespace().items
            return [ns.metadata.name for ns in ns_list]
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

    def get_pods(self, namespace: str = None):
        try:
            if namespace:
                pods = self.v1.list_namespaced_pod(namespace).items
            else:
                pods = self.v1.list_pod_for_all_namespaces().items
                
            pod_list = []
            for pod in pods:
                # Count restarts
                restarts = 0
                container_states = {}
                if pod.status.container_statuses:
                    for status in pod.status.container_statuses:
                        restarts += status.restart_count
                        # Determine container state
                        if status.state.waiting:
                            container_states[status.name] = {
                                "state": "Waiting",
                                "reason": status.state.waiting.reason,
                                "message": status.state.waiting.message
                            }
                        elif status.state.running:
                            container_states[status.name] = {"state": "Running"}
                        elif status.state.terminated:
                            container_states[status.name] = {
                                "state": "Terminated",
                                "reason": status.state.terminated.reason
                            }
                
                pod_list.append({
                    "name": pod.metadata.name,
                    "namespace": pod.metadata.namespace,
                    "status": pod.status.phase,
                    "ip": pod.status.pod_ip or "N/A",
                    "node": pod.spec.node_name or "N/A",
                    "restarts": restarts,
                    "container_states": container_states,
                    "created_at": pod.metadata.creation_timestamp.isoformat() if pod.metadata.creation_timestamp else None
                })
            return pod_list
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

    def get_deployments(self, namespace: str = None):
        try:
            if namespace:
                deployments = self.apps_v1.list_namespaced_deployment(namespace).items
            else:
                deployments = self.apps_v1.list_deployment_for_all_namespaces().items
                
            deploy_list = []
            for deploy in deployments:
                replicas = deploy.status.replicas or 0
                ready_replicas = deploy.status.ready_replicas or 0
                available_replicas = deploy.status.available_replicas or 0
                
                # Check status
                status = "Healthy"
                if replicas != available_replicas:
                    status = "Degraded"
                
                deploy_list.append({
                    "name": deploy.metadata.name,
                    "namespace": deploy.metadata.namespace,
                    "status": status,
                    "replicas": {
                        "desired": deploy.spec.replicas,
                        "updated": deploy.status.updated_replicas or 0,
                        "ready": ready_replicas,
                        "available": available_replicas
                    },
                    "created_at": deploy.metadata.creation_timestamp.isoformat() if deploy.metadata.creation_timestamp else None
                })
            return deploy_list
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

    def get_events(self):
        try:
            # Get latest 25 events
            events = self.v1.list_event_for_all_namespaces(limit=25).items
            # Sort by last timestamp or creation timestamp
            events.sort(key=lambda x: x.last_timestamp or x.event_time or x.metadata.creation_timestamp or '', reverse=True)
            
            event_list = []
            for event in events[:25]:
                # Format timestamps safely
                t = event.last_timestamp or event.event_time or event.metadata.creation_timestamp
                t_str = t.isoformat() if t else "N/A"
                
                event_list.append({
                    "namespace": event.metadata.namespace,
                    "type": event.type,
                    "reason": event.reason,
                    "message": event.message,
                    "source": event.source.component or "N/A",
                    "object": f"{event.involved_object.kind}/{event.involved_object.name}",
                    "timestamp": t_str
                })
            return event_list
        except ApiException as e:
            raise Exception(f"Kubernetes API error: {e}")

```

---

### backend/app/alerts.py
*Alert store in-memory caching and webhook payload receiver*

``` python
class AlertStore:
    def __init__(self):
        self.active_alerts = []
        self.alert_history = []

    def add_alerts(self, payload: dict):
        status = payload.get("status", "firing")
        for alert in payload.get("alerts", []):
            alert_name = alert.get("labels", {}).get("alertname", "UnknownAlert")
            namespace = alert.get("labels", {}).get("namespace", "N/A")
            pod = alert.get("labels", {}).get("pod", "N/A")
            severity = alert.get("labels", {}).get("severity", "warning")
            summary = alert.get("annotations", {}).get("summary", "")
            description = alert.get("annotations", {}).get("description", "")
            starts_at = alert.get("startsAt")
            ends_at = alert.get("endsAt")
            
            alert_data = {
                "name": alert_name,
                "namespace": namespace,
                "pod": pod,
                "severity": severity,
                "summary": summary,
                "description": description,
                "status": alert.get("status", status),
                "starts_at": starts_at,
                "ends_at": ends_at
            }
            
            # If resolved, remove from active alerts and add to history
            if alert.get("status") == "resolved" or status == "resolved":
                # Find matching active alert and remove it
                self.active_alerts = [a for a in self.active_alerts if not (a["name"] == alert_name and a["namespace"] == namespace and a["pod"] == pod)]
                alert_data["status"] = "resolved"
                self.alert_history.append(alert_data)
            else:
                # Add to active alerts if not already present
                exists = any(a["name"] == alert_name and a["namespace"] == namespace and a["pod"] == pod for a in self.active_alerts)
                if not exists:
                    self.active_alerts.append(alert_data)
                    self.alert_history.append(alert_data)
            
            # Keep history capped at 100 items
            if len(self.alert_history) > 100:
                self.alert_history.pop(0)

    def get_active(self):
        return self.active_alerts

    def get_history(self):
        return self.alert_history

alert_store = AlertStore()

```

---

### frontend/Dockerfile
*Streamlit UI container build configuration*

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0", "--server.enableCORS=false", "--server.enableXsrfProtection=false", "--client.toolbarMode=minimal"]

```

---

### frontend/requirements.txt
*Streamlit UI Python packages requirements*

``` text
streamlit==1.35.0
requests==2.32.3

```

---

### frontend/app.py
*Streamlit UI dashboard code with tabs, grids, and auto-refresh logic*

``` python
import streamlit as st
import requests
import os
import time

# Set up page configurations
st.set_page_config(
    page_title="KubePulse | Kubernetes Dashboard",
    page_icon="⚡",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Backend URL resolver
BACKEND_URL = os.getenv("BACKEND_URL", "http://kubepulse-backend:8000")

# Fetch helpers
def fetch_api(endpoint):
    try:
        r = requests.get(f"{BACKEND_URL}/api{endpoint}", timeout=3)
        if r.status_code == 200:
            return r.json()
        return None
    except Exception:
        return None

# Page Header
st.markdown("""
<div style="text-align: center; margin-bottom: 25px; margin-top: -30px;">
    <h1 style="font-size: 3.8rem; font-weight: 900; letter-spacing: -2px; margin: 0 0 8px 0; background: linear-gradient(135deg, #00f2fe 0%, #4facfe 35%, #ec4899 100%); -webkit-background-clip: text; -webkit-text-fill-color: transparent; filter: drop-shadow(0px 4px 12px rgba(79, 172, 254, 0.25)); font-family: 'Inter', sans-serif;">
        ⚡ KubePulse
    </h1>
    <span style="background-color: rgba(16, 185, 129, 0.12); color: #10b981; padding: 6px 16px; border-radius: 9999px; font-size: 0.8rem; font-weight: 700; letter-spacing: 1px; border: 1px solid rgba(16, 185, 129, 0.25); text-transform: uppercase; font-family: 'Inter', sans-serif;">
        ACTIVE CLUSTER MONITORING
    </span>
</div>
""", unsafe_allow_html=True)

# Main Navigation Link Buttons (Centered below title using a flat layout to avoid React nested columns rendering loops)
btn_col1, btn_col2, btn_col3, btn_col4 = st.columns([2.5, 1.5, 1.5, 2.5])
with btn_col2:
    st.link_button("📊 Open Grafana", "http://localhost/grafana/", use_container_width=True)
with btn_col3:
    st.link_button("🔥 Open Prometheus", "http://localhost/prometheus/", use_container_width=True)

# Fetch current state data
summary = fetch_api("/cluster/summary")
namespaces_list = fetch_api("/cluster/namespaces") or ["all"]
nodes = fetch_api("/cluster/nodes")
alerts = fetch_api("/cluster/alerts")
events = fetch_api("/cluster/events")

# Sidebar Controls
st.sidebar.title("KubePulse Controls")
selected_ns = st.sidebar.selectbox("Filter Namespace", ["all"] + [ns for ns in namespaces_list if ns != "all"])

# Add manual refresh and auto refresh options
st.sidebar.markdown("---")
auto_refresh = st.sidebar.checkbox("Auto Refresh (5s)", value=True)

# Summary Metrics Row
if summary:
    nodes_ready = summary["nodes"]["ready"]
    nodes_total = summary["nodes"]["total"]
    pods_running = summary["pods"]["running"]
    pods_total = summary["pods"]["total"]
    active_alerts = summary["active_alerts"]
    
    col1, col2, col3, col4, col5 = st.columns(5)
    col1.metric("Nodes (Ready/Total)", f"{nodes_ready} / {nodes_total}", delta=f"{nodes_total - nodes_ready} unready" if nodes_total != nodes_ready else None, delta_color="inverse")
    col2.metric("Namespaces", summary["namespaces"])
    col3.metric("Deployments", summary["deployments"])
    col4.metric("Pods (Running/Total)", f"{pods_running} / {pods_total}", delta=f"{pods_total - pods_running} failing" if pods_total != pods_running else None, delta_color="inverse")
    
    # Alert metric color matches severity
    if active_alerts > 0:
        col5.markdown(f"""
        <div style="background-color: rgba(239, 68, 68, 0.12); padding: 10px; border-radius: 8px; border: 1px solid rgba(239, 68, 68, 0.3); text-align: center;">
            <span style="font-size: 0.8rem; color: #94a3b8; font-weight: bold; text-transform: uppercase;">Incidents / Alerts</span>
            <div style="font-size: 1.8rem; color: #ef4444; font-weight: bold; margin-top: 5px;">🔥 {active_alerts} Active</div>
        </div>
        """, unsafe_allow_html=True)
    else:
        col5.metric("Incidents / Alerts", "0 Active")
else:
    st.error("Could not fetch metrics summary from KubePulse backend. Check if the backend API is running.")

st.markdown("---")

# Main tabs layout
tab1, tab2, tab3, tab4, tab5 = st.tabs(["📦 Pods", "🚀 Deployments", "🖥️ Nodes", "🚨 Incidents", "📜 Event Stream"])

# Tab 1: Pods View
with tab1:
    st.subheader("Workload Pods")
    pods_url = "/cluster/pods" if selected_ns == "all" else f"/cluster/pods?namespace={selected_ns}"
    pods = fetch_api(pods_url)
    if pods:
        pods_data = []
        for p in pods:
            pods_data.append({
                "Pod Name": p["name"],
                "Namespace": p["namespace"],
                "Status": p["status"],
                "IP Address": p["ip"],
                "Node": p["node"],
                "Restarts": p["restarts"]
            })
        st.dataframe(pods_data, use_container_width=True)
    else:
        st.info("No active pods found in this namespace.")

# Tab 2: Deployments View
with tab2:
    st.subheader("Deployments Status")
    deploy_url = "/cluster/deployments" if selected_ns == "all" else f"/cluster/deployments?namespace={selected_ns}"
    deployments = fetch_api(deploy_url)
    if deployments:
        deploy_data = []
        for d in deployments:
            deploy_data.append({
                "Deployment Name": d["name"],
                "Namespace": d["namespace"],
                "Status": d["status"],
                "Desired": d["replicas"]["desired"],
                "Available": d["replicas"]["available"],
                "Ready": d["replicas"]["ready"]
            })
        st.dataframe(deploy_data, use_container_width=True)
    else:
        st.info("No deployments found in this namespace.")

# Tab 3: Nodes View
with tab3:
    st.subheader("Cluster Node Infrastructure")
    if nodes:
        for node in nodes:
            status_color = "green" if node["status"] == "Ready" else "red"
            st.markdown(f"""
            <div style="background-color: rgba(255,255,255,0.02); border: 1px solid rgba(255,255,255,0.08); padding: 15px; border-radius: 10px; margin-bottom: 15px;">
                <div style="display: flex; justify-content: space-between; align-items: center;">
                    <span style="font-weight: bold; font-size: 1.1rem;">🖥️ {node["name"]}</span>
                    <span style="background-color: {status_color}; color: white; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; font-weight: bold;">{node["status"]}</span>
                </div>
                <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 10px; font-size: 0.9rem; color: #94a3b8;">
                    <span><b>CPU Capacity:</b> {node["cpu_capacity"]} cores</span>
                    <span><b>Memory Capacity:</b> {node["memory_capacity"]}</span>
                    <span><b>Kubelet Version:</b> {node["kubelet_version"]}</span>
                    <span><b>Kernel Version:</b> {node["kernel_version"]}</span>
                    <span style="grid-column: span 2;"><b>OS Image:</b> {node["os_image"]}</span>
                </div>
            </div>
            """, unsafe_allow_html=True)
    else:
        st.info("No node information available.")

# Tab 4: Incidents View
with tab4:
    st.subheader("Firing Alerts & Webhook Incidents")
    if alerts and len(alerts) > 0:
        for alert in alerts:
            border_color = "#ef4444" if alert["severity"] == "critical" else "#f59e0b"
            st.markdown(f"""
            <div style="border-left: 5px solid {border_color}; background-color: rgba(255,255,255,0.02); padding: 15px; border-radius: 4px; margin-bottom: 15px;">
                <div style="display: flex; justify-content: space-between; align-items: center;">
                    <span style="font-weight: bold; font-size: 1.05rem; color: {border_color};">🚨 {alert["name"]}</span>
                    <span style="background-color: {border_color}; color: white; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; font-weight: bold;">{alert["severity"].upper()}</span>
                </div>
                <div style="margin-top: 8px; font-size: 0.95rem;">
                    <b>Summary:</b> {alert["summary"]}<br>
                    <b>Description:</b> {alert["description"]}
                </div>
                <div style="margin-top: 8px; font-size: 0.8rem; color: #94a3b8;">
                    Namespace: {alert["namespace"]} | Pod: {alert["pod"]} | Firing since: {alert["starts_at"]}
                </div>
            </div>
            """, unsafe_allow_html=True)
    else:
        st.success("No active incidents or alerts detected. The cluster is healthy!")

# Tab 5: Event Stream View
with tab5:
    st.subheader("Live Kubernetes Resource Events")
    if events:
        for e in events:
            indicator_color = "red" if e["type"] == "Warning" else "#3b82f6"
            st.markdown(f"""
            <div style="border-left: 3px solid {indicator_color}; padding-left: 10px; margin-bottom: 15px; font-size: 0.9rem;">
                <div style="display: flex; justify-content: space-between; color: #94a3b8; font-size: 0.8rem;">
                    <span><b>{e["object"]}</b></span>
                    <span>{e["timestamp"]}</span>
                </div>
                <div style="font-weight: bold; margin-top: 2px;">{e["reason"]} [{e["source"]}]</div>
                <div style="color: #cbd5e1; margin-top: 2px;">{e["message"]}</div>
            </div>
            """, unsafe_allow_html=True)
    else:
        st.info("No recent Kubernetes events found.")

# Trigger auto-refresh at the end of the script execution so the page actually renders first!
if auto_refresh:
    time.sleep(5)
    st.rerun()

```

---

## 5. Building & Loading Docker Images

Before applying the manifests to the cluster, build the frontend and backend Docker images locally on your host machine and load them directly into the Kind cluster nodes:

### Step 1: Build Backend Container
```powershell
# Navigate to the workspace and build the backend image
docker build -t kubepulse-backend:latest ./backend
```

### Step 2: Build Frontend Container
```powershell
# Build the frontend image
docker build -t kubepulse-frontend:latest ./frontend
```

### Step 3: Load Images into Kind Node
```powershell
# Load the backend image
kind load docker-image kubepulse-backend:latest --name kubepulse

# Load the frontend image
kind load docker-image kubepulse-frontend:latest --name kubepulse
```

---

## 6. Applying Kubernetes Manifests

Apply the manifests to the Kind cluster in the following logical order:

### Step 1: Create Namespace and RBAC permissions
```powershell
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/rbac.yaml
```

### Step 2: Deploy Backend API and Frontend Dashboard
```powershell
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

### Step 3: Deploy Monitoring Stack (Alertmanager, Prometheus, Grafana)
```powershell
kubectl apply -f k8s/monitoring/alertmanager.yaml
kubectl apply -f k8s/monitoring/prometheus.yaml
kubectl apply -f k8s/monitoring/grafana.yaml
```

### Step 4: Deploy Ingress Configurations
```powershell
kubectl apply -f k8s/ingress.yaml
```

---

## 8. Verification and Testing

### 1. Test Ingress and Dashboard
- Open your browser and navigate to `http://localhost/`. The KubePulse Streamlit Dashboard should load.
- Ensure Pods, Deployments, Nodes, and Events load correctly.

### 2. Verify Prometheus Scrapers
- Navigate to `http://localhost/prometheus/targets`.
- Ensure all 4 scrape targets (`kubernetes-apiservers`, `kubernetes-cadvisor`, `kubernetes-nodes`, `kubernetes-service-endpoints`) show as **UP**.
- Execute a query on `http://localhost/prometheus/graph` like `up` to verify data returns.

### 3. Verify Grafana Dashboard
- Navigate to `http://localhost/grafana/`.
- The Grafana page will directly load the **KubePulse Kubernetes Monitoring** dashboard pre-configured with Gauges and Timeseries panels for CPU, Memory, Pod count, and Network throughput rates.

### 4. Trigger Webhook Mock Alert
You can simulate a warning alert to check Alertmanager webhook notifications:
```powershell
Invoke-RestMethod -Uri "http://localhost/api/alerts/webhook" -Method Post -ContentType "application/json" -Body '{"status":"firing","alerts":[{"labels":{"alertname":"PodCrashLooping","namespace":"kubepulse","pod":"test-error-pod","severity":"critical"},"annotations":{"summary":"Pod test-error-pod is crashlooping","description":"Container restarting repeatedly"},"startsAt":"2026-06-19T10:00:00Z"}]}'
```
- Open the Streamlit dashboard `http://localhost/` and check the **Incidents** tab. You should see the custom warning incident card showing active status!
