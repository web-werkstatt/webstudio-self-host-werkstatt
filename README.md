# Webstudio Self-Host Werkstatt

Wiederverwendbares Docker-Compose-Template fuer **Webstudio Builder** auf eigenem Server, hinter Caddy/Traefik-Reverse-Proxy.

Basiert auf [`webstudio-community/webstudio-self-host`](https://github.com/webstudio-community/webstudio-self-host) — angepasst an Werkstatt-Patterns:
- `proxy-network` extern, kein Host-Port-Binding
- Bind-Mounts auf `/data/webstudio/{db,minio,uploads}` statt Named Volumes
- Caddy-Block-Beispiel mit basic_auth (Defense-in-Depth) **+ Wildcard-Subdomain
  fuer Project-Canvas (on-demand TLS, kein DNS-Challenge noetig)**
- `extra_hosts`-Loopback-Fix fuer OAuth-Token-Tausch (Hairpin-NAT-Workaround)
- Publisher- und Nginx-Service weggelassen (V1 ohne Publish)

> **WICHTIG:** Webstudio braucht eine **Wildcard-Subdomain** (`*.webstudio.your-domain.com`)
> fuer Project-Canvas-Iframes — pro Projekt eine eigene Sub-Subdomain. Ohne Wildcard
> kannst Du keine Projekte anlegen oder oeffnen (`NS_ERROR_UNKNOWN_HOST`).
> Volle Anleitung: **[`docs/REVERSE-PROXY.md`](docs/REVERSE-PROXY.md)**.

> **Use-Case:** Single-User-Designwerkstatt, hinter `webstudio.your-domain.com`, fuer visuelles Authoring + statischen Export. **Nicht** fuer Multi-User-Production.

---

## Voraussetzungen

| | |
|---|---|
| OS | Linux (getestet auf Debian 12) |
| Docker | ≥ 24 |
| Docker Compose | v2 |
| RAM | ≥ 2 GB frei |
| Disk | ≥ 10 GB frei |
| DNS | A-Record `webstudio.your-domain.com` → Server-IP **+ Wildcard `*.webstudio.your-domain.com` → Server-IP** |
| Reverse-Proxy | Caddy 2.x mit `on_demand_tls` (Beispiel) oder Traefik/Nginx mit Wildcard-Cert |
| Externes Docker-Network | `proxy-network` (anpassbar, siehe `docker-compose.yml`) |

---

## Quick Start

```bash
# 1. Clonen
git clone https://github.com/web-werkstatt/webstudio-self-host-werkstatt.git
cd webstudio-self-host-werkstatt

# 2. Konfiguration vorbereiten
cp .env.example .env
# .env editieren — alle "change-me"-Werte ersetzen:
#   - APP_FQDN  (Deine Domain)
#   - POSTGRES_PASSWORD  (openssl rand -hex 32)
#   - PGRST_JWT_SECRET   (openssl rand -hex 64, ≥ 64 Zeichen)
#   - AUTH_SECRET        (openssl rand -hex 32) ← bei DEV_LOGIN gleichzeitig Login-Passwort
#   - TRPC_SERVER_API_TOKEN  (openssl rand -hex 32)
#   - MINIO_ROOT_PASSWORD + S3_SECRET_ACCESS_KEY (gleicher Wert)
#   - DEV_LOGIN_EMAIL    (Deine Login-Email)

# 3. Daten-Verzeichnisse anlegen
sudo mkdir -p /data/webstudio/{db,minio,uploads}
sudo chown -R "$USER:$USER" /data/webstudio

# 4. Externes Docker-Network sicherstellen (falls noch nicht vorhanden)
docker network inspect proxy-network >/dev/null 2>&1 || docker network create proxy-network

# 5. Stack starten
docker compose up -d --wait
```

Nach ~1 Minute sind alle Services healthy. Test:

```bash
docker exec webstudio-app wget -qO- http://127.0.0.1:3000/health
# Erwartet: OK
```

---

## Reverse-Proxy einrichten

**Volle Schritt-fuer-Schritt-Anleitung:** [`docs/REVERSE-PROXY.md`](docs/REVERSE-PROXY.md)
— Wildcard-DNS, Caddy-Snippets, Loopback-Fix, Verify-Tests.

### Caddy (Kurzfassung)

`caddy-block.example.txt` enthaelt vier Bausteine, die alle ins Caddyfile gehoeren:

1. **Globaler Block** mit `on_demand_tls` (kein Wildcard-Cert noetig)
2. **`:9001` Ask-Endpoint** (Schutz gegen Cert-Issuing-Spam)
3. **Apex** `webstudio.your-domain.com` mit `basic_auth @needs_auth not path /oauth/* /auth/*`
4. **Wildcard** `*.webstudio.your-domain.com` mit `tls { on_demand }` — KEIN basic_auth

bcrypt-Hash erzeugen:

```bash
docker run --rm httpd:alpine htpasswd -nbBC 14 USERNAME 'PASSWORT'
```

Caddy reload:

```bash
caddy reload --config /etc/caddy/Caddyfile
```

### Traefik

Vorlage siehe `webstudio-community/webstudio-self-host` README (`docker-compose.coolify.yml`).
Wichtig: Wildcard-Cert via DNS-Challenge fuer `*.webstudio.your-domain.com` (Canvas-Iframes).

### Nginx

Nicht out-of-the-box dabei — Caddy-Block-Pattern ist analog uebertragbar (basic_auth,
gzip, security-headers, proxy_pass mit WebSocket-Upgrade-Header). Wichtig: `Sec-Fetch-Site`
beim proxy_pass auf `same-origin` setzen, sonst blockiert Webstudio mit 400.

---

## Architektur (Kurzform)

```
[Reverse-Proxy mit basic_auth]
        ↓
   webstudio-app  ← Production-Build aus Community-Fork (Port 3000)
        ↓
   webstudio-postgrest  ← REST-API
        ↓                ┌─ webstudio-db (postgres:15-alpine + init-SQL)
        └────────────────┴─ webstudio-minio (S3-kompatibler Asset-Storage)

Run-once Init-Services:
  - webstudio-db-setup  (anon-Rolle + Permissions)
  - webstudio-migrate   (Prisma-Migrations)
  - webstudio-minio-init (S3-Bucket erstellen)
```

Detail in [`docs/ARCHITEKTUR.md`](docs/ARCHITEKTUR.md).

---

## Erstanmeldung

1. Browser oeffnen: `https://webstudio.your-domain.com/`
2. Falls basic_auth aktiv: User+Passwort eingeben
3. Webstudio-DEV_LOGIN-Form:
   - **Auth secret**: Wert von `AUTH_SECRET` aus `.env`
   - **Email**: Wert von `DEV_LOGIN_EMAIL`
   - **Default plan**: `Default` (entspricht `USER_PLAN=Pro` aus ENV)

Browser-Passwort-Manager nutzen, dann ist's beim 2. Mal automatisch.

---

## Update

```bash
docker compose pull
docker compose up -d --wait
```

`webstudio-migrate` faehrt automatisch neue Migrations.

---

## Backup

| Was | Wo | Empfehlung |
|---|---|---|
| Postgres-Daten | `/data/webstudio/db/` | `pg_dump` per Cron oder VM-Snapshot |
| MinIO-Assets | `/data/webstudio/minio/` | rsync nach Backup-Storage |
| Uploads | `/data/webstudio/uploads/` | rsync |
| `.env` | im Repo-Verzeichnis (gitignored) | sicher in Passwort-Manager |
| Caddy-Cert | extern (Caddy-Storage) | nicht im Repo |

---

## Troubleshooting

Bekannte Stolpersteine in [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md). Highlights:

- **HTTP 403 trotz richtiger Caddy-Konfig** → `APP_FQDN` falsch gesetzt; muss exakt zu Reverse-Proxy-Domain passen
- **PostgREST `could not look up local user ID 1000`** → `PGRST_DB_URI` ist nicht gesetzt (PostgREST 12 kennt kein `_FILE`-Suffix); ENV-Variable inline setzen
- **`pnpm dev` aus dem Source-Klon scheitert** → Builder-Source hat hardcoded `wstd.dev` in `vite.config.ts`; Self-Host-Image vom Community-Fork nutzen (das Image hat bereits die Patches)
- **`NS_ERROR_UNKNOWN_HOST` beim Project-Anlegen** → Wildcard-DNS + Wildcard-Caddy-Block fehlt, siehe [`docs/REVERSE-PROXY.md`](docs/REVERSE-PROXY.md)
- **`/error 400` "Cross-origin request"** → `header_up Sec-Fetch-Site "same-origin"` im reverse_proxy fehlt; basic_auth blockt OAuth-Pfade
- **`fetch failed` beim Project-Open** → Hairpin-NAT, `extra_hosts` mit Reverse-Proxy-Container-IP setzen (siehe `.env.example` + `docker-compose.yml`)

---

## Lizenz

MIT — siehe [`LICENSE`](LICENSE).

Webstudio selbst ist AGPL-3.0; das Community-Fork-Image stammt aus
[`webstudio-community/webstudio-fork`](https://github.com/webstudio-community/webstudio-fork)
und steht unter derselben Lizenz.

---

## Quellen / Danksagung

- [`webstudio-is/webstudio`](https://github.com/webstudio-is/webstudio) — Webstudio-Source (AGPL)
- [`webstudio-community/webstudio-self-host`](https://github.com/webstudio-community/webstudio-self-host) — Setup-Vorlage (Coolify-fokussiert)
- [`webstudio-community/webstudio-fork`](https://github.com/webstudio-community/webstudio-fork) — Image mit Self-Host-Patches
- [Webstudio Self-Hosting FAQ](https://docs.webstudio.is/university/self-hosting)
