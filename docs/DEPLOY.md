# Self-Hosting-Anleitung — Webstudio Builder Schritt fuer Schritt

Komplette Anleitung von „leerer Server" zu „laufender Builder unter eigener Domain".

> **Aufwand:** ~30-45 Min (haengt an Image-Pull-Geschwindigkeit + DNS-Propagation)
> **Erfahrungslevel:** Linux-Kommandozeile, Docker-Compose-Grundlagen

---

## Inhaltsverzeichnis

1. [Server vorbereiten](#1-server-vorbereiten)
2. [DNS einrichten](#2-dns-einrichten)
3. [Repo klonen + .env anpassen](#3-repo-klonen--env-anpassen)
4. [Daten-Verzeichnisse + Network](#4-daten-verzeichnisse--network)
5. [Stack starten](#5-stack-starten)
6. [Reverse-Proxy konfigurieren (Caddy)](#6-reverse-proxy-konfigurieren-caddy)
7. [Erstanmeldung](#7-erstanmeldung)
8. [Verify-Checkliste](#8-verify-checkliste)
9. [Optional: Wildcard-Subdomain fuer Canvas-Preview](#9-optional-wildcard-subdomain-fuer-canvas-preview)
10. [Optional: GitHub-OAuth statt DEV_LOGIN](#10-optional-github-oauth-statt-dev_login)

---

## 1. Server vorbereiten

### Mindest-Spezifikation

| | |
|---|---|
| OS | Linux mit aktuellem Kernel (Debian 12, Ubuntu 22.04+, etc.) |
| RAM | 2 GB frei (Production-Build, deutlich genuegsamer als Dev-Mode) |
| Disk | 10 GB frei (Postgres + MinIO + Images) |
| Docker | ≥ 24 |
| Compose | v2 (`docker compose`-Syntax, kein `docker-compose`) |

### Docker installieren (falls noch nicht vorhanden)

```bash
# Debian/Ubuntu
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# danach neu einloggen oder: newgrp docker
```

Test:
```bash
docker --version              # erwartet: Docker version 24.x oder neuer
docker compose version        # erwartet: Docker Compose version v2.x
```

---

## 2. DNS einrichten

A-Record bei Deinem DNS-Provider:

| Type | Name | Content | TTL |
|---|---|---|---|
| A | `webstudio.your-domain.com` | Server-IP | Auto (Cloudflare) oder 300 |

**Cloudflare-User:** Proxy-Status auf **DNS only** (graue Wolke), nicht orange. Caddy/Traefik macht TLS selbst — Cloudflare-Proxy davor verhindert Let's Encrypt-Cert-Issuing.

DNS-Propagation pruefen:
```bash
host webstudio.your-domain.com
# erwartet: webstudio.your-domain.com has address <Server-IP>
```

---

## 3. Repo klonen + .env anpassen

```bash
git clone https://github.com/web-werkstatt/webstudio-self-host-werkstatt.git
cd webstudio-self-host-werkstatt
cp .env.example .env
```

### Geheimnisse generieren

```bash
echo "POSTGRES_PASSWORD=$(openssl rand -hex 32)"
echo "PGRST_JWT_SECRET=$(openssl rand -hex 64)"
echo "AUTH_SECRET=$(openssl rand -hex 32)"
echo "TRPC_SERVER_API_TOKEN=$(openssl rand -hex 32)"
echo "MINIO_ROOT_PASSWORD=$(openssl rand -hex 24)"
```

Alle 5 Werte in `.env` eintragen. **Wichtig:** `S3_SECRET_ACCESS_KEY` muss exakt denselben Wert wie `MINIO_ROOT_PASSWORD` haben (MinIO ist der S3-Server).

### Domain + Login-Email

In `.env`:
```env
APP_FQDN=webstudio.your-domain.com
DEV_LOGIN_EMAIL=admin@your-domain.com
```

`APP_FQDN` muss **exakt** mit der Reverse-Proxy-Domain uebereinstimmen — Webstudio prueft den Origin gegen diesen Wert und lehnt sonst mit HTTP 403 ab.

### Permissions setzen

```bash
chmod 600 .env
```

---

## 4. Daten-Verzeichnisse + Network

```bash
sudo mkdir -p /data/webstudio/{db,minio,uploads}
sudo chown -R "$USER:$USER" /data/webstudio
```

Externes Docker-Network anlegen (falls noch nicht vom Reverse-Proxy bereitgestellt):

```bash
docker network inspect proxy-network >/dev/null 2>&1 || docker network create proxy-network
```

Wer ein anders benanntes Network nutzt: in `docker-compose.yml` (Block am Ende) den Namen anpassen.

---

## 5. Stack starten

```bash
docker compose up -d --wait --wait-timeout 300
```

`--wait` wartet bis alle Container `healthy` sind (oder Init-Services `Exited 0`). Dauert beim ersten Mal ~2-5 Min (Image-Pulls + Migrations).

Erfolgreicher Output sieht so aus (verkuerzt):
```
 Container webstudio-db          Healthy
 Container webstudio-minio       Healthy
 Container webstudio-postgrest   Healthy
 Container webstudio-app         Healthy
 Container webstudio-db-setup    Exited 0
 Container webstudio-migrate     Exited 0
 Container webstudio-minio-init  Exited 0
```

### Internen Healthcheck pruefen

```bash
docker exec webstudio-app wget -qO- http://127.0.0.1:3000/health
# erwartet: OK
```

### Migrations-Log pruefen

```bash
docker logs webstudio-migrate 2>&1 | tail -5
# erwartet: "All migrations have been successfully applied."
```

---

## 6. Reverse-Proxy konfigurieren (Caddy)

### bcrypt-Hash fuer basic_auth

```bash
docker run --rm httpd:alpine htpasswd -nbBC 14 USERNAME 'PASSWORT'
# Output: USERNAME:$2y$14$XXX...
```

Den `$2y$14$XXX...`-Teil im naechsten Schritt verwenden.

### Caddy-Block ergaenzen

Das Beispiel aus `caddy-block.example.txt` ans zentrale Caddyfile anhaengen — User+Hash + Domain ersetzen. Zwei Varianten:
- **Variante A** (mit basic_auth) — empfohlen fuer Werkstatt-Setup
- **Variante B** (ohne basic_auth) — wenn DEV_LOGIN/OAuth allein genuegen soll

### Caddy reload

```bash
caddy reload --config /etc/caddy/Caddyfile
```

Caddy holt automatisch ein Let's Encrypt-Zertifikat. Erste Cert-Ausstellung dauert ~30 Sekunden, ist im Caddy-Log sichtbar.

### Externer Test

```bash
curl -sI https://webstudio.your-domain.com/
# erwartet (Variante A): HTTP/2 401 + www-authenticate: Basic realm="restricted"
# erwartet (Variante B): HTTP/2 302 → /login

# TLS-Cert pruefen
echo | openssl s_client -connect webstudio.your-domain.com:443 \
  -servername webstudio.your-domain.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
# erwartet: subject=CN=webstudio.your-domain.com, issuer=Let's Encrypt
```

---

## 7. Erstanmeldung

1. Browser oeffnen: `https://webstudio.your-domain.com/`
2. Wenn Caddy basic_auth aktiv: Username + Passwort eingeben
3. Webstudio-DEV_LOGIN-Form erscheint:
   - **Auth secret**: Wert von `AUTH_SECRET` aus `.env`
   - **Email**: Wert von `DEV_LOGIN_EMAIL` (sollte vorausgefuellt sein)
   - **Default plan**: `Default` (= dein konfigurierter `USER_PLAN=Pro` Plan)
4. → Webstudio-Dashboard

**Browser-Passwort-Manager nutzen** — speichert beim ersten Login das Auth-Secret und fuellt es beim naechsten Mal automatisch ein.

---

## 8. Verify-Checkliste

| Test | Erwartet |
|---|---|
| `docker ps --filter name=webstudio` | 4 langlaufend (Up + healthy), 3 Exited 0 |
| `docker exec webstudio-app wget -qO- http://127.0.0.1:3000/health` | `OK` |
| `curl -sI https://webstudio.your-domain.com/` | HTTP 401 oder 302 (je nach Variante) |
| TLS-Cert via `openssl s_client` | Let's Encrypt, gueltig 90 Tage |
| Browser-Login → Dashboard | Webstudio-UI bedienbar |
| Test-Projekt anlegen + speichern | Projekt erscheint in Dashboard-Liste, Reload behaelt State |
| Bild hochladen (Asset) | Upload erfolgreich, Bild im Canvas sichtbar |

---

## 9. Optional: Wildcard-Subdomain fuer Canvas-Preview

Webstudio nutzt fuer die Live-Canvas-Preview Sub-Subdomains (eine pro Projekt). Ohne Wildcard kann das Canvas-Iframe broken sein — aber das **Authoring** (Layout, Style-Editor) funktioniert auch ohne.

Falls Du Wildcard brauchst:

### DNS

```
A  *.webstudio.your-domain.com  → Server-IP
```

### Caddy mit DNS-Challenge

Caddy braucht ein DNS-Plugin fuer Wildcard-Cert. Beispiel mit Cloudflare:

```caddyfile
*.webstudio.your-domain.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
    reverse_proxy webstudio-app:3000 {
        flush_interval -1
    }
}
```

`CF_API_TOKEN` als ENV in den Caddy-Container injecten. Cloudflare-Token bei *Cloudflare-Dashboard → My Profile → API Tokens* (Template "Edit zone DNS", auf Zone beschraenken).

---

## 10. Optional: GitHub-OAuth statt DEV_LOGIN

Fuer Multi-User-Setup oder bequemeres Login.

### OAuth-App bei GitHub erstellen

https://github.com/settings/developers → **New OAuth App**

| Feld | Wert |
|---|---|
| Application name | `Webstudio` (oder beliebig) |
| Homepage URL | `https://webstudio.your-domain.com` |
| Authorization callback URL | `https://webstudio.your-domain.com/auth/github/callback` |

→ **Register**. Client-ID und Client-Secret notieren.

### .env anpassen

```env
DEV_LOGIN=
DEV_LOGIN_EMAIL=
GH_CLIENT_ID=Iv1.xxxxxxx
GH_CLIENT_SECRET=xxxxxxxxxxxxxxxx
```

### Container neu starten

```bash
docker compose up -d --force-recreate webstudio-app
```

Nach Login mit GitHub: Cookie haelt 30 Tage.

---

## Update-Prozess (sicher)

### Was passiert bei einem Update?

| Komponente | Wird ueberschrieben? |
|---|---|
| Container-Image (App, DB, MinIO, PostgREST) | ja — neue Image-Layer |
| Bind-Mount-Daten in `/data/webstudio/{db,minio,uploads}` | **nein** — bleiben unveraendert |
| Schema-Migrations | werden idempotent neu gefahren (Prisma ueberspringt bereits applied) |
| `.env` | bleibt — wird vor jedem `up` gelesen |

→ **Daten + Konfig sind sicher**, Container-Layer werden ersetzt.

### Risiko: `:latest`-Tag

Der Default-Tag `BUILDER_IMAGE=ghcr.io/webstudio-community/builder:latest` ist **nicht** Update-sicher: Wenn der Community-Fork ein Breaking Change pushed (neue Pflicht-ENV, geaendertes Schema, andere Default-Ports), zieht `docker compose pull` das ungefiltert ein.

**Loesung — Image-SHA pinnen:**

```bash
# aktuell deployten Digest auslesen
docker inspect ghcr.io/webstudio-community/builder:latest \
  --format '{{json .RepoDigests}}'
# z.B. ["ghcr.io/webstudio-community/builder@sha256:d380263690eada56...d1381"]
```

In `.env` ersetzen:

```env
# vorher (unsicher):
BUILDER_IMAGE=ghcr.io/webstudio-community/builder:latest

# nachher (gepinnt):
BUILDER_IMAGE=ghcr.io/webstudio-community/builder@sha256:d380263690eada56292656f4f7f6196bc38103798c6ba635bc5c43c1776d1381
```

Ab jetzt zieht `docker compose pull` exakt diesen Build, egal wo `:latest` hinwandert. Updates werden zur **bewussten Entscheidung**: SHA aktualisieren, nicht passiv mitnehmen.

### Drei-Schichten-Schutz vor Updates

| Schicht | Aufwand | Schuetzt vor |
|---|---|---|
| 1. Image-SHA pinnen | einmalig 10 Sek | Unbeabsichtigte Breaking Changes |
| 2. `pg_dump` vor jedem Update | 30 Sek | Migrations-Schaden |
| 3. VM-Snapshot (Proxmox/Hetzner) | 1 Min | Komplett-Restore bei Total-Crash |

### Sicherer Update-Workflow

```bash
# 1. Backup
docker exec webstudio-db pg_dump -U postgres webstudio | gzip > backup-$(date +%Y%m%d-%H%M).sql.gz

# 2. (Optional) MinIO-Assets sichern
rsync -av /data/webstudio/minio/ /backup-target/minio-$(date +%Y%m%d)/

# 3. Aktuellen SHA notieren — fuer Rollback
docker inspect ghcr.io/webstudio-community/builder:latest --format '{{.Id}}' > .last-image-sha

# 4. Update fahren
docker compose pull
docker compose up -d --wait

# 5. Migrations-Log pruefen
docker logs webstudio-migrate 2>&1 | tail -10
# erwartet: "All migrations have been successfully applied."

# 6. Smoke-Test
curl -sI https://webstudio.your-domain.com/
docker exec webstudio-app wget -qO- http://127.0.0.1:3000/health
# erwartet: 401/302 + "OK"
```

### Rollback

Falls Update Probleme macht:

```bash
# In .env die SHA auf den alten Wert (aus .last-image-sha) zuruecksetzen,
# dann recreate:
docker compose up -d --force-recreate webstudio-app

# Falls DB-Schaden (selten, nur bei Migrations-Bugs):
docker compose stop webstudio-app webstudio-postgrest
docker exec -i webstudio-db psql -U postgres -d postgres -c "DROP DATABASE webstudio; CREATE DATABASE webstudio;"
docker exec -i webstudio-db psql -U postgres webstudio < postgres-init.sql
gunzip -c backup-YYYYMMDD-HHMM.sql.gz | docker exec -i webstudio-db psql -U postgres webstudio
docker compose up -d --wait
```

### Wann updaten?

- **Security-Patches** im Community-Fork (Watch im GitHub-Repo) → zeitnah
- **Feature-Updates** → testen, evtl. erst auf Test-Setup probieren
- **Major-Webstudio-Versionen** (z.B. 0.2x → 0.3x) → erst Changelog lesen, Schema-Migrations checken

---

## Stack stoppen

```bash
docker compose stop          # Container stoppen, Daten bleiben
docker compose down          # Container entfernen, Volumes bleiben (Bind-Mounts in /data unangetastet)
docker compose down -v       # GEFAEHRLICH — auch named Volumes loeschen (wir haben aber Bind-Mounts → /data/webstudio bleibt trotzdem)
```

---

## Komplett zuruecksetzen (Destruktiv!)

```bash
docker compose down
sudo rm -rf /data/webstudio/{db,minio,uploads}/*
docker compose up -d --wait    # frisches Setup mit leerem DB-State
```

→ Alle Projekte, User, Assets gehen verloren. Nur fuer Test-Setups.
