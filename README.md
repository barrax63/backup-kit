# ZeroByte Backup Kit

This Docker setup provides a production-ready ZeroByte deployment, optimized for security and resource management.

## Features

- ZeroByte service with FUSE mount support
- Nginx reverse proxy with HTTPS (self-signed certificates)
- Virtual host routing
- HTTP to HTTPS automatic redirect
- Security hardened: no-new-privileges, AppArmor, dropped capabilities
- Minimal added capabilities: SYS_ADMIN for FUSE, NET_BIND_SERVICE for nginx
- Structured logging: JSON logging with rotation (10MB max, 5 files)
- Resource limits: CPU/memory constraints for all services
- Health checks for all services

## Directory Structure

```
.
├── docker-compose.yml     # Service orchestration
├── nginx/
│   ├── nginx.conf         # Nginx configuration
│   └── certs/
│       ├── server.crt     # SSL certificate
│       └── server.key     # SSL private key
├── LICENSE                # License file
└── README.md              # This file
```

## Setup Instructions

### 1. Clone the repository

```bash
git clone https://github.com/barrax63/backup-kit.git
cd backup-kit
```

### 2. Generate self-signed certificates

```bash
mkdir -p nginx/certs

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=zerobyte.fritz.box" \
  -addext "subjectAltName=DNS:zerobyte.fritz.box,DNS:*.fritz.box,DNS:localhost,IP:127.0.0.1"

chmod 644 nginx/certs/server.key
chmod 644 nginx/certs/server.crt
```

### 3. Configure DNS

Add a DNS entry in your Fritz!Box or local DNS server:

| Hostname           | Target IP           |
|--------------------|---------------------|
| zerobyte.fritz.box | Your Docker host IP |

### 4. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f zerobyte
docker compose logs -f nginx
```

### 5. Access the service

Open in browser: `https://zerobyte.fritz.box`

## Maintenance

### Update

```bash
git pull
docker compose up -d --pull always
```

### Restart

```bash
docker compose restart zerobyte nginx
```

### Renew certificates

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=zerobyte.fritz.box" \
  -addext "subjectAltName=DNS:zerobyte.fritz.box,DNS:*.fritz.box,DNS:localhost,IP:127.0.0.1"

chmod 644 nginx/certs/server.key
chmod 644 nginx/certs/server.crt
docker compose restart nginx
```

### View logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f zerobyte
docker compose logs -f nginx
```

## Adding More Virtual Hosts

To add another service (e.g., `newservice.fritz.box`), add a new server block in `nginx/nginx.conf`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name newservice.fritz.box;

    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    # ... SSL and security headers ...

    location / {
        set $upstream_newservice newservice:8080;
        proxy_pass http://$upstream_newservice;
        # ... proxy headers ...
    }
}
```

Then regenerate the certificate to include the new domain and restart nginx.

## Security Considerations

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only required capabilities are re-added.
2. **AppArmor**: Default Docker AppArmor profile is enforced.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Immutable Config**: Rclone and nginx configurations are mounted read-only.
5. **Resource Limits**: CPU/memory limits and reservations are set for all services.
6. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
7. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.
8. **Rate Limiting**: API rate limiting (10 req/s with burst of 20).
9. **Catch-all Server**: Unknown domains receive no response (HTTP 444).

## Notes and Tips

- The `/dev/fuse` device is required for FUSE mount operations.
- Unknown domains hitting the server will receive a closed connection (444 response).
