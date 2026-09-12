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

The API listens on port 3000 and the dashboard on port 80. Putting a TLS
reverse proxy in front and routing `/api` to the backend is tracked in the
project roadmap and is not yet part of this stack.

## Secrets

`.env` holds the database passwords, the JWT signing secret, the token the
Notehub route uses to reach the API, and the SMTP settings for incident
emails. Every value is documented in `.env.example`. It is
git-ignored and must never be committed. Generate values with
`openssl rand -base64 32`.

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
