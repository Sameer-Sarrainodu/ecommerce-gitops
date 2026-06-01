# RoboShop AKS GitOps Deployment Guide

## Overview

This repository contains the complete GitOps deployment for RoboShop on Azure Kubernetes Service (AKS) using:

* Terraform
* AKS
* Azure Container Registry (ACR)
* ArgoCD
* Helm
* NGINX Ingress Controller

---

# Phase 1 - Final Code Review

Complete these checks before deploying any Azure resources.

## Verify Helm Charts

Ensure all charts are present and valid:

```text
mongodb
mysql
redis
rabbitmq

catalogue-schema
shipping-schema

catalogue
user
cart
shipping
payment
frontend

roboshop-ingress
ingress-nginx
```

Run:

```bash
helm lint helm/mongodb
helm lint helm/mysql
helm lint helm/redis
helm lint helm/rabbitmq

helm lint helm/catalogue-schema
helm lint helm/shipping-schema

helm lint helm/catalogue
helm lint helm/user
helm lint helm/cart
helm lint helm/shipping
helm lint helm/payment
helm lint helm/frontend

helm lint helm/roboshop-ingress
helm lint helm/ingress-nginx
```

---

## Verify ArgoCD Configuration

Validate:

```text
argocd/
├── app-of-apps.yaml
└── applicationset.yaml
```

Check:

* Repository URL is correct
* Helm chart paths are correct
* Sync waves are configured correctly

Deployment Order:

```text
Wave -1
└── ingress-nginx

Wave 0
├── mongodb
├── mysql
├── redis
└── rabbitmq

Wave 1
├── catalogue-schema
└── shipping-schema

Wave 2
├── catalogue
├── user
├── cart
├── shipping
└── payment

Wave 3
└── frontend

Wave 4
└── roboshop-ingress
```

---

## Push GitOps Repository

```bash
git add .
git commit -m "Complete AKS GitOps setup"
git push
```

---

# Phase 2 - Deploy Infrastructure

Apply Terraform in sequence.

---

## 00-resourcegroup

```bash
terraform init
terraform apply
```

---

## 10-network

```bash
terraform init
terraform apply
```

---

## 20-acr

```bash
terraform init
terraform apply
```

Verify ACR:

```bash
az acr show -n sameerroboshopacr
```

---

## 30-aks

```bash
terraform init
terraform apply
```

Wait until AKS provisioning completes.

---

## 40-dns

```bash
terraform init
terraform apply
```

---

# Phase 3 - Connect To AKS

Retrieve cluster credentials:

```bash
az aks get-credentials \
  --resource-group <resource-group-name> \
  --name <aks-name>
```

Verify cluster nodes:

```bash
kubectl get nodes
```

Expected:

```text
systempool
apppool
```

---

# Phase 4 - Bootstrap

## Create Namespaces

```bash
kubectl apply -f bootstrap/namespace.yaml
```

Verify:

```bash
kubectl get ns
```

Expected:

```text
argocd
ingress-nginx
roboshop
```

---

## Install ArgoCD

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all ArgoCD pods are healthy:

```bash
kubectl get pods -n argocd
```

Expected:

```text
Running
```

---

# Phase 5 - Deploy GitOps Applications

Apply ArgoCD resources:

```bash
kubectl apply -f argocd/applicationset.yaml
kubectl apply -f argocd/app-of-apps.yaml
```

Monitor:

```bash
kubectl get applications -n argocd
```

---

# Phase 6 - Debug

Check application pods:

```bash
kubectl get pods -n roboshop
```

Common Issues:

```text
ImagePullBackOff
CrashLoopBackOff
Pending
```

Investigate:

```bash
kubectl describe pod <pod-name> -n roboshop

kubectl logs <pod-name> -n roboshop
```

---

# Phase 7 - Configure Ingress

Verify ingress controller:

```bash
kubectl get svc -n ingress-nginx
```

Retrieve external IP:

```bash
kubectl get svc -n ingress-nginx
```

Update DNS:

```text
shop.sharkdev.shop
        ↓
Ingress External IP
```

Verify DNS resolution:

```bash
nslookup shop.sharkdev.shop
```

---

# Phase 8 - End-To-End Testing

Open:

```text
http://shop.sharkdev.shop
```

Verify:

* Frontend loads
* Product listing works
* User login works
* Cart operations work
* Shipping calculation works
* Payment processing works

---

## Verify Service Logs

Catalogue:

```bash
kubectl logs deployment/catalogue -n roboshop
```

User:

```bash
kubectl logs deployment/user -n roboshop
```

Cart:

```bash
kubectl logs deployment/cart -n roboshop
```

Shipping:

```bash
kubectl logs deployment/shipping -n roboshop
```

Payment:

```bash
kubectl logs deployment/payment -n roboshop
```

Frontend:

```bash
kubectl logs deployment/frontend -n roboshop
```

---

# Success Criteria

All pods running:

```bash
kubectl get pods -n roboshop
```

All applications synced:

```bash
kubectl get applications -n argocd
```

Frontend accessible:

```text
shop.sharkdev.shop
```

Application fully functional through:

* Catalogue
* User
* Cart
* Shipping
* Payment

---

# Future Enhancements

## Security

```text
50-security
├── Azure Key Vault
├── Workload Identity
├── CSI Driver
├── Network Policies
└── Private Endpoints
```

---

## Ingress Refactoring

Current:

```text
Ingress
↓
Frontend NGINX
↓
API Routing
```

Future:

```text
Ingress
├── / → frontend
├── /api/catalogue → catalogue
├── /api/user → user
├── /api/cart → cart
├── /api/shipping → shipping
└── /api/payment → payment
```

Remove API routing from frontend NGINX.

---

## Observability

```text
Prometheus
Grafana
Loki
Alertmanager
```

---

## CI/CD

```text
Build
Test
Docker Build
Push ACR
Update Helm Tag
GitOps Commit
```

---

## Production AKS

```text
Private AKS
Azure Bastion
NAT Gateway
Private DNS
WAF
Key Vault Integration
```

---

## Multi-Cloud

Reuse GitOps repository for:

```text
AKS
EKS
GKE
```

Only infrastructure Terraform changes. Helm and ArgoCD remain largely unchanged.
