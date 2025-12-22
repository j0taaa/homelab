# 🏠 Homelab

Self-hosted Docker infrastructure with Traefik reverse proxy and Cloudflare Tunnel.

## Architecture

```
Internet → Cloudflare Tunnel → Traefik → Services
                                  ↓
                    [bepasty, test-nginx, ...]
```

## Services

### Infrastructure (`infra/`)

| Service | Description | Access |
|---------|-------------|--------|
| **Traefik v3** | Reverse proxy with automatic Docker discovery | Dashboard: `<tailscale-ip>:8080` |
| **Portainer** | Docker management UI | `<tailscale-ip>:9000` |
| **Cloudflared** | Secure tunnel to Cloudflare (no port forwarding needed) | - |

### Applications

| Service | Description | URL |
|---------|-------------|-----|
| **bepasty** | File sharing / pastebin | `bepasty.jaypussy.site` |
| **test-nginx** | Test web server | `test.jaypussy.site` |

## Quick Start

### Prerequisites

- Docker & Docker Compose
- Cloudflare account with a tunnel configured
- (Optional) Tailscale for secure access to management UIs

### Setup

1. **Create the shared network:**
   ```bash
   docker network create proxy
   ```

2. **Configure Cloudflare Tunnel token:**
   ```bash
   # Create .env file in infra/
   echo "CF_TUNNEL_TOKEN=your-token-here" > infra/.env
   ```

3. **Start infrastructure:**
   ```bash
   cd infra && docker compose up -d
   ```

4. **Start applications:**
   ```bash
   cd bepasty && docker compose up -d
   cd test-nginx && docker compose up -d
   ```

## Adding New Services

Create a new directory with a `compose.yml`:

```yaml
services:
  my-service:
    image: some-image
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-service.rule=Host(`my-service.jaypussy.site`)"
      - "traefik.http.services.my-service.loadbalancer.server.port=8080"

networks:
  proxy:
    external: true
```

Then add the subdomain in Cloudflare DNS pointing to your tunnel.

## Security

- Management UIs (Traefik, Portainer) are bound to Tailscale IP only
- Public services are exposed via Cloudflare Tunnel (no direct internet exposure)
- All inter-service communication happens on the isolated `proxy` network

