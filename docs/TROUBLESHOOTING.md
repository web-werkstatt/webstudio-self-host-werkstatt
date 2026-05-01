# Troubleshooting

Bekannte Stolpersteine + Loesungen aus dem ersten Setup.

## Inhaltsverzeichnis

1. [HTTP 403 trotz richtiger Caddy-Konfig](#1-http-403-trotz-richtiger-caddy-konfig)
2. [`could not look up local user ID 1000`](#2-could-not-look-up-local-user-id-1000)
3. [`pnpm dev` scheitert mit `wstd.dev`-Hardcode](#3-pnpm-dev-scheitert-mit-wstddev-hardcode)
4. [Postgres-Volume inkompatibel nach Image-Wechsel](#4-postgres-volume-inkompatibel-nach-image-wechsel)
5. [TLS-Cert-Issuing scheitert](#5-tls-cert-issuing-scheitert)
6. [Builder zeigt nach Login leere Seite](#6-builder-zeigt-nach-login-leere-seite)
7. [Canvas-Iframe broken](#7-canvas-iframe-broken)
8. [DEV_LOGIN-Page fragt jedes Mal nach Auth-Secret](#8-dev_login-page-fragt-jedes-mal-nach-auth-secret)
9. [Migrations fail mit `extensions.uuid_generate_v4 does not exist`](#9-migrations-fail-mit-extensionsuuid_generate_v4-does-not-exist)
10. [Container-Restart-Loop mit ENV-Validation-Error](#10-container-restart-loop-mit-env-validation-error)

---

## 1. HTTP 403 trotz richtiger Caddy-Konfig

**Symptom:** Caddy zeigt 200 im Log, aber Builder antwortet 403. Im Browser leere Seite oder „Forbidden".

**Ursache:** `APP_FQDN` in `.env` passt nicht zur Reverse-Proxy-Domain. Webstudio prueft den `Host`-Header und verwirft alles was nicht `APP_FQDN` ist.

**Loesung:**
```bash
grep APP_FQDN .env
# muss exakt zu Caddy/Traefik-Domain passen, ohne Protokoll, ohne Port
# RICHTIG:  APP_FQDN=webstudio.example.com
# FALSCH:   APP_FQDN=https://webstudio.example.com/

docker compose up -d --force-recreate webstudio-app
```

**Verify:**
```bash
docker logs webstudio-app 2>&1 | grep "Cross-origin"
# sollte still sein nach Fix
```

---

## 2. `could not look up local user ID 1000`

**Symptom:** PostgREST-Container im Restart-Loop, Logs zeigen `PGRST000: could not look up local user ID 1000`.

**Ursache:** PostgREST 12.x kennt **kein** `_FILE`-Suffix fuer Secret-Files (das ist Postgres-Image-Konvention, nicht universal). Wenn `PGRST_DB_URI_FILE` gesetzt ist statt `PGRST_DB_URI`, ignoriert PostgREST das, faellt auf libpq-Default zurueck und versucht Peer-Auth ueber Unix-Socket → User-Lookup auf nicht-existenten UID.

**Loesung:** In `docker-compose.yml` muss bei Postgrest `PGRST_DB_URI` als ENV-Variable mit eingebettetem Passwort gesetzt sein:

```yaml
webstudio-postgrest:
  environment:
    PGRST_DB_URI: postgresql://postgres:${POSTGRES_PASSWORD}@webstudio-db:5432/webstudio
```

Nicht `PGRST_DB_URI_FILE` mit Secret-File-Verweis.

---

## 3. `pnpm dev` scheitert mit `wstd.dev`-Hardcode

**Symptom:** Wenn man versucht, den Builder direkt aus dem Source-Klon `webstudio-is/webstudio` mit `pnpm dev` hinter eigenem Reverse-Proxy zu betreiben:
- HTTPS-Connection-Reset oder HTTP 403
- Builder-Logs zeigen `Cross-origin request to ...` und Vite startet auf `https://wstd.dev:5173`

**Ursache:** `apps/builder/vite.config.ts` ist hardcoded auf `server.host: "wstd.dev"` und liest TLS-Cert aus `../../https/`. CORS-Default ist `origin: false`. Webstudios offizielle Doku sagt explizit: *"Self-hosting the Builder in production is more difficult and currently not recommended."*

**Loesung:** Source-Klon-Dev-Mode-Pfad **nicht** weiter verfolgen. Stattdessen das Production-Image vom Community-Fork nutzen:

```yaml
webstudio-app:
  image: ghcr.io/webstudio-community/builder:latest  # statt node:22-alpine + pnpm dev
  environment:
    NODE_ENV: production
    APP_FQDN: webstudio.your-domain.com
    # ... rest siehe docker-compose.yml in diesem Repo
```

---

## 4. Postgres-Volume inkompatibel nach Image-Wechsel

**Symptom:** Nach Wechsel von `ghcr.io/supabase/postgres` zu `postgres:15-alpine` (oder umgekehrt) startet `webstudio-db` nicht, Logs zeigen `incompatible PG_VERSION` oder `database files are incompatible with server`.

**Ursache:** Verschiedene Postgres-Distros (Supabase vs. plain) haben unterschiedliche Init-Schemata. Volume vom alten Image kann vom neuen nicht gelesen werden.

**Loesung (Daten-Verlust!):**
```bash
docker compose down
sudo mv /data/webstudio/db /data/webstudio/db.OLD-$(date +%Y%m%d)
sudo mkdir -p /data/webstudio/db
sudo chown -R "$USER:$USER" /data/webstudio/db
docker compose up -d --wait
```

Migrations werden vom `webstudio-migrate`-Service automatisch neu ausgefuehrt — bei frischer Werkstatt = kein echter Verlust. Bei Bestandsdaten: vorher `pg_dump` aus dem alten Container, nach neuem Init `psql ... < dump.sql` zurueckspielen.

---

## 5. TLS-Cert-Issuing scheitert

**Symptom:** Caddy-Log zeigt `acme: Error 403` oder `unauthorized` beim Cert-Issuing.

**Ursachen + Loesungen:**

| Ursache | Diagnose | Fix |
|---|---|---|
| DNS zeigt nicht auf Server | `host webstudio.your-domain.com` zeigt falsche IP | DNS-A-Record korrigieren, TTL warten |
| Cloudflare-Proxy (orange Wolke) aktiv | Cloudflare-Dashboard zeigt orange Cloud | Auf grau (DNS only) umstellen |
| Port 80 blockiert | `nc -zv server-ip 80` schlaegt fehl | Firewall/iptables: Port 80 + 443 zur Container/Caddy oeffnen |
| Rate-Limit erreicht | 1 Cert pro Domain, 5 dupes/Woche bei Let's Encrypt | Warten oder Staging-CA verwenden |

**Caddy-Log lesen:**
```bash
docker logs caddy 2>&1 | grep -i "acme\|tls\|error"
```

---

## 6. Builder zeigt nach Login leere Seite

**Symptom:** DEV_LOGIN erfolgreich, aber Dashboard ist weiss/leer. JS-Fehler in Browser-Konsole.

**Ursache (haeufig):** MinIO/S3-Endpoint nicht erreichbar fuer den Browser (Webstudio versucht Asset-URLs zu laden).

**Diagnose:**
```bash
# In Browser-DevTools → Network → Failed Requests pruefen
# Falls Failed Requests gegen `webstudio-minio:9000` oder `localhost:9000` gehen:
# → MinIO-Endpoint muss vom Browser erreichbar sein, nicht nur vom Server
```

**Loesung:** S3-Assets ueber den App-Server proxyen (Default-Verhalten im Self-Host) statt direkt MinIO exposen. In `.env`:
```env
S3_ENDPOINT=http://webstudio-minio:9000     # interner Endpoint, App proxyt nach aussen
```

Falls externer MinIO-Zugriff gewollt: separate Subdomain mit Caddy-Block fuer MinIO-API + Console.

---

## 7. Canvas-Iframe broken

**Symptom:** Builder laedt, Layout-Editor sichtbar, aber Live-Canvas-Preview ist leer/Iframe-Error.

**Ursache:** Webstudio Canvas-Iframes nutzen Sub-Subdomains (`<projekt>.webstudio.your-domain.com`). Ohne Wildcard-DNS + Wildcard-TLS-Cert scheitert der Iframe-Load.

**Loesung:** Wildcard-Subdomain einrichten (siehe `docs/DEPLOY.md` Abschnitt 9). Erfordert:
- DNS-A-Record `*.webstudio.your-domain.com`
- Caddy mit DNS-Challenge fuer Wildcard-Cert (Cloudflare-API-Token o.ae.)

V1-Workaround ohne Wildcard: Canvas-Preview in eigenem Tab oeffnen statt Iframe.

---

## 8. DEV_LOGIN-Page fragt jedes Mal nach Auth-Secret

**Symptom:** Nach Tab-Schliessen oder Browser-Restart muss man Email + Auth-Secret erneut eingeben.

**Ursache:** Cookie persistiert nicht (Session-Cookie statt Persistent-Cookie, oder Browser-Privacy-Modus).

**Loesungen:**
1. **Browser-Passwort-Manager** speichert Auth-Secret beim ersten Login → automatisches Eintragen ab dann
2. **GitHub-OAuth einrichten** (siehe `docs/DEPLOY.md` Abschnitt 10) — Cookie haelt 30 Tage
3. **Caddy basic_auth entfernen** — wenn das die nervigste Schicht ist; Webstudio-DEV_LOGIN allein reicht (Auth-Secret = 64 hex chars = 256 bit Entropy)

---

## 9. Migrations fail mit `extensions.uuid_generate_v4 does not exist`

**Symptom:** `webstudio-migrate` exit-code != 0, Logs:
```
ERROR: function extensions.uuid_generate_v4() does not exist
```

**Ursache:** Webstudio-Migrations rufen UUID-Funktionen explizit aus dem `extensions`-Schema auf (Supabase-Konvention). Auf plain Postgres fehlt das Schema, wenn `postgres-init.sql` nicht beim ersten DB-Start gelaufen ist.

**Diagnose:**
```bash
docker exec webstudio-db psql -U postgres webstudio -c "\dn"
# erwartet: extensions-Schema in der Liste
```

**Loesung:** `postgres-init.sql` muss als Bind-Mount nach `/docker-entrypoint-initdb.d/init.sql` reingeschoben werden — laeuft aber **nur beim ersten Start** (leeres Volume). Wenn DB schon initialisiert: manuell nachziehen:
```bash
docker exec -i webstudio-db psql -U postgres webstudio < postgres-init.sql
docker compose up -d --force-recreate webstudio-migrate
```

---

## 10. Container-Restart-Loop mit ENV-Validation-Error

**Symptom:** `webstudio-app` startet wiederholt + crasht. Logs zeigen z.B. `Invalid env: AUTH_SECRET is required`.

**Ursache:** ENV-Variable fehlt oder ist leer in `.env`.

**Diagnose:**
```bash
docker compose config | grep -E "AUTH_SECRET|POSTGRES_PASSWORD|PGRST_JWT_SECRET|TRPC"
# erwartet: alle haben non-empty Werte hinter dem Doppelpunkt
```

**Loesung:** `.env` neu pruefen, alle „change-me"-Werte ersetzen, dann `docker compose up -d --force-recreate webstudio-app`.

---

## Allgemeine Diagnose-Befehle

```bash
# Status aller Container
docker ps -a --filter name=webstudio --format "table {{.Names}}\t{{.Status}}"

# Logs eines Services
docker logs --tail=50 -f webstudio-app

# Healthcheck-Status
docker inspect webstudio-app --format '{{.State.Health.Status}}'

# In den Container einsteigen
docker exec -it webstudio-app sh

# DB-Tabellen pruefen
docker exec webstudio-db psql -U postgres webstudio -c "\dt"

# Anzahl Migrations geprueft
docker exec webstudio-db psql -U postgres webstudio -c "SELECT count(*) FROM _prisma_migrations;"

# MinIO-Bucket-Inhalt
docker run --rm --network=proxy-network minio/mc \
  mc alias set local http://webstudio-minio:9000 minioadmin "$(grep MINIO_ROOT_PASSWORD .env | cut -d= -f2)" && \
  mc ls local/webstudio-assets
```
