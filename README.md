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
| Grafana | Monitoring | http://grafana.internal |
| Prometheus | Metrics | http://prometheus.internal |

## Architecture

Your homelab runs a declarative GitOps engine inside a single-node K3s context, managing structured tenant isolation across namespaces. Local DNS resolutions point wildcard internal traffic directly to the host IP, where Traefik handles Layer 7 route matching.

```mermaid
graph TD
    %% Global Styling
    classDef clusterStyle fill:#1a1a1a,stroke:#333,stroke-width:2px,color:#fff;
    classDef nsStyle fill:#24292e,stroke:#444,stroke-width:1px,color:#fff;
    classDef mediaStyle fill:#1D4ED8,stroke:#2563EB,color:#fff;
    classDef monitorStyle fill:#78350F,stroke:#92400E,color:#fff;
    classDef systemStyle fill:#4C1D95,stroke:#5B21B6,color:#fff;
    classDef netStyle fill:#064E3B,stroke:#047857,color:#fff;

    %% External & Network Ingress Flow
    subgraph Traffic_Routing [Network Edge & Ingress]
        direction LR
        Internet((Internet)) -->|Port 80/443| Router[Router NAT]
        LAN[Local LAN] -->|DNS Query :53| AG_DNS[AdGuard DNS<br/>*.internal &rarr; 192.168.1.108]
        
        Router -->|Forward| Traefik[Traefik Ingress Controller<br/>Ports :80 / :443]
        LAN -->|HTTP Traffic| Traefik
    end
    class Traefik,AG_DNS netStyle;

    %% GitOps Automation Loop
    subgraph GitOps_Engine [Continuous Delivery]
        Git[GitHub Repository<br/>kbdevops/homelab-k8s] -->|Webhook / Poll| Argo[ArgoCD Engine<br/>Auto-Sync 60s]
    end
    class Argo systemStyle;

    %% K3s Compute Boundary
    subgraph K3s_Cluster [K3s Single Node Cluster — obiwan @ 192.168.1.108]
        direction TB

        subgraph ns_media [Namespace: media]
            Jellyfin[jellyfin.internal]
            Sonarr[sonarr.internal]
            Radarr[radarr.internal]
            Prowlarr[prowlarr.internal]
            qBit[qbit.internal]
            Bazarr[bazarr.internal]
            HA[ha.internal]
            Homepage[homepage.internal]
        end

        subgraph ns_monitoring [Namespace: monitoring]
            Prom[prometheus.internal]
            Graf[grafana.internal]
            Loki[Loki Engine]
            Tail[Promtail Collector]
            AM[Alertmanager]
        end

        subgraph ns_adguard [Namespace: adguard]
            AGH[adguard.internal]
        end

        subgraph ns_argocd [Namespace: argocd]
            ArgoCore[argocd.internal]
        end
    end
    
    %% Apply Colors to Namespaces
    class K3s_Cluster clusterStyle;
    class ns_media mediaStyle;
    class ns_monitoring monitorStyle;
    class ns_adguard,AGH fill:#7C2D12,stroke:#9A3412,color:#fff;
    class ns_argocd systemStyle;

    %% Connect Ingress and GitOps Engines to Context Targets
    Argo -->|Deploys Manifests| K3s_Cluster
    Traefik -->|Route: /| Homepage
    Traefik -->|Route: /| Jellyfin
    Traefik -->|Route: /| Graf
    Traefik -->|Route: /| ArgoCore
    
    %% Internal Scrape Path Representation
    Tail -.->|Gathers Logs| Loki
    Prom -.->|Scrapes Telemetry| K3s_Cluster

```

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

### 6. ArgoCD deploys everything automatically
ArgoCD watches the repo and deploys:
- base/ -> media + adguard namespaces
- bootstrap/monitoring/ -> monitoring namespace

## Data Layout

/home/obiwan/k8s_data/    - All service config (backed up)
/home/obiwan/movies/      - Movie files
/home/obiwan/tv/          - TV show files
/home/obiwan/media/       - Downloads directory

## Monitoring

- Grafana: http://grafana.internal (obiwan/kenobi)
- Prometheus: http://prometheus.internal
- Dashboards: Node Exporter, Kubernetes Pods, ArgoCD, Traefik, Jellyfin Traffic
- Logs: Loki + Promtail collecting all pod logs
- Jellyfin traffic dashboard shows source IPs, response codes, slow requests

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
- grafana.internal
- prometheus.internal

## Router

- Primary DNS: 192.168.1.108 (AdGuard)
- Port forward: 80 -> 192.168.1.108:80 (Traefik)
- External access: http://161.8.205.5 -> Jellyfin (catch-all)

## Credentials

- ArgoCD: admin / (reset via kubectl patch if forgotten)
- Grafana: obiwan / kenobi
README

cd ~/homelab-k8s
git add .
git commit -m "docs: update README with monitoring stack and full architecture"
git push origin main