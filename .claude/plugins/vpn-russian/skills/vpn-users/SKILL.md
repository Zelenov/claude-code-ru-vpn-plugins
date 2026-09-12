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

def add_client(s, H, base, inb_id, email, flow="xtls-rprx-vision"):
    """Add a client by rewriting the whole inbound. Works on every 3x-ui version.
    Do NOT use POST /panel/api/inbounds/addClient - it returns HTTP 404 (empty body)
    on 3x-ui 3.4+, which surfaces as a confusing JSONDecodeError."""
    inb = get_inbound(s, H, base, inb_id)
    st = inb["settings"]
    st = json.loads(st) if isinstance(st, str) else st   # 3.4+ returns parsed objects
    for c in st["clients"]:
        if c["email"] == email:
            return c["id"]                                # idempotent
    client = new_client(email, flow)
    st["clients"].append(client)
    body = dict(inb)
    for k in ("streamSettings", "sniffing", "allocate"):  # must go back as JSON strings
        if isinstance(body.get(k), (dict, list)):
            body[k] = json.dumps(body[k])
    body["settings"] = json.dumps(st)
    for k in ("clientStats", "created_at", "updated_at"):
        body.pop(k, None)
    r = update_inbound(s, H, base, inb_id, body)
    assert r.get("success"), r
    return client["id"]
```

After any client change the panel has saved to its DB, but **xray keeps serving the old
client list until `systemctl restart x-ui`**. Always restart, then verify with the
end-to-end test in `vpn-bridge` (a throwaway xray client on the relay) - a passing
`openssl s_client` probe proves nothing about Reality.

## Scenario A: Bridge user (Russian -> foreign, split routing)

Bridge users use split routing: `.ru`/`.рф` domains and Russian IPs (`geoip:ru`) go direct through the relay; everything else goes to the foreign exit.

1. `uuid = add_client(s, H, BASE, inb_id, email)` on the Russian panel (`inb_id` usually `2` — verify with list below).
2. Append **three** routing rules to `extra.json` on Russian server (base64 write — `vpn-server-access`):
   ```python
   {"type": "field", "user": [email], "domain": ["ru", "рф"], "outboundTag": "direct"},
   {"type": "field", "user": [email], "ip": ["geoip:ru"], "outboundTag": "direct"},
   {"type": "field", "user": [email], "outboundTag": "exit-foreign"},
   ```
3. `systemctl restart x-ui`.
4. Build link: `vless://{uuid}@<RELAY_IP>:443?...&sni=<RU_SNI>&pbk=...&sid=...&flow=xtls-rprx-vision#{email}`
5. Test as that user from the relay (`vpn-bridge` § End-to-end test): exit IP must be the foreign server, a `.ru` site must return 200.

## Naming the connection in the app

The text after `#` in the `vless://` link is what Happ / Hiddify / v2RayTun show as the
connection name. Put the **user's** email there (`#alice`), not the inbound remark - otherwise
every person sees the same generic name and support calls become guesswork. When you make a
QR code, encode the full link including the `#name`, so scanning imports the right name too.
Never hand out the panel's default subscription/QR for a shared inbound.

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
