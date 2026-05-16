---
name: vpn-bridge
description: >-
  Configure Xray wrapper and extra.json so a 3x-ui relay forwards selected users to a
  foreign VLESS+Reality exit. Use for VPN bridge, relay leg, Russian→foreign routing, or
  debugging relay vs exit failures.
---

# VPN Bridge / Relay Setup

```
Client → Relay:443 (VLESS+Reality, Russian SNI) → Exit:443 (VLESS+Reality, foreign SNI) → Internet
```

3x-ui regenerates `config.json` on restart — manual outbound edits are lost. **Fix:** replace the xray binary with a wrapper that appends `extra.json`.

## Install wrapper

```python
BINARY = "xray-linux-amd64"   # verify: ps aux | grep xray

run(f"test -f /usr/local/x-ui/bin/{BINARY}.real || mv /usr/local/x-ui/bin/{BINARY} /usr/local/x-ui/bin/{BINARY}.real")

wrapper = f"#!/bin/bash\nexec /usr/local/x-ui/bin/{BINARY}.real \"$@\" -c /usr/local/x-ui/bin/extra.json\n"
run(f"""python3 -c "
with open('/usr/local/x-ui/bin/{BINARY}', 'w') as f:
    f.write({repr(wrapper)})
" """)
run(f"chmod +x /usr/local/x-ui/bin/{BINARY}")
```

## extra.json shape

Bridge users get **split routing**: `.ru`/`.рф` and Russian IPs go direct through the relay; everything else goes to the foreign exit.

```python
extra = {
    "outbounds": [{
        "tag": "exit-foreign",
        "protocol": "vless",
        "settings": {
            "vnext": [{
                "address": "<EXIT_IP>",
                "port": 443,
                "users": [{
                    "id": "<EXIT_UUID>",
                    "flow": "xtls-rprx-vision",
                    "encryption": "none"
                }]
            }]
        },
        "streamSettings": {
            "network": "tcp",
            "security": "reality",
            "realitySettings": {
                "serverName": "<EXIT_SNI>",
                "fingerprint": "chrome",
                "shortId": "<SHORT_ID>",
                "publicKey": "<PUBLIC_KEY>",
                "spiderX": "/"
            }
        }
    }],
    "routing": {
        "rules": [
            # bridge user: .ru/.рф domains → direct (Russian server)
            {"type": "field", "user": ["alice"], "domain": ["ru", "рф"], "outboundTag": "direct"},
            # bridge user: Russian IPs → direct
            {"type": "field", "user": ["alice"], "ip": ["geoip:ru"], "outboundTag": "direct"},
            # bridge user: everything else → foreign exit
            {"type": "field", "user": ["alice"], "outboundTag": "exit-foreign"},
            # direct user: all traffic → Russian server
            {"type": "field", "user": ["alice-direct"], "outboundTag": "direct"}
        ]
    }
}
```

Write via base64 (see `vpn-server-access`). Rules append after defaults in `config.json` (api/private/bittorrent only).

## Split routing: domain matching

**Why not `geosite:ru`:** The bundled `geosite.dat` from 3x-ui does not include a `ru` category — it silently fails.
`ext:geosite_RU.dat:ru` also fails — the file exists but the tag name differs.

**Working solution:**
- `"domain": ["ru", "рф"]` — xray subdomain matching by TLD (matches `*.ru` and `*.рф`)
- `"ip": ["geoip:ru"]` — standard `geoip.dat` has `ru`, works correctly

## Add bridge user routing

For each new bridge user: add three rules (domain→direct, ip→direct, catch-all→exit-foreign), base64-write, restart x-ui. See `vpn-users` Scenario A.

## Debug: bypass exit

```python
run("echo '{}' > /usr/local/x-ui/bin/extra.json")
run("systemctl restart x-ui")
# If user now works (shows relay IP), problem was exit outbound config
```

## Verify after restart

```bash
journalctl -u x-ui --no-pager -n 10
# Must see: Reading config: ...extra.json
# Must see: prepend outbound with tag: exit-foreign

ps aux | grep xray | grep -v grep
# Must show: xray-linux-amd64.real ... config.json ... extra.json
```

Full Russian two-phase flow: `russian-vpn-setup`.
