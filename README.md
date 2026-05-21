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

## Bootstrap a fresh cluster

### 1. Install k3s
curl -sfL https://get.k3s.io | sh -
echo 'write-kubeconfig-mode: "0644"' | sudo tee /etc/rancher/k3s/config.yaml
sudo systemctl restart k3s

### 2. Create required data directories
mkdir -p /home/obiwan/k8s_data/jellyfin/config
mkdir -p /home/obiwan/k8s_data/jellyfin/cache
mkdir -p /home/obiwan/k8s_data/sonarr/config
mkdir -p /home/obiwan/k8s_data/radarr/config
mkdir -p /home/obiwan/k8s_data/prowlarr/config
mkdir -p /home/obiwan/k8s_data/qbittorrent/config
mkdir -p /home/obiwan/k8s_data/bazarr/config
mkdir -p /home/obiwan/k8s_data/homeassistant
mkdir -p /home/obiwan/k8s_data/homepage/config/icons
mkdir -p /home/obiwan/k8s_data/adguardhome/conf
mkdir -p /home/obiwan/k8s_data/adguardhome/work
mkdir -p /home/obiwan/movies
mkdir -p /home/obiwan/tv
mkdir -p /home/obiwan/media/downloads

### 3. Restore backups (if rebuilding)
#### Jellyfin
cp jellyfin-backup-YYYYMMDD.zip /home/obiwan/k8s_data/jellyfin/config/data/data/backups/
sudo chown -R 1000:1000 /home/obiwan/k8s_data/jellyfin/config

#### Sonarr/Radarr/Prowlarr
cp sonarr_backup.zip /home/obiwan/k8s_data/sonarr/config/Backups/manual/
cp radarr_backup.zip /home/obiwan/k8s_data/radarr/config/Backups/manual/
cp prowlarr_backup.zip /home/obiwan/k8s_data/prowlarr/config/Backups/manual/

#### Bazarr
cp bazarr_backup.zip /home/obiwan/k8s_data/bazarr/backup/

#### Home Assistant
cp -r /path/to/old/homeassistant/config /home/obiwan/k8s_data/homeassistant/
sudo chown -R obiwan:obiwan /home/obiwan/k8s_data/homeassistant/

### 4. Bootstrap ArgoCD
kubectl apply -k bootstrap/argocd/

### 5. Add GitHub repo secret
kubectl create secret generic homelab-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/kbdevops/homelab-k8s \
  --from-literal=username=kbdevops \
  --from-literal=password=YOUR_GITHUB_TOKEN \
  -n argocd
kubectl label secret -n argocd homelab-repo argocd.argoproj.io/secret-type=repository

### 6. ArgoCD will deploy everything automatically

## Data Layout

/home/obiwan/k8s_data/    - All service config (backed up)
/home/obiwan/movies/      - Movie files
/home/obiwan/tv/          - TV show files
/home/obiwan/media/       - Downloads directory

## AdGuard DNS Rewrites

Point these to 192.168.1.108 in AdGuard:
- jellyfin.internal
- sonarr.internal
- radarr.internal
- prowlarr.internal
- qbit.internal
- bazarr.internal
- ha.internal
- homepage.internal
- adguard.internal
- argocd.internal

## Router

Set primary DNS to 192.168.1.108 (AdGuard) in DHCP settings.
