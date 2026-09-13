# biobot-infrastructure

Docker Compose stack for the BioBot backend: MariaDB, the `biobot-cloud`
API, and the `biobot-dashboard` web app. The two applications are git
submodules.

## First run

```sh
git clone --recurse-submodules https://github.com/biobotproject-org/biobot-infrastructure
cd biobot-infrastructure
cp .env.example .env        # then fill in real values
docker compose up -d --build
```

The API listens on port 3000 and the dashboard on port 80. The dashboard's
nginx proxies `/api/` to the backend container, so in production only the
dashboard port needs to be exposed (see *Production* below).

## Secrets

`.env` holds the database passwords, the JWT signing secret, and the SMTP
settings for incident emails. Every value is documented in `.env.example`.
It is git-ignored and must never be committed. Generate values with
`openssl rand -base64 32`.

The Notehub route authenticates with an **ingest key** created in the
dashboard (API Keys, New key, scope Notehub ingest). Such a key can only
deliver sensor data to `POST /ingest/notehub`; it is stored hashed and can
be revoked from the same page. `NOTEHUB_INGEST_TOKEN` in `.env` is
optional and legacy: if set, the API still accepts it as a fallback and
logs a deprecation warning.

## Simulating a node

```sh
INGEST_TOKEN=<ingest key from the dashboard> python3 biobot-cloud/scripts/notehub-sim.py --device biobot-010 --name "Test Ridge"
```

Add `--leave-open` to stop after the escalation and keep the incident
open for dashboard work.

Earlier revisions of this repository committed a real `.env` and a zip
archive containing another one. Those values must be treated as public:
rotate the database passwords and the JWT secret on any server that used
them, and purge the files from history with `git filter-repo` so clones
stop carrying them.

## Updating the applications

```sh
git submodule update --remote
docker compose up -d --build
```

## Production

The public instance runs at <https://app.biobotproject.org> on the same host
as the Laravel site. The stack lives in `/opt/biobot/biobot-infrastructure`
and is reached through the host nginx and a Let's Encrypt certificate; the
containers themselves are bound to loopback only.

```
browser -> host nginx (443, app.biobotproject.org)
        -> 127.0.0.1:8085 frontend container (nginx, React build)
             -> /api/ -> backend:3000 (compose network only)
                           -> db:3306   (compose network only)
```

### Override file

`docker-compose.override.yml` is server-specific and git-ignored. Copy
`docker-compose.prod.example.yml` to that name on the server. It uses the
Compose `!override` tag to replace the `ports` lists of the base file:
the frontend is published on `127.0.0.1:8085`, the backend is not published,
the database is never published. Check the merged result with
`docker compose config | grep -A4 ports`.

### Reverse proxy and TLS

Host nginx vhost `/etc/nginx/sites-available/app-biobot` proxies
`app.biobotproject.org` to `http://127.0.0.1:8085` with the usual
`X-Forwarded-*` headers. The certificate is a separate Certbot lineage
(`/etc/letsencrypt/live/app.biobotproject.org`) obtained with
`certbot --nginx -d app.biobotproject.org` and renewed by the existing
`certbot.timer`. Nothing in the other vhosts is touched.

### Updating

```sh
cd /opt/biobot/biobot-infrastructure
git pull --recurse-submodules
docker compose up -d --build
docker compose ps
```

`git pull --recurse-submodules` moves the submodules to the commits the
infrastructure repo points at. The backend applies schema changes on start
(`sequelize.sync({ alter: true })`), so no migration step exists yet.

### Backups

A daily cron (03:30 UTC, user `ubuntu`) dumps the database and keeps 14 days:

```
30 3 * * * cd /opt/biobot/biobot-infrastructure && docker compose exec -T db sh -c 'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --single-transaction biobot' | gzip > /opt/biobot/backups/biobot-$(date +\%F).sql.gz && find /opt/biobot/backups -name 'biobot-*.sql.gz' -mtime +14 -delete
```

Restore: `gunzip -c biobot-YYYY-MM-DD.sql.gz | docker compose exec -T db sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" biobot'`.

### Rollback

```sh
sudo rm -f /etc/nginx/sites-enabled/app-biobot /etc/nginx/sites-available/app-biobot
sudo nginx -t && sudo systemctl reload nginx
cd /opt/biobot/biobot-infrastructure && docker compose down      # add -v to drop the DB volume
```
