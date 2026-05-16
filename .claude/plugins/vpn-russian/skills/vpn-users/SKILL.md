---
name: vpn-users
description: >-
  Add, list, and disable VPN users on 3x-ui inbounds; sync per-user routing rules in
  extra.json for bridge vs direct exits. Use when adding VPN clients to Russian relay
  or foreign exit servers.
---

# VPN Users Setup

Run whenever a new person needs access. Requires `vpn-3xui-common` login helpers.

## Naming

```
<prefix>-<server>-<number|role>

alice             Russian relay -> foreign bridge
alice-direct      Russian relay -> direct exit
alice-foreign     Foreign direct (bypasses Russian relay)
exit-relay-1      Internal: Russian relay outbound to foreign exit (not for clients)
```

## Helpers

```python
import requests, json, uuid as uuidlib

def login(base, password="<YOUR_PASSWORD>"):
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

def new_client(email, flow="xtls-rprx-vision", existing_uuid=None):
    return {
        "id": existing_uuid or str(uuidlib.uuid4()),
        "flow": flow,
        "email": email,
        "enable": True,
        "expiryTime": 0,
        "totalGB": 0,
        "limitIp": 0,
        "reset": 0,
        "subId": "",
        "tgId": 0,
        "comment": ""
    }

def get_inbound(s, H, base, inb_id):
    r = s.get(f"{base}/panel/api/inbounds/get/{inb_id}", headers=H, timeout=10)
    return r.json()["obj"]

def update_inbound(s, H, base, inb_id, inb):
    r = s.post(f"{base}/panel/api/inbounds/update/{inb_id}", json=inb,
        headers={**H, "Content-Type": "application/json"}, timeout=10)
    return r.json()
```

## Scenario A: Bridge user (Russian -> foreign, split routing)

Bridge users use split routing: `.ru`/`.рф` domains and Russian IPs (`geoip:ru`) go direct through the relay; everything else goes to the foreign exit.

1. Add client to RU inbound via API (`inb_id` usually `2` — verify with list below).
2. Append **three** routing rules to `extra.json` on Russian server (base64 write — `vpn-server-access`):
   ```python
   {"type": "field", "user": [email], "domain": ["ru", "рф"], "outboundTag": "direct"},
   {"type": "field", "user": [email], "ip": ["geoip:ru"], "outboundTag": "direct"},
   {"type": "field", "user": [email], "outboundTag": "exit-foreign"},
   ```
3. `systemctl restart x-ui`.
4. Build link: `vless://{uuid}@<RELAY_IP>:443?...&sni=<RU_SNI>&pbk=...&sid=...&flow=xtls-rprx-vision#{email}`

## Scenario B: Direct RU user

Same as A, but routing rule uses `"outboundTag": "direct"`.

## Scenario C: Direct foreign user

Add to foreign inbound only — no `extra.json` changes. Link uses foreign server IP and foreign SNI.

## Inbound ID lookup

```python
r = s.get(f"{BASE}/panel/api/inbounds/list", headers=H, timeout=10)
for inb in r.json().get("obj", []):
    print(inb["id"], inb["port"], inb["tag"], inb["remark"])
```

## Disable user

Set `cl["enable"] = False` in inbound settings, `update_inbound`, restart x-ui.

For a fully worked Russian relay example (same user/routing model), see `russian-vpn-setup/reference.md`.
