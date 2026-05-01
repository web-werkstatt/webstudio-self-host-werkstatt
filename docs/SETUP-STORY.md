# Setup-Story — Wie dieses Template entstanden ist

> Echter Erfahrungsbericht eines Webstudio-Self-Host-Setups am 2026-05-01.
> Geschrieben fuer Leute die vor dem gleichen Problem stehen — was wir
> probiert haben, woran es gescheitert ist, was am Ende funktioniert hat.
> Lesedauer ~10 Min.

## Ausgangslage

- Ziel: Webstudio Builder als visuelle Designwerkstatt auf eigener Subdomain
  (`webstudio.example.com`), hinter Caddy mit basic_auth, fuer Single-User-Authoring
- Hardware: Hetzner-VM, Debian 12, Docker 24 + Compose v2
- Plan-Annahme: *„Webstudio ist Open-Source, da klone ich das Repo und
  starte `pnpm dev` in einem Container — fertig in einer Stunde."*

Spoiler: das hat **nicht** funktioniert. Drei Stunden spaeter und mit
einem komplett anderen Ansatz lief es. Hier ist der Verlauf.

---

## V1 — Plan: Source-Klon + `pnpm dev` im Dev-Mode-Container

### Was wir gemacht haben

Idee war pragmatisch: Das offizielle `webstudio-is/webstudio` Repo lokal
klonen, in einem `node:22-alpine`-Container mit Volume-Mount betreiben,
`pnpm install + dev` als Container-Command, fertig.

```yaml
webstudio-builder:
  image: node:22-alpine
  volumes:
    - /path/to/webstudio-clone:/app
    - webstudio_node_modules:/app/node_modules
  environment:
    DATABASE_URL_FILE: /run/secrets/database_url
    AUTH_SECRET_FILE: /run/secrets/auth_secret
  command: >
    sh -c "pnpm install && pnpm migrations && pnpm --filter=@webstudio-is/builder dev"
```

Dazu Postgres + PostgREST in zwei weiteren Services. Schien sauber.

### Stolperstein 1 — Untracked Files im Klon

```
git status zeigte:
  webstudio-ai-service/   (eigenes Sub-Projekt, ~50 Dateien, Oktober 2025-Januar 2026)
  TODO.md
  .pnpm-store/
  .coderabbit.yaml
  .github/workflows/{coderabbit,security-review,code-quality}.yml
  +24 weitere project.json (Nx-Workspace-Setup)
```

Der Klon war nicht „jungfraeulich" — frueheres Tooling-Setup + ein
parallel angelegtes AI-Microservice-Projekt im selben Verzeichnis. Erste
Diskussion mit dem User: was tun mit der eigenen Substanz?

**Loesung:** Pragmatisch — `git pull` schadet untracked Files nicht.
Container ignoriert fremde Verzeichnisse weil pnpm-workspace strikt
ist (sieht nur was in `pnpm-workspace.yaml` steht).

### Stolperstein 2 — PostgREST 12 kennt kein `_FILE`-Suffix

Wir hatten Postgres-Passwort als Docker-Secret geplant, mit
`POSTGRES_PASSWORD_FILE` (Postgres-Image-Konvention). Naheliegend, das
gleiche fuer PostgREST zu machen:

```yaml
PGRST_DB_URI_FILE: /run/secrets/postgrest_db_uri
```

Container startete und ging in Restart-Loop:

```
PGRST000: could not look up local user ID 1000: No such file or directory
```

**Diagnose:** PostgREST 12.x **ignoriert** `_FILE`-Suffix-Variablen.
Das ist Postgres-Image-Konvention, nicht universell. PostgREST liest
nur die direkten ENV-Vars; wenn `PGRST_DB_URI` fehlt, faellt libpq
auf Peer-Auth zurueck und lookup-t den UID 1000 in `/etc/passwd` —
existiert im Container nicht → Fehler.

**Loesung:** Inline-ENV-Substitution mit `${VAR}` aus einer
`.env`-Datei neben dem Compose:

```yaml
webstudio-postgrest:
  environment:
    PGRST_DB_URI: "postgresql://postgres:${POSTGRES_PASSWORD}@webstudio-db:5432/webstudio"
```

Postgres bleibt mit `_FILE` (funktioniert dort), App und PostgREST
nutzen `${VAR}`-Pattern.

### Stolperstein 3 — Bind-Mount zeigt ins Leere

Nach Compose-Fix: Builder-Container startet, dann sofort:
```
ERR_PNPM_NO_PKG_MANIFEST  No package.json found in /app
```

**Diagnose:** Lokaler Klon war auf meinem Laptop unter
`/mnt/projects/webstudio-builder/`. Compose mountete `/mnt/projects/
webstudio-builder:/app` — aber der Container laeuft auf der **Server-VM**.
Die VM hat den Pfad nicht.

Ein Konzeptfehler im urspruenglichen Plan: Pfade waren so notiert
als waeren Laptop und Server der gleiche Host.

**Loesung:** Klon auf die VM legen, neuen Pfad mounten:
```bash
ssh server 'sudo mkdir -p /data/webstudio/source && \
  sudo chown joshko:joshko /data/webstudio/source && \
  cd /data/webstudio && \
  git clone https://github.com/webstudio-is/webstudio.git source'
```

### Stolperstein 4 — Workspace-Packages nicht gebaut

Container startet weiter, Vite versucht zu starten, dann:
```
Failed to resolve entry for package "@webstudio-is/http-client".
The package may have incorrect main/module/exports specified in its package.json.
```

**Diagnose:** Webstudio ist ein pnpm-Monorepo. Internal-Workspace-Packages
muessen vor `pnpm dev` gebaut sein. Der offizielle `.devcontainer/postinstall.sh`
zeigt die Reihenfolge:
```
pnpm install
pnpm build               # ← der fehlte uns
pnpm migrations migrate  # ← richtig: "migrate" Sub-Command, nicht "migrations" allein
```

**Loesung:** Container-Command erweitert, idempotent (Build erkennt
unveraenderte Files):
```bash
pnpm install --frozen-lockfile=false &&
pnpm build &&
(pnpm migrations migrate || true) &&
pnpm --filter=@webstudio-is/builder dev --host 0.0.0.0 --port 5173
```

Das laeuft 5+ Min beim ersten Start (pnpm install + pnpm build ueber
~50 Workspace-Packages), bei Restarts 30 Sek (alles cached).

### Stolperstein 5 — der Showstopper

Nach erfolgreichem Build:
```
Local:   https://wstd.dev:5173/
Local:   https://vite.wstd.dev:5173/
```

Vite-Server startete — auf `wstd.dev`. **Hardcoded.** Curl gegen
`https://webstudio-builder:5173/`:
- Ohne Host-Header: TLS-Reset
- Mit `Host: wstd.dev`: HTTP **403** mit leerem Body

```typescript
// apps/builder/vite.config.ts:
server: {
  host: "wstd.dev",                    // hardcoded
  https: {
    key: readFileSync("../../https/privkey.pem"),    // Self-Signed Cert fuer wstd.dev
    cert: readFileSync("../../https/fullchain.pem"),
  },
  cors: req => callback(null, { origin: false }),    // CORS aus
}
```

Plus Auth-Middleware die Origin gegen `wstd.dev` prueft.

Webstudio's offizielle Doku sagt es klar:

> Self-hosting the Builder in production is more difficult and currently
> not recommended.

Der Dev-Server ist auf lokales Setup mit `/etc/hosts` Mapping
`127.0.0.1 wstd.dev` + Self-Signed Cert designed. **Reverse-Proxy
auf eigener Domain ist explizit nicht supportet.**

### Optionen die wir verworfen haben

| Option | Warum nicht |
|---|---|
| Vite-Config patchen (host, https, cors) | Klon-Drift; bei `git pull` Konflikte; OAuth-Pfade brechen weiter |
| Caddy `header_up Host wstd.dev` + `tls_insecure_skip_verify` | Browser-Redirects/Links zeigen auf wstd.dev → unerreichbar |
| Production-Build (`remix-serve`) aus Source | Kein offizielles Image, eigener Build noetig, Test-Aufwand offen |
| Akzeptieren dass V1 nicht geht | … |

---

## V2 — Recherche-Pivot

An dem Punkt war klar: Brute-Force funktioniert nicht. Wir haben
einen Recherche-Agent losgeschickt mit konkretem Auftrag:

> Findet jemand der Webstudio-Self-Hosting hinter Reverse-Proxy
> tatsaechlich am Laufen hat? GitHub-Issues, Discord, Forks.

Antwort nach 2 Min: **Es existiert ein Community-Fork.**

- Repo: [`webstudio-community/webstudio-self-host`](https://github.com/webstudio-community/webstudio-self-host)
- Image: `ghcr.io/webstudio-community/builder:latest`
- Production-Remix-Build (kein Vite-Dev), `wstd.dev`-Hardcodes raus,
  `APP_FQDN` als sauberer ENV-Konfiguration
- Wird **offiziell vom Webstudio-FAQ verlinkt** (im Self-Hosting-Abschnitt)

Game-Changer. Re-Setup mit dem Image dauerte ~30 Minuten und lief
beim ersten Versuch sauber durch.

---

## V2-Setup (was am Ende funktioniert hat)

### Architektur

7 Services:
- `webstudio-app` (Production-Image vom Community-Fork)
- `webstudio-db` (`postgres:15-alpine` + `postgres-init.sql` Bind-Mount fuer
  `extensions`-Schema + `anon`-Rolle)
- `webstudio-postgrest` (`postgrest/postgrest:v12.2.0`)
- `webstudio-minio` (S3-Asset-Storage)
- 3 Run-Once-Init-Container (`db-setup`, `migrate`, `minio-init`)

Alle in `proxy-network` extern, kein Host-Port-Binding, Caddy mit
basic_auth davor.

### Inkrementelle Verifikation (was wirklich half)

Statt alle 7 Services parallel hochzuziehen: einer nach dem anderen.

```bash
docker compose up -d webstudio-db
# verify pg_isready, dann:
docker compose up -d webstudio-postgrest
# verify "Successfully connected to PostgreSQL", dann:
docker compose up -d webstudio-app
# verify /health endpoint
```

Bei Stolperstein 2 (PostgREST `_FILE`-Suffix) hatten wir die Diagnose
**sofort** weil nur ein neuer Service hinzugekommen war. Bei einem
parallelen `up` waere die Fehlersuche viel laenger gewesen.

### Stolperstein 6 — Postgres-Schema-Migration zwischen Image-Wechseln

Zwischen V1 (Supabase-Postgres-Image) und V2 (plain `postgres:15-alpine`)
ist das Daten-Volume inkompatibel — anderer init.

**Loesung:** Volume umbenennen statt loeschen — reversibel:
```bash
mv /data/webstudio/postgres /data/webstudio/postgres.OLD-supabase-2026-05-01
mkdir -p /data/webstudio/db
```

Nach erfolgreichem V2-Setup koennte man `*.OLD*` loeschen — wir liessen
es aus Vorsicht.

### Stolperstein 7 — Origin-Check bei wrong Host

Mit Caddy davor und richtigem `APP_FQDN=webstudio.example.com` lief
alles. Direkt-curl gegen Container intern:
```bash
curl http://webstudio-app:3000/   # HTTP 403
```

War kurz Schreck-Moment, bis im Log:
```
Cross-origin request to http://webstudio-app:3000/ blocked
[ ['host', 'webstudio-app:3000'] ]
```

→ **Erwartetes Verhalten**. Das Image prueft Host-Header gegen
`APP_FQDN`. Caddy reicht den richtigen Host durch, dann kommt 200/302.

---

## Was uns geholfen hat

### Inkrementelles Hochziehen
Service fuer Service starten, jeweils healthy verifizieren, **dann erst
weiter**. Spart Stunden bei Fehlersuche.

### Beweisblock am Ende jedes Schritts
Nach jedem `up` wurde gepruefelt:
- `docker ps` zeigt Status
- `pg_isready` / `wget /health` interne Probe
- Logs nach Errors gegrept

Wenn ein Schritt fehlschlug, war klar **welcher** und **wo**. Ohne diese
Disziplin verliert man sich in „irgendwas geht nicht".

### Reversible Aktionen statt destruktive
- `mv` statt `rm` fuer alte Daten
- `docker compose stop` statt `down -v`
- Backup vor jedem irreversiblen Schritt

Nichts ging waehrend des Setups verloren, weil jeder Schritt rueckwaerts
abrollbar war.

### Stufe-3-Vorgehen mit User
Manuelle Freigabe pro Schritt, kein Auto-Run, vor jedem Compose-Edit
oder Caddy-Deploy kurz „was + warum" erklaeren. **Kostet Zeit, spart
Zeit** — Fehler werden frueh gefangen, nicht erst nach 30 Min.

### Recherche-Agent statt Brute-Force
Als V1 stuckte: nicht weiter probieren, sondern fragen ob das Problem
bekannt ist und ob es Forks gibt. Das fand den Community-Fork in 2 Min.

---

## Was wir lieber anders gemacht haetten

### Webstudio-Doku zuerst lesen
Die Warnung *„Self-hosting the Builder in production is more difficult
and currently not recommended"* stand schwarz auf weiss in der offiziellen
Doku. Wir hatten sie gelesen und **interpretiert** als „aufwendig aber
machbar". Stimmt nicht — sie heisst „eigentlich nicht supportet, geh ueber
Forks". Erkenntnis: Doku-Warnungen ernster nehmen.

### Image-Tag pinnen von Anfang an
Wir hatten zuerst `:latest` benutzt — funktioniert, aber bei spaeteren
Updates riskant. Im Bericht ist die SHA-Pin-Empfehlung in `docs/DEPLOY.md`
explizit drin.

### Nicht zu viele Compose-Re-Designs
Wir hatten zwischen V1-Iterationen 3 verschiedene Compose-Dateien
geschrieben. Im Nachhinein: nach Stolperstein 5 sofort Recherche-Pivot,
nicht erst weitere Source-Klon-Versuche.

---

## Zeitaufwand grob

| Phase | Dauer |
|---|---|
| V1 Source-Klon-Setup mit Stolpersteinen 1-5 | ~2 h |
| Recherche-Pivot | ~10 Min |
| V2 Self-Host-Pattern | ~30 Min |
| Caddy + Verifikation | ~15 Min |
| **Gesamt** | **~3 h** |

Mit dem Wissen das wir jetzt haben (= dieses Template-Repo) ist das
Setup in **~30 Min** machbar — vorausgesetzt Server, DNS, Caddy/Traefik
sind schon vorhanden.

---

## TL;DR fuer Eilige

1. **Nicht** `webstudio-is/webstudio` direkt klonen + `pnpm dev` betreiben
2. **Stattdessen** dieses Template-Repo nehmen (basiert auf
   `webstudio-community/webstudio-self-host` Production-Image)
3. `.env` ausfuellen, `docker compose up -d --wait`, Caddy davor
4. Stolpersteine die garantiert kommen → siehe `TROUBLESHOOTING.md`
5. Update-Sicherheit → `BUILDER_IMAGE` mit SHA pinnen, nicht `:latest`
