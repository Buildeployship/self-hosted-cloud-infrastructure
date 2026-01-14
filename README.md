# Self-Hosted Cloud Infrastructure

Personal cloud storage platform replacing Google Drive.
Fully self-hosted on Arch Linux with 16TB storage.

**Live:** https://drive.buildeployship-nas.dev

## Architecture

- **Nextcloud** — File sync, sharing, multi-user access
- **PostgreSQL 16** — Database backend
- **Docker Compose** — Container orchestration
- **Cloudflare Tunnel** — Secure remote access (no exposed ports)
- **Custom Domain** — drive.buildeployship-nas.dev

```
┌────────────────────────────────────────────────────────┐
│                   Arch Linux Host                      │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │            Docker Compose Stack                  │  │
│  │                                                  │  │
│  │  ┌─────────────┐      ┌─────────────┐           │  │
│  │  │  Nextcloud  │◄────►│ PostgreSQL  │           │  │
│  │  └─────────────┘      └─────────────┘           │  │
│  │                                                  │  │
│  └──────────────────────────────────────────────────┘  │
│                          │                             │
│                          ▼                             │
│               ┌─────────────────────┐                  │
│               │  16TB HDD Storage   │                  │
│               │  (LUKS Encrypted)   │                  │
│               └─────────────────────┘                  │
│                                                        │
└────────────────────────────────────────────────────────┘
                           │
                           │ Cloudflare Tunnel
                           ▼
┌────────────────────────────────────────────────────────┐
│              Cloudflare Edge Network                   │
│                                                        │
│    • SSL/TLS termination                               │
│    • DDoS protection                                   │
│    • DNS routing                                       │
│                                                        │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
                       Internet
              (drive.buildeployship-nas.dev)
```

## Tech Stack / Infrastructure

| Component | Technology |
|-----------|------------|
| OS | Arch Linux |
| Containers | Docker, Docker Compose |
| Application | Nextcloud |
| Database | PostgreSQL 16 |
| Storage | 16TB LUKS-encrypted HDD |
| Proxy | Cloudflare Tunnel |
| DNS/SSL | Cloudflare |

## Features

- Multi-user support with granular permissions
- Mobile and desktop sync clients
- Secure remote access (no exposed ports/IP)
- Automatic SSL certificates
- File versioning and trash recovery

## File Structure
```
~/nextcloud/                        # Config (docker-compose, db)
/mnt/16TB_HDD_1/nextcloud-data/     # Server storage
/mnt/16TB_HDD_1/nextcloud-sync/     # Desktop sync folder
```

## Setup Guide

<details>
<summary><strong>1. Install Docker</strong></summary>
```bash
sudo pacman -S docker docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```
Log out and back in for group changes.

</details>

<details>
<summary><strong>2. Create Project Structure</strong></summary>
```bash
mkdir -p ~/nextcloud
cd ~/nextcloud
```

</details>

<details>
<summary><strong>3. Configure docker-compose.yml</strong></summary>
```yaml
services:
  nextcloud:
    image: nextcloud
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - /mnt/YOUR_HDD/nextcloud-data:/var/www/html
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=CHANGE_ME

  db:
    image: postgres:16
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=CHANGE_ME
    volumes:
      - ./db:/var/lib/postgresql/data
```

</details>

<details>
<summary><strong>4. Launch Containers</strong></summary>
```bash
docker-compose up -d
```
Access at `http://localhost:8080`

</details>

<details>
<summary><strong>5. Configure Cloudflare Tunnel</strong></summary>
```bash
# Install
yay -S cloudflared

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create nextcloud

# Configure (~/.cloudflared/config.yml)
tunnel: YOUR_TUNNEL_ID
credentials-file: /home/USER/.cloudflared/YOUR_TUNNEL_ID.json

ingress:
  - hostname: drive.yourdomain.com
    service: http://localhost:8080
  - service: http_status:404

# Create DNS route
cloudflared tunnel route dns nextcloud drive.yourdomain.com

# Run
cloudflared tunnel run nextcloud
```

</details>

## What I Learned

- Container orchestration with Docker Compose
- Database selection and migration (MariaDB → PostgreSQL)
- Volume management and persistent storage
- Reverse proxy and tunnel architecture
- DNS configuration and domain management
- User access control and sharing permissions
- Infrastructure-as-code patterns
- Secure remote access without port forwarding