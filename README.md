# plex-nginx

nginx reverse proxy configuration for Plex Media Server. Optimized for:

- Fast Plex Web UI loading (HTTP/2, gzip, upstream keepalive)
- Zero nginx-induced buffering on direct-play streams
- Thumbnail caching to reduce repeated Plex transcoding
- Forcing all clients (including Shield TV) through the proxy

## Assumptions

- nginx installed from **nginx.org packages** (not distro packages): version 1.25+ required for `http2 on` directive
- Plex Media Server running on the same host, listening on `127.0.0.1:32400`
- TLS certificate at `/etc/letsencrypt/live/plex.example.com/` (Let's Encrypt / Certbot)
- Debian/Ubuntu-based host with `/etc/nginx/sites-enabled/` layout

## File Layout

```
/etc/nginx/
├── nginx.conf                  ← replace the default
├── include/
│   ├── ssl.conf                ← TLS settings and security headers
│   ├── gzip.conf               ← compression (excludes video/audio)
│   └── proxy-headers.conf      ← proxy_set_header, Connection/keepalive handling
└── sites-enabled/
    └── plex.conf               ← the Plex vhost
```

## Installation

### 1. Install nginx (nginx.org packages)

```bash
# Debian/Ubuntu
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
  http://nginx.org/packages/debian $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list

sudo apt update && sudo apt install nginx
```

### 2. Deploy the config files

```bash
sudo cp nginx.conf /etc/nginx/nginx.conf
sudo mkdir -p /etc/nginx/include
sudo cp include/ssl.conf /etc/nginx/include/ssl.conf
sudo cp include/gzip.conf /etc/nginx/include/gzip.conf
sudo cp include/proxy-headers.conf /etc/nginx/include/proxy-headers.conf
sudo mkdir -p /etc/nginx/sites-enabled
sudo cp sites-enabled/plex.conf /etc/nginx/sites-enabled/plex.conf
```

Edit `sites-enabled/plex.conf` and update:
- `server_name`: your public domain
- `ssl_certificate` paths: wherever your cert lives

### 3. Create the nginx cache directory

```bash
sudo mkdir -p /var/cache/nginx
sudo chown nginx:nginx /var/cache/nginx
```

### 4. Test and reload

```bash
sudo nginx -t && sudo nginx -s reload
```

### 5. Configure Plex

In **Settings → Network**:

- **Custom server access URLs**: add `https://your-domain.com`
- **Preferred Network Interface**: set to your LAN interface (e.g. `eth0`)

This tells Plex clients to use the proxy URL instead of advertising the direct IP.

## Force All Clients Through the Proxy (iptables)

By default clients on the LAN can reach Plex directly on port 32400, bypassing nginx entirely (Shield TV does this). Block direct access with iptables:

> **Note:** The last rule below drops all direct connections to port 32400 that aren't explicitly allowed. Confirm every host that needs direct access, including nginx itself on `127.0.0.1`, is already in an ACCEPT rule before you apply it, or you'll cut off that host's access.

```bash
# Allow localhost (nginx → Plex)
sudo iptables -A INPUT -p tcp -s 127.0.0.1 --dport 32400 -j ACCEPT

# Allow any other trusted hosts that need direct access (e.g. another proxy)
sudo iptables -A INPUT -p tcp -s 192.0.2.10 --dport 32400 -j ACCEPT

# Drop all other direct connections to port 32400
sudo iptables -A INPUT -p tcp --dport 32400 -j DROP
```

Persist the rules across reboots:

```bash
sudo apt install iptables-persistent
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

Verify:

```bash
sudo iptables -L INPUT -n --line-numbers | grep 32400
```

To remove later:

```bash
sudo iptables -D INPUT -p tcp -s 127.0.0.1 --dport 32400 -j ACCEPT
sudo iptables -D INPUT -p tcp -s 192.0.2.10 --dport 32400 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 32400 -j DROP
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

## How It Works

### Location routing

| Path | Behaviour |
|------|-----------|
| `/:/websockets/` | WebSocket upgrade, real-time Plex notifications |
| `~ ^/(library/parts\|video)/` | Direct-play stream, `proxy_buffering off`, 2 h timeout |
| `/photo/:/` | Thumbnail/poster, nginx cache, 1 h cache lifetime |
| `/` | Everything else (UI, API, metadata), pass-through, `proxy_buffering off` |

### Why `proxy_buffering off` everywhere except thumbnails

With buffering on, nginx holds the full upstream response before relaying anything to the client. Plex's API responses (metadata, poster lists) can be large enough that this delay causes browser timeouts, so buffering is off everywhere except the thumbnail cache. Streaming needs it off too, so nginx relays bytes to the client as they arrive instead of holding the whole stream. The thumbnail cache location keeps buffering on because nginx has to buffer the response before it can write it to disk cache.

### WebSocket map

```nginx
map $http_upgrade $connection_upgrade {
    default  "upgrade";
    ""       "";
}
```

Non-WebSocket requests send `Connection: ""` to clear the header and allow upstream keepalive. Hardcoding `Connection: upgrade` on all requests (a common mistake) prevents keepalive reuse.

## Troubleshooting

**Verify thumbnail cache is working**

```bash
curl -si https://your-domain.com/photo/:/transcode/universal/thumb?... | grep x-cache-status
# x-cache-status: HIT  (after first request)
```

**Check nginx error log**

```bash
sudo tail -f /var/log/nginx/error.log
```

**Test Plex is reachable locally**

```bash
curl -s http://127.0.0.1:32400/identity | grep -o 'version="[^"]*"'
```
