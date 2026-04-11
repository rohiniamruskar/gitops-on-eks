<div align="center">

# AI BankApp — End-to-End GitOps on EKS

A Spring Boot banking application with an AI chatbot, deployed on AWS EKS using Terraform, ArgoCD, Gateway API, and Prometheus monitoring. Built as a complete GitOps reference architecture.

[![Java Version](https://img.shields.io/badge/Java-21-blue.svg)](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.1-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Kubernetes](https://img.shields.io/badge/EKS-1.35-326CE5.svg)](https://aws.amazon.com/eks/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange.svg)](https://argo-cd.readthedocs.io/)

</div>

---

## Architecture

### GitOps Flow

```
Developer Push → GitHub Actions CI → DockerHub → ArgoCD → EKS Cluster
                                                    ↑
                                          watches k8s/ manifests
```

### EKS Infrastructure

```
                    ┌─────────────────────────────────────────────────┐
                    │                 VPC (10.0.0.0/16)               │
                    │                                                 │
                    │  ┌──────────────┐ ┌──────────────┐ ┌────────┐  │
                    │  │ Public Sub    │ │ Public Sub    │ │Pub Sub │  │
                    │  │ 10.0.1.0/24  │ │ 10.0.2.0/24  │ │.3.0/24 │  │
                    │  │  (us-west-2a) │ │  (us-west-2b) │ │ (2c)  │  │
                    │  └──────┬───────┘ └──────┬───────┘ └───┬────┘  │
                    │         │ NAT GW                       │        │
                    │  ┌──────┴───────┐ ┌──────────────┐ ┌───┴────┐  │
                    │  │ Private Sub   │ │ Private Sub   │ │Priv Sub│  │
                    │  │ 10.0.4.0/24  │ │ 10.0.5.0/24  │ │.6.0/24 │  │
                    │  │ (Worker Nodes)│ │ (Worker Nodes)│ │(Nodes) │  │
                    │  └──────────────┘ └──────────────┘ └────────┘  │
                    │  ┌──────────────┐ ┌──────────────┐ ┌────────┐  │
                    │  │  Intra Sub    │ │  Intra Sub    │ │Intra   │  │
                    │  │  10.0.7.0/24  │ │  10.0.8.0/24  │ │.9.0/24│  │
                    │  │ (Ctrl Plane)  │ │ (Ctrl Plane)  │ │(Ctrl)  │  │
                    │  └──────────────┘ └──────────────┘ └────────┘  │
                    └─────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Tool |
|-------|------|
| **Application** | Java 21, Spring Boot 3.4.1, Spring Security, Thymeleaf |
| **AI Chatbot** | Ollama (TinyLlama) — self-hosted, zero cost |
| **Database** | MySQL 8.0 with EBS gp3 persistent storage |
| **Infrastructure** | Terraform (VPC + EKS + ArgoCD) |
| **Kubernetes** | Amazon EKS 1.35, 3x t3.medium nodes across 3 AZs |
| **GitOps / CD** | ArgoCD — auto-sync from `k8s/` manifests |
| **CI Pipeline** | GitHub Actions → DockerHub |
| **Ingress** | Gateway API + Envoy Gateway (AWS NLB) |
| **TLS** | cert-manager + Let's Encrypt (auto-provisioned) |
| **Monitoring** | kube-prometheus-stack (Prometheus + Grafana) |
| **Storage** | EBS CSI Driver with gp3 dynamic provisioning |

---

## What Gets Created

### Terraform (Infrastructure)

| Resource | Details |
|----------|---------|
| **VPC** | `10.0.0.0/16` across 3 AZs with public, private, and intra subnets |
| **NAT Gateway** | Single NAT for private subnet internet access |
| **EKS Cluster** | `bankapp-eks` running Kubernetes 1.35 |
| **Node Group** | `bankapp-ng` — 3x `t3.medium` on AL2023 |
| **Add-ons** | CoreDNS, kube-proxy, VPC-CNI, Pod Identity Agent, EBS CSI Driver |
| **ArgoCD** | Installed via Helm, exposed as LoadBalancer |

### ArgoCD (Application Manifests from `k8s/`)

| Manifest | What it does |
|----------|-------------|
| `namespace.yml` | Creates `bankapp` namespace |
| `pv.yml` | gp3 StorageClass (EBS CSI dynamic provisioning) |
| `pvc.yml` | PVCs for MySQL (5Gi) and Ollama (10Gi) |
| `configmap.yml` | DB host, port, Ollama URL, proxy headers |
| `secrets.yml` | DB credentials |
| `mysql-deployment.yml` | MySQL 8.0 with EBS volume |
| `ollama-deployment.yml` | Ollama AI with TinyLlama model |
| `bankapp-deployment.yml` | BankApp (2 replicas, HPA enabled) |
| `service.yml` | ClusterIP services for all components |
| `gateway.yml` | Gateway API + HTTPS + session persistence |
| `cert-manager.yml` | Let's Encrypt ClusterIssuer |
| `hpa.yml` | Auto-scale BankApp 2–4 replicas |

---

## CI/CD Pipeline (GitOps)

```
Code Push → GitHub Actions → Build & Push to DockerHub → Update k8s manifest → ArgoCD auto-sync → EKS
```

1. Developer pushes code to `feat/gitops`
2. GitHub Actions ([`.github/workflows/gitops-ci.yml`](.github/workflows/gitops-ci.yml)):
   - Builds the Spring Boot app with Maven
   - Builds and pushes Docker image to DockerHub (`trainwithshubham/ai-bankapp-eks:<sha>`)
   - Updates `k8s/bankapp-deployment.yml` with new image tag
   - Commits with `[skip ci]`
3. ArgoCD detects manifest change → auto-syncs to EKS
4. EKS performs rolling update with zero downtime

### GitHub Secrets Required

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | DockerHub username |
| `DOCKERHUB_TOKEN` | DockerHub access token |

---

## Quick Start

> Full step-by-step commands are in [`DEPLOYMENT.md`](DEPLOYMENT.md)

```bash
# 1. Provision infrastructure (~15 min)
cd terraform && terraform init && terraform apply

# 2. Configure kubectl
aws eks update-kubeconfig --name bankapp-eks --region us-west-2

# 3. Install Gateway API + Envoy Gateway + cert-manager
#    (see DEPLOYMENT.md Steps 4-5)

# 4. Install monitoring
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace --set grafana.service.type=LoadBalancer --wait

# 5. Deploy app via ArgoCD
kubectl apply -f argocd/application.yml

# 6. Pull AI model
kubectl exec -n bankapp deploy/ollama -- ollama pull tinyllama
```

---

## Project Structure

```
.
├── terraform/                 # Infrastructure as Code
│   ├── provider.tf            # Terraform + AWS + Helm providers
│   ├── variables.tf           # Configurable inputs
│   ├── vpc.tf                 # VPC — public, private, intra subnets
│   ├── eks.tf                 # EKS — cluster, node group, add-ons, IRSA
│   ├── argocd.tf              # ArgoCD Helm release
│   ├── outputs.tf             # Cluster info, kubectl command
│   └── terraform.tfvars       # Default values
├── k8s/                       # Application manifests (ArgoCD syncs this)
│   ├── namespace.yml
│   ├── pv.yml                 # StorageClass
│   ├── pvc.yml                # PVCs for MySQL + Ollama
│   ├── configmap.yml
│   ├── secrets.yml
│   ├── mysql-deployment.yml
│   ├── ollama-deployment.yml
│   ├── bankapp-deployment.yml
│   ├── service.yml
│   ├── gateway.yml            # Gateway + HTTPRoute + session persistence
│   ├── cert-manager.yml       # Let's Encrypt ClusterIssuer
│   └── hpa.yml
├── argocd/
│   └── application.yml        # ArgoCD Application pointing to k8s/
├── .github/workflows/
│   └── gitops-ci.yml          # CI → DockerHub → manifest update
├── DEPLOYMENT.md              # Step-by-step deployment playbook
└── terraform/README.md        # Detailed setup with troubleshooting
```

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| **Envoy Gateway** (not nginx ingress) | Native Gateway API support, future-proof, auto-provisions AWS NLB |
| **Cookie-based session persistence** | Envoy Gateway bypasses K8s Service sessionAffinity — cookie hashing pins browsers to pods for CSRF/session consistency |
| **ServerSideApply in ArgoCD** | Avoids field ownership conflicts with Gateway API CRDs |
| **sslip.io for TLS** | Free wildcard DNS tied to NLB IP — no domain registrar needed for demos |
| **EBS gp3 for storage** | Cost-effective persistent volumes for MySQL and Ollama model data |
| **Single NAT Gateway** | Sufficient for dev/demo; production would use one per AZ |

---

## Documentation

| Document | Purpose |
|----------|---------|
| [`DEPLOYMENT.md`](DEPLOYMENT.md) | Step-by-step deployment commands + gotchas |
| [`terraform/README.md`](terraform/README.md) | Detailed infrastructure setup + troubleshooting |

---

<div align="center">

Happy Learning

**TrainWithShubham**

</div>
