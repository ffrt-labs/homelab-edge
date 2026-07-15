# homelab-edge

The single HTTPS entry point for this home server. One Caddy container, one wildcard
certificate, one place to see every app that's reachable from the LAN.

## Scope

This repo owns exactly one thing: the edge proxy. It does **not** own application containers,
monitoring, backups, or anything else — those live in their own repos and deploy independently.
Keeping this repo small and boring is deliberate: it's the one component that, if broken, takes
every app on the server down at once.

## How it works

- **One wildcard cert** (`*.yourdomain.com`) via Let's Encrypt DNS-01, using a Cloudflare API
  token. DNS-01 means Caddy never needs an inbound port 80 — it proves domain ownership by
  writing a temporary TXT record to your Cloudflare zone, which Let's Encrypt reads. No HTTP
  challenge, no port 80 needed at all.
  - Why wildcard instead of a cert per app: certificate issuance is published to public
    Certificate Transparency logs (see crt.sh). A cert per app leaks the name of every service
    you run. A wildcard only ever publishes "`*.yourdomain.com` exists."
- **One shared Docker network, `edge`.** Every app joins it as `external: true` and publishes
  **no** ports of its own to the host. Caddy is the only container with a published port (443),
  and reaches each app by its Compose service name over Docker's internal DNS
  (`reverse_proxy mycoach:8000`).
- **Explicit routing table.** The `Caddyfile` in this repo lists one `handle` block per app. It's
  the single file that answers "what does this server serve, and where does it go" — no Docker
  labels, no socket access, no magic.
- **Public DNS, not local (Pi-hole) DNS.** Even though this server runs Pi-hole, we deliberately
  use a public wildcard A record rather than a Pi-hole-only override. Android's Private DNS and
  Chrome's Secure DNS both bypass the system resolver (and therefore Pi-hole) by default — a
  Pi-hole-only record would silently fail to resolve on phones with either enabled. A public
  record pointed at a private LAN IP is robust regardless of resolver, and is not itself an
  exposure: the IP simply isn't routable from outside the LAN.

## One-time setup

1. **Static IP for the server.** Router admin → DHCP → give this box a reserved/static lease so
   the DNS record below never goes stale.
2. **Cloudflare DNS record:**
   - Type `A`, name `*` (→ `*.yourdomain.com`), value = the server's static LAN IP.
   - **Proxy status: DNS only (grey cloud).** If it's orange ("Proxied"), Cloudflare fronts the
     record with its own edge, which cannot reach a private IP. Must be grey.
3. **Cloudflare API token:**
   - Cloudflare dashboard → profile icon → **My Profile** → **API Tokens** → **Create Token**
   - Use the **"Edit zone DNS"** template, scoped to **this one zone only** (not "All zones").
   - Not the Global API Key.
4. **`docker network create edge`** — one-time, on the server itself. (The deploy workflow also
   creates this idempotently on every run, so a cold rebuild in either order still works — but do
   it manually once so the very first deploy of any app isn't racing this repo's first deploy.)
5. **GitHub repo settings** → Settings → Secrets and variables → Actions:
   - **Variables**: `EDGE_BASE_DOMAIN` = `yourdomain.com` (bare — no subdomain, the Caddyfile
     prepends `*.` itself).
   - **Secrets**: `CLOUDFLARE_API_TOKEN` = the token from step 3.
6. Push to `main` (or re-run the workflow). Watch it come up:
   ```bash
   docker compose logs -f caddy   # watch the DNS-01 challenge and cert issuance
   ```

## Adding a new app

1. In the app's own `docker-compose.yml`: join the shared network, publish no host ports.
   ```yaml
   services:
     myapp:
       # ... no `ports:` section ...
       networks:
         - edge

   networks:
     edge:
       external: true
   ```
   Use a specific, non-generic service name (not `app` or `web`) — Compose registers the service
   name as a DNS alias on every network it joins, and a generic name risks colliding with another
   app on the shared `edge` network.
2. In this repo's `Caddyfile`, add a matcher + handle block:
   ```
   @myapp host myapp.{$EDGE_BASE_DOMAIN}
   handle @myapp {
       reverse_proxy myapp:PORT
   }
   ```
   Add it *above* the catch-all `handle { abort }` block at the end.
3. No DNS change, no new cert — the wildcard already covers the new subdomain.
4. `docker compose up -d` here to reload Caddy with the new route.

## Verification

```bash
# Config is valid
docker compose config

# Caddyfile parses and the Cloudflare module is present (fails only on a
# dummy token, which is expected without real credentials)
docker compose build caddy
docker run --rm -e EDGE_BASE_DOMAIN=example.com -e CLOUDFLARE_API_TOKEN=dummy \
  -v "$(pwd)/Caddyfile:/etc/caddy/Caddyfile:ro" \
  homelab-edge-caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile

# Once deployed, from any LAN machine:
curl -v https://mycoach.yourdomain.com/api/system/status   # 200, valid wildcard cert
curl -sk https://nonsense.yourdomain.com                    # connection dropped (catch-all)
sudo ss -ltnp | grep -E ':(80|443) '                         # Pi-hole still on :80, caddy on :443
```

## Why this exists

Originally, MyCoach ran its own Caddy container to get HTTPS for its offline PWA logger. That was
fine with one app. It stopped being fine once more apps needed HTTPS too: every app defining its
own `caddy` service means every one of them fights over host port 443 (this is literally how the
project's first deploy to this server failed — `Bind for 0.0.0.0:80 failed: port is already
allocated`, because Pi-hole already owned port 80). This repo exists so there's exactly one
process on the server that ever binds 80/443, decoupled from any single app's release cycle.
