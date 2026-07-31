# KubePulse Project Tasks and Commands Checklist

This checklist tracks your tasks and serves as a quick reference sheet for all the installation, build, and deployment commands used in the project.

---

## Phase 1: Tooling and Local Environment Setup
- [x] **Install Docker Desktop**
  * Windows download link: https://www.docker.com/products/docker-desktop/
  * Ensure the Docker service is running on your machine.
- [x] **Install Kubectl CLI**
  ```powershell
  winget install Kubernetes.kubectl
  ```
- [x] **Install Kind (Kubernetes in Docker)**
  ```powershell
  winget install Kubernetes.kind
  ```
- [x] **Verify local Docker daemon is active**
  ```bash
  docker ps
  ```

---

## Phase 2: Cluster Provisioning
- [x] **Create Kind Cluster config with Ingress port-mappings (`kind-config.yaml`)**
- [x] **Create Kind Cluster `kubepulse`**
  ```powershell
  kind create cluster --config kind-config.yaml --name kubepulse
  ```
- [x] **Deploy raw NGINX Ingress controller manifest for Kind**
  ```powershell
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
  ```
- [x] **Wait for Ingress controller readiness**
  ```powershell
  kubectl wait --namespace ingress-nginx \
    --for=condition=ready pod \
    --selector=app.kubernetes.io/component=controller \
    --timeout=120s
  ```

---

## Phase 3: Project Architecture & Namespace Config
- [x] **Create Namespace `kubepulse`**
  ```powershell
  kubectl apply -f k8s/namespace.yaml
  ```
- [x] **Configure RBAC permissions**
  ```powershell
  kubectl apply -f k8s/rbac.yaml
  ```
- [x] **Setup workspace directories**
  Make folders: `backend/app`, `frontend`, `k8s/monitoring`.

---

## Phase 4: Backend API Development
- [x] **Write backend Python dependencies (`requirements.txt`)**
- [x] **Implement Kubernetes client wrappers (`k8s_client.py`)**
- [x] **Implement Webhook Alert Store module (`alerts.py`)**
- [x] **Implement REST Endpoints & Webhook receiver (`main.py`)**
- [x] **Wrote backend `Dockerfile`**

---

## Phase 5: Streamlit Frontend Development
- [x] **Write frontend Python dependencies (`requirements.txt`)**
- [x] **Develop Streamlit UI dashboard with full component tabs (`app.py`)**
- [x] **Flatten columns layout to prevent React rendering loops**
- [x] **Wrote frontend `Dockerfile`**

---

## Phase 6: Monitoring & Observability Stack
- [x] **Configure Prometheus scraping targets & RBAC nodes/proxy permissions (`prometheus.yaml`)**
- [x] **Configure Alertmanager Webhook receiver routing (`alertmanager.yaml`)**
- [x] **Create default Kubernetes Monitoring dashboard JSON**
- [x] **Configure Grafana datasources, dashboards provider, and dashboard ConfigMaps (`grafana.yaml`)**
- [x] **Configure Grafana anonymous viewer roles and default home dashboard path**

---

## Phase 7: Build & Deployment Commands
- [x] **Build backend Docker image**
  ```powershell
  docker build -t kubepulse-backend:latest ./backend
  ```
- [x] **Build frontend Docker image**
  ```powershell
  docker build -t kubepulse-frontend:latest ./frontend
  ```
- [x] **Load backend Docker image into Kind cluster**
  ```powershell
  kind load docker-image kubepulse-backend:latest --name kubepulse
  ```
- [x] **Load frontend Docker image into Kind cluster**
  ```powershell
  kind load docker-image kubepulse-frontend:latest --name kubepulse
  ```
- [x] **Deploy backend services and frontend manifests**
  ```powershell
  kubectl apply -f k8s/backend.yaml
  kubectl apply -f k8s/frontend.yaml
  ```
- [x] **Deploy monitoring config maps, deployments, and services**
  ```powershell
  kubectl apply -f k8s/monitoring/alertmanager.yaml
  kubectl apply -f k8s/monitoring/prometheus.yaml
  kubectl apply -f k8s/monitoring/grafana.yaml
  ```
- [x] **Deploy Ingress routing rules**
  ```powershell
  kubectl apply -f k8s/ingress.yaml
  ```
- [x] **Rollout restart command (if updating configurations)**
  ```powershell
  kubectl rollout restart deployment/grafana -n kubepulse
  kubectl rollout restart deployment/kubepulse-frontend -n kubepulse
  ```

---

## Phase 8: Verification & Verification Checks
- [x] **Verify Ingress is routing traffic correctly on port 80**
  Open: `http://localhost/`
- [x] **Verify Prometheus active scraper targets are all reporting UP**
  Open: `http://localhost/prometheus/targets`
- [x] **Verify Grafana database connection is healthy**
  Open: `http://localhost/grafana/`
- [x] **Verify Grafana auto-loads Kubernetes Monitoring dashboard on home page**
- [x] **Verify Streamlit fetches data from backend and updates in real-time**
- [x] **Simulate an active incident using PowerShell webhook POST request**
  ```powershell
  Invoke-RestMethod -Uri "http://localhost/api/alerts/webhook" -Method Post -ContentType "application/json" -Body '{"status":"firing","alerts":[{"labels":{"alertname":"PodCrashLooping","namespace":"kubepulse","pod":"test-error-pod","severity":"critical"},"annotations":{"summary":"Pod test-error-pod is crashlooping","description":"Container restarting repeatedly"},"startsAt":"2026-06-19T10:00:00Z"}]}'
  ```
