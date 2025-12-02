# ZeroByte Backup Kit

This Docker setup provides a production-ready ZeroByte deployment, optimized for security and resource management.

## Features

- ZeroByte service with FUSE mount support
- Nginx reverse proxy with HTTPS (self-signed certificates)
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
├── docker-compose.yml     # Varialbe configuration
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
  -addext "subjectAltName=DNS:zerobyte.fritz.box,DNS:localhost,IP:127.0.0.1"

chmod 644 nginx/certs/server.key
chmod 644 nginx/certs/server.crt
```

### 3. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f zerobyte
docker compose logs -f nginx
```

### 4. Access the service

Open in browser: `https://zerobyte.fritz.box:4096`

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

### View logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f zerobyte
docker compose logs -f nginx
```

## Security Considerations

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only required capabilities are re-added.
2. **AppArmor**: Default Docker AppArmor profile is enforced.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Immutable Config**: Rclone and nginx configurations are mounted read-only.
5. **Resource Limits**: CPU/memory limits and reservations are set for all services.
6. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
7. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.
8. **Rate Limiting**: API rate limiting (10 req/s with burst of 20).

## Notes

- The `/dev/fuse` device is required for FUSE mount operations.
