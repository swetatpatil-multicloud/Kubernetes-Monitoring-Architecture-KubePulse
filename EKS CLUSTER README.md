
# ⚡ KubePulse: Kubernetes Cluster Health Checker & Observability Platform on AWS EKS 
### A system that continuously checks the “pulse” of your Kubernetes cluster and applications.

[![AWS](https://img.shields.io/badge/AWS-EKS-orange?logo=amazonaws&style=flat-square)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.33-blue?logo=kubernetes&style=flat-square)](https://kubernetes.io)
[![Docker](https://img.shields.io/badge/Container-Docker-blue?logo=docker&style=flat-square)](https://docker.com)
[![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-orange?logo=prometheus&style=flat-square)](https://prometheus.io)
[![Grafana](https://img.shields.io/badge/Dashboard-Grafana-orange?logo=grafana&style=flat-square)](https://grafana.com)
[![Alertmanager](https://img.shields.io/badge/Alerting-Alertmanager-red?logo=prometheus&style=flat-square)](https://prometheus.io/docs/alerting/latest/alertmanager/)
[![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&style=flat-square)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/UI-React-61DAFB?logo=react&style=flat-square)](https://react.dev/)
[![License](https://img.shields.io/badge/License-MIT-success?style=flat-square)](LICENSE)

---

# Project Overview

**KubePulse** is a production-style Kubernetes observability platform deployed on **Amazon Elastic Kubernetes Service (AWS EKS)** that provides centralized monitoring, visualization, health analysis, and alert management for Kubernetes workloads.

The platform combines **React**, **FastAPI**, **Prometheus**, **Grafana**, and **Alertmanager** into a unified monitoring dashboard capable of displaying real-time Kubernetes cluster information while collecting metrics, evaluating alert rules, and receiving Alertmanager webhook notifications.

Unlike a traditional monitoring stack where Prometheus and Grafana are used independently, KubePulse integrates Kubernetes APIs, Prometheus metrics, Alertmanager notifications, and a custom backend into a single operational dashboard.

The project was completely developed using **Visual Studio Code** and deployed onto an **AWS EKS cluster** using Docker containers and Kubernetes manifests.

---

# Project Objectives

The primary objectives of KubePulse are:

- Deploy a production-ready Kubernetes monitoring platform on AWS EKS.
- Monitor Kubernetes cluster health in real time.
- Visualize cluster resources through a custom React dashboard.
- Collect infrastructure metrics using Prometheus.
- Display historical and live metrics through Grafana.
- Receive alerts from Alertmanager using custom webhooks.
- Provide a centralized backend API for Kubernetes resource management.
- Demonstrate production deployment practices on AWS Cloud.

---

# Technology Stack

| Layer | Technology |
|---------|------------|
| Cloud Platform | AWS |
| Kubernetes | Amazon EKS |
| Container Runtime | Docker |
| IDE | Visual Studio Code |
| Frontend | React.js |
| Backend | FastAPI (Python) |
| Monitoring | Prometheus |
| Visualization | Grafana |
| Alerting | Alertmanager |
| Metrics Exporters | kube-state-metrics, cAdvisor |
| Kubernetes Client | Python Kubernetes Client |
| Deployment | Kubernetes YAML Manifests |

---

# AWS Architecture

The entire monitoring platform is deployed inside an Amazon EKS cluster.

```
                    Internet
                        │
                        │
                AWS Load Balancer
                        │
        ┌───────────────┴───────────────┐
        │                               │
        │          AWS VPC              │
        │                               │
        │      Amazon EKS Cluster       │
        │                               │
        │ ┌───────────────────────────┐ │
        │ │ Worker Node               │ │
        │ │                           │ │
        │ │ React Frontend            │ │
        │ │ FastAPI Backend           │ │
        │ │ Prometheus                │ │
        │ │ Alertmanager              │ │
        │ │ Grafana                   │ │
        │ │ kube-state-metrics        │ │
        │ │ cAdvisor                  │ │
        │ └───────────────────────────┘ │
        │                               │
        └───────────────────────────────┘
```

---

# High-Level Architecture

```text
                 +-----------------------+
                 |    React Dashboard    |
                 +-----------+-----------+
                             |
                      REST API Calls
                             |
                             ▼
                +-------------------------+
                |     FastAPI Backend     |
                +-----------+-------------+
                            |
        -----------------------------------------------
        |              |              |               |
        ▼              ▼              ▼               ▼
 Kubernetes API   Prometheus     Alertmanager     Grafana
        |              |              |               |
        |              |              |               |
        ▼              ▼              ▼               ▼
 Nodes / Pods     Metrics DB     Webhook Events   Dashboards
```

---

# Production Workflow

```
Developer

     │

     ▼

Visual Studio Code

     │

     ▼

Docker Build

     │

     ▼

Docker Image

     │

     ▼

Amazon EKS

     │

     ▼

Kubernetes Pods

     │

     ▼

Prometheus Metrics

     │

     ▼

Alertmanager

     │

     ▼

FastAPI Webhook

     │

     ▼

React Dashboard
```

---

# AWS Infrastructure

The project was deployed on AWS using the following infrastructure.

| Resource | Purpose |
|-----------|----------|
| AWS Account | Cloud Infrastructure |
| Amazon VPC | Networking |
| Public Subnets | Worker Node Connectivity |
| Internet Gateway | Internet Access |
| Amazon EKS | Managed Kubernetes Cluster |
| EC2 Worker Nodes | Run Kubernetes Workloads |
| IAM Roles | Cluster Authentication |
| Security Groups | Network Security |
| Elastic Load Balancer | External Access |

---

# Amazon EKS Cluster

The monitoring platform runs entirely inside an Amazon Elastic Kubernetes Service (EKS) cluster.

The EKS cluster provides:

- Managed Kubernetes Control Plane
- Automatic Kubernetes API management
- Worker node orchestration
- Service discovery
- High availability
- Rolling deployments
- Self-healing workloads

Worker nodes host the following Kubernetes workloads:

- Frontend
- Backend
- Prometheus
- Grafana
- Alertmanager
- kube-state-metrics

---

# Development Environment

The complete application was developed using **Visual Studio Code**.

Development workflow:

1. Source code edited in VS Code
2. Docker images built locally
3. Images pushed to Kubernetes
4. Deployments updated using kubectl
5. Services verified on AWS EKS
6. Monitoring stack validated
7. Alert pipeline tested

---

# Docker Workflow

Every application component is containerized.

```
Source Code

     │

     ▼

Docker Build

     │

     ▼

Docker Image

     │

     ▼

Kubernetes Deployment

     │

     ▼

Running Pod
```

Example build commands:

```bash
docker build -t kubepulse-backend .
docker build -t kubepulse-frontend .
```

Deploy updated images:

```bash
kubectl rollout restart deployment kubepulse-backend -n kubepulse

kubectl rollout restart deployment kubepulse-frontend -n kubepulse
```

---

# Project Structure

```text
KubePulse
│
├── frontend/
│   ├── src/
│   ├── components/
│   ├── pages/
│   └── services/
│
├── backend/
│   ├── app/
│   │    ├── alerts.py
│   │    ├── k8s_client.py
│   │    ├── main.py
│   │    └── metrics.py
│   │
│   ├── Dockerfile
│   └── requirements.txt
│
├── kubernetes/
│   ├── backend-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── prometheus.yaml
│   ├── grafana.yaml
│   ├── alertmanager.yaml
│   └── rbac.yaml
│
├── diagrams/
│
└── README.md
```

---

# Major Components

The platform consists of six primary services:

- React Frontend
- FastAPI Backend
- Kubernetes API
- Prometheus
- Grafana
- Alertmanager

Each component performs a dedicated responsibility while working together as an integrated observability platform.

---

# React Frontend

The React frontend serves as the primary user interface for KubePulse. It provides real-time visibility into the Kubernetes cluster by consuming REST APIs exposed by the FastAPI backend.

Unlike directly querying Kubernetes or Prometheus, the frontend communicates only with the backend service, making the architecture secure, modular, and easier to maintain.

The dashboard is designed to present cluster information in a clean and intuitive manner, allowing operators to quickly identify unhealthy workloads, monitor infrastructure health, and review active alerts.

---

## Frontend Responsibilities

The React application provides:

- Cluster Overview Dashboard
- Node Health Monitoring
- Namespace Information
- Pod Status Monitoring
- Deployment Status
- Kubernetes Events
- Active Alert List
- Historical Alerts
- Grafana Dashboard Integration
- Auto-refresh Dashboard
- Responsive User Interface

---

## Frontend Architecture

```text
                 React Frontend
                        │
                        │ REST API
                        ▼
               FastAPI Backend
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
 Kubernetes API   Prometheus API   Alert Store
```

---

## Dashboard Overview

The dashboard displays the following information:

### Cluster Summary

- Total Nodes
- Healthy Nodes
- Namespaces
- Running Pods
- Failed Pods
- Deployments
- Active Alerts

---

### Kubernetes Resources

Users can browse:

- Nodes
- Pods
- Namespaces
- Deployments
- Services
- Events

---

### Monitoring

The dashboard also displays:

- CPU Usage
- Memory Usage
- Pod Health
- Deployment Availability
- Cluster Events
- Alert Notifications

---

# FastAPI Backend

The backend is implemented using **FastAPI** and acts as the central API layer for the platform.

Instead of allowing the frontend to communicate directly with Kubernetes or Prometheus, all requests pass through the backend.

This architecture improves:

- Security
- Maintainability
- Scalability
- Authentication
- RBAC
- Future extensibility

---

## Backend Responsibilities

The backend is responsible for:

- Reading Kubernetes resources
- Querying Prometheus metrics
- Receiving Alertmanager webhooks
- Storing active alerts
- Maintaining alert history
- Returning cluster summary
- Providing REST APIs to the frontend

---

## Backend Architecture

```text
             React Dashboard
                     │
                     ▼
              FastAPI Backend
                     │
     ┌───────────────┼────────────────┐
     ▼               ▼                ▼
 Kubernetes API  Prometheus API  Alertmanager
```

---

## Backend Folder Structure

```text
backend/

├── app/
│
├── alerts.py
├── k8s_client.py
├── main.py
├── metrics.py
├── Dockerfile
├── requirements.txt
└── README.md
```

---

## Core Backend Modules

### main.py

Acts as the FastAPI application entry point.

Responsibilities:

- API registration
- Alert webhook endpoint
- Cluster APIs
- Health endpoint
- CORS configuration

---

### alerts.py

Maintains application alert state.

Responsibilities:

- Store active alerts
- Store alert history
- Remove resolved alerts
- Deduplicate alerts
- Expose alert APIs

---

### k8s_client.py

Responsible for Kubernetes API communication.

Uses:

- Kubernetes Python Client

Retrieves:

- Nodes
- Pods
- Deployments
- Events
- Namespaces

---

# Backend REST APIs

The backend exposes the following REST endpoints.

| Method | Endpoint | Description |
|----------|---------------------------|------------------------------|
| GET | /health | Backend health |
| GET | /api/cluster/summary | Cluster overview |
| GET | /api/cluster/nodes | Node list |
| GET | /api/cluster/pods | Pod list |
| GET | /api/cluster/deployments | Deployment list |
| GET | /api/cluster/namespaces | Namespace list |
| GET | /api/cluster/events | Cluster events |
| GET | /api/cluster/alerts | Active alerts |
| POST | /api/alerts/webhook | Alertmanager webhook |

---

# Kubernetes Integration

The backend communicates directly with the Kubernetes API Server using the Kubernetes Python client.

Since the backend runs inside Amazon EKS, it authenticates using an in-cluster ServiceAccount rather than a kubeconfig file.

This follows Kubernetes production best practices.

---

## Kubernetes Objects Retrieved

The backend retrieves:

- Nodes
- Namespaces
- Pods
- Deployments
- ReplicaSets
- Events

Additional resources can be added in future releases.

---

## Kubernetes Communication Flow

```text
React Dashboard

       │

       ▼

FastAPI Backend

       │

       ▼

Kubernetes Python Client

       │

       ▼

Kubernetes API Server

       │

       ▼

Amazon EKS Cluster
```

---

# Kubernetes Resource Monitoring

The backend continuously retrieves Kubernetes resource information including:

## Nodes

Displays:

- Node Name
- Status
- Kubernetes Version
- Roles
- Internal IP

---

## Pods

Displays:

- Namespace
- Pod Name
- Status
- Restart Count
- Node Assignment

---

## Deployments

Displays:

- Desired Replicas
- Available Replicas
- Ready Replicas
- Deployment Status

---

## Events

Displays:

- Type
- Reason
- Namespace
- Object
- Timestamp
- Message

---

# Prometheus Monitoring

Prometheus is responsible for collecting and storing time-series metrics from the Kubernetes cluster.

It periodically scrapes exporters running inside the cluster and stores metrics in its internal TSDB database.

---

## Metrics Sources

Prometheus scrapes metrics from:

- kube-state-metrics
- cAdvisor
- Kubernetes API
- Application Metrics
- Node Metrics

---

## Prometheus Architecture

```text
              Prometheus

                   │

       ┌───────────┼───────────┐

       ▼           ▼           ▼

kube-state    cAdvisor     Applications

 metrics

```

---

## Metrics Collected

Infrastructure metrics include:

- CPU Usage
- Memory Usage
- Disk Usage
- Network Usage
- Node Status
- Pod Status
- Deployment Availability

---

## Prometheus Targets

| Target | Purpose |
|-------------|-----------------------------|
| kube-state-metrics | Kubernetes Objects |
| cAdvisor | Container Metrics |
| Kubernetes API | Cluster Information |
| Backend Metrics | Custom Metrics |

---

# kube-state-metrics

kube-state-metrics exposes Kubernetes object information in Prometheus format.

Unlike cAdvisor, it focuses on Kubernetes resource states rather than infrastructure utilization.

It provides metrics for:

- Deployments
- Pods
- Nodes
- ReplicaSets
- StatefulSets
- Services
- Namespaces

---

# cAdvisor

cAdvisor provides container-level resource utilization metrics.

These metrics include:

- CPU Usage
- Memory Usage
- Filesystem Usage
- Network Statistics

These metrics are used by Prometheus to generate alert rules.

---

# Grafana Integration

Grafana connects directly to Prometheus and provides visualization dashboards.

The React dashboard focuses on operational status, while Grafana provides advanced metric exploration.

---

## Grafana Dashboards

Available dashboards include:

- Cluster Overview
- Node Resources
- Pod Resources
- Namespace Metrics
- CPU Usage
- Memory Usage
- Network Traffic
- Container Utilization

---

## Grafana Data Flow

```text
Prometheus

      │

      ▼

Grafana

      │

      ▼

Interactive Dashboards
```

---

## Why Both React and Grafana?

The React dashboard provides:

- Operational status
- Kubernetes resources
- Alerts
- Cluster overview

Grafana provides:

- Time-series analysis
- Historical trends
- Interactive charts
- Advanced PromQL visualizations

Together they create a complete observability platform.

---

---

# 6. Monitoring & Observability Stack

KubePulse uses a cloud-native observability stack to continuously monitor Kubernetes resources, collect metrics, generate alerts, and provide real-time operational visibility.

The monitoring stack is deployed directly inside the Amazon EKS cluster.

### Components

| Component | Purpose |
|------------|----------|
| Prometheus | Collects and stores Kubernetes metrics |
| kube-state-metrics | Exposes Kubernetes object metrics |
| cAdvisor | Container CPU and memory metrics |
| Kubelet Metrics | Node and Pod resource metrics |
| Alertmanager | Processes Prometheus alerts and forwards notifications |
| Grafana | Visualizes metrics through dashboards |
| FastAPI Backend | Provides REST APIs to the frontend |
| React Dashboard | Displays cluster health in real time |

---

# 7. Prometheus Metrics Collection

Prometheus continuously scrapes metrics from Kubernetes components using pull-based scraping.

### Scrape Targets

- Kubernetes API Server
- kube-state-metrics
- kubelet
- cAdvisor
- Node Metrics
- FastAPI Backend
- Custom Application Metrics

Example metrics collected:

- CPU Usage
- Memory Usage
- Disk Usage
- Pod Restart Count
- Running Pods
- Failed Pods
- Deployment Availability
- Node Health
- Network Traffic

---

## Example Prometheus Configuration

```yaml
scrape_configs:

- job_name: kubernetes-cadvisor

  kubernetes_sd_configs:
    - role: node

  metrics_path: /metrics/cadvisor

  scheme: https

  tls_config:
    insecure_skip_verify: true
```

---

# 8. Prometheus Alert Rules

Prometheus evaluates alert rules continuously.

Example production alert rules:

### Target Down

```yaml
alert: PrometheusTargetMissing

expr: up == 0

for: 1m

labels:
  severity: critical

annotations:
  summary: "Target is down"
```

---

### High CPU Usage

```yaml
alert: HighCPUUsage

expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8

for: 2m

labels:
  severity: warning
```

---

### High Memory Usage

```yaml
alert: HighMemoryUsage

expr: container_memory_usage_bytes > 500000000

for: 2m

labels:
  severity: warning
```

---

# 9. Alertmanager Integration

Alertmanager receives alerts from Prometheus and routes them to the KubePulse Backend through a webhook.

## Alert Flow

```text
Prometheus

      │

      ▼

Alertmanager

      │

Webhook

      │

      ▼

FastAPI Backend

      │

Stores Active Alerts

      │

      ▼

React Dashboard
```

---

## Alertmanager Configuration

Example receiver:

```yaml
receivers:

- name: kubepulse-webhook

  webhook_configs:

  - url: http://kubepulse-backend:8000/api/alerts/webhook
```

The backend receives webhook payloads and stores:

- Active Alerts
- Alert History
- Alert Severity
- Alert Status
- Namespace
- Pod Information

---

# 10. Grafana Dashboards

Grafana is integrated with Prometheus to visualize cluster metrics.

Available dashboards include:

- Kubernetes Cluster Overview
- Node Resource Usage
- CPU Utilization
- Memory Consumption
- Deployment Status
- Pod Health
- Namespace Usage
- Container Metrics
- Alert Status

Grafana connects directly to Prometheus using PromQL.

---

# 11. Backend REST APIs

The FastAPI backend exposes Kubernetes and monitoring data through REST APIs.

### Cluster APIs

| Endpoint | Description |
|-----------|-------------|
| /health | Backend health check |
| /api/cluster/summary | Cluster summary |
| /api/cluster/nodes | Node details |
| /api/cluster/namespaces | Namespace information |
| /api/cluster/pods | Pod details |
| /api/cluster/deployments | Deployment information |
| /api/cluster/events | Kubernetes events |
| /api/cluster/alerts | Active alerts |

---

## Alert Webhook API

Alertmanager sends alerts to:

```
POST

/api/alerts/webhook
```

Payload contains:

- Alert Name
- Severity
- Namespace
- Pod
- Description
- Status
- Start Time
- End Time

The backend processes:

- firing alerts
- resolved alerts
- alert history
- active alert list

---

# 12. Dashboard Features

The React dashboard provides a centralized operational view of the Kubernetes cluster.

Dashboard cards include:

- Total Nodes
- Ready Nodes
- Total Pods
- Running Pods
- Failed Pods
- Deployments
- Namespaces
- Active Alerts

Additional pages include:

### Nodes

Displays:

- Node Name
- Status
- CPU Capacity
- Memory Capacity
- Kubernetes Version

---

### Pods

Displays:

- Pod Name
- Namespace
- Status
- Restarts
- Node

---

### Deployments

Displays:

- Desired Replicas
- Available Replicas
- Updated Replicas
- Deployment Status

---

### Events

Displays Kubernetes events including:

- Failed Scheduling
- Pod Restarts
- Image Pull Errors
- Scaling Events
- Node Events

---

### Alerts

Displays

- Alert Name
- Severity
- Status
- Namespace
- Summary
- Description
- Timestamp

---

# 13. Kubernetes RBAC

The backend authenticates to the Kubernetes API using a ServiceAccount.

Permissions include:

- get
- list
- watch

Resources:

- Pods
- Nodes
- Namespaces
- Deployments
- Events

This follows Kubernetes least-privilege security practices.

---

# 14. Security

Production security best practices implemented include:

- IAM-based EKS authentication
- Kubernetes RBAC
- Service Accounts
- Kubernetes Secrets
- ConfigMaps
- Private Cluster Communication
- ClusterIP Services
- Read-only Kubernetes API access
- Namespace Isolation

Future improvements:

- OIDC Authentication
- External Secrets
- HashiCorp Vault
- Network Policies

---

# 15. High Availability

Amazon EKS provides a highly available Kubernetes control plane.

Worker nodes are distributed across multiple Availability Zones.

Benefits include:

- Automatic Control Plane Recovery
- Multi-AZ Worker Nodes
- Managed Kubernetes
- Automatic API Server Scaling
- Cloud-native Networking

---

# 16. Production Architecture

```text
                        Users

                          │

                          ▼

                  React Dashboard

                          │

                    REST API Calls

                          │

                          ▼

                 FastAPI Backend Pod

          (Runs inside Amazon EKS)

          │                 │

          │                 │

          ▼                 ▼

 Kubernetes API      Alertmanager Webhook

          │                 ▲

          │                 │

          ▼                 │

     Amazon EKS Cluster      │

          │                 │

          ▼                 │

      kube-state-metrics     │

      kubelet / cAdvisor     │

          │                 │

          ▼                 │

       Prometheus ----------┘

          │

          ▼

       Grafana
```

---

# 17. Benefits of Amazon EKS Deployment

Deploying KubePulse on Amazon EKS provides several production-grade advantages:

- Managed Kubernetes Control Plane
- High Availability
- Multi-AZ Deployment
- IAM Integration
- Auto Scaling Support
- AWS Load Balancer Integration
- CloudWatch Compatibility
- Secure Networking
- Production-grade Monitoring
- Simplified Cluster Management

---

# 18. Project Highlights

✔ Built on Amazon EKS

✔ Kubernetes-native architecture

✔ Production-ready deployment

✔ FastAPI backend

✔ React frontend

✔ Prometheus monitoring

✔ Grafana dashboards

✔ Alertmanager webhook integration

✔ Real-time Kubernetes monitoring

✔ REST API architecture

✔ Containerized using Docker

✔ Kubernetes RBAC

✔ Production cloud deployment on AWS

---

---

# 14. Deployment Architecture (AWS EKS)

KubePulse is designed as a cloud-native platform and is deployed on **Amazon Elastic Kubernetes Service (Amazon EKS)** to provide scalability, reliability, and high availability.

The entire monitoring stack runs inside the Kubernetes cluster while exposing only the frontend and monitoring dashboards through Kubernetes Services or AWS Load Balancers.

## Deployment Architecture

```text
                    ┌────────────────────────────┐
                    │        End Users           │
                    │    Browser / DevOps Team   │
                    └─────────────┬──────────────┘
                                  │
                          AWS Application
                           Load Balancer
                                  │
                     Kubernetes Ingress / Service
                                  │
             ┌────────────────────┴────────────────────┐
             │                                         │
     React Dashboard                          Grafana Dashboard
             │                                         │
             └────────────────────┬────────────────────┘
                                  │
                           FastAPI Backend
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
      Kubernetes API        Prometheus API      Alertmanager
             │                    │                    │
             │                    │                    │
     Amazon EKS Cluster      Metrics Storage     Webhook API
             │                    │                    │
             └───────────────┬────┴────────────────────┘
                             │
                      Worker Node Metrics
                             │
      kube-state-metrics • cAdvisor • Node Exporter
```

---

## AWS Infrastructure

The project was deployed on AWS using the following managed services.

| AWS Service | Purpose |
|-------------|----------|
| Amazon EKS | Managed Kubernetes Cluster |
| Amazon EC2 | Kubernetes Worker Nodes |
| Elastic Load Balancer | Exposes frontend and monitoring services |
| Amazon VPC | Cluster networking |
| IAM Roles | Secure Kubernetes permissions |
| Amazon ECR *(Optional)* | Docker Image Registry |
| CloudShell / VS Code | Deployment and management |
| kubectl | Kubernetes management |
| Docker Desktop | Image building |

---

## Amazon EKS Cluster

The Kubernetes cluster consists of:

- Managed Control Plane
- EC2 Worker Nodes
- Kubernetes Services
- Deployments
- ReplicaSets
- ConfigMaps
- Secrets
- RBAC Policies

The FastAPI backend communicates with the Kubernetes API Server using an in-cluster ServiceAccount.

---

# 15. Deployment Process

The application is containerized using Docker and deployed onto Amazon EKS.

## Step 1 – Build Docker Images

```bash
docker build -t kubepulse-backend .
docker build -t kubepulse-frontend .
```

---

## Step 2 – Push Images

Images can be pushed to:

- Docker Hub
- Amazon ECR

Example:

```bash
docker tag kubepulse-backend <repository>/kubepulse-backend:v1
docker push <repository>/kubepulse-backend:v1
```

---

## Step 3 – Deploy to Amazon EKS

```bash
kubectl apply -f k8s/
```

This deploys:

- Backend Deployment
- Frontend Deployment
- Prometheus
- Grafana
- Alertmanager
- Services
- ConfigMaps
- RBAC
- Monitoring Stack

---

## Step 4 – Verify Deployments

```bash
kubectl get pods

kubectl get svc

kubectl get deployments
```

Example:

```
NAME                               READY
kubepulse-backend                  1/1
kubepulse-frontend                 1/1
prometheus                         1/1
grafana                            1/1
alertmanager                       1/1
node-exporter                      Running
kube-state-metrics                 Running
```

---

## Step 5 – Verify Monitoring

Verify Prometheus targets.

```
Status → Targets
```

Ensure:

- Kubernetes API
- kube-state-metrics
- Node Exporter
- cAdvisor

are all UP.

---

## Step 6 – Verify Alertmanager

Open Alertmanager.

```
Status → Alerts
```

Verify active alerts.

---

## Step 7 – Verify Backend API

Example:

```
GET /health

GET /api/cluster/summary

GET /api/cluster/pods

GET /api/cluster/events

GET /api/cluster/alerts
```

---

# 16. Security Architecture

KubePulse follows Kubernetes security best practices.

## Kubernetes RBAC

Backend access is controlled through:

- ServiceAccount
- ClusterRole
- ClusterRoleBinding

Only read-only permissions are granted for monitoring resources.

Example permissions:

```
pods
nodes
deployments
services
events
namespaces
```

---

## Kubernetes Secrets

Sensitive configuration is stored using Kubernetes Secrets.

Examples:

- API Tokens
- SMTP Credentials
- Slack Webhooks
- Database Passwords

Secrets are never hardcoded.

---

## IAM Integration

Amazon EKS integrates with AWS IAM.

Benefits include:

- Secure cluster authentication
- IAM Role based access
- Fine-grained permissions
- AWS best practices

---

## Network Security

Traffic is restricted using:

- Kubernetes Services
- Internal Cluster Networking
- Security Groups
- VPC Isolation

---

## Container Security

Docker containers are:

- Lightweight
- Stateless
- Immutable
- Independently deployable

---

# 17. CI/CD Pipeline

The project is designed for automated deployment using CI/CD.

Example pipeline:

```text
GitHub
   │
   ▼
GitHub Actions
   │
   ▼
Docker Build
   │
   ▼
Docker Hub / Amazon ECR
   │
   ▼
Amazon EKS Deployment
```

Pipeline stages include:

- Code Checkout
- Dependency Installation
- Docker Build
- Image Push
- Kubernetes Deployment
- Rollout Verification

---

# 18. Production Hardening

The following best practices are recommended for production deployments.

## High Availability

Deploy multiple replicas.

```yaml
replicas: 3
```

---

## Resource Limits

Each workload should define:

- CPU Requests
- CPU Limits
- Memory Requests
- Memory Limits

---

## Health Checks

Use:

- Liveness Probe
- Readiness Probe

to ensure application availability.

---

## Persistent Storage

Use Persistent Volumes for:

- Grafana Dashboards
- Prometheus TSDB

---

## HTTPS

Ingress should terminate TLS using:

- AWS ACM
- NGINX Ingress
- Traefik

---

## Monitoring the Monitoring Stack

The monitoring stack itself should also be monitored.

Track:

- Prometheus Health
- Grafana Health
- Alertmanager Availability
- Backend API Health

---

# 19. Future Enhancements

Future versions of KubePulse can include:

- Multi-Cluster Monitoring
- Helm Chart Packaging
- GitOps with Argo CD
- Loki Log Aggregation
- Tempo Distributed Tracing
- OpenTelemetry Integration
- AI-Powered Incident Summaries
- Slack and Microsoft Teams Notifications
- Automated Pod Restart Suggestions
- Self-Healing Workflows
- Predictive Capacity Planning
- Cost Optimization Dashboards
- SLO / SLA Monitoring
- Kubernetes Upgrade Advisor

---

# 20. Project Highlights

✔ Built using **Amazon EKS**

✔ Cloud-native Kubernetes monitoring platform

✔ FastAPI backend for Kubernetes API integration

✔ React frontend dashboard

✔ Prometheus metrics collection

✔ Grafana visualization

✔ Alertmanager webhook integration

✔ Real-time cluster monitoring

✔ Kubernetes Events monitoring

✔ Namespace, Pod and Deployment visibility

✔ Alert history and active alerts

✔ Dockerized microservices

✔ Kubernetes-native deployment

✔ Production-ready architecture

✔ Designed with scalability and extensibility in mind

---

# 21. Conclusion

KubePulse demonstrates the implementation of a production-style Kubernetes observability platform deployed on **Amazon Elastic Kubernetes Service (Amazon EKS)**.

The platform combines Kubernetes resource monitoring, Prometheus-based metrics collection, Grafana dashboards, and Alertmanager-driven notifications into a unified web application. By leveraging FastAPI for backend services and React for the frontend, KubePulse provides real-time visibility into cluster health, application status, and infrastructure performance.

This project highlights practical DevOps and Cloud Engineering skills, including containerization with Docker, orchestration using Kubernetes, cloud deployment on AWS EKS, infrastructure monitoring, alert management, and production-ready architectural design.

It serves as a strong foundation for enterprise-scale Kubernetes operations and can be extended with GitOps, AI-assisted incident response, centralized logging, distributed tracing, and multi-cluster management to meet modern cloud-native operational requirements.

---

# Screenshots:

## Grafana dashboard:

<img width="1905" height="1037" alt="image" src="https://github.com/user-attachments/assets/fa6929a7-562a-46d2-b5e3-ade9e610fe84" />

## Alert Manager:

<img width="1757" height="563" alt="image" src="https://github.com/user-attachments/assets/e4e918e4-4d01-4454-8eb3-10306f138c48" />
<img width="1895" height="1078" alt="image" src="https://github.com/user-attachments/assets/05e2d9e1-9dd2-468d-a206-b93ce399c0ee" />

## AlertManager on local host :

<img width="1911" height="1090" alt="image" src="https://github.com/user-attachments/assets/50b1e920-13f3-4b64-a69b-bd99e30fa44c" />

## AlertManager through aws:

<img width="1891" height="1063" alt="image" src="https://github.com/user-attachments/assets/56ab6857-cd09-4578-a48f-40bb5986234d" />

## Prometheus:

### http://afc913fcab5f84cdbb42f27d4e4ac264-1293831530.ap-south-1.elb.amazonaws.com/prometheus

<img width="1897" height="1037" alt="image" src="https://github.com/user-attachments/assets/778cdda1-ae85-45a4-859a-1178fbcb636d" />

<img width="1891" height="1048" alt="image" src="https://github.com/user-attachments/assets/c8d99266-609a-40cb-a884-91a393b389e0" />

<img width="1913" height="552" alt="image" src="https://github.com/user-attachments/assets/7e4c0dc6-d387-4101-9f91-959783cf96e0" />

## Prometheus Rule:
<img width="1902" height="1041" alt="image" src="https://github.com/user-attachments/assets/27f7e50a-1306-4407-a03a-f2322abff33d" />

## Prometheus Alerts:

<img width="1907" height="1081" alt="image" src="https://github.com/user-attachments/assets/dec4d72b-4ae7-4faa-848d-5653d4adf41a" />

<img width="1900" height="1088" alt="image" src="https://github.com/user-attachments/assets/02588934-69ba-4cd9-adb3-29384d990d49" />

## Alertmanager:

<img width="1915" height="1068" alt="image" src="https://github.com/user-attachments/assets/cd71fc73-f656-4326-a502-6814779e60c3" />

## Grafana dashboard through AWS EKS:

<img width="1911" height="1085" alt="image" src="https://github.com/user-attachments/assets/e3dd4329-d2f1-4bc3-b2d4-1c0e5fecf0ab" />

## KubePulse dashboard: 

<img width="1906" height="1085" alt="image" src="https://github.com/user-attachments/assets/70807f99-1288-4631-8b89-febf6f93d671" />



=======
---

## 11. Monitoring Stack

### `k8s/monitoring/prometheus.yaml`
Deploys Prometheus to scrape time-series metrics from nodes and containers.
* **Scraper Configs**:
  1. `kubernetes-apiservers`: Monitors API server latency and loads.
  2. `kubernetes-nodes`: Gathers operating system usage metrics (CPU, Memory, Disk, Network) from each node.
  3. `kubernetes-cadvisor`: Fetches system usage metrics for every container running on the node.

### `k8s/monitoring/grafana.yaml`
Deploys Grafana for metric visualization.
* **Automated Data Sources**: Uses a ConfigMap to pre-configure connection settings for Prometheus and Alertmanager data sources automatically on start.
* **Default Dashboard Provisioning**: Automatically loads a default cluster monitoring dashboard JSON file from a ConfigMap, ensuring dashboards are populated immediately at `http://localhost/grafana` without manual configuration.

### `k8s/monitoring/alertmanager.yaml`
Alertmanager processes alert signals from Prometheus and routes them to alert endpoints.
* **Receivers**: A webhook receiver routes alerts directly to the FastAPI API backend at `/api/alerts/webhook`, allowing the dashboard to display critical alerts instantly.

---

## 12. Docker Containers

KubePulse uses containerized execution environments to ensure consistent behavior across environments:

1. **Backend Container**:
   * Runs Python 3.12-slim.
   * Exposes port `8000`.
   * Executable: `uvicorn app.main:app`.
2. **Frontend Container**:
   * Runs Streamlit.
   * Exposes port `8501` (mapped to `80` by Ingress).
>>>>>>> 92368514b81dfe6e466fae3beb3c4a41ff7ab21b

---

## 13. Deployment & Verification Workflow

Follow these steps to deploy KubePulse to a local Kind cluster:

### Step 1: Create the Kind Cluster
Use the Kind configuration file to create the cluster with port mappings:
```powershell
kind create cluster --config kind-config.yaml --name kubepulse
```

### Step 2: Deploy the Ingress Controller
Apply the NGINX Ingress Controller manifest and wait for it to become ready:
```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for Ingress pods to be running
kubectl wait --namespace ingress-nginx `
  --for=condition=ready pod `
  --selector=app.kubernetes.io/component=controller `
  --timeout=120s
```

### Step 3: Create the Namespace & RBAC Roles
```powershell
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/rbac.yaml
```

### Step 4: Build and Load Docker Images
Build images locally and load them directly into the Kind cluster nodes (eliminating the need for an external container registry):
```powershell
# Build Backend
docker build -t kubepulse-backend:latest ./backend
kind load docker-image kubepulse-backend:latest --name kubepulse

# Build Frontend
docker build -t kubepulse-frontend:latest ./frontend
kind load docker-image kubepulse-frontend:latest --name kubepulse
```

### Step 5: Deploy the Workloads
Deploy backend, frontend, and database services:
```powershell
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

### Step 6: Deploy the Monitoring Stack
Apply Prometheus, Grafana, and Alertmanager configurations:
```powershell
kubectl apply -f k8s/monitoring/prometheus.yaml
kubectl apply -f k8s/monitoring/grafana.yaml
kubectl apply -f k8s/monitoring/alertmanager.yaml
```

### Step 7: Apply Ingress Routes
Activate external HTTP access pathways:
```powershell
kubectl apply -f k8s/ingress.yaml
```

### Step 8: Verify the Deployment
Verify that the services are accessible:

* **Dashboard**: `http://localhost/` (Streamlit UI loads)
* **Backend API**: `http://localhost/api/cluster/summary` (returns JSON status output)
* **Grafana**: `http://localhost/grafana` (access Grafana dashboards)
* **Prometheus**: `http://localhost/prometheus` (view metrics targets and run queries)

---

## 14. DevOps Concepts Demonstrated

This project showcases a wide range of production-grade Cloud-Native and DevOps practices:

* **Local Kubernetes Clusters**: Kind orchestration with multi-container nodes.
* **In-Cluster & Local Client Auth**: Dynamic Kubeconfig resolution.
* **Traffic Routing**: NGINX Ingress path-based routing.
* **Kubernetes RBAC**: ClusterRoles, ClusterRoleBindings, and ServiceAccounts.
* **Health and Healing**: Readiness and liveness probes.
* **Monitoring as Code**: Auto-provisioned Grafana datasources, Grafana dashboards, and Alertmanager hooks.
* **Containerization**: Optimized Docker builds.
* **Alerting Pipelines**: Prometheus metrics alerting routed to webhook receivers.
