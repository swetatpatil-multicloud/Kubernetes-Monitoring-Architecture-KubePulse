# KubePulse: Production Observability & Self-Healing Setup Guide (AWS EKS)

This document provides a complete production-grade implementation guide for deploying **KubePulse** on **Amazon Elastic Kubernetes Service (Amazon EKS)**.

Unlike the local Kind deployment, this guide provisions a fully managed Kubernetes cluster on AWS and deploys the entire KubePulse observability platform using Amazon ECR, EKS, ALB Ingress Controller, Persistent Storage, Prometheus, Grafana, Alertmanager, FastAPI Backend, and Streamlit Frontend.

By following this guide from start to finish, anyone can successfully deploy the project onto AWS without referring to external documentation.

---

# Table of Contents

1. AWS Prerequisites
2. Install Required Tools
3. Configure AWS CLI
4. Create IAM User
5. Configure IAM Permissions
6. Create Amazon ECR Repositories
7. Project Directory Structure
8. Build Docker Images
9. Push Images to Amazon ECR
10. Install eksctl
11. Create EKS Cluster
12. Configure kubectl
13. Verify Cluster
14. Install Metrics Server
15. Install EBS CSI Driver
16. Install AWS Load Balancer Controller
17. Create Namespace
18. Create RBAC
19. Next Steps

---

# 1. AWS Prerequisites

Before beginning deployment, ensure the following requirements are satisfied.

## AWS Account

Create an AWS account.

https://aws.amazon.com/

The account should have permissions to create:

- VPC
- IAM
- EC2
- EKS
- ECR
- CloudFormation
- ELB
- Auto Scaling
- Route53 (Optional)
- ACM Certificates (Optional)

---

## Recommended Region

For this project, use

```

us-east-1

```

or

```

ap-south-1

```

---

## Recommended EC2 Worker Nodes

For demonstration purposes

```

t3.medium

```

For production

```

m5.large

```

or

```

m5.xlarge

```

---

## Kubernetes Version

Recommended

```

1.30

```

---

# 2. Install Required Tools

The following tools must be installed on your workstation.

---

## Install Docker Desktop

Windows

Download Docker Desktop

```

https://www.docker.com/products/docker-desktop/

```

Verify

```bash
docker --version

docker ps
```

Expected

```text
Docker version xx.xx.x

CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
```

---

## Install AWS CLI

Windows

Download

```

https://aws.amazon.com/cli/

```

Verify

```bash
aws --version
```

Example

```text
aws-cli/2.xx.x
```

---

## Install kubectl

Windows

```powershell
winget install Kubernetes.kubectl
```

Verify

```bash
kubectl version --client
```

---

## Install eksctl

Windows

```powershell
winget install Weaveworks.eksctl
```

Verify

```bash
eksctl version
```

---

## Install Helm

```powershell
winget install Helm.Helm
```

Verify

```bash
helm version
```

---

## Install Git

```powershell
winget install Git.Git
```

Verify

```bash
git --version
```

---

# 3. Configure AWS CLI

Configure your AWS account.

```bash
aws configure
```

Enter

```text
AWS Access Key ID

AWS Secret Access Key

Region

Output Format
```

Example

```text
AWS Access Key ID : AKIAxxxxxxxxxxxx

AWS Secret Access Key : ********************

Default region : us-east-1

Default output : json
```

Verify credentials

```bash
aws sts get-caller-identity
```

Expected

```json
{
  "UserId": "...",
  "Account": "...",
  "Arn": "arn:aws:iam::xxxxxxxx:user/admin"
}
```

---

# 4. Create IAM User

Open

```
AWS Console

IAM

Users

Create User
```

User Name

```
kubepulse-admin
```

Attach Policy

```
AdministratorAccess
```

For production environments, create a least-privilege policy instead of using AdministratorAccess.

Generate

- Access Key
- Secret Key

Store them securely.

---

# 5. Create Amazon ECR Repositories

KubePulse uses two Docker images.

Backend

Frontend

Create Backend Repository

```bash
aws ecr create-repository \
--repository-name kubepulse-backend
```

Create Frontend Repository

```bash
aws ecr create-repository \
--repository-name kubepulse-frontend
```

Verify

```bash
aws ecr describe-repositories
```

Example

```text
kubepulse-backend

kubepulse-frontend
```

---

# 6. Authenticate Docker to Amazon ECR

Retrieve login password

```bash
aws ecr get-login-password \
--region us-east-1 \
| docker login \
--username AWS \
--password-stdin \
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

Example

```text
Login Succeeded
```

---

# 7. Project Directory Structure

The project structure for AWS deployment is shown below.

```text
KubePulse/

│

├── implementation-aws-eks.md

├── eks-cluster.yaml

├── backend/

│   ├── Dockerfile

│   ├── requirements.txt

│   └── app/

│       ├── main.py

│       ├── alerts.py

│       ├── k8s_client.py

│       └── __init__.py

│

├── frontend/

│   ├── Dockerfile

│   ├── requirements.txt

│   └── app.py

│

├── k8s/

│   ├── namespace.yaml

│   ├── rbac.yaml

│   ├── backend.yaml

│   ├── frontend.yaml

│   ├── ingress.yaml

│   └── monitoring/

│       ├── prometheus.yaml

│       ├── grafana.yaml

│       └── alertmanager.yaml

│

└── scripts/

    ├── build.sh

    ├── deploy.sh

    └── cleanup.sh
```

---

# 8. Build Docker Images

Backend

```bash
docker build \
-t kubepulse-backend:latest \
./backend
```

Frontend

```bash
docker build \
-t kubepulse-frontend:latest \
./frontend
```

Verify

```bash
docker images
```

Expected

```text
kubepulse-backend

kubepulse-frontend
```

---

# 9. Tag Docker Images for Amazon ECR

Backend

```bash
docker tag \
kubepulse-backend:latest \
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/kubepulse-backend:latest
```

Frontend

```bash
docker tag \
kubepulse-frontend:latest \
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/kubepulse-frontend:latest
```

---

# 10. Push Images to Amazon ECR

Backend

```bash
docker push \
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/kubepulse-backend:latest
```

Frontend

```bash
docker push \
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/kubepulse-frontend:latest
```

Verify

```bash
aws ecr list-images \
--repository-name kubepulse-backend

aws ecr list-images \
--repository-name kubepulse-frontend
```

---

# 11. Create Amazon EKS Cluster

Create a file

```
eks-cluster.yaml
```

```yaml
apiVersion: eksctl.io/v1alpha5

kind: ClusterConfig

metadata:

  name: kubepulse-cluster

  region: us-east-1

  version: "1.30"

managedNodeGroups:

- name: kubepulse-workers

  instanceType: t3.medium

  desiredCapacity: 2

  minSize: 2

  maxSize: 5

  volumeSize: 50

  ssh:

    allow: false

  iam:

    withAddonPolicies:

      imageBuilder: true

      autoScaler: true

      cloudWatch: true

      albIngress: true
```

Create the cluster

```bash
eksctl create cluster \
-f eks-cluster.yaml
```

Provisioning time

```
20–30 minutes
```

During provisioning, eksctl automatically creates:

- VPC
- Internet Gateway
- Public Subnets
- Private Subnets
- Security Groups
- CloudFormation Stack
- IAM Roles
- Node Groups
- EC2 Instances
- EKS Control Plane

---

# 12. Configure kubectl

Update kubeconfig

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name kubepulse-cluster
```

Verify

```bash
kubectl config current-context
```

Expected

```text
arn:aws:eks:us-east-1:<ACCOUNT_ID>:cluster/kubepulse-cluster
```

---

# 13. Verify Cluster

Nodes

```bash
kubectl get nodes
```

Expected

```text
NAME                       STATUS   ROLES

ip-192-168-xx-xx           Ready

ip-192-168-xx-xx           Ready
```

Namespaces

```bash
kubectl get ns
```

System Pods

```bash
kubectl get pods -A
```

All system pods should be in

```
Running
```

state.

---

## End of Part 1

Part 2 will continue with:

- Metrics Server
- IAM OIDC Provider
- IAM Roles for Service Accounts (IRSA)
- AWS Load Balancer Controller
- Amazon EBS CSI Driver
- Storage Classes
- Persistent Volumes
- Namespace
- RBAC
- AWS-specific Kubernetes configuration

# 14. Install Kubernetes Metrics Server

The Kubernetes Metrics Server is required for:

- Horizontal Pod Autoscaler (HPA)
- kubectl top commands
- Resource monitoring
- Cluster autoscaling

Deploy Metrics Server using the official manifest.

```bash
kubectl apply \
-f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Wait until deployment is ready.

```bash
kubectl rollout status deployment metrics-server \
-n kube-system
```

Verify installation.

```bash
kubectl get deployment metrics-server \
-n kube-system
```

Expected

```text
NAME             READY

metrics-server   1/1
```

Verify node metrics.

```bash
kubectl top nodes
```

Expected

```text
NAME

CPU

MEMORY

ip-...

125m

18%

650Mi

42%
```

Verify pod metrics.

```bash
kubectl top pods -A
```

---

# 15. Associate IAM OIDC Provider

Amazon EKS uses IAM Roles for Service Accounts (IRSA) to securely allow Kubernetes workloads to access AWS resources.

Associate the cluster with an IAM OIDC Provider.

```bash
eksctl utils associate-iam-oidc-provider \
--cluster kubepulse-cluster \
--approve
```

Verify.

```bash
aws iam list-open-id-connect-providers
```

Expected

```text
arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/xxxxxxxx
```

---

# 16. Install Amazon EBS CSI Driver

Persistent volumes for Prometheus and Grafana will use Amazon Elastic Block Storage.

Create IAM Service Account.

```bash
eksctl create iamserviceaccount \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster kubepulse-cluster \
--role-name AmazonEKS_EBS_CSI_DriverRole \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve
```

Install EBS CSI Addon.

```bash
eksctl create addon \
--name aws-ebs-csi-driver \
--cluster kubepulse-cluster \
--service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole \
--force
```

Verify.

```bash
kubectl get pods \
-n kube-system
```

Expected

```text
ebs-csi-controller

Running

ebs-csi-node

Running
```

---

# 17. Create Default Storage Class

Verify storage classes.

```bash
kubectl get storageclass
```

Expected.

```text
gp2

gp3
```

If no default StorageClass exists, create one.

Create

```
storageclass.yaml
```

```yaml
apiVersion: storage.k8s.io/v1

kind: StorageClass

metadata:

  name: gp3

  annotations:

    storageclass.kubernetes.io/is-default-class: "true"

provisioner: ebs.csi.aws.com

volumeBindingMode: WaitForFirstConsumer

allowVolumeExpansion: true

parameters:

  type: gp3

reclaimPolicy: Delete
```

Deploy.

```bash
kubectl apply \
-f storageclass.yaml
```

Verify.

```bash
kubectl get storageclass
```

---

# 18. Install AWS Load Balancer Controller

The AWS Load Balancer Controller provisions an Application Load Balancer (ALB) automatically from Kubernetes Ingress resources.

Download IAM Policy.

```bash
curl -O \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create IAM Policy.

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

Create IAM Service Account.

```bash
eksctl create iamserviceaccount \
--cluster kubepulse-cluster \
--namespace kube-system \
--name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

Install Helm Repository.

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update
```

Install Controller.

```bash
helm install aws-load-balancer-controller \
eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=kubepulse-cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```

Verify.

```bash
kubectl get deployment \
-n kube-system
```

Expected.

```text
aws-load-balancer-controller

READY 2/2
```

---

# 19. Verify AWS Load Balancer Controller

```bash
kubectl get pods \
-n kube-system
```

Expected.

```text
aws-load-balancer-controller

Running
```

Check logs.

```bash
kubectl logs deployment/aws-load-balancer-controller \
-n kube-system
```

There should be no errors.

---

# 20. Create KubePulse Namespace

Create

```
k8s/namespace.yaml
```

```yaml
apiVersion: v1

kind: Namespace

metadata:

  name: kubepulse
```

Deploy.

```bash
kubectl apply \
-f k8s/namespace.yaml
```

Verify.

```bash
kubectl get ns
```

Expected.

```text
kubepulse
```

---

# 21. Create Backend Service Account

Create

```
backend-serviceaccount.yaml
```

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: kubepulse-backend

  namespace: kubepulse
```

Deploy.

```bash
kubectl apply \
-f backend-serviceaccount.yaml
```

Verify.

```bash
kubectl get sa \
-n kubepulse
```

---

# 22. Configure RBAC

Create

```
k8s/rbac.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRole

metadata:

  name: kubepulse-reader

rules:

- apiGroups: [""]

  resources:

  - pods

  - pods/log

  - services

  - events

  - nodes

  - namespaces

  - persistentvolumes

  - persistentvolumeclaims

  verbs:

  - get

  - list

  - watch

- apiGroups: ["apps"]

  resources:

  - deployments

  - daemonsets

  - replicasets

  - statefulsets

  verbs:

  - get

  - list

  - watch

---

apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRoleBinding

metadata:

  name: kubepulse-reader-binding

subjects:

- kind: ServiceAccount

  name: kubepulse-backend

  namespace: kubepulse

roleRef:

  apiGroup: rbac.authorization.k8s.io

  kind: ClusterRole

  name: kubepulse-reader
```

Deploy.

```bash
kubectl apply \
-f k8s/rbac.yaml
```

Verify.

```bash
kubectl get clusterrole

kubectl get clusterrolebinding
```

---

# 23. Verify Kubernetes Access Permissions

Launch a temporary pod.

```bash
kubectl run test \
--image=amazonlinux \
-it \
--rm \
-- bash
```

Inside the container.

```bash
curl https://kubernetes.default.svc
```

The Kubernetes API should respond.

Exit.

```bash
exit
```

---

# 24. Validate Cluster Readiness

Run the following commands before deploying KubePulse.

```bash
kubectl get nodes

kubectl get pods -A

kubectl get storageclass

kubectl get svc -A

kubectl get deployment -A
```

Everything should report healthy.

The cluster is now fully prepared for deploying:

- FastAPI Backend
- Streamlit Frontend
- Prometheus
- Alertmanager
- Grafana
- Application Load Balancer (ALB)
- Persistent Volumes
- Self-Healing Infrastructure

---

## End of Part 2

Part 3 will include the complete production Kubernetes manifests for AWS EKS, including:

- backend.yaml
- frontend.yaml
- ingress.yaml (AWS ALB)
- Prometheus (Persistent Volume enabled)
- Alertmanager
- Grafana
- PersistentVolumeClaims
- Amazon ECR image configuration
- AWS-specific annotations
- Health probes
- Resource limits
- Production deployment configuration

# Part 3 Kubernetes Deployment on AWS EKS

## 8. Kubernetes Namespace

Create the application namespace.

apiVersion: v1
kind: Namespace

metadata:
  name: kubepulse

## Deploy

kubectl apply -f k8s/namespace.yaml

## Verify

kubectl get namespaces

## Output

kubepulse
default
kube-system

# 9. RBAC Configuration

Create ServiceAccount

ClusterRole

ClusterRoleBinding

Exactly the same as local deployment.

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
  resources:
  - pods
  - pods/log
  - nodes
  - namespaces
  - services
  - events
  - persistentvolumes
  - persistentvolumeclaims
  verbs:
  - get
  - list
  - watch

- apiGroups:
  - apps

  resources:

  - deployments

  - replicasets

  - daemonsets

  - statefulsets

  verbs:

  - get

  - list

  - watch

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

## Deploy

## kubectl apply -f k8s/rbac.yaml

Verify

kubectl get sa -n kubepulse

kubectl get clusterrole kubepulse-reader

kubectl get clusterrolebinding kubepulse-backend-binding

# 10. Backend Deployment

Unlike Kind, the image must be pulled from Amazon ECR.

## Replace

image: kubepulse-backend:latest

with

image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/kubepulse-backend:latest

## Example

apiVersion: apps/v1

kind: Deployment

metadata:

  name: kubepulse-backend

  namespace: kubepulse

spec:

  replicas: 2

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

        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/kubepulse-backend:latest

        imagePullPolicy: Always

        ports:

        - containerPort: 8000

        env:

        - name: IN_CLUSTER

          value: "true"

        resources:

          requests:

            cpu: "250m"

            memory: "256Mi"

          limits:

            cpu: "1"

            memory: "1Gi"

---

apiVersion: v1

kind: Service

metadata:

  name: kubepulse-backend

  namespace: kubepulse

spec:

  selector:

    app: kubepulse-backend

  ports:

  - port: 8000

    targetPort: 8000

  type: ClusterIP

## Deploy

kubectl apply -f k8s/backend.yaml

## Verify

kubectl get deployment -n kubepulse

kubectl get pods -n kubepulse

kubectl get svc -n kubepulse

# 11. Frontend Deployment

Again replace the image with ECR.

image:
123456789012.dkr.ecr.us-east-1.amazonaws.com/kubepulse-frontend:latest

## Example

apiVersion: apps/v1

kind: Deployment

metadata:

  name: kubepulse-frontend

  namespace: kubepulse

spec:

  replicas: 2

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

        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/kubepulse-frontend:latest

        imagePullPolicy: Always

        ports:

        - containerPort: 8501

        env:

        - name: BACKEND_URL

          value: http://kubepulse-backend:8000

---

apiVersion: v1

kind: Service

metadata:

  name: kubepulse-frontend

  namespace: kubepulse

spec:

  selector:

    app: kubepulse-frontend

  ports:

  - port: 8501

    targetPort: 8501

  type: ClusterIP

## Deploy

kubectl apply -f k8s/frontend.yaml

## Verify

kubectl get pods -n kubepulse

kubectl get svc -n kubepulse

# 12. Deploy kube-state-metrics

EKS does NOT expose many metrics by default.

## Deploy kube-state-metrics

kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/latest/download/standard.yaml

## Verify

kubectl get pods -n kube-system | grep kube-state

# 13. Install Metrics Server
kubectl apply -f \
https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

## Patch for EKS

kubectl patch deployment metrics-server \
-n kube-system \
--type=json \
-p='[
{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'

## Verify

kubectl top nodes

kubectl top pods -A

# 14. Install Prometheus using Helm

## Add repository

helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts

## Update

helm repo update

## Install

helm install prometheus prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--create-namespace

## Verify

kubectl get pods -n monitoring

## Expected

Prometheus

Grafana

Alertmanager

Node Exporter

Prometheus Operator

kube-state-metrics

# 15. Install AWS Load Balancer Controller

## Associate IAM OIDC provider

eksctl utils associate-iam-oidc-provider \
--cluster kubepulse-cluster \
--approve

## Download IAM policy

curl -O \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

## Create policy

aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json

## Create service account

eksctl create iamserviceaccount \
--cluster kubepulse-cluster \
--namespace kube-system \
--name aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve

## Install Helm chart

helm repo add eks https://aws.github.io/eks-charts

helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=kubepulse-cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller

## Verify

kubectl get deployment -n kube-system aws-load-balancer-controller

# This completes Part 3, covering the Kubernetes deployment on EKS: application workloads, metrics components, Prometheus/Grafana stack installation, and the AWS Load Balancer Controller required for exposing services.

# In Part 4, we'll cover:

ALB Ingress configuration
External DNS (optional)
TLS certificates with ACM
Alertmanager configuration
Prometheus rules
HPA (Horizontal Pod Autoscaler)
Cluster Autoscaler
Persistent storage with EBS CSI
Final deployment commands and validation steps
Production best practices and troubleshooting.

# Part 4 — Production Deployment, Autoscaling & Validation on AWS EKS

## 16. Install Amazon EBS CSI Driver

Unlike Kind, EKS requires a CSI driver for Persistent Volumes.

## Verify Add-on

aws eks describe-addon \
--cluster-name kubepulse-cluster \
--addon-name aws-ebs-csi-driver

## If it doesn't exist:

aws eks create-addon \
--cluster-name kubepulse-cluster \
--addon-name aws-ebs-csi-driver

## Verify

kubectl get pods -n kube-system | grep ebs

## Expected

ebs-csi-controller
ebs-csi-node

# 17. Create StorageClass

## storageclass.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: gp3-storage

provisioner: ebs.csi.aws.com

volumeBindingMode: WaitForFirstConsumer

allowVolumeExpansion: true

parameters:
  type: gp3

## Deploy

kubectl apply -f storageclass.yaml

## Verify

kubectl get storageclass

## Expected

gp3-storage

# 18. Persistent Volume Claims

Grafana should not lose dashboards after pod restart.

## Example PVC

apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: grafana-storage

  namespace: monitoring

spec:

  accessModes:

  - ReadWriteOnce

  storageClassName: gp3-storage

  resources:

    requests:

      storage: 20Gi

## Deploy

kubectl apply -f grafana-pvc.yaml

## Verify

kubectl get pvc -n monitoring

## Expected

grafana-storage
Bound

# 19. Horizontal Pod Autoscaler (HPA)

## Backend

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: kubepulse-backend-hpa

  namespace: kubepulse

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: kubepulse-backend

  minReplicas: 2

  maxReplicas: 10

  metrics:

  - type: Resource

    resource:

      name: cpu

      target:

        type: Utilization

        averageUtilization: 70

## Frontend

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: kubepulse-frontend-hpa

  namespace: kubepulse

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: kubepulse-frontend

  minReplicas: 2

  maxReplicas: 10

  metrics:

  - type: Resource

    resource:

      name: cpu

      target:

        type: Utilization

        averageUtilization: 70

## Deploy

kubectl apply -f hpa-backend.yaml

kubectl apply -f hpa-frontend.yaml

## Verify

kubectl get hpa -n kubepulse

# 20. Install Cluster Autoscaler

## Download

curl -O \
https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

## Replace

<YOUR CLUSTER NAME>

with

kubepulse-cluster

## Deploy

kubectl apply -f cluster-autoscaler-autodiscover.yaml

## Verify

kubectl get deployment \
-n kube-system cluster-autoscaler

# 21. Create ACM SSL Certificate

## Request certificate

aws acm request-certificate \
--domain-name kubepulse.example.com \
--validation-method DNS

Validate through Route53.

## After validation

aws acm list-certificates

Copy

Certificate ARN

# 22. Create ALB Ingress

## Replace

ACCOUNT VALUES

HOSTNAME

CERTIFICATE ARN

## with your values.

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: kubepulse-ingress

  namespace: kubepulse

  annotations:

    kubernetes.io/ingress.class: alb

    alb.ingress.kubernetes.io/scheme: internet-facing

    alb.ingress.kubernetes.io/target-type: ip

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'

    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...

    alb.ingress.kubernetes.io/ssl-redirect: '443'

spec:

  rules:

  - host: kubepulse.example.com

    http:

      paths:

      - path: /

        pathType: Prefix

        backend:

          service:

            name: kubepulse-frontend

            port:

              number: 8501

      - path: /api

        pathType: Prefix

        backend:

          service:

            name: kubepulse-backend

            port:

              number: 8000

## Deploy

kubectl apply -f ingress.yaml

## Verify

kubectl get ingress -n kubepulse

## Expected

ADDRESS

xxxxxxxx.ap-south-1.elb.amazonaws.com

# 23. Route53 DNS Mapping

## Create an Alias Record

kubepulse.example.com

↓

ALB DNS Name

## Verify

nslookup kubepulse.example.com

# 24. Alertmanager Configuration

## Deploy Alertmanager ConfigMap

kubectl apply -f monitoring/alertmanager.yaml

## Restart

kubectl rollout restart deployment alertmanager \
-n monitoring

## Verify

kubectl logs deployment/alertmanager \
-n monitoring

# 25. Prometheus Rules

## Deploy

kubectl apply -f monitoring/prometheus-rules.yaml

## Verify

kubectl get prometheusrules \
-n monitoring

# 26. Restart Monitoring Stack
kubectl rollout restart deployment grafana \
-n monitoring

kubectl rollout restart deployment prometheus-kube-prometheus-prometheus \
-n monitoring

kubectl rollout restart deployment alertmanager \
-n monitoring

# 27. Verify Cluster

## Nodes

kubectl get nodes

## Pods

kubectl get pods -A

## Services

kubectl get svc -A

## Ingress

kubectl get ingress -A

## PVC

kubectl get pvc -A

# 28. Verify Metrics

Check

kubectl top nodes

kubectl top pods -A

# 29. Verify Prometheus

## Port Forward

kubectl port-forward \
svc/prometheus-operated \
9090:9090 \
-n monitoring

## Open

http://localhost:9090

## Query

up

## Expected

All targets UP

# 30. Verify Grafana

## Port Forward

kubectl port-forward \
svc/prometheus-grafana \
3000:80 \
-n monitoring

## Open

http://localhost:3000

## Login

admin

prom-operator

## Verify dashboards

Cluster CPU
Memory
Pods
Nodes
Deployments
Network
Kubernetes Overview

# 31. Test Backend

## Health

curl https://kubepulse.example.com/api/health

## Expected

{
  "status":"healthy",
  "service":"kubepulse-backend"
}

# 32. Test Frontend

## Open

https://kubepulse.example.com

## Verify

Dashboard loads
Pods tab
Deployments
Nodes
Events
Alerts
Grafana button
Prometheus button

# 33. Test Alertmanager

curl \
-X POST \
https://kubepulse.example.com/api/alerts/webhook \
-H "Content-Type: application/json" \
-d '{
"status":"firing",
"alerts":[
{
"labels":{
"alertname":"HighCPUUsage",
"namespace":"kubepulse",
"pod":"backend-test",
"severity":"critical"
},
"annotations":{
"summary":"CPU exceeds threshold",
"description":"CPU usage exceeded 90%"
},
"startsAt":"2026-06-20T12:00:00Z"
}
]
}'

## Expected Dashboard

Incidents

↓

HighCPUUsage

↓

Critical

# 34. Scaling Test

## Increase replicas

kubectl scale deployment kubepulse-backend \
--replicas=6 \
-n kubepulse

## Verify

kubectl get pods -n kubepulse

## Decrease

kubectl scale deployment kubepulse-backend \
--replicas=2 \
-n kubepulse

# 35. Rolling Update

## Push a new Docker image

docker build \
-t kubepulse-backend:v2 \
./backend

## Push

docker push \
<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/kubepulse-backend:v2

## Update Deployment

kubectl set image deployment/kubepulse-backend \
backend=<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/kubepulse-backend:v2 \
-n kubepulse

## Verify rollout

kubectl rollout status deployment/kubepulse-backend \
-n kubepulse

# 36. Final Validation Checklist

Before declaring the deployment production-ready, verify the following:

✅ EKS Cluster status is ACTIVE
✅ Worker Nodes are in Ready state
✅ Backend and Frontend pods are Running
✅ Prometheus, Grafana, Alertmanager, kube-state-metrics, and Metrics Server are healthy
✅ AWS Load Balancer Controller is operational
✅ ALB Ingress is provisioned with an external DNS name
✅ Route53 correctly resolves the application domain
✅ ACM SSL certificate is issued and HTTPS works
✅ Grafana dashboards display live Kubernetes metrics
✅ Prometheus targets are all UP
✅ Alertmanager successfully forwards alerts to the backend webhook
✅ Horizontal Pod Autoscaler (HPA) scales workloads under load
✅ Cluster Autoscaler adds/removes EC2 worker nodes as required
✅ EBS-backed Persistent Volumes retain Grafana data after pod restarts
✅ Rolling updates complete without downtime
✅ All Kubernetes resources report healthy status with kubectl get all -n kubepulse

## This completes the AWS EKS implementation guide, bringing the project to a production-grade deployment with scalable workloads, persistent storage, HTTPS via ACM, ALB ingress, monitoring, alerting, and autoscaling.