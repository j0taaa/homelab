# Homelab

Self-hosted Docker infrastructure with Traefik reverse proxy and optional Cloudflare Tunnel.

## Architecture

```
Internet → (Cloudflare Tunnel) → Traefik → Services
                                  ↓
                    [bepasty, divyde, crafty, ...]
```

## Services

### Infrastructure (`infra/`)

| Service | Description | Access |
|---------|-------------|--------|
| **Traefik** | Reverse proxy with automatic Docker discovery | Dashboard: `127.0.0.1:8080` |
| **Portainer** | Docker management UI | `127.0.0.1:9000` |
| **Cloudflared** | Secure tunnel to Cloudflare (no port forwarding needed) | Optional profile |

### Applications

| Service | Description | URL |
|---------|-------------|-----|
| **bepasty** | File sharing / pastebin | `bepasty.<domain>` |
| **divyde** | Expense splitting app | `divyde.<domain>` |
| **crafty-controller** | Minecraft server manager | `crafty.<domain>` |
| **word-mastermind** | Word guessing game | `wordmastermind.<domain>` |
| **dev-env** | Development environment | `dev.<domain>` |

## Quick Start

### Prerequisites

- Docker & Docker Compose
- (Optional) Cloudflare account with a tunnel configured

### Setup

1. **Create the shared network:**
   ```bash
   docker network create proxy
   ```

2. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with your values
   ```

3. **Start infrastructure:**
   ```bash
   docker compose -f infra/compose.yml up -d
   
   # With Cloudflare Tunnel:
   docker compose -f infra/compose.yml --profile cloudflare up -d
   ```

4. **Start applications:**
   ```bash
   docker compose -f bepasty/compose.yml up -d
   docker compose -f divyde/compose.yml up -d
   # etc.
   ```

## Environment Variables

All configuration lives in a single `.env` file at the root. Variables use double underscore to separate project from variable name:

```env
# Shared across all services
shared__base_domain=example.com
shared__tz=Etc/UTC

# Infrastructure specific
infra__cf_tunnel_token=xxx
infra__traefik_version=v2.11

# Dev environment specific
dev_env__github_pat=ghp_xxx
```

See `.env.example` for all available options.

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
      - "traefik.http.routers.my-service.rule=Host(`my-service.${shared__base_domain:-example.com}`)"
      - "traefik.http.services.my-service.loadbalancer.server.port=8080"

networks:
  proxy:
    external: true
```

Then add the subdomain in your DNS pointing to your server/tunnel.

## Security

- Management UIs (Traefik, Portainer) are bound to localhost only by default
- Public services are exposed via Cloudflare Tunnel (no direct internet exposure)
- All inter-service communication happens on the isolated `proxy` network
