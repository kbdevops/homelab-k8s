# Homelab K8s

Personal homelab running on k3s with GitOps via ArgoCD.

## Stack

| Service | Description | URL |
|---------|-------------|-----|
| Jellyfin | Media Server | http://jellyfin.internal |
| Sonarr | TV Management | http://sonarr.internal |
| Radarr | Movie Management | http://radarr.internal |
| Prowlarr | Indexer Management | http://prowlarr.internal |
| qBittorrent | Torrent Client | http://qbit.internal |
| Bazarr | Subtitle Management | http://bazarr.internal |
| Home Assistant | Home Automation | http://ha.internal |
| Homepage | Dashboard | http://homepage.internal |
| AdGuard Home | DNS Ad Blocker | http://adguard.internal |
| ArgoCD | GitOps | http://argocd.internal |

## Structure

base/ - All k8s manifests
bootstrap/ - One-time cluster bootstrap (ArgoCD)

## Bootstrap a fresh cluster

1. Install k3s: curl -sfL https://get.k3s.io | sh -
2. Bootstrap ArgoCD: kubectl apply -k bootstrap/argocd/
3. ArgoCD will automatically deploy everything else from base/

## Data

All persistent data stored in /home/obiwan/k8s_data/ on the host.

## GitOps Flow

git push to main -> ArgoCD detects change -> kubectl apply -k base/
