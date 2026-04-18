# MW Docker Stack — Migration Runbook

## Directory layout expected on the new server

```
/opt/mw/
├── docker-compose.yml      ← from this repo
├── .env                    ← fill from .env.template (never committed)
├── nginx/conf.d/           ← from this repo
├── timecheck/              ← git clone HammazoneRecords/Time-Check-App
├── solob-portal/           ← git clone HammazoneRecords/Book-of-Solobility (solob-portal/ subdir)
├── mindwaveja/             ← git clone HammazoneRecords/Mindwaveja.com (website/ subdir)
├── marcus-app/             ← git clone HammazoneRecords/MarcusGarvey-App-WWMD
└── melissa/                ← git clone HammazoneRecords/melissa_relief_index
```

---

## Step 1 — New server setup

```bash
# Install Docker only — nothing else needed
apt update && apt install -y docker.io docker-compose-plugin

# Create working directory
mkdir -p /opt/mw && cd /opt/mw
```

---

## Step 2 — Clone all repos

```bash
cd /opt/mw

git clone https://github.com/HammazoneRecords/Time-Check-App.git timecheck
git clone https://github.com/HammazoneRecords/Book-of-Solobility.git solob-portal-repo
cp -r solob-portal-repo/solob-portal ./solob-portal

git clone https://github.com/HammazoneRecords/Mindwaveja.com.git mindwaveja-repo
cp -r mindwaveja-repo/website ./mindwaveja

git clone https://github.com/HammazoneRecords/MarcusGarvey-App-WWMD.git marcus-app
git clone https://github.com/HammazoneRecords/melissa_relief_index.git melissa

# Clone this infra repo
git clone https://github.com/HammazoneRecords/vps-infra.git .
```

---

## Step 3 — Restore data volumes

```bash
# Restore Time-Check PostgreSQL
# (Run after postgres container is created but before timecheck starts)
docker-compose up -d postgres
sleep 5
docker exec -i mw-postgres psql -U timecheck timecheck < /path/to/timecheck-YYYY-MM-DD.sql

# Restore solobility.db
docker-compose up -d solob
docker cp /path/to/solobility-YYYY-MM-DD.db mw-solob:/app/data/solobility.db

# Restore Marcus DBs
mkdir -p marcus-app/backend/data
cp /path/to/marcus-memory-YYYY-MM-DD.db marcus-app/backend/data/memory.db
cp /path/to/marcus-nodes-YYYY-MM-DD.db marcus-app/backend/data/nodes.db
```

---

## Step 4 — Set environment variables

```bash
cp .env.template .env
nano .env
# Fill in all values from vault/vps-backups or 1Password
```

Values needed:
- `POSTGRES_PASSWORD` — from old VPS pm2 env (tc_prod_2026_secure)
- `SESSION_SECRET` — from old VPS pm2 env
- `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_EMAIL` — from old VPS
- `ZITADEL_MASTERKEY` — from Zitadel setup
- `ZITADEL_DB_PASSWORD` — from old Zitadel docker-compose

---

## Step 5 — Point DNS to new server IP

Update A records for all domains before starting containers.
Wait for DNS propagation (~5 min for Cloudflare, up to 30 min others).

---

## Step 6 — Start everything

```bash
cd /opt/mw

# First run — build all images
docker-compose up -d --build

# Watch logs for errors
docker-compose logs -f
```

---

## Step 7 — Issue SSL certs

```bash
# Run Certbot for each domain
docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d whatissolob.com -d www.whatissolob.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d mindwaveja.com -d www.mindwaveja.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d timecheck.mindwaveja.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d mri.mindwaveja.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d marcusgarvey876.com -d www.marcusgarvey876.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d auth.mindwaveja.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d loveref.mindwaveja.com \
  --email ovandobrown@gmail.com --agree-tos --no-eff-email

# Reload nginx to pick up certs
docker-compose exec nginx nginx -s reload
```

---

## Step 8 — Verify all apps

```bash
curl -s -o /dev/null -w "%{http_code}" https://whatissolob.com
curl -s -o /dev/null -w "%{http_code}" https://mindwaveja.com
curl -s -o /dev/null -w "%{http_code}" https://timecheck.mindwaveja.com/api/auth/user
curl -s -o /dev/null -w "%{http_code}" https://mri.mindwaveja.com
curl -s -o /dev/null -w "%{http_code}" https://marcusgarvey876.com
curl -s -o /dev/null -w "%{http_code}" https://auth.mindwaveja.com
```

---

## Step 9 — Decommission old VPS

Only after all 8 checks above return 200:

```bash
# On old VPS — stop everything cleanly
pm2 stop all
docker stop $(docker ps -q)
```

Then cancel the Contabo subscription.

---

## Ongoing — Deploy updates

```bash
# Update a single app
cd /opt/mw/timecheck && git pull
docker-compose up -d --build timecheck

# Update all
docker-compose pull && docker-compose up -d --build
```
