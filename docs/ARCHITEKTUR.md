# Architektur

## Service-Uebersicht

Der Stack besteht aus **7 Services** (4 langlaufend + 3 run-once Init).

### Langlaufende Services

| Service | Image | Port (intern) | Rolle |
|---|---|---|---|
| `webstudio-app` | `ghcr.io/webstudio-community/builder:latest` | 3000 | Webstudio Builder UI (Production-Remix) |
| `webstudio-db` | `postgres:15-alpine` | 5432 | PostgreSQL fuer Builder-State + Project-Daten |
| `webstudio-postgrest` | `postgrest/postgrest:v12.2.0` | 3000 (intern) | REST-API auf der DB (Builder spricht damit) |
| `webstudio-minio` | `minio/minio:latest` | 9000 + 9001 | S3-kompatibler Asset-Storage (Bilder, Fonts) |

### Run-Once Init-Services

| Service | Image | Aufgabe |
|---|---|---|
| `webstudio-db-setup` | `postgres:15-alpine` | Erstellt `anon`-Rolle und gewaehrt Permissions (idempotent) |
| `webstudio-migrate` | `ghcr.io/webstudio-community/builder:latest` | `prisma migrate deploy` — Prisma-Schema-Migrations |
| `webstudio-minio-init` | `minio/mc:latest` | Erstellt S3-Bucket `webstudio-assets` + setzt anonymous-Read |

Alle Init-Services starten beim ersten `up`, terminieren mit Exit 0, und `app` wartet via `depends_on: condition: service_completed_successfully`.

---

## Datenfluss

```
Browser
   |
   v  HTTPS (TLS-Cert via Reverse-Proxy)
[Caddy/Traefik mit basic_auth]
   |
   v  HTTP (intern, proxy-network)
[webstudio-app:3000]  ← Origin-Check gegen APP_FQDN
   |
   ├─→ [webstudio-postgrest:3000]
   │       |
   │       v  postgresql://...
   │   [webstudio-db:5432]
   │
   ├─→ [webstudio-db:5432]  (direkter Prisma-Client)
   │
   └─→ [webstudio-minio:9000]  (S3-API fuer Assets)
```

---

## Persistenz

Drei Bind-Mounts auf dem Host:

| Pfad | Inhalt | Backup-Prio |
|---|---|---|
| `/data/webstudio/db` | PostgreSQL Data Directory | hoch — enthaelt alle Projekt-Daten |
| `/data/webstudio/minio` | MinIO Object Storage | hoch — Bilder, Fonts |
| `/data/webstudio/uploads` | Builder-internes Upload-Cache | mittel |

`docker-compose.yml` mountet zusaetzlich `./postgres-init.sql` read-only in den DB-Container — wird **nur beim ersten Start** ausgefuehrt (Standard-Verhalten von `/docker-entrypoint-initdb.d/`).

---

## Network

Alle Container haengen an einem externen Docker-Network (`proxy-network` per Default). Das erlaubt:
- Reverse-Proxy (Caddy/Traefik in eigenem Compose) erreicht `webstudio-app:3000` per Container-DNS
- Keine Host-Port-Bindings (App-Stack ist nur via Reverse-Proxy erreichbar)
- Andere Services (z.B. Postiz, BentoPDF) im selben Network bleiben isoliert (keine direkte Kommunikation, ausser explizit gewollt)

Wer kein zentrales Reverse-Proxy-Netzwerk hat: in `docker-compose.yml` `external: true` durch ein internes Network ersetzen und Reverse-Proxy ins selbe Compose nehmen.

---

## ENV-Variablen-Verbreitung

```
.env  ──→  docker-compose.yml  ──→  ${VAR}-Substitution  ──→  Container-ENV
```

Wichtige Querverweise:
- `POSTGRES_PASSWORD` taucht in 5 Containern auf (`db`, `db-setup`, `migrate`, `postgrest`, `app`) — muss konsistent sein
- `PGRST_JWT_SECRET` muss zwischen `postgrest` und `app` identisch sein (sonst koennen JWTs nicht validiert werden)
- `MINIO_ROOT_PASSWORD` und `S3_SECRET_ACCESS_KEY` muessen identisch sein (MinIO ist gleichzeitig der S3-Server fuer den App)
- `APP_FQDN` wird vom App fuer Origin-Checks und Canvas-Iframe-URL-Bauen genutzt — muss zur Reverse-Proxy-Domain passen

---

## Was bewusst weggelassen ist

| Service | Grund |
|---|---|
| `publisher` (Webstudio-Self-Host) | V1 ohne Publish-Funktion. Wer das spaeter braucht: aus `webstudio-community/webstudio-self-host/docker-compose.yml` uebernehmen + zweite Subdomain (Wildcard) konfigurieren |
| `nginx` (Webstudio-Self-Host) | Nur fuer Publisher relevant (statische Site-Auslieferung) |
| Wildcard-Subdomain `*.webstudio.your-domain.com` | Canvas-Iframes funktionieren oft auch ohne — falls Live-Preview im Builder broken ist, mit DNS-Challenge nachziehen |
