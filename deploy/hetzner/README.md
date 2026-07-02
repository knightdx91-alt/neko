# Neko on a Hetzner VPS (Tor + optional desktop)

A step-by-step to run neko on a cheap Hetzner box and open it from a card in
your RetroPlay hub (`index.html` in the pokemon-game repo). The neko session
runs on the **server**, so reloading the tab on your phone reconnects to the
still-running browser — it does **not** reset anything.

## 1. Create the server

- Hetzner Cloud → **CX22** (2 vCPU / 4 GB RAM, ~€4–5/mo) — comfortable for Tor
  Browser + an optional XFCE desktop. (CX11/2 GB works for Tor-only but is tight.)
- Image: **Ubuntu 24.04**.
- Note the server's **public IPv4**.

## 2. Open the firewall

Neko needs its web port **and** the WebRTC UDP range. In Hetzner Cloud →
**Firewalls**, allow inbound:

| Protocol | Port          | Why                       |
|----------|---------------|---------------------------|
| TCP      | 22            | SSH                       |
| TCP      | 80, 443       | web UI / HTTPS (Caddy)    |
| TCP      | 8080          | neko web UI (if no HTTPS) |
| UDP      | 52000–52100   | WebRTC media (Tor)        |
| UDP      | 52200–52300   | WebRTC media (desktop)    |

If you skip HTTPS you can drop 80/443; if you use HTTPS you can drop 8080.

## 3. Install Docker

```bash
ssh root@YOUR.SERVER.PUBLIC.IP
curl -fsSL https://get.docker.com | sh
```

## 4. Deploy neko

Copy this `deploy/hetzner/` folder to the server (or just recreate
`docker-compose.yml`), then edit it:

- Set both `CHANGE_ME_*` **passwords**.
- Set `NEKO_WEBRTC_NAT1TO1` to your **public IPv4**.
- (Optional) uncomment the `neko-desktop` service for a full XFCE desktop.

```bash
docker compose up -d
docker compose logs -f   # watch it come up
```

Open `http://YOUR.SERVER.PUBLIC.IP:8080`, log in with the **admin** password,
and you'll see Tor Browser running. The desktop (if enabled) is on `:8081`.

## 5. (Recommended) HTTPS + a domain

Installing the hub as an app on your phone and some browser features want a
**secure origin**. Easiest path: point a domain (or a free subdomain) at the
server IP and put **Caddy** in front — it gets a Let's Encrypt cert
automatically. Minimal `Caddyfile`:

```
neko.example.com {
    reverse_proxy localhost:8080
}
```

```bash
docker run -d --name caddy --restart unless-stopped --network host \
  -v $PWD/Caddyfile:/etc/caddy/Caddyfile -v caddy_data:/data caddy
```

Then your neko URL is `https://neko.example.com`. (WebRTC UDP still goes direct
to the server IP on 52000–52100, so keep those ports open.)

## 6. Wire it into the hub

In the RetroPlay hub, open the **Neko — Virtual Browser** card. First tap
prompts for the server URL — enter `https://neko.example.com` (or
`http://YOUR.IP:8080`). It's saved in `localStorage`; **long-press** (or
right-click) the card to change it later.

## Why your session persists

- **Tab reload / phone sleeps the tab** → reconnects to the running container.
  Nothing lost.
- **`docker restart` / server reboot** → the `neko-profile` volume keeps your
  Tor Browser profile, so you come back to the same setup (open tabs in Tor are
  cleared by Tor itself for anonymity, but bookmarks/settings persist).
- The only thing that truly resets state is destroying the container **and** the
  volume (`docker compose down -v`) — don't do that unless you mean it.

## Notes

- Tor Browser image = anonymized browsing out of the box. For a full desktop
  where you can install apps, enable the `neko-desktop` (XFCE) service.
- Available images if you want a different browser:
  `ghcr.io/m1k1o/neko/{tor-browser,firefox,chromium,brave,xfce,kde}:latest`.
- Full neko docs: https://neko.m1k1o.net/docs/v3/
