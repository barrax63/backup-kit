# ZeroByte Backup Kit

This Docker setup provides a production-ready ZeroByte deployment with rclone integration, optimized for security and resource management.

## Features

- ZeroByte service with FUSE mount support
- Rclone configuration integration for cloud storage backends
- Nginx reverse proxy with HTTPS (self-signed certificates)
- HTTP to HTTPS automatic redirect
- Security hardened: no-new-privileges, AppArmor, dropped capabilities
- Minimal added capabilities: SYS_ADMIN for FUSE, NET_BIND_SERVICE for nginx
- Structured logging: JSON logging with rotation (10MB max, 5 files)
- Resource limits: CPU/memory constraints for all services
- Health checks for nginx service

## Directory Structure

```
.
├── docker-compose.yml     # Service orchestration
├── .env.example           # Environment variables example
├── .env                   # Environment variables (required)
├── rclone/
│   └── rclone.conf        # rclone configuration (add this!)
├── nginx/
│   ├── nginx.conf         # Nginx configuration
│   └── certs/
│       ├── server.crt     # SSL certificate (add this!)
│       └── server.key     # SSL private key (add this!)
├── LICENSE                # License file
└── README.md              # This file
```

## Setup Instructions

### 1. Clone the repository

```bash
git clone https://github.com/barrax63/backup-kit.git
cd backup-kit
```

### 2. Configure environment variables

Create a `.env` file with the following variables:

```bash
TZ=Europe/London
RCLONE_CONFIG_PASS=your_rclone_config_password
RCLONE_CONFIG_PATH=/path/to/your/rclone/dir
```

### 3. Generate self-signed certificates

```bash
mkdir -p nginx/certs

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=zerobyte.local" \
  -addext "subjectAltName=DNS:zerobyte.local,DNS:localhost,IP:127.0.0.1"

chmod 600 nginx/certs/server.key
chmod 644 nginx/certs/server.crt
```

### 4. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f zerobyte
docker compose logs -f nginx
```

### 5. Access the service

```bash
# Via HTTPS (recommended)
curl -k https://localhost

# HTTP will redirect to HTTPS automatically
```

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
  -subj "/C=US/ST=State/L=City/O=Organization/CN=zerobyte.local" \
  -addext "subjectAltName=DNS:zerobyte.local,DNS:localhost,IP:127.0.0.1"

docker compose restart nginx
```

## Security Considerations

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only required capabilities are re-added.
2. **AppArmor**: Default Docker AppArmor profile is enforced.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Immutable Config**: Rclone and nginx configurations are mounted read-only.
5. **Resource Limits**: CPU/memory limits and reservations are set for all services.
6. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
7. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.

## Notes and Tips

- The `/dev/fuse` device is required for FUSE mount operations.
- The rclone config password is used to decrypt an encrypted rclone configuration file.
- The `-k` flag in curl skips certificate verification for self-signed certs.
