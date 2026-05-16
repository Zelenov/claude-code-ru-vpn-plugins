---
name: vpn-3xui-common
description: >-
  Install and operate 3x-ui on any VPS: install script, admin password via x-ui binary
  (never raw SQL), panel HTTPS via nginx+Let's Encrypt. Use for 3x-ui panel setup on
  Russian or foreign VPN servers.
---

# Common 3x-ui Server Setup

## Install

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
/usr/local/x-ui/x-ui setting -username admin -password '<YOUR_PASSWORD>'
systemctl restart x-ui
```

**Never** set the bcrypt password via raw SQL — shell `$` corrupts hashes.

Default panel: `http://<IP>:54321/<PANEL_PATH>`. For production use HTTPS below (v3.0.2 built-in HTTPS is broken).

## Panel HTTPS (nginx + Let's Encrypt)

Prerequisites: port 80 open, subdomain DNS → server IP, nginx installed.

### 1. Bind panel to localhost

```bash
sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='2053' WHERE key='webPort';"
sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='127.0.0.1' WHERE key='webListen';"
systemctl restart x-ui
# ss -tlnp | grep 2053  → 127.0.0.1:2053
```

### 2. nginx TLS terminator on 54321

Write an nginx site config with:
- `listen 54321 ssl` and `server_name <panel.example.com>`
- `location /` -> `proxy_pass http://127.0.0.1:2053`
- `listen 80` ACME server block with `/.well-known/acme-challenge/`
- redirect `http://` to `https://<panel.example.com>:54321`

When running remotely through SSH, write the config with the base64 pattern from `vpn-server-access`.

### 3. certbot

```bash
certbot certonly --webroot -w /var/www/html -d <panel.example.com>
# Point ssl_certificate paths to /etc/letsencrypt/live/.../fullchain.pem and privkey.pem
nginx -t && systemctl reload nginx
```

API base becomes: `https://<panel.example.com>:54321/<PANEL_PATH>/`

## API login helper

```python
import requests

def login(base, password):
    s = requests.Session()
    s.get(f"{base}/", timeout=10)
    csrf = s.get(f"{base}/csrf-token",
        headers={"X-Requested-With": "XMLHttpRequest"}, timeout=10).json()["obj"]
    s.post(f"{base}/login",
        json={"username": "admin", "password": password},
        headers={"X-CSRF-Token": csrf, "X-Requested-With": "XMLHttpRequest"}, timeout=10)
    csrf = s.get(f"{base}/csrf-token",
        headers={"X-Requested-With": "XMLHttpRequest"}, timeout=10).json()["obj"]
    return s, {"X-CSRF-Token": csrf, "X-Requested-With": "XMLHttpRequest"}
```

Extended end-to-end Russian relay flow with the same panel assumptions: `russian-vpn-setup` + `russian-vpn-setup/reference.md`.
