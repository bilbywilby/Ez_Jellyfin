# Jellyfin Production Infrastructure

Zero-trust, hardware-accelerated Jellyfin deployment with async telemetry sidecar, staged readiness probing, and full observability.

## Layout

```
docker-compose.yml          Root stack: jellyfin, webhook sidecar, probe, prometheus, nginx
.env.example                 Copy to .env and fill in
config/jellyfin/             config.json (client), logging.json (Serilog structured)
config/nginx/                Reverse proxy: HTTP/2, buffering off, /socket upgrade
deployment/                  Declarative schemas: backup, ingress, metrics, transcode, OIDC, SQLite tuning, security, library naming, ECS task def
sidecar/                     FastAPI webhook ingestion microservice
probe/                       Standalone packaged Python daemon — staged readiness gate + Prometheus exporter
scripts/                     deploy.sh, init-db.sh, backup.sh, healthcheck.py, install-backup-cron.sh
monitoring/                  Prometheus scrape config + alert rules, Grafana dashboard
```

## Bring-up

```bash
cp .env.example .env   # fill in JELLYFIN_API_KEY, NFS paths, OIDC secret
./scripts/deploy.sh
```

`deploy.sh` pulls images, starts the stack, waits for the Jellyfin healthcheck, then applies WAL/mmap pragmas via `init-db.sh`.

## Database

SQLite is forced into WAL with `synchronous=NORMAL`, 2GB mmap, and `read_uncommitted` isolation per `deployment/database_tuning.json`. `scripts/init-db.sh` applies these pragmas idempotently; `scripts/backup.sh` runs `VACUUM` pre-backup and excludes `/cache`, `/log`, `/transcodes`, and `/data/metadata/People` to avoid backup bloat from scraped actor images.

## Hardware transcode

`deployment/hardware_transcode_profiles.json` defines zero-copy QSV (`iHD` driver) and NVENC pipelines with explicit BT.2390 tone-mapping filters — no CPU fallback paths.

## Probe daemon

`probe/` is a self-contained packaged service implementing a seven-stage readiness gate (transport → auth → discovery → administrative-state → liveness → capability → negative-test) rather than treating a single 200 OK as proof of health. Exports Prometheus metrics on `:9108`. See `probe/README.md`.

## Backups

```bash
./scripts/install-backup-cron.sh   # nightly 02:00, matches backup_policy.json
```

## Security baseline

Container runs non-privileged with `cap_drop: ALL` and only `SYS_ADMIN, SETUID, SETGID` added back. Remote API auth enforced (`security_hardening.json`: 14-char minimum password, 5-attempt lockout, 90-day token expiry). OIDC federation template in `identity_provisioning.json`.
