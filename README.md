# 202512221823-keycloak-prod

Production Keycloak setup with Docker Compose, Traefik reverse proxy, PostgreSQL database, and monitoring stack (Grafana, Loki, Portainer).

## Features

- **Keycloak**: Open Source Identity and Access Management
- **PostgreSQL**: Database backend for Keycloak
- **Traefik**: Reverse proxy with automatic SSL/TLS certificates
- **Grafana**: Monitoring and visualization
- **Loki**: Log aggregation
- **Portainer**: Docker container management UI

## Quick Start

1. Copy the example environment file and configure your settings:
   ```bash
   cp env_example .env
   ```

2. Edit `.env` file with your specific configuration:
   - Update passwords (all `changeme_*` values)
   - Set your domain names
   - Configure email for Let's Encrypt

3. Start the services:
   ```bash
   docker-compose up -d
   ```

4. Access the services:
   - Keycloak: `https://<KC_HOSTNAME>`
   - Grafana: `https://<GF_SERVER_DOMAIN>`
   - Portainer: `https://portainer.<KC_HOSTNAME>`
   - Traefik Dashboard: `https://traefik.<KC_HOSTNAME>`

## Configuration

All configuration is managed through environment variables in the `.env` file. See `env_example` for all available options.

### Generating Hashed Passwords

For Traefik dashboard BasicAuth, you need to generate a hashed password:

```bash
# Using htpasswd (recommended)
echo $(htpasswd -nb admin yourpassword) | sed -e s/\\$/\\$\\$/g

# Or use an online generator and escape $ as $$
# https://hostingcanada.org/htpasswd-generator/
```

Then set `TRAEFIK_DASHBOARD_PASSWORD` in `.env` to the generated hash (e.g., `admin:$$apr1$$...`).

### Important Security Notes

- **Change all default passwords** in `.env` file before deployment
- **Use hashed passwords** for Traefik BasicAuth (not plain text)
- The `.env` file is gitignored and should never be committed to version control
- Data directories are gitignored to prevent accidental commits of sensitive data
- For production use:
  - Ensure proper firewall rules are in place
  - Consider restricting Portainer access or using alternative authentication methods
  - Review and harden Traefik configuration based on your security requirements
  - Use strong, unique passwords for all services
  - Regularly update container images for security patches

## Data Persistence

Data is persisted in the following directories:
- `keycloak/data/` - Keycloak data
- `postgres/data/` - PostgreSQL database
- `portainer/data/` - Portainer configuration
- `loki/data/` - Loki logs
- `grafana/data/` - Grafana dashboards and configuration
- `traefik/letsencrypt/` - SSL certificates

## Maintenance

### Backup

To backup your data, backup the data directories listed above along with your `.env` file (store securely).

### Updates

To update the services:
```bash
docker-compose pull
docker-compose up -d
```

### Logs

View logs for all services:
```bash
docker-compose logs -f
```

View logs for a specific service:
```bash
docker-compose logs -f keycloak
```

## Troubleshooting

If services fail to start:
1. Check logs: `docker-compose logs`
2. Verify `.env` file configuration
3. Ensure required ports (80, 443) are available
4. Check data directory permissions

## License

See repository license file.