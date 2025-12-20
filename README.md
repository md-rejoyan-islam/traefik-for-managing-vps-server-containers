# Traefik Reverse Proxy - Production Setup

A production-grade Traefik reverse proxy for managing Docker containers with automatic SSL certificates.

## Quick Start

```bash
# 1. Create the shared network
docker network create traefik-proxy

# 2. Start Traefik
docker-compose up -d

# 3. Access dashboard
# Local: http://localhost:8080/dashboard/
# HTTPS: https://traefik.localhost/dashboard/ (admin:admin)
```

## Directory Structure

```
traefik/
├── docker-compose.yml     # Main Traefik service
├── .env                   # Environment variables & credentials
├── acme.json              # SSL certificates (auto-populated)
├── certs/                 # Local self-signed certificates
│   ├── local-cert.pem
│   └── local-key.pem
├── config/
│   ├── traefik.yml        # Static configuration
│   └── dynamic/
│       ├── middlewares.yml # Reusable middlewares
│       └── tls.yml        # TLS settings
└── logs/                  # Access & error logs
```

## Configuration Files

### .env - Environment Variables

```env
TZ=Asia/Dhaka
TRAEFIK_DASHBOARD_CREDENTIALS=admin:$$2y$$05$$...  # admin:admin
ACME_EMAIL=your-email@example.com
TRAEFIK_DOMAIN=traefik.yourdomain.com
```

**Change dashboard password:**

```bash
# Generate new password hash
htpasswd -nbB admin YOUR_NEW_PASSWORD

# Copy the output to .env (escape $ as $$)
# Example: admin:$2y$... becomes admin:$$2y$$...
```

## Connecting Your Containers

### Step 1: Add Network and Labels

Add to your service's `docker-compose.yml`:

```yaml
version: "3.9"

services:
  your-app:
    image: your-image
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.your-app.rule=Host(`your-app.localhost`)"
      - "traefik.http.routers.your-app.entrypoints=websecure"
      - "traefik.http.routers.your-app.tls=true"
      - "traefik.http.services.your-app.loadbalancer.server.port=3000"
    networks:
      - traefik-proxy

networks:
  traefik-proxy:
    external: true
```

### Step 2: For Production with Auto SSL

Add `certresolver` for automatic Let's Encrypt certificates:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.your-app.rule=Host(`app.yourdomain.com`)"
  - "traefik.http.routers.your-app.entrypoints=websecure"
  - "traefik.http.routers.your-app.tls=true"
  - "traefik.http.routers.your-app.tls.certresolver=letsencrypt"
  - "traefik.http.services.your-app.loadbalancer.server.port=3000"
```

**Requirements for Auto SSL:**

- Domain must point to server IP (DNS A record)
- Port 80 must be accessible from internet

## Common Routing Patterns

### Pattern 1: Simple Host-Based Routing

```yaml
# Access: https://myapp.localhost
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`myapp.localhost`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls=true"
  - "traefik.http.services.myapp.loadbalancer.server.port=80"
```

### Pattern 2: Path-Based Routing

```yaml
# Access: https://example.com/api/*
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.api.rule=Host(`example.com`) && PathPrefix(`/api`)"
  - "traefik.http.routers.api.entrypoints=websecure"
  - "traefik.http.routers.api.tls=true"
  - "traefik.http.services.api.loadbalancer.server.port=5000"
  # Strip /api prefix before forwarding
  - "traefik.http.middlewares.api-strip.stripprefix.prefixes=/api"
  - "traefik.http.routers.api.middlewares=api-strip"
```

### Pattern 3: Frontend + Backend Same Domain

```yaml
# Frontend: https://app.localhost
# Backend: https://app.localhost/api/*

services:
  frontend:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`app.localhost`)"
      - "traefik.http.routers.frontend.priority=1"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls=true"
      - "traefik.http.services.frontend.loadbalancer.server.port=3000"

  backend:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`app.localhost`) && PathPrefix(`/api`)"
      - "traefik.http.routers.backend.priority=2"
      - "traefik.http.routers.backend.entrypoints=websecure"
      - "traefik.http.routers.backend.tls=true"
      - "traefik.http.services.backend.loadbalancer.server.port=5000"
```

### Pattern 4: With Middlewares

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.secure-app.rule=Host(`secure.localhost`)"
  - "traefik.http.routers.secure-app.entrypoints=websecure"
  - "traefik.http.routers.secure-app.tls=true"
  - "traefik.http.services.secure-app.loadbalancer.server.port=80"
  # Apply middlewares from config/dynamic/middlewares.yml
  - "traefik.http.routers.secure-app.middlewares=security-headers@file,rate-limit@file,compress@file"
```

## Available Middlewares

Defined in `config/dynamic/middlewares.yml`:

| Middleware               | Usage       | Description                |
| ------------------------ | ----------- | -------------------------- |
| `security-headers@file`  | Security    | HSTS, XSS protection, etc. |
| `compress@file`          | Performance | Gzip compression           |
| `rate-limit@file`        | Security    | 100 req/s limit            |
| `rate-limit-strict@file` | Security    | 10 req/s limit             |
| `cors-headers@file`      | API         | CORS headers               |
| `basic-auth@file`        | Auth        | Basic authentication       |
| `ip-whitelist@file`      | Security    | IP whitelist               |

## SSL Certificates

### Local Development (Self-Signed)

Already configured. Access via `https://*.localhost`

### Production (Let's Encrypt - HTTP Challenge)

For single domains:

```yaml
- "traefik.http.routers.app.tls.certresolver=letsencrypt"
```

### Production (Let's Encrypt - DNS Challenge)

For wildcard certificates (\*.yourdomain.com):

1. Add Cloudflare credentials to `docker-compose.yml`:

```yaml
environment:
  - CF_API_EMAIL=your-cloudflare-email
  - CF_API_KEY=your-cloudflare-api-key
```

2. Use DNS resolver:

```yaml
- "traefik.http.routers.app.tls.certresolver=letsencrypt-dns"
```

## Dashboard Access

| Method     | URL                                  | Credentials |
| ---------- | ------------------------------------ | ----------- |
| Direct     | http://localhost:8080/dashboard/     | None        |
| HTTPS      | https://traefik.localhost/dashboard/ | admin:admin |
| Production | https://TRAEFIK_DOMAIN/dashboard/    | From .env   |

## Useful Commands

```bash
# Start Traefik
docker-compose up -d

# View logs
docker-compose logs -f traefik

# Restart after config changes
docker-compose restart traefik

# Stop Traefik
docker-compose down

# Check running routers
curl -s http://localhost:8080/api/http/routers | jq

# Check services
curl -s http://localhost:8080/api/http/services | jq
```

## Troubleshooting

### Container not appearing in dashboard

1. Check `traefik.enable=true` label exists
2. Verify container is on `traefik-proxy` network
3. Container must be running: `docker ps`

### 502 Bad Gateway

1. Check service port matches container's exposed port
2. Verify container is healthy
3. Ensure same network as Traefik

### SSL Certificate not working

1. Domain must resolve to server IP
2. Port 80 must be open for HTTP challenge
3. Check `acme.json` permissions: `chmod 600 acme.json`
4. View logs: `docker-compose logs traefik | grep acme`

### Access logs location

```bash
# Traefik logs
tail -f logs/traefik.log

# Access logs
tail -f logs/access.log
```

## Production Checklist

- [ ] Change dashboard password in `.env`
- [ ] Update `TRAEFIK_DOMAIN` in `.env`
- [ ] Update email in `config/traefik.yml`
- [ ] Configure DNS A record for your domain
- [ ] Enable `certresolver=letsencrypt` for production domains
- [ ] Consider removing port 8080 exposure
- [ ] Set up log rotation for `logs/` directory
