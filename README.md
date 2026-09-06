# 🏠 Home Server

A public version of my self-hosted home server, running on a [k3s](https://k3s.io/) Kubernetes cluster and managed entirely with [Argo CD](https://argo-cd.readthedocs.io/) using a **GitOps** approach.

> ℹ️ This repository is a sanitized mirror of the private repository that runs in production. The only meaningful differences are the redaction of private information (domains, emails, secrets, etc.).

> 🤖 This documentation was written with AI assistance and reviewed by me. The manifests remain the source of truth — if something here disagrees with them, trust the manifests.

## 🚀 Overview

Everything in this cluster is declarative. The desired state of every workload lives in this Git repository, and Argo CD continuously reconciles the cluster to match it. There is no manual `kubectl apply` — to change something, you change the manifests and push.

- **Orchestration:** k3s (lightweight Kubernetes)
- **GitOps / CD:** Argo CD (self-managed)
- **Ingress:** Traefik (k3s default)
- **TLS:** cert-manager + Let's Encrypt (DNS-01 via Cloudflare)
- **Databases:** CloudNativePG (CNPG) operator
- **Config management:** Kustomize + Helm

## 🧩 GitOps Architecture

The repo follows an **app-of-apps** pattern:

```
argocd-sources/          # Argo CD itself (install manifests + Traefik ingress + config patches)
argocd-applications/     # The root Kustomization referencing every Argo CD Application
```

- [`argocd-sources/`](argocd-sources/) bootstraps and self-manages Argo CD. The `argocd-self` Application points back at this directory, so Argo CD keeps **itself** in sync from Git.
- [`argocd-applications/kustomization.yaml`](argocd-applications/kustomization.yaml) is the entrypoint that aggregates every workload. It doubles as the **on/off switch**: a workload is disabled by commenting out its line, not by deleting its manifests.
- Each app is typically split into:
  - `*_app.yaml` — the Argo CD `Application`, usually an upstream Helm chart. Several are **multi-source** apps that combine the upstream chart with manifests from this repo (Home Assistant, Nextcloud, monitoring).
  - `*_resources_app.yaml` — an `Application` for the extra Kubernetes resources (ingresses, volumes, network policies, …) that live in this repo.
- Where several workloads need the same shape of resources, an Argo CD **`ApplicationSet`** generates them instead — see [`postgres/pg_dbs_resources_app.yaml`](argocd-applications/postgres/pg_dbs_resources_app.yaml), which fans out one Application per database from a list generator.

## 📦 Deployed Workloads

| Category | Service | Status | Description |
|----------|---------|--------|-------------|
| 📸 Photos | **Immich** | ✅ enabled | Self-hosted photo & video backup, backed by a CNPG Postgres cluster (with the `vchord` vector extension) and Valkey. |
| ☁️ Files | **Nextcloud** | ✅ enabled | File sync & share, backed by its own CNPG Postgres cluster, with the background cronjob enabled. |
| 🎬 Media | **Jellyfin** | ✅ enabled | Media streaming server. |
| 🎬 Media | **qBittorrent** | ✅ enabled | Torrent client, routed through a **Gluetun** VPN sidecar (all containers in the pod share its network namespace). |
| 🎬 Media | **Prowlarr** | ⏸️ disabled | Indexer manager for the *arr stack; multi-source app that also deploys FlareSolverr. |
| 🎬 Media | **Radarr** | ⏸️ disabled | Movie collection management. |
| 🛡️ Network | **Pi-hole** | ✅ enabled | Network-wide DNS sinkhole / ad-blocking. See [pihole/README.md](argocd-applications/pihole/README.md) for freeing port 53 on Ubuntu. |
| 🏡 Home | **Home Assistant** | ⏸️ disabled | Home automation on `hostNetwork` for device discovery, plus a **Matter server** deployment for Matter/Thread devices. |
| 🔐 TLS | **cert-manager** | ✅ enabled | Automated Let's Encrypt certificates via Cloudflare DNS-01. |
| 🗄️ Database | **CloudNativePG** | ✅ enabled | Operator-managed PostgreSQL clusters (Immich + Nextcloud). |
| 📊 Monitoring | **kube-prometheus-stack** | ✅ enabled | Prometheus + Grafana + Alertmanager. |
| 📊 Monitoring | **fail2ban exporter** | ✅ enabled | Exposes fail2ban metrics to Prometheus. |
| 🎮 Games | **Minecraft** | ⏸️ disabled | Game server. |

*Status reflects whether the Application is currently referenced (uncommented) in [`argocd-applications/kustomization.yaml`](argocd-applications/kustomization.yaml).*

## 🗄️ Databases

PostgreSQL is provided by the **CloudNativePG** operator, installed once by [`pg_db_operator_app.yaml`](argocd-applications/postgres/pg_db_operator_app.yaml) into `cnpg-system`.

Per-application `Cluster` resources live in `postgres/<app>_pg_db_resources/` and are rolled out by the [`pg-dbs-resources` ApplicationSet](argocd-applications/postgres/pg_dbs_resources_app.yaml). Adding a database is a one-line change to its list generator:

```yaml
generators:
- list:
    elements:
    - name: immich
    - name: nextcloud
```

Each entry creates an Application named `<name>-pg-db-resources` that syncs `argocd-applications/postgres/<name>_pg_db_resources` into the `<name>` namespace.

- **Immich** runs a VectorChord-enabled Postgres image, with `vchord.so` preloaded and the `vchord` / `earthdistance` extensions created through `postInitSQL`.
- **Nextcloud** runs a stock CNPG Postgres 18 image and bootstraps its database and owner from the `nextcloud-db-creds` secret.

## 💾 Storage

Persistent data uses **statically provisioned `hostPath` PersistentVolumes** paired with matching PVCs (`storageClassName: ""` plus an explicit `volumeName`), with `persistentVolumeReclaimPolicy: Retain` so data survives an Application being deleted and re-synced. Volumes are defined next to the app that uses them, e.g. [`nextcloud_volumes.yaml`](argocd-applications/nextcloud/nextcloud_resources/nextcloud_volumes.yaml) and [`home-assistant volumes.yaml`](argocd-applications/home-assistant/home-assistant-resources/volumes.yaml).

## 🔐 TLS & Networking

- **Traefik** (shipped with k3s) handles ingress routing. Argo CD itself is exposed through a Traefik `IngressRoute`, with `server.insecure: "true"` set deliberately — Traefik terminates TLS externally and forwards plain HTTP internally.
- **cert-manager** issues certificates from Let's Encrypt using a `ClusterIssuer` with the **Cloudflare DNS-01** solver, so services get valid public TLS without exposing HTTP-01 challenge endpoints.
- Sensitive workloads (e.g. the Immich Postgres database) are restricted with Kubernetes **NetworkPolicies** — only Immich components and the CNPG operator namespace may reach port 5432.
- A few workloads need the host network on purpose: **Home Assistant** and the **Matter server** (device / mDNS discovery), while **qBittorrent** sends all of its traffic through the Gluetun VPN sidecar.

## 🛠️ Tech Stack

- **Kubernetes:** k3s
- **GitOps:** Argo CD (Applications + ApplicationSets)
- **Templating:** Kustomize, Helm
- **Ingress:** Traefik
- **Certificates:** cert-manager, Let's Encrypt, Cloudflare
- **Databases:** CloudNativePG (PostgreSQL)
- **Observability:** Prometheus, Grafana, Alertmanager

## 📁 Repository Structure

```
.
├── argocd-sources/                 # Argo CD bootstrap (self-managed)
│   ├── kustomization.yaml          # Pinned upstream install.yaml + patches
│   ├── argocd_traefik_ingress.yaml
│   └── patches/
└── argocd-applications/            # App-of-apps root
    ├── kustomization.yaml          # References every Application below
    ├── argo/                       # argocd-self Application
    ├── cert-manager/               # Chart + ACME ClusterIssuer
    ├── home-assistant/             # Chart + Matter server deployment & volumes
    ├── immich/                     # Chart + ingresses, volumes
    ├── media/                      # jellyfin, qbittorrent (+ gluetun), prowlarr, radarr
    ├── minecraft/
    ├── monitoring/                 # kube-prometheus-stack, fail2ban exporter
    ├── nextcloud/                  # Chart + volumes
    ├── pihole/
    └── postgres/                   # CNPG operator + per-database ApplicationSet
```

## 🧾 License

[MIT](LICENSE)
