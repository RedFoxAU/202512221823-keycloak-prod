# Keycloak Production Stack

Production-ready Keycloak deployment with Traefik reverse proxy, PostgreSQL, centralized logging (Loki + Grafana), and Cloudflare Tunnel integration.

## Stack Components

- **Traefik v3.4** - Reverse proxy with automatic HTTPS via Cloudflare DNS-01 challenge
- **Keycloak 26.1** - Identity and access management at `auth.${DOMAIN}`
- **PostgreSQL 17** - Backend database with ZFS optimization
- **Portainer CE** - Container management at `manage.${DOMAIN}`
- **Loki 3.2.1** - Log aggregation
- **Grafana 11.2.0** - Monitoring and visualization at `grafana.${DOMAIN}`
- **Cloudflared** - Zero-trust tunnel for secure remote access

## Prerequisites

### Infrastructure
- Proxmox VE host
- Debian 13 (Trixie) LXC container (unprivileged)
**Minimal (1-10 users):** 2 CPU cores, 4GB RAM minimum
- **Recommended (10-100 users):** 4 CPU cores, 8GB RAM minimum
- ZFS pool with `recordsize=16k` for PostgreSQL

### External Services
- Cloudflare account with:
  - Domain managed by Cloudflare DNS
  - API token with DNS edit permissions
  - Cloudflare Tunnel configured

## Deployment

### 1. Create LXC Container

```bash
# On Proxmox host
pct create 300 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname keycloak-prod \
  --cores 2
  --memory 4096
  --swap 1024
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-zfs \
  --rootfs local-z16:32 \
  --unprivileged 1 \
  --features nesting=1
```
```bash
# Set ZFS recordsize for PostgreSQL
zfs set recordsize=16k rpool/data/subvol-300-disk-0

# Start container
pct start 300
pct enter 300


```

### 2. Install Docker

```bash
# Update system
# Install Docker
apt update && apt upgrade -y
apt install -y curl git locales

# Generate locale
locale-gen en_US.UTF-8

# Apply locale settings
export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
curl -fsSL https://get.docker.com | sh

# Enable Docker
systemctl enable --now docker
```

### 3. Clone Repository

```bash
mkdir -p /opt/keycloak
cd /opt/keycloak
git clone https://github.com/RedFoxAU/202512221823-keycloak-prod.git .
```

### 4. Configure Environment

```bash
# Copy example env file
cp .env.example .env

# Edit configuration
nano .env
```

Required variables:
```env
DOMAIN=your-domain.com
CF_DNS_API_TOKEN=your_cloudflare_api_token
CF_TUNNEL_TOKEN=your_cloudflare_tunnel_token
POSTGRES_PASSWORD=strong_random_password
KEYCLOAK_ADMIN_PASSWORD=strong_random_password
GRAFANA_ADMIN_PASSWORD=strong_random_password
```

### 5. Prepare Directories

```bash
# Create required directories
mkdir -p traefik/logs postgres/data portainer/data loki/data grafana/data

# Set proper permissions
chmod 600 traefik/acme.json 2>/dev/null || touch traefik/acme.json && chmod 600 traefik/acme.json
chown -R 472:472 grafana/data  # Grafana UID

# Loki Plugin
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

### 6. Deploy Stack

```bash
# Start all services
docker compose up -d

# Check logs
docker compose logs -f

# Verify all containers running
docker compose ps
```

## Service URLs

After deployment, services are accessible at:

- **Keycloak**: `https://auth.${DOMAIN}`
- **Traefik Dashboard**: `https://traefik.${DOMAIN}`
- **Portainer**: `https://manage.${DOMAIN}`
- **Grafana**: `https://grafana.${DOMAIN}`

## Network Architecture

- **proxy**: External-facing network for Traefik and web services
- **backend**: Internal network for PostgreSQL (isolated)
- **monitoring**: Internal network for Loki and Grafana (isolated)

## Backup Strategy

### PBS Integration

All data is stored in bind mounts at `./` relative paths:
```
/opt/keycloak/
├── postgres/data/       # PostgreSQL database
├── portainer/data/      # Portainer configuration
├── grafana/data/        # Grafana dashboards and settings
├── loki/data/          # Log storage
├── traefik/acme.json   # SSL certificates
└── traefik/logs/       # Access and error logs
```

Proxmox Backup Server will backup the entire LXC container, including all data.

### Manual Backup

```bash
# Stop services
cd /opt/keycloak
docker compose down

# Backup to tarball
tar -czf keycloak-backup-$(date +%Y%m%d).tar.gz \
  .env docker-compose.yml traefik/ postgres/ portainer/ grafana/ loki/

# Restart services
docker compose up -d
```

## Maintenance

### Update Containers

```bash
cd /opt/keycloak
docker compose pull
docker compose up -d
```

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f keycloak

# Via Grafana
# Access Grafana at https://grafana.${DOMAIN}
# Add Loki datasource: http://loki:3100
```

### Database Backup

```bash
# PostgreSQL backup
docker compose exec postgres pg_dump -U keycloak keycloak > backup.sql

# Restore
docker compose exec -T postgres psql -U keycloak keycloak < backup.sql
```

## Troubleshooting

### Traefik Certificate Issues

```bash
# Check Traefik logs
docker compose logs traefik | grep acme

# Verify Cloudflare API token permissions
# Required: Zone:DNS:Edit for all zones

# Reset certificates if needed
rm traefik/acme.json
touch traefik/acme.json && chmod 600 traefik/acme.json
docker compose restart traefik
```

### Keycloak Database Connection

```bash
# Check PostgreSQL health
docker compose exec postgres pg_isready -U keycloak

# Check Keycloak logs
docker compose logs keycloak
```

### Container Won't Start

```bash
# Check container status
docker compose ps

# View errors
docker compose logs [service-name]

# Restart single service
docker compose restart [service-name]
```

## Security Considerations

- All passwords should be strong random strings (32+ characters)
- Traefik dashboard is protected by Cloudflare Tunnel
- Backend database is on isolated internal network
- Monitoring stack is on isolated internal network
- All external services use HTTPS with automatic certificate renewal
- Cloudflare proxy provides DDoS protection and WAF

## License

MIT
