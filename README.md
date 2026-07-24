# ZeroByte Backup Kit

This Docker setup provides a production-ready ZeroByte deployment, optimized for security and resource management.

## Features

- ZeroByte service with FUSE mount support
- Nginx reverse proxy with HTTPS (self-signed certificates), TLS-only (no plain-HTTP listener is exposed)
- Security hardened: no-new-privileges, AppArmor (where supported by the host), dropped capabilities
- Minimal added capabilities: SYS_ADMIN/DAC_OVERRIDE for zerobyte's FUSE/backup access, CHOWN/SETUID/SETGID for nginx
- Structured logging: JSON logging with rotation (10MB max, 5 files)
- Resource limits: CPU/memory constraints for all services
- Health checks for all services

## Directory Structure

```
.
├── docker-compose.yml     # Service orchestration
├── .env.example           # Variable configuration
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

### 3. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set at minimum:

- `APP_SECRET` — generate with `openssl rand -hex 32`
- `BASE_URL` — the URL you will access the instance at (e.g. `https://zerobyte.fritz.box:4096`)
- `NGINX_PORT` — host port to expose HTTPS on (default `4096`)

### 4. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f zerobyte
docker compose logs -f nginx
```

### 5. Access the service

Open in browser: `https://zerobyte.fritz.box:${NGINX_PORT}` (default: `https://zerobyte.fritz.box:4096`)

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

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only required capabilities are re-added (`SYS_ADMIN`/`DAC_OVERRIDE` for zerobyte's FUSE and full-filesystem backup access; `CHOWN`/`SETUID`/`SETGID` for nginx to drop privileges to its worker user).
2. **AppArmor**: Docker's default AppArmor profile is enforced for nginx. It is **not** used for zerobyte because that profile unconditionally denies the `mount()` syscall, which would break zerobyte's FUSE mounts even with `SYS_ADMIN` granted; zerobyte therefore runs with `apparmor:unconfined`.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Read-Only Filesystem**: nginx's container root filesystem is mounted read-only (with `tmpfs` for its cache/runtime directories); its configuration is also mounted read-only from the host.
5. **Resource Limits**: CPU/memory limits and reservations are set for all services.
6. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
7. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS, Permissions-Policy enabled.
8. **High-risk zerobyte privileges**: zerobyte runs as root inside its container with `SYS_ADMIN`, `DAC_OVERRIDE`, and a read-write bind mount of the entire host filesystem at `/rootfs`. This is required for it to back up (and restore) arbitrary host files, but it means a compromise of the zerobyte process/image is effectively equivalent to root access on the host. Mitigate by:
   - Only running this stack on a host dedicated to backups, not co-located with other workloads.
   - Restricting network/UI access to the zerobyte service (e.g. VPN, firewall, or an authenticating reverse proxy in front of nginx) — the built-in nginx proxy here does not add authentication.
   - Keeping the `zerobyte` image pinned and updated deliberately, and reviewing changelogs before upgrading given its elevated privileges.
   - If your workflow never needs to restore *to* the host filesystem, consider mounting `/rootfs` as `:ro` — verify this does not break intended restore functionality before changing it.

## Notes

- The `/dev/fuse` device is required for FUSE mount operations.
- `APP_SECRET`, `BASE_URL`, and `TRUST_PROXY` are read from `.env` (see `.env.example`); `TRUST_PROXY=true` should be set since zerobyte is always accessed through the bundled nginx reverse proxy.
