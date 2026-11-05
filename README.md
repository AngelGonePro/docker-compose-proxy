# WARNING: NGINX-UI REQUIRES THE PASSWORD TO BE UNDER 20 CHARACTERS, DO NOT GO OVER IN THE .env!!

# docker-compose-proxy

> **WARNING: nginx-ui requires the admin password to be under 20 characters.
> Don't go over in `.env`.**

Two nginx-ui reverse-proxy stacks joined by an FRP tunnel:

| Zip | Runs on | What it does |
|---|---|---|
| `linode-nginx-stack.zip` | the Linode (public edge) | nginx-ui with a wildcard cert, frps, fail2ban, connection rate limiting for Minecraft and other tunneled ports, and an inbound blocklist for the whole host |
| `proxy-nginx.zip` | debian-proxy at home (`10.0.0.100`) | nginx-ui with its own copy of the wildcard cert, frpc (the tunnel to the Linode), and site configs for home services |

Each zip has its own detailed README inside (`linode-nginx-stack/README.md`,
`proxy-nginx/README.md`).

## Install / update: one command each

The same command installs a fresh copy and updates an existing one. It
downloads the zip from GitHub and applies it without touching your `.env`
values, files you've hand-edited or runtime state. Then it pulls and builds
the versions pinned in `.env`, recreates the containers and checks they all
came back up.

**Linode:**
```bash
curl -fsSL --retry 3 -o /tmp/linode-nginx-stack.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/linode-nginx-stack.zip && python3 -c "import zipfile;open('/tmp/linode-nginx-stack-deploy.sh','wb').write(zipfile.ZipFile('/tmp/linode-nginx-stack.zip').read('linode-nginx-stack/deploy-update.sh'))" && sh /tmp/linode-nginx-stack-deploy.sh file:///tmp/linode-nginx-stack.zip ~/linode-nginx-stack
```

**Home (debian-proxy):**
```bash
curl -fsSL --retry 3 -o /tmp/proxy-nginx.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/proxy-nginx.zip && python3 -c "import zipfile;open('/tmp/proxy-nginx-deploy.sh','wb').write(zipfile.ZipFile('/tmp/proxy-nginx.zip').read('proxy-nginx/deploy-update.sh'))" && sh /tmp/proxy-nginx-deploy.sh file:///tmp/proxy-nginx.zip ~/proxy-nginx
```

After uploading a new zip to GitHub, wait 5 minutes before running the
command: GitHub serves the previous copy for up to 5 minutes.

## Fresh install, step by step

Run everything as root (`sudo -s`). Set up the Linode first, then home.

### 1. Prerequisites (both machines)

```bash
apt update && apt install -y curl python3 openssl
command -v docker || curl -fsSL https://get.docker.com | sh
df -h /var /home
```

If `/var` is its own small partition (a few GB), move Docker's storage off it
first; see [Disk full on /var](#disk-full-on-var-no-space-left-on-device).

### 2. The Linode

1. Download the files. The first run puts them in `~/linode-nginx-stack` and
   stops:
   ```bash
   curl -fsSL --retry 3 -o /tmp/linode-nginx-stack.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/linode-nginx-stack.zip && python3 -c "import zipfile;open('/tmp/linode-nginx-stack-deploy.sh','wb').write(zipfile.ZipFile('/tmp/linode-nginx-stack.zip').read('linode-nginx-stack/deploy-update.sh'))" && sh /tmp/linode-nginx-stack-deploy.sh file:///tmp/linode-nginx-stack.zip ~/linode-nginx-stack
   ```
2. Fill in the settings:
   ```bash
   nano ~/linode-nginx-stack/.env
   ```
   | Key | What to put |
   |---|---|
   | `ROOT_DOMAIN` | `cosmoscraft.net` |
   | `CF_API_TOKEN` | Cloudflare API token with **Zone:DNS:Edit** for that zone only |
   | `LE_EMAIL` | your email, for certificate notices |
   | `NGINX_UI_ADMIN_EMAIL`, `NGINX_UI_ADMIN_USER`, `NGINX_UI_ADMIN_PASSWORD` | the nginx-ui admin account (password **under 20 characters**) |
   | `FRP_TOKEN` | a long random secret: `openssl rand -hex 24`. Home uses the same value. |
   | `FRP_DASHBOARD_PASSWORD` | a password for the frps dashboard |
   | `COMPOSE_PROFILES` | `firewall` (turns on fail2ban and the blocklist) |
   | `FAIL2BAN_WHITELIST_IPS` | your own static IPs, so you can't lock yourself out |
   | `RATE_LIMIT_EXCLUDE_PORTS` | public ports that should never be rate-limited |
3. Start it: run the command from step 1 again. It ends with
   `Update complete and verified`.
4. Check:
   ```bash
   cd ~/linode-nginx-stack && docker compose ps
   docker logs bootstrap --tail 5
   docker logs blocklist --tail 3
   ```
   `bootstrap` should say the cert was imported, and `blocklist` should say
   `active — dropping new connections from ... networks`.
5. In the Linode Cloud Firewall, allow inbound TCP 80, 443 and
   `FRP_BIND_PORT` (7000), plus the public game ports you tunnel.

### 3. Home (debian-proxy)

1. Download the files (first run stops):
   ```bash
   curl -fsSL --retry 3 -o /tmp/proxy-nginx.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/proxy-nginx.zip && python3 -c "import zipfile;open('/tmp/proxy-nginx-deploy.sh','wb').write(zipfile.ZipFile('/tmp/proxy-nginx.zip').read('proxy-nginx/deploy-update.sh'))" && sh /tmp/proxy-nginx-deploy.sh file:///tmp/proxy-nginx.zip ~/proxy-nginx
   ```
2. Fill in the settings:
   ```bash
   nano ~/proxy-nginx/.env
   ```
   | Key | What to put |
   |---|---|
   | `ROOT_DOMAIN`, `CF_API_TOKEN`, `LE_EMAIL` | same as the Linode |
   | `NGINX_UI_ADMIN_*` | this nginx-ui's admin account (password **under 20 characters**) |
   | `FRP_SERVER_ADDR` | `45.79.25.46` (the Linode) |
   | `FRP_SERVER_PORT`, `FRP_TOKEN`, `TUNNEL_LOCAL_PORT` | exactly the Linode's `FRP_BIND_PORT`, `FRP_TOKEN`, `TUNNEL_LOCAL_PORT` |
3. Add your tunnels (Minecraft and so on):
   ```bash
   nano ~/proxy-nginx/frpc/frpc.toml
   ```
   Proxy names must end in the public port, e.g.
   `minecraft-java-25565-realip`. The Linode's rate limiter finds them by
   that name. Examples are in the file.
4. Start it: run the command from step 1 again.
5. Check the tunnel:
   ```bash
   docker logs frpc 2>&1 | grep -i "login to server success" | tail -1
   ```

### 4. Site configs (in the nginx-ui panels)

Templates are in each stack's `nginx-sites/` folder. In each panel: **Sites →
Add Site**, paste the file, change the backend IP/port, save and enable.

| Panel | Add |
|---|---|
| Home, `http://10.0.0.100:9001` | `home-tunnel-relay.conf` (traffic from the Linode), `home-direct-lan-vpn.conf` (LAN/VPN), `threatmap.conf` (optional, the wg-vpn-gateway threat map), the Pterodactyl ones if needed |
| Linode | `linode-tunneled-site.conf` for each domain whose backend is at home, the Pterodactyl ones if needed |

For a site served through Cloudflare Tunnel instead, add a public hostname in
Cloudflare Zero Trust pointing at `http://10.0.0.100:80`.

### 5. Threat map: send the Linode's blocks home (optional)

Needs wg-vpn-gateway on debian-proxy with the split tunnel and the
`dashboard` profile (see the wireguard-upstream-vpn README, steps 2 and 5).

1. **On the Linode**, turn on the read-only stats and copy the token it
   prints:
   ```bash
   cd ~/linode-nginx-stack && TOK=$(openssl rand -hex 24) && sed -i "s|^BLOCKLIST_STATS_LISTEN=.*|BLOCKLIST_STATS_LISTEN=192.168.209.180:9180|; s|^BLOCKLIST_STATS_TOKEN=.*|BLOCKLIST_STATS_TOKEN=$TOK|" .env && docker compose up -d blocklist && sleep 15 && docker logs blocklist --tail 2; echo "TOKEN: $TOK"
   ```
2. **On debian-proxy**, with that token in place of `PASTE_TOKEN`:
   ```bash
   cd ~/wg-vpn-gateway && TOK=PASTE_TOKEN && sed -i "s|^DASHBOARD_UPSTREAM_URL=.*|DASHBOARD_UPSTREAM_URL=http://192.168.209.180:9180/stats|; s|^DASHBOARD_UPSTREAM_TOKEN=.*|DASHBOARD_UPSTREAM_TOKEN=$TOK|" .env && docker compose up -d
   ```
3. A minute later, `docker logs wg-vpn-gateway-dashboard-collector 2>&1 | grep -i upstream | tail -1`
   should say `upstream: ok`.

Use `192.168.209.180`, not the WireGuard address `10.13.13.1`: StratumVPN
sends every reply from `10.13.13.0/24` out through Windscribe, so home would
never get an answer.

## Updating

Run the same install command. What it reports:

- **"Hand-edited by you ... left untouched"**: the file differs from what was
  last installed, so your version is kept. The `diff` line it prints shows
  what the new release would change.
- **"New .env keys added"**: new settings, added at the bottom of `.env` with
  their defaults.
- **"These .env keys are no longer used"**: nothing in the current release
  reads them. They're left in place. Delete them only once you're sure your
  own edited files don't use them either.
- **"Update complete and verified"**: done. The current `.env` is saved as
  `.env.last-good`.
- **"[FAIL]"**: details are in `update-failures.log` in the stack folder. The
  version pins (`*_VERSION`) go back to the last working ones and the stack
  is redeployed. Every other `.env` setting stays as you set it.

To redeploy the pinned versions without downloading a new release:
```bash
cd ~/linode-nginx-stack && ./update.sh
cd ~/proxy-nginx && ./update.sh
```

## Troubleshooting

### Disk full on /var ("no space left on device")

Debian can put `/var` on its own small partition, and Docker keeps everything
under `/var/lib/docker` and `/var/lib/containerd`. Check with `df -h /var /home`.
To move Docker to `/home` (all containers stop for a few minutes, then start
again):

```bash
systemctl stop docker.socket docker containerd && \
mkdir -p /home/docker-data/docker /home/docker-data/containerd && \
cp -a /var/lib/docker/. /home/docker-data/docker/ && \
cp -a /var/lib/containerd/. /home/docker-data/containerd/ && \
mv /var/lib/docker /var/lib/docker.old && mv /var/lib/containerd /var/lib/containerd.old && \
mkdir /var/lib/docker /var/lib/containerd && \
echo '/home/docker-data/docker /var/lib/docker none bind 0 0' >> /etc/fstab && \
echo '/home/docker-data/containerd /var/lib/containerd none bind 0 0' >> /etc/fstab && \
systemctl daemon-reload && mount /var/lib/docker && mount /var/lib/containerd && \
systemctl start containerd docker && sleep 15 && docker ps --format '{{.Names}}: {{.Status}}'
```

If any step fails, the rest is skipped; start Docker again with
`systemctl start containerd docker`. When everything shows `Up`, free the old
space: `rm -rf /var/lib/docker.old /var/lib/containerd.old`

### Checks

| What | Command |
|---|---|
| Everything running | `docker compose ps` (in the stack folder) |
| Why an update failed | `tail -40 update-failures.log` |
| Tunnel up (home) | `docker logs frpc 2>&1 \| grep -i "login to server success" \| tail -1` |
| Blocklist loaded (Linode) | `docker logs blocklist --tail 3` |
| Blocklist firewall table present (Linode) | `nft list table inet linode_blocklist \| grep -c elements` |
| Rate limiter (Linode) | `docker logs ratelimit-gen --tail 5` |
| fail2ban bans (Linode) | `docker exec fail2ban fail2ban-client status nginx-http-auth` |
