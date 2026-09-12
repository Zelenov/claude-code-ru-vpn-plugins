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

## End-to-end test from the relay (no phone needed)

Run a throwaway xray client **on the relay itself**, pointed at the relay's own inbound with
the user's UUID, and curl through it. It shows exactly what that user would see.

```python
import base64, json
test = {"log": {"loglevel": "warning"},
 "inbounds": [{"port": 18081, "listen": "127.0.0.1", "protocol": "http", "tag": "in"}],
 "outbounds": [{"protocol": "vless",
   "settings": {"vnext": [{"address": "<RELAY_IP>", "port": 443,
                "users": [{"id": "<USER_UUID>", "flow": "xtls-rprx-vision", "encryption": "none"}]}]},
   "streamSettings": {"network": "tcp", "security": "reality",
     "realitySettings": {"serverName": "<RU_SNI>", "fingerprint": "chrome",
                         "shortId": "<SHORT_ID>", "publicKey": "<PUBLIC_KEY>"}}}]}
enc = base64.b64encode(json.dumps(test).encode()).decode()
run(f"python3 -c \"import base64;open('/tmp/utest.json','w').write(base64.b64decode('{enc}').decode())\"")
run("(/usr/local/x-ui/bin/xray-linux-amd64.real run -c /tmp/utest.json >/tmp/utest.log 2>&1 &); sleep 2; "
    "printf 'exit IP: '; curl -sS -m 15 -x http://127.0.0.1:18081 https://api.ipify.org; echo; "
    "printf 'ru site: '; curl -sS -m 15 -x http://127.0.0.1:18081 -o /dev/null -w '%{http_code}\\n' https://www.gosuslugi.ru/; "
    "pkill -f utest.json; rm -f /tmp/utest.json", timeout=60)
```

Expect: bridge user → exit IP and `.ru` → 200 (goes direct); `*-direct` user → relay IP.
`openssl s_client` is only a reachability probe - it passes even when Reality itself is
broken (see the donor-certificate note in `vpn-foreign-exit`).

## Troubleshooting: relay → exit leg silently filtered

Seen 2026-09 on a Russia → Netherlands pair after months of working. Symptoms:

- Every bridge user has no internet, `*-direct` users work, and the exit works for clients
  outside Russia.
- On the relay `ss -tno state established | grep <EXIT_IP>:443` shows hundreds of sockets
  with non-zero `Send-Q` and `timer:(on,…,12)` retransmit counters.
- `journalctl -k | grep 'Out of memory'` on the relay shows xray OOM-killed, often weekly for
  a long time - the stuck sockets eat RAM, so the "VPN died" moment is usually the OOM.
- From the relay, `openssl s_client -connect <EXIT_IP>:443 -servername <FOREIGN_SNI>` hangs.
  So does `curl http://<EXIT_IP>/`, and even reading the SSH banner from `<EXIT_IP>:22`.
  Ping, large-MTU ping and `mtr` are clean, neighbouring IPs in the exit's /24 answer.
- Packet captures on both ends: the TCP handshake completes, but data segments never arrive,
  in either direction. Connections initiated **from the exit to the relay** carry data fine.

Diagnosis (run on the relay unless noted):

```python
run("ss -tn state established | grep -c <EXIT_IP>:443; ss -tno state established | grep <EXIT_IP>:443 | head -3")
run("journalctl -k --no-pager | grep 'Out of memory' | tail -5")
run("timeout 8 openssl s_client -connect <EXIT_IP>:443 -servername <FOREIGN_SNI> </dev/null 2>&1 | grep -E 'Cipher is' || echo RELAY->EXIT TLS FAILS")
run("timeout 5 bash -c 'exec 3<>/dev/tcp/<EXIT_IP>/22 && head -c 25 <&3' || echo NO-SSH-BANNER")
run("ping -M do -c 2 -s 1400 <EXIT_IP> | tail -1")          # MTU sanity
# reverse direction - run ON THE EXIT. Must succeed if the relay itself is healthy:
run("timeout 8 openssl s_client -connect <RELAY_IP>:443 -servername <RU_SNI> </dev/null 2>&1 | grep 'Cipher is'")
```

If relay-initiated probes fail while exit-initiated ones pass, no config on either box will
fix it: the path is filtered for that direction. Options: move the exit to a new IP (it may
get filtered again), or reuse the direction that works.

### Workaround: reverse SSH tunnel (exit → relay)

The exit keeps a persistent SSH session into the relay and publishes its own `:443` as
`127.0.0.1:10443` on the relay. The bridge outbound then targets localhost. Reality keys,
SNI, users and client links do not change. Measured ~25 MB/s on 1-vCPU VPSes.

1. On the **exit**: `ssh-keygen -t ed25519 -N '' -f /root/.ssh/rtun -C exit-reverse-tunnel`
2. On the **relay**, append the public key to `/root/.ssh/authorized_keys`, restricted to
   this one forward so the key can neither run commands nor open other ports:
   ```
   restrict,port-forwarding,permitlisten="127.0.0.1:10443" ssh-ed25519 AAAA... exit-reverse-tunnel
   ```
   `sshd -T | grep -E 'allowtcpforwarding|permitlisten'` must say `yes` / `any` (Ubuntu default).
3. On the **exit**, write `/etc/systemd/system/rtun-relay.service`. Keep the options one per
   line - a single long `ExecStart` gets wrapped by terminal paste and ssh then dies with
   `option requires an argument -- o`:
   ```ini
   [Unit]
   Description=Reverse SSH tunnel exit->relay (relay 127.0.0.1:10443 -> exit :443)
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   ExecStart=/usr/bin/ssh -N -T \
     -o ServerAliveInterval=15 \
     -o ServerAliveCountMax=3 \
     -o ExitOnForwardFailure=yes \
     -o StrictHostKeyChecking=accept-new \
     -o IdentitiesOnly=yes \
     -i /root/.ssh/rtun \
     -R 127.0.0.1:10443:127.0.0.1:443 \
     root@<RELAY_IP>
   Restart=always
   RestartSec=5
   StartLimitIntervalSec=0

   [Install]
   WantedBy=multi-user.target
   ```
   `systemctl daemon-reload && systemctl enable --now rtun-relay && systemctl status rtun-relay`
4. On the **relay**: `ss -tlnp | grep 10443` must show sshd listening, and
   `openssl s_client -connect 127.0.0.1:10443 -servername <FOREIGN_SNI>` must complete.
5. In `extra.json` on the relay set the `exit-foreign` outbound to
   `"address": "127.0.0.1", "port": 10443` (nothing else changes). Back up the old file,
   `systemctl restart x-ui`, run the end-to-end test above: bridge user must show the exit IP.

Revert: point the outbound back at `<EXIT_IP>:443`, restart x-ui, then
`systemctl disable --now rtun-relay` on the exit.

Steps 1-3 create a key and a service on remote hosts; Claude Code's auto-mode safety
classifier refuses to run those. Give the user the exact commands to paste (one host at a
time - check the prompt shows the right hostname), then continue with steps 4-5 yourself.

Full Russian two-phase flow: `russian-vpn-setup`.
