# Skill 17: Nginx Reverse Proxy and SSL

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 15

## Input Contract
- Node.js API server running on port 3000 inside Docker
- Docker Compose setup from Skill 15
- Domain name (or localhost for development)
- Server accessible on ports 80 and 443

## Output Contract
- `nginx/nginx.conf` — Full reverse proxy configuration with upstream, security headers, rate limiting, WebSocket support
- `nginx/ssl.conf` — Shared TLS configuration with modern cipher suite, HSTS, OCSP stapling
- Self-signed certificate for local development
- Instructions for Let's Encrypt certificate provisioning in production
- Updated `docker-compose.yml` with nginx service

## Files to Create
- `nginx/nginx.conf` — Main nginx configuration
- `nginx/ssl.conf` — Shared SSL/TLS settings
- `nginx/conf.d/upstream.conf` — Upstream server definitions

## Steps

### Step 1: Create Directory Structure

```bash
mkdir -p nginx/conf.d nginx/certs nginx/snippets
```

### Step 2: Create `nginx/nginx.conf`

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 2048;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format with request ID
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20m;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=upload:10m rate=10r/m;

    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=connlimit:10m;

    include /etc/nginx/conf.d/upstream.conf;

    # HTTP → HTTPS redirect
    server {
        listen 80;
        server_name _;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    # HTTPS server
    server {
        listen 443 ssl;
        http2 on;
        server_name _;

        include /etc/nginx/ssl.conf;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'" always;
        add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

        # API routes — rate limited
        location /api/ {
            limit_req zone=api burst=50 nodelay;
            limit_conn connlimit 20;

            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 60s;
        }

        # Auth routes — stricter rate limiting
        location /api/v1/auth/ {
            limit_req zone=auth burst=10 nodelay;

            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Upload routes — separate limit with body size
        location /api/v1/images/upload {
            limit_req zone=upload burst=20 nodelay;

            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            client_max_body_size 20m;
        }

        # WebSocket support
        location /ws {
            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;
        }

        # Health check — no rate limit for monitoring
        location /api/v1/health {
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            access_log off;
        }

        # Default — return 404
        location / {
            return 404;
        }
    }
}
```

### Step 3: Create `nginx/ssl.conf`

```nginx
# TLS configuration — shared across server blocks
ssl_certificate /etc/nginx/certs/fullchain.pem;
ssl_certificate_key /etc/nginx/certs/privkey.pem;

# Modern TLS only (Mozilla Modern configuration)
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

# Session settings
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# Diffie-Hellman parameter (generate with: openssl dhparam -out dhparam.pem 2048)
# ssl_dhparam /etc/nginx/certs/dhparam.pem;

# HSTS (6 months, include subdomains, preload)
add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload" always;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

### Step 4: Create `nginx/conf.d/upstream.conf`

```nginx
upstream api_backend {
    server api:3000;
    keepalive 32;
}
```

### Step 5: Generate Self-Signed Certificate for Development

```bash
# Generate self-signed certificate valid for 365 days
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout nginx/certs/privkey.pem \
  -out nginx/certs/fullchain.pem \
  -subj "/C=US/ST=Dev/L=Dev/O=Dev/CN=localhost"

# Add to .gitignore to never commit certificates
echo "nginx/certs/*.pem" >> .gitignore
```

**IMPORTANT:** Ensure `nginx/certs/*.pem` is in `.gitignore` — self-signed and production certificates must never be committed.

### Step 6: Update `docker-compose.yml` — Add Nginx Service

Add the nginx service to `docker-compose.yml`:

```yaml
services:
  # ... existing services (postgres, redis, api, worker) ...

  nginx:
    image: nginx:1.25-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl.conf:/etc/nginx/ssl.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - certbot_webroot:/var/www/certbot
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-kf", "https://localhost/api/v1/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - app-network

volumes:
  # ... existing volumes ...
  certbot_webroot:

networks:
  app-network:
    driver: bridge
```

### Step 7: Let's Encrypt Instructions for Production

Create a helper script `scripts/certbot.sh`:

```bash
#!/bin/bash
set -e

DOMAIN=${1:-example.com}
EMAIL=${2:-admin@example.com}

echo "Requesting certificate for $DOMAIN ..."

docker run --rm \
  -v certbot_webroot:/var/www/certbot \
  -v certbot_certs:/etc/letsencrypt \
  certbot/certbot:latest \
  certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email "$EMAIL" \
  --agree-tos \
  --no-eff-email \
  -d "$DOMAIN"

echo "Certificate obtained. Update nginx/ssl.conf to point to Let's Encrypt certs:"
echo "  ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;"
echo "  ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;"
echo ""
echo "Then update docker-compose.yml to mount certbot_certs into nginx:"
echo "  - certbot_certs:/etc/letsencrypt:ro"
echo ""
echo "Set up auto-renewal with a cron job:"
echo "  0 0 * * * docker run --rm -v certbot_webroot:/var/www/certbot -v certbot_certs:/etc/letsencrypt certbot/certbot:latest renew --quiet && docker compose exec nginx nginx -s reload"
```

```bash
chmod +x scripts/certbot.sh
```

### Step 8: Test Nginx Configuration Locally

```bash
docker compose config --quiet
docker compose up -d nginx
curl -k https://localhost/api/v1/health
```

Expected: Returns `{"status":"ok",...}` over HTTPS.

Test HTTP redirect:
```bash
curl -I http://localhost/api/v1/health
```
Expected: `301 Moved Permanently` with `Location: https://localhost/api/v1/health`

## Verification

1. **Validate nginx config:**
   ```bash
   docker compose exec nginx nginx -t
   ```
   Expected: `syntax is ok` and `test is successful`

2. **Test HTTPS access:**
   ```bash
   curl -k https://localhost/api/v1/health
   ```
   Expected: `200 OK` with JSON response

3. **Test HTTP → HTTPS redirect:**
   ```bash
   curl -I http://localhost/api/v1/health
   ```
   Expected: `301` redirect to `https://`

4. **Test security headers:**
   ```bash
   curl -kI https://localhost/api/v1/health | grep -i "x-frame\|strict-transport\|x-content-type"
   ```
   Expected: `X-Frame-Options`, `Strict-Transport-Security`, `X-Content-Type-Options` headers present

5. **Test rate limiting:**
   ```bash
   for i in $(seq 1 60); do curl -sk -o /dev/null -w "%{http_code}\n" https://localhost/api/v1/health; done
   ```
   Expected: First ~50 requests return `200`, subsequent requests return `429 Too Many Requests` (burst allows 50 beyond the 30/s rate)

6. **Test large file rejection:**
   ```bash
   dd if=/dev/zero bs=1M count=25 | curl -k -X POST -H "Content-Type: application/json" --data-binary @- https://localhost/api/v1/images/upload
   ```
   Expected: `413 Request Entity Too Large` (exceeds 20MB limit)

## Rollback

1. Remove nginx service from `docker-compose.yml`
2. Delete nginx configuration:
   ```bash
   rm -rf nginx/
   ```
3. Remove certbot volumes:
   ```bash
   docker volume rm certbot_webroot certbot_certs 2>/dev/null; true
   ```
4. Remove certbot script:
   ```bash
   rm scripts/certbot.sh
   ```
5. Remove `nginx/certs/*.pem` from `.gitignore` if added only for this skill
6. Restore direct port exposure on the `api` service (port `3000:3000`)

## ADR-017: Nginx as Reverse Proxy with TLS Termination

**Decision:** Use Nginx as a reverse proxy placed in front of the Node.js application, handling TLS termination, rate limiting, and security headers.

**Reason:** Nginx provides battle-tested HTTP handling, efficient TLS termination, and flexible rate limiting. By offloading these concerns from the Node.js process, the application server focuses solely on business logic. The keepalive connection pool (32 connections) between Nginx and the API reduces TCP overhead.

**Consequences:**
- All external traffic must go through Nginx — the API port should not be exposed directly
- TLS certificates must be managed (self-signed for dev, Let's Encrypt for prod)
- Nginx adds a network hop, but the latency impact is negligible (<1ms local)
- Configuration changes require Nginx reload (`nginx -s reload`)
- WebSocket connections require explicit upgrade handling

**Alternatives Considered:**
- **Caddy:** Automatic HTTPS with simpler config, but less widely adopted and fewer tuning options
- **HAProxy:** Excellent for load balancing but weaker as a reverse proxy for static content and rewrites
- **Node.js handles TLS directly:** Removes Nginx dependency but loses rate limiting, connection pooling, and defense-in-depth
- **Traefik:** Good for dynamic environments but overkill for a single-service deployment