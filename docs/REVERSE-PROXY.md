# Reverse-Proxy-Setup fuer Webstudio Self-Host

Diese Anleitung beschreibt das **vollstaendige** Reverse-Proxy-Setup fuer
Webstudio Builder, inkl. Wildcard-Subdomain fuer Project-Canvas-Iframes,
OAuth-Internal-Flow und Hairpin-NAT-Workaround.

Ohne diese Schritte kannst Du nach erfolgreichem Container-Start zwar die
Builder-UI sehen, aber **keine Projekte anlegen oder oeffnen**.

> Caddy 2.x mit `on_demand_tls` ist die einfachste Loesung — kein Wildcard-
> Cert noetig, keine DNS-Challenge, kein API-Token beim DNS-Provider. Caddy
> holt sich pro genutzter Project-Subdomain ein einzelnes Lets-Encrypt-Cert
> beim ersten Aufruf (HTTP-01).

---

## Warum so kompliziert?

Webstudio's Architektur:

```
Browser
  │
  ├──► webstudio.example.com         (Builder-UI, basic_auth ok)
  │       └─ /oauth/* /auth/*        (OAuth-Flow, MUSS frei sein)
  │
  └──► p-<uuid>.webstudio.example.com (Canvas-Iframe pro Projekt, KEIN basic_auth)
```

Beim Anlegen oder Oeffnen eines Projekts:
1. Browser laedt Apex `webstudio.example.com/builder/<id>`
2. Builder leitet auf `p-<id>.webstudio.example.com` um (eigene Subdomain)
3. Project-Subdomain macht OAuth-Internal-Roundtrip zur Apex
4. Apex (Container) fetcht **selbst** seine public URL fuer Token-Tausch
   → **Hairpin-NAT-Problem** auf vielen Docker-Hosts

Daraus folgen die vier Knackpunkte:
- **Wildcard-DNS** (Punkt 2)
- **on-demand TLS** (TLS-Cert pro Project-Subdomain — Punkt 2)
- **basic_auth-Bypass fuer OAuth-Pfade** (Punkt 3)
- **Loopback-Fix via `extra_hosts`** (Punkt 4)

Plus: Webstudio's `preventCrossOriginCookie` lehnt Requests mit
`Sec-Fetch-Site != "same-origin"` ab. Caddy verschluckt die browser-eigenen
`Sec-Fetch-*`-Headers — wir muessen sie explizit weiterreichen.

---

## Schritt-fuer-Schritt

### 1) DNS-Wildcard setzen

Beim Domain-Provider zwei A-Records anlegen:

```
webstudio.example.com.       A   <server-ip>
*.webstudio.example.com.     A   <server-ip>
```

Wildcard-Record bei vielen Providern als `*.webstudio` oder `*.webstudio.example.com.`
einzutragen. Bei Cloudflare, AutoDNS, InternetX, etc. moeglich ohne API.

**Verify:**
```bash
dig +short webstudio.example.com         # → server-ip
dig +short test123.webstudio.example.com # → server-ip
```

DNS-Propagation kann 5–60 Minuten dauern.

---

### 2) Caddyfile-Snippets einbinden

Die kompletten Bausteine stehen in `caddy-block.example.txt`. Vier Blöcke:

#### 2a) Globaler Block — `on_demand_tls`

```caddy
{
    on_demand_tls {
        ask http://localhost:9001/check
    }
}
```

`ask`-Endpoint schuetzt Caddy davor, dass Fremde durch Punkten ihrer Domain
auf Deine IP beliebig viele Certs anfordern koennen (Lets-Encrypt-Rate-Limit).

#### 2b) `:9001` Ask-Endpoint

```caddy
:9001 {
    @allowed expression `{http.request.uri.query.domain}.endsWith(".webstudio.example.com")`
    handle @allowed {
        respond 200
    }
    respond 404
}
```

Caddy fragt diesen Endpoint vor jedem Cert-Request: nur Subdomains von
`webstudio.example.com` werden erlaubt.

#### 2c) Apex-Block

```caddy
webstudio.example.com {
    encode gzip zstd

    @needs_auth not path /oauth/* /auth/*
    basic_auth @needs_auth {
        joshko $2y$14$REPLACE-WITH-YOUR-BCRYPT-HASH
    }

    header {
        X-Content-Type-Options nosniff
        X-Frame-Options SAMEORIGIN
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }

    reverse_proxy webstudio-app:3000 {
        flush_interval -1
        header_up Sec-Fetch-Site "same-origin"
    }
}
```

**Wichtig:**
- `@needs_auth not path /oauth/* /auth/*` — `basic_auth` darf **nicht** auf
  OAuth-Pfade greifen, sonst bekommt der Browser 401 beim Token-Tausch
- `header_up Sec-Fetch-Site "same-origin"` — sonst lehnt Webstudio mit 400 ab
- `flush_interval -1` — fuer WebSocket/SSE (Realtime-Collab in Webstudio)

#### 2d) Wildcard-Block

```caddy
*.webstudio.example.com {
    encode gzip zstd

    tls {
        on_demand
    }

    header {
        X-Content-Type-Options nosniff
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }

    reverse_proxy webstudio-app:3000 {
        flush_interval -1
        header_up Sec-Fetch-Site "same-origin"
    }
}
```

**Wichtig:**
- KEIN `basic_auth` — Iframes koennen keine basic_auth-Credentials uebergeben.
  Auth wird durch Webstudio's OAuth-Cookie gemacht.
- KEIN `X-Frame-Options SAMEORIGIN` — der Apex-Builder embedded das per Iframe
- `tls { on_demand }` aktiviert Per-Subdomain-Cert-Issuing

bcrypt-Hash erzeugen (nur fuer Apex):
```bash
docker run --rm httpd:alpine htpasswd -nbBC 14 USERNAME 'PASSWORT'
```

Caddy reload:
```bash
caddy reload --config /etc/caddy/Caddyfile
# oder bei Container-Caddy:
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

---

### 3) Loopback-Fix (Hairpin-NAT-Workaround)

Der Builder fetcht beim OAuth-Token-Tausch seine eigene public URL. Auf
vielen Docker-Hosts kann ein Container die public IP des Hosts nicht
erreichen (Hairpin-NAT-Problem) → `fetch failed`.

Workaround: `APP_FQDN` container-intern auf die Reverse-Proxy-Container-IP
mappen. Token-Request laeuft dann ueber das interne Proxy-Netz, das Cert
ist trotzdem gueltig.

#### 3a) Reverse-Proxy-Container-IP ermitteln

```bash
docker inspect <proxy-container> \
  --format '{{(index .NetworkSettings.Networks "proxy-network").IPAddress}}'
# Beispiel-Output: 172.21.0.3
```

#### 3b) In `.env` setzen

```env
REVERSE_PROXY_IP=172.21.0.3
```

#### 3c) Container neu starten

```bash
docker compose up -d --force-recreate webstudio-app
```

Die `extra_hosts`-Eintrag in `docker-compose.yml` macht den Rest:

```yaml
webstudio-app:
  extra_hosts:
    - "${APP_FQDN}:${REVERSE_PROXY_IP:-172.21.0.3}"
```

> **Heads-up:** Bei Reverse-Proxy-Restart kann sich die Container-IP aendern.
> Wenn `fetch failed` ploetzlich wieder auftritt: IP neu pruefen, `.env`
> updaten, `docker compose up -d --force-recreate webstudio-app`.

---

### 4) Browser-Cache loeschen + testen

Nach allen Aenderungen:

1. Browser-Cache fuer die Domain loeschen (oder Inkognito-Fenster)
2. `https://webstudio.example.com/` aufrufen
3. basic_auth + DEV_LOGIN durchgehen
4. Neues Projekt anlegen → sollte auf `https://p-<uuid>.webstudio.example.com/...`
   weiterleiten und Builder-Canvas zeigen

---

## Verify (kompletter Smoke-Test)

```bash
# 1. DNS
dig +short webstudio.example.com           # → server-ip
dig +short xx.webstudio.example.com        # → server-ip

# 2. Apex erreichbar (basic_auth verlangt → 401)
curl -sI https://webstudio.example.com/   # 401 (ohne creds)

# 3. OAuth-Pfad NICHT durch basic_auth blockiert
curl -sI https://webstudio.example.com/oauth/ws/token  # 200/302/405 — KEIN 401

# 4. Wildcard live, neues Cert wird issued
curl -sI https://test-$(date +%s).webstudio.example.com/  # 200 oder 404 vom Builder
docker logs caddy 2>&1 | tail -20 | grep "certificate obtained"

# 5. Container kann eigene public URL aufloesen
docker exec webstudio-app sh -c \
  "wget -qO- --header='Host: webstudio.example.com' \
   https://webstudio.example.com/health"
# erwartet: HTTP 200 oder 401 — kein "Connection refused" / "fetch failed"

# 6. Sec-Fetch-Site wird durchgereicht
docker logs webstudio-app 2>&1 | grep -i "cross-origin"
# erwartet: leer
```

---

## Bei Problemen

Section-Verweise in [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md):

| Symptom | Section |
|---|---|
| `NS_ERROR_UNKNOWN_HOST` | 7 |
| `/error 400` "Cross-origin request" | 11 |
| `fetch failed` bei Project-Open | 12 |
| TLS-Cert-Issuing scheitert | 5 |

---

## Fuer Traefik / Nginx

**Traefik:** Wildcard-Cert via DNS-Challenge (Cloudflare/Route53/etc.) statt
on-demand. Traefik unterstuetzt kein on-demand-TLS analog zu Caddy.

**Nginx:** Wildcard-Cert manuell holen (acme.sh + DNS-Challenge), in zwei
`server`-Bloecke einbinden (Apex + Wildcard). Wichtig:
- `proxy_set_header Sec-Fetch-Site "same-origin";` in beiden server-Bloecken
- Auth-Bypass fuer `/oauth/*` und `/auth/*` per `location`-Match
- `proxy_buffering off` fuer SSE/WebSocket
