# Agent Instructions

This document describes project conventions for AI agents working on this homelab codebase.

## Project Structure

```
homelab/
├── .env.example      # Template for all environment variables
├── .env              # Actual config (gitignored)
├── infra/            # Core infrastructure (Traefik, Portainer, Cloudflared)
├── <service>/        # Each service gets its own directory
│   └── compose.yml
└── README.md
```

## Environment Variables

### Naming Convention

All variables use **double underscore** (`__`) to separate project/scope from variable name:

```
<scope>__<variable_name>=value
```

Examples:
- `shared__base_domain` - Used across multiple services
- `infra__cf_tunnel_token` - Infrastructure-specific
- `dev_env__github_pat` - Dev environment specific

### Variable Scopes

| Scope | Purpose |
|-------|---------|
| `shared__` | Variables used by multiple services (domain, timezone) |
| `infra__` | Infrastructure config (Traefik, Portainer, Cloudflare) |
| `dev_env__` | Development environment secrets and config |
| `<service>__` | Service-specific variables when needed |

### Adding New Variables

1. Add to `.env.example` with a descriptive comment
2. Group under the appropriate section header
3. Use lowercase with underscores: `service__my_variable`

## Adding New Services

Every new service must follow this pattern:

### 1. Create Directory Structure

```bash
mkdir <service-name>
touch <service-name>/compose.yml
```

### 2. Compose File Template

```yaml
services:
  <service-name>:
    image: <image>
    container_name: <service-name>
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.<service>.rule=Host(`<subdomain>.${shared__base_domain:-jaypussy.site}`)"
      - "traefik.http.routers.<service>.entrypoints=web"
      - "traefik.http.services.<service>.loadbalancer.server.port=<port>"

networks:
  proxy:
    external: true
```

### 3. Traefik Labels (Required)

Every public-facing service MUST have these labels:

| Label | Purpose |
|-------|---------|
| `traefik.enable=true` | Enable Traefik discovery |
| `traefik.http.routers.<name>.rule=Host(...)` | Domain routing |
| `traefik.http.routers.<name>.entrypoints=web` | Use HTTP entrypoint |
| `traefik.http.services.<name>.loadbalancer.server.port=<port>` | Container port |

### 4. Network Configuration

- All services connect to the `proxy` network (external, created manually)
- Internal-only networks (like databases) should be service-specific:

```yaml
networks:
  proxy:
    external: true
  <service>-internal:
    driver: bridge
```

### 5. Domain Variables

Always use the shared domain variable with a fallback:

```yaml
Host(`myservice.${shared__base_domain:-jaypussy.site}`)
```

## Special Cases

### HTTPS Backend Services

For services with self-signed certs (like Crafty):

```yaml
labels:
  - "traefik.http.services.<name>.loadbalancer.server.scheme=https"
```

### WebSocket Support

Add header middleware for WebSocket-based services:

```yaml
labels:
  - "traefik.http.middlewares.<name>-headers.headers.customrequestheaders.X-Forwarded-Proto=https"
  - "traefik.http.routers.<name>.middlewares=<name>-headers"
```

### Multiple Ports/Subdomains

Define separate routers and services for each:

```yaml
labels:
  # Main service
  - "traefik.http.routers.myapp.rule=Host(`myapp.${shared__base_domain}`)"
  - "traefik.http.services.myapp.loadbalancer.server.port=8080"
  # Secondary service (e.g., API)
  - "traefik.http.routers.myapp-api.rule=Host(`api.${shared__base_domain}`)"
  - "traefik.http.services.myapp-api.loadbalancer.server.port=3000"
```

## Running Services

```bash
# Start infrastructure first
docker compose -f infra/compose.yml up -d

# Start individual services
docker compose -f <service>/compose.yml up -d

# With Cloudflare tunnel
docker compose -f infra/compose.yml --profile cloudflare up -d
```

## Common Patterns

### Database Sidecar

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
    networks:
      - proxy
      - app-internal

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-internal

networks:
  proxy:
    external: true
  app-internal:
    driver: bridge
```

### Build Args from Env

```yaml
services:
  app:
    build:
      context: .
      args:
        MY_ARG: ${service__my_arg}
```

## GitHub Actions Deployment

The workflow at `.github/workflows/deploy.yml` handles automated deployments.

### How It Works

1. Generates `.env` file from GitHub secrets/variables
2. Deploys infrastructure first, then each service
3. Cleans up old Docker images

### When Adding a New Service

You MUST update the workflow:

1. **Add a deploy step** for the new service:

```yaml
- name: Deploy <service-name>
  run: |
    cd <service-name>
    docker compose pull
    docker compose up -d
```

2. **Add after** the infrastructure step but before cleanup

### When Adding New Environment Variables

You MUST update the workflow's "Generate .env file" step:

1. **For secrets** (tokens, API keys, passwords):
   - Add to GitHub repository secrets with UPPERCASE and double underscores
   - Example: `INFRA__CF_TUNNEL_TOKEN`, `DEV_ENV__GITHUB_PAT`
   - Reference in workflow: `${{ secrets.INFRA__CF_TUNNEL_TOKEN }}`

2. **For non-sensitive config** (ports, versions, domains):
   - Add to GitHub repository variables
   - Example: `INFRA__TRAEFIK_VERSION`, `SHARED__BASE_DOMAIN`
   - Reference in workflow: `${{ vars.INFRA__TRAEFIK_VERSION || 'default' }}`

3. **Add the variable** to the heredoc in the "Generate .env file" step:

```yaml
- name: Generate .env file
  run: |
    cat > .env << 'ENVFILE'
    # ... existing vars ...
    new_scope__new_variable=${{ secrets.NEW_SCOPE__NEW_VARIABLE }}
    ENVFILE
```

### GitHub Secrets/Variables Naming

| Local `.env` | GitHub Secret/Variable |
|--------------|------------------------|
| `infra__cf_tunnel_token` | `INFRA__CF_TUNNEL_TOKEN` (secret) |
| `shared__base_domain` | `SHARED__BASE_DOMAIN` or `BASE_DOMAIN` (variable) |
| `dev_env__github_pat` | `DEV_ENV__GITHUB_PAT` (secret) |

### Checklist for New Services

- [ ] Create `<service>/compose.yml` with Traefik labels
- [ ] Add variables to `.env.example` if needed
- [ ] Add deploy step to `.github/workflows/deploy.yml`
- [ ] Add any new secrets/variables to GitHub repository settings
- [ ] Update "Generate .env file" step if new variables added
