---
name: media-downloader
description: Use this skill to download any movie or TV show using Radarr/Sonarr on the k3s cluster. It ensures quality (1080p), size constraints (<4GB), and automatically cleans up original download files after import.
---

# Media Downloader Workflow

This skill provides a standardized, automated workflow for downloading media while adhering to specific quality and storage constraints.

## Core Rules
1. **Quality**: Always prefer 1080p (BluRay or Web-DL).
2. **Size**: Never exceed 4GB for a single movie file.
3. **Storage**: Data is stored in `/home/obiwan/k8s_data/`.
4. **Cleanup**: Always delete the original torrent and files from qBittorrent once the file has been imported to the final library folder (e.g., `/home/obiwan/movies/`).

## Credentials & Endpoints
- **Radarr**: `http://radarr.media.svc.cluster.local:7878` (API Key: `fdd37144015d40b18a12bab9b000434b`)
- **Prowlarr**: `http://prowlarr.media.svc.cluster.local:9696` (API Key: `4e15b52c12134a65aac2ad3e58b92935`)
- **qBittorrent**: `http://qbittorrent.media.svc.cluster.local:8080` (API Key: N/A - uses local auth or no-auth)

## Standardized Steps

### 1. Lookup & Add
Search Radarr/Sonarr for the requested title. If not found in the library, add it with:
- **Quality Profile**: HD-1080p (ID: 4)
- **Root Folder**: `/movies` (ID: 2)
- **Monitoring**: Enabled
- **Search on Add**: True

### 2. Monitor Search
Trigger a search for the title and monitor the releases. Filter for:
- `1080p` in quality name.
- `size <= 4GB`.
- `seeders >= 1`.

### 3. Verification & Cleanup
Check history for `movieFileImported`. Once the file exists in `/home/obiwan/movies/<TITLE>/`, find the torrent in qBittorrent by hash or name and delete it including files:
```bash
curl -X POST "http://qbittorrent.media.svc.cluster.local:8080/api/v2/torrents/delete" -d "hashes=<HASH>&deleteFiles=true"
```
