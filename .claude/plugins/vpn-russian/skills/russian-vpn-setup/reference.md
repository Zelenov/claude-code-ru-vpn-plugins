# Russian VPN Setup — Reference

Full commands and API payloads. Assumes paramiko session `c` and `run()` from `vpn-server-access`.

## Step 1: Server prep

```python
import paramiko, os, time

key = paramiko.Ed25519Key.from_private_key_file(os.path.expanduser("~/.ssh/vpn_servers"))
c = paramiko.SSHClient()
c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
c.connect("<RELAY_IP>", username="root", pkey=key, timeout=20)

def run(cmd, timeout=30):
    stdin, stdout, stderr = c.exec_command(cmd, timeout=timeout)
    out = stdout.read().decode(); err = stderr.read().decode()
    return stdout.channel.recv_exit_status(), out, err

run("apt update && apt upgrade -y && apt install -y curl ufw sqlite3", timeout=120)

for cmd in ["ufw allow 22/tcp", "ufw allow 80/tcp", "ufw allow 443/tcp", "ufw allow 54321/tcp", "ufw --force enable"]:
    run(cmd)
```

## Step 2: Install 3x-ui

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
/usr/local/x-ui/x-ui setting -username admin -password '<YOUR_PASSWORD>'
systemctl restart x-ui
```

Panel HTTPS: see `vpn-3xui-common`.

## Step 3: Generate Reality keys

```python
rc, out, err = run("/usr/local/x-ui/bin/xray-linux-amd64 x25519")
# Parse PrivateKey: and Password (PublicKey): lines — see main SKILL.md

import secrets
short_id = secrets.token_hex(4)
SNI = "www.wildberries.ru"
```

## Phase 1: Create inbound

```python
import requests, json

BASE = "https://<panel.example.com>:54321/<PANEL_PATH>"
PASS = "<YOUR_PASSWORD>"

def login(base, password):
    s = requests.Session()
    s.get(f"{base}/", timeout=10)
    csrf = s.get(f"{base}/csrf-token", headers={"X-Requested-With": "XMLHttpRequest"}, timeout=10).json()["obj"]
    s.post(f"{base}/login", json={"username": "admin", "password": password},
        headers={"X-CSRF-Token": csrf, "X-Requested-With": "XMLHttpRequest"}, timeout=10)
    csrf = s.get(f"{base}/csrf-token", headers={"X-Requested-With": "XMLHttpRequest"}, timeout=10).json()["obj"]
    return s, {"X-CSRF-Token": csrf, "X-Requested-With": "XMLHttpRequest"}

s, H = login(BASE, PASS)

inbound = {
    "remark": "ru-relay-clients",
    "enable": True,
    "listen": "",
    "port": 443,
    "protocol": "vless",
    "settings": json.dumps({
        "clients": [],
        "decryption": "none",
        "fallbacks": []
    }),
    "streamSettings": json.dumps({
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
            "show": False,
            "xver": 0,
            "dest": f"{SNI}:443",
            "serverNames": [SNI],
            "privateKey": priv_key,
            "shortIds": [short_id],
            "settings": {
                "publicKey": pub_key,
                "fingerprint": "chrome",
                "serverName": "",
                "spiderX": "/"
            }
        },
        "tcpSettings": {"acceptProxyProtocol": False, "header": {"type": "none"}}
    }),
    "sniffing": json.dumps({
        "enabled": True,
        "destOverride": ["http", "tls", "quic"],
        "metadataOnly": False,
        "routeOnly": False
    }),
    "tag": "inbound-443"
}

r = s.post(f"{BASE}/panel/api/inbounds/add", json=inbound,
    headers={**H, "Content-Type": "application/json"}, timeout=10)
print(r.json())
```

Add users with `vpn-users` (`alice`, `alice-direct`).

## Phase 2: Xray wrapper

```python
rc, out, _ = run("ps aux | grep xray | grep -v grep | head -2")
print(out)
BINARY = "xray-linux-amd64"

run(f"test -f /usr/local/x-ui/bin/{BINARY}.real || mv /usr/local/x-ui/bin/{BINARY} /usr/local/x-ui/bin/{BINARY}.real")

wrapper = f"#!/bin/bash\nexec /usr/local/x-ui/bin/{BINARY}.real \"$@\" -c /usr/local/x-ui/bin/extra.json\n"
run(f"""python3 -c "
with open('/usr/local/x-ui/bin/{BINARY}', 'w') as f:
    f.write({repr(wrapper)})
" """)
run(f"chmod +x /usr/local/x-ui/bin/{BINARY}")
```

## Phase 2: extra.json

```python
import base64, json

EXIT_IP       = "<EXIT_IP>"
EXIT_UUID     = "<EXIT_UUID>"
EXIT_PUB_KEY  = "<PUBLIC_KEY>"
EXIT_SHORT_ID = "<SHORT_ID>"
FOREIGN_SNI   = "gateway.icloud.com"   # small cert chain; www.microsoft.com breaks Reality (vpn-foreign-exit)

extra = {
    "outbounds": [{
        "tag": "exit-foreign",
        "protocol": "vless",
        "settings": {
            "vnext": [{
                "address": EXIT_IP,
                "port": 443,
                "users": [{
                    "id": EXIT_UUID,
                    "flow": "xtls-rprx-vision",
                    "encryption": "none"
                }]
            }]
        },
        "streamSettings": {
            "network": "tcp",
            "security": "reality",
            "realitySettings": {
                "serverName": FOREIGN_SNI,
                "fingerprint": "chrome",
                "shortId": EXIT_SHORT_ID,
                "publicKey": EXIT_PUB_KEY,
                "spiderX": "/"
            }
        }
    }],
    "routing": {
        "rules": [
            # Bridge user split routing: .ru/.рф and Russian IPs → direct; rest → foreign exit.
            # geosite:ru does NOT work (not in 3x-ui bundled geosite.dat).
            # Use domain TLD matching + geoip:ru instead.
            {"type": "field", "user": ["alice"], "domain": ["ru", "рф"], "outboundTag": "direct"},
            {"type": "field", "user": ["alice"], "ip": ["geoip:ru"], "outboundTag": "direct"},
            {"type": "field", "user": ["alice"], "outboundTag": "exit-foreign"},
            {"type": "field", "user": ["alice-direct"], "outboundTag": "direct"}
        ]
    }
}

extra_json = json.dumps(extra, indent=2)
encoded = base64.b64encode(extra_json.encode()).decode()
run(f"""python3 -c "
import base64, json
data = base64.b64decode('{encoded}').decode()
json.loads(data)
with open('/usr/local/x-ui/bin/extra.json', 'w') as f:
    f.write(data)
print('written ok')
" """)
```

## Phase 2: Verify

```python
run("systemctl restart x-ui", timeout=30)
time.sleep(5)
rc, out, _ = run("journalctl -u x-ui --no-pager -n 10")
print(out)
rc, out, _ = run("ps aux | grep xray | grep -v grep")
print(out)
```

Expect: `Reading config: ...extra.json`, `prepend outbound with tag: exit-foreign`, process line with both `config.json` and `extra.json`.
