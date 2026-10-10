# Home Server

Public mirror of my self-hosted home server: a [k3s](https://k3s.io/) cluster managed with [Argo CD](https://argo-cd.readthedocs.io/) using GitOps.

> [!NOTE]
> This is a sanitized mirror of the private production repo. The only differences are redacted values (domains, emails, secrets).

> [!NOTE]
> Documentation written with AI assistance and reviewed by me. The manifests are the source of truth.

## Stack

| Concern | Tool |
|---------|------|
| Kubernetes | k3s |
| GitOps | Argo CD v3.5.4 (self-managed, Applications + ApplicationSets) |
| Config | Kustomize, Helm |
| Ingress | Traefik (k3s default) |
| TLS | cert-manager + Let's Encrypt (Cloudflare DNS-01) |
| Databases | CloudNativePG |
| Monitoring | kube-prometheus-stack, fail2ban exporter |
| Updates | Renovate (self-hosted CronJob) |
| Backups | rsync CronJobs to a Hetzner Storage Box |

## Architecture

App-of-apps pattern. Nothing is applied by hand; change the manifests and push.

- [`argocd-sources/`](argocd-sources/) installs Argo CD from a pinned upstream `install.yaml`. The `argocd-self` Application points back at it, so Argo CD manages itself.
- [`argocd-applications/kustomization.yaml`](argocd-applications/kustomization.yaml) references every Application. Disable a workload by commenting out its line.
- `*_app.yaml` is the Argo CD Application (usually an upstream Helm chart, often multi-source with values from this repo). `*_resources_app.yaml` deploys the extra manifests (ingresses, volumes, network policies, cronjobs).
- [`pg_dbs_resources_app.yaml`](argocd-applications/postgres/pg_dbs_resources_app.yaml) is an ApplicationSet that creates one `<name>-pg-db-resources` Application per database from a list generator.

## Workloads

| Service | Version | Status | Notes |
|---------|---------|--------|-------|
| Immich | chart 0.12.0, server v3.3.1 | enabled | CNPG Postgres with `vchord`, Valkey. Backed up daily. |
| Nextcloud | chart 9.4.0 | enabled | CNPG Postgres 18. Backed up daily (DB dump + data). |
| Jellyfin | chart 3.2.0 | enabled | Media server. |
| qBittorrent | 5.2.4 | enabled | All traffic through a Gluetun VPN sidecar. |
| Pi-hole | chart 2.38.0, image 2026.09.0 | enabled | DNS ad-blocking. See [pihole/README.md](argocd-applications/pihole/README.md) for freeing port 53. |
| cert-manager | v1.21.2 | enabled | Let's Encrypt via Cloudflare DNS-01. |
| CloudNativePG | chart 0.29.1 | enabled | Postgres operator for Immich and Nextcloud. |
| kube-prometheus-stack | chart 92.2.0 | enabled | Prometheus, Grafana, Alertmanager. |
| fail2ban exporter | 0.10.3 | enabled | fail2ban metrics for Prometheus. |
| Renovate | chart 46.352.0 | enabled | Dependency update PRs. |
| Home Assistant | chart 0.3.85 | disabled | `hostNetwork` for discovery, plus a Matter server. |
| Prowlarr, Radarr | latest | disabled | *arr stack (Prowlarr also deploys FlareSolverr). |
| Minecraft | chart 5.2.0 | disabled | Game server. |

Status reflects whether the Application is referenced in [`argocd-applications/kustomization.yaml`](argocd-applications/kustomization.yaml).

## Storage and Databases

- Persistent data uses statically provisioned `hostPath` PVs bound to PVCs (`storageClassName: ""`, explicit `volumeName`) with `Retain`, so data survives an Application being deleted.
- The CNPG operator runs in `cnpg-system`. Per-app `Cluster` resources live in `postgres/<app>_pg_db_resources/`. Add a database by adding a `name` to the ApplicationSet list generator.

## Backups

Daily CronJobs at 03:00 `Europe/Zurich` rsync data over SSH (port 23) to a Hetzner Storage Box.

| Job | What is backed up | Remote path |
|-----|-------------------|-------------|
| [`immich-backup`](argocd-applications/immich/immich_resources/cronjob.yaml) | Immich volume | `immich/` |
| [`nextcloud-backup`](argocd-applications/nextcloud/nextcloud_resources/cronjob.yaml) | `pg_dump` of the Nextcloud DB (init container), then the Nextcloud volume | `nextcloud/db/`, `nextcloud/data/` |

> [!WARNING]
> rsync runs with `--delete`, so the remote is a mirror of the current state, not a versioned history. Only the latest Nextcloud DB dump is kept.

- The DB dump is a custom-format archive: restore with `pg_restore`, not `gunzip`.
- Volumes are mounted read-only. Nextcloud files are read through `supplementalGroups: [33]` (www-data) instead of extra capabilities.

## Security

- **Pod hardening** on the backup jobs, fail2ban exporter and Renovate: `automountServiceAccountToken: false`, all capabilities dropped, no privilege escalation, `seccompProfile: RuntimeDefault`, and `readOnlyRootFilesystem` / `runAsNonRoot` where the image allows.
- **NetworkPolicies:** only Immich components and the CNPG operator can reach the Immich Postgres on 5432. The fail2ban exporter accepts traffic only from Prometheus on 9191 and has no egress.
- **Pinned images:** enabled workloads use explicit tags (the fail2ban exporter no longer uses `latest`), so every change goes through a Renovate PR.
- **TLS:** Traefik terminates TLS; Argo CD runs with `server.insecure: "true"` behind it on purpose. DNS-01 avoids exposing HTTP-01 challenge endpoints.
- **Host network** is used only where needed: Home Assistant and the Matter server (mDNS discovery).

## Dependency Updates

Self-hosted [Renovate](https://docs.renovatebot.com/) from `ghcr.io/renovatebot/charts/renovate`, configured in [`values.yaml`](argocd-applications/renovate/renovate_resources/values.yaml).

- Runs daily at 02:00 `Europe/Zurich`.
- Targets only the private production repo (autodiscovery off), max 5 open PRs, with a dependency dashboard issue.
- No `renovate.json` in the repo; all config is global.

| Manager | Matches |
|---------|---------|
| `argocd` | `chart` / `targetRevision` in Applications |
| `kubernetes` | `image: repo:tag` in plain manifests |
| custom regex | Tags inside an Application's `helm.values`, marked with a `# renovate:` comment |
| custom regex | The Argo CD `install.yaml` URL in [`argocd-sources/kustomization.yaml`](argocd-sources/kustomization.yaml) |

```yaml
# renovate: datasource=docker depName=pihole/pihole versioning=loose
tag: "2026.09.0"
```

Chart bumps are labelled `helm-chart`, `lscr.io/linuxserver/*` images are grouped into one PR, and major bumps of cert-manager, CloudNativePG, Immich and Nextcloud need dashboard approval.

## Repository Structure

```
.
├── argocd-sources/                 # Argo CD install (pinned) + ingress + patches
└── argocd-applications/
    ├── kustomization.yaml          # Root: references every Application
    ├── argo/                       # argocd-self
    ├── cert-manager/               # Chart + ClusterIssuer
    ├── home-assistant/             # Chart + Matter server
    ├── immich/                     # Chart + ingresses, volume, backup cronjob
    ├── media/                      # jellyfin, qbittorrent, prowlarr, radarr
    ├── minecraft/
    ├── monitoring/                 # kube-prometheus-stack, fail2ban exporter
    ├── nextcloud/                  # Chart + volumes, backup cronjob
    ├── pihole/
    ├── postgres/                   # CNPG operator + per-database ApplicationSet
    └── renovate/                   # Chart + values
```

## License

[MIT](LICENSE)
