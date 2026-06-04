# 🏠 Home Server

A public version of my self-hosted home server, running on a [k3s](https://k3s.io/) Kubernetes cluster and managed entirely with [Argo CD](https://argo-cd.readthedocs.io/) using a **GitOps** approach.

> ℹ️ This repository is a sanitized mirror of the private repository that runs in production. The only meaningful differences are the redaction of private information (domains, emails, secrets, etc.).

## 🚀 Overview

Everything in this cluster is declarative. The desired state of every workload lives in this Git repository, and Argo CD continuously reconciles the cluster to match it. There is no manual `kubectl apply` — to change something, you change the manifests and push.

- **Orchestration:** k3s (lightweight Kubernetes)
- **GitOps / CD:** Argo CD (self-managed)
- **Ingress:** Traefik (k3s default)
- **TLS:** cert-manager + Let's Encrypt (DNS-01 via Cloudflare)
- **Database:** CloudNativePG (CNPG) operator
- **Config management:** Kustomize + Helm

## 🧩 GitOps Architecture

The repo follows an **app-of-apps** pattern:

```
argocd-sources/          # Argo CD itself (install manifests + Traefik ingress + config patches)
argocd-applications/     # The root Kustomization referencing every Argo CD Application
```

- [`argocd-sources/`](argocd-sources/) bootstraps and self-manages Argo CD. The `argocd-self` Application points back at this directory, so Argo CD keeps **itself** in sync from Git.
- [`argocd-applications/kustomization.yaml`](argocd-applications/kustomization.yaml) is the entrypoint that aggregates every workload. Each app is typically split into:
  - `*_app.yaml` — the Argo CD `Application` (often a Helm chart from upstream).
  - `*_resources_app.yaml` — an `Application` for the extra Kubernetes resources (ingresses, volumes, network policies, …) that live in this repo.

## 📦 Deployed Workloads

| Category | Service | Description |
|----------|---------|-------------|
| 📸 Photos | **Immich** | Self-hosted photo & video backup, backed by a CNPG Postgres cluster (with the `vchord` vector extension) and Valkey. |
| 🎬 Media | **Jellyfin** | Media streaming server. |
| 🎬 Media | **Prowlarr** | Indexer manager for the *arr stack. |
| 🎬 Media | **Radarr** | Movie collection management. |
| 🎬 Media | **qBittorrent** | Torrent client (with Prometheus metrics exporter). |
| 🛡️ Network | **Pi-hole** | Network-wide DNS sinkhole / ad-blocking. |
| 🔐 TLS | **cert-manager** | Automated Let's Encrypt certificates via Cloudflare DNS-01. |
| 🗄️ Database | **CloudNativePG** | Operator-managed PostgreSQL clusters. |
| 📊 Monitoring | **kube-prometheus-stack** | Prometheus + Grafana + Alertmanager. |
| 📊 Monitoring | **fail2ban exporter** | Exposes fail2ban metrics to Prometheus. |
| 🎮 Games | **Minecraft** | Game server (currently disabled). |

## 🔐 TLS & Networking

- **Traefik** (shipped with k3s) handles ingress routing.
- **cert-manager** issues certificates from Let's Encrypt using a `ClusterIssuer` with the **Cloudflare DNS-01** solver, so services get valid public TLS without exposing HTTP-01 challenge endpoints.
- Sensitive workloads (e.g. the Immich Postgres database) are restricted with Kubernetes **NetworkPolicies**.

## 🛠️ Tech Stack

- **Kubernetes:** k3s
- **GitOps:** Argo CD
- **Templating:** Kustomize, Helm
- **Ingress:** Traefik
- **Certificates:** cert-manager, Let's Encrypt, Cloudflare
- **Databases:** CloudNativePG (PostgreSQL)
- **Observability:** Prometheus, Grafana, Alertmanager

## 📁 Repository Structure

```
.
├── argocd-sources/                 # Argo CD bootstrap (self-managed)
│   ├── kustomization.yaml
│   ├── argocd_traefik_ingress.yaml
│   └── patches/
└── argocd-applications/            # App-of-apps root
    ├── kustomization.yaml          # References every Application below
    ├── argo/                       # argocd-self Application
    ├── cert-manager/
    ├── immich/
    ├── postgres/                   # CNPG cluster for Immich
    ├── media/                      # jellyfin, prowlarr, radarr, qbittorrent
    ├── monitoring/                 # kube-prometheus-stack, fail2ban exporter
    ├── pihole/
    └── minecraft/
```

## 🧾 License

[MIT](LICENSE)
