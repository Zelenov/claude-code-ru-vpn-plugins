---
name: russian-vpn-setup
description: >-
  Provisions a Russian VPS as a VLESS+Reality relay on 3x-ui (Ubuntu 22.04/24.04).
  Phase 1: standalone direct exit for testing. Phase 2: Xray wrapper + extra.json bridge
  to a foreign exit. Use when setting up a Russian VPN relay, RU server, VLESS Reality
  Russia, TSPU bypass, or continuing from a prepared foreign exit setup.
---

# Russian VPN Relay Setup

Two-phase setup on a Russian VPS. **Always complete Phase 1 and verify before Phase 2.**

## Gather inputs first

Ask the user for anything missing:

| Variable | Example | Notes |
|----------|---------|-------|
| `RELAY_IP` | `138.x.x.x` | Russian VPS public IP |
| `PANEL_URL` | `https://panel.example.com:54321/<path>/` | HTTPS panel base (see `vpn-3xui-common`) |
| `PANEL_PASSWORD` | — | Set via x-ui binary, never raw SQL |
| `RU_SNI` | `www.wildberries.ru` | Russian domain only (TSPU 2026) |
| `EXIT_IP` | — | Foreign exit (phase 2 only) |
| `EXIT_UUID` | — | Relay user on foreign inbound |
| `EXIT_PUB_KEY` | — | From `vpn-foreign-exit` |
| `EXIT_SHORT_ID` | — | From `vpn-foreign-exit` |
| `FOREIGN_SNI` | `gateway.icloud.com` | Foreign exit donor; must have a small cert chain (`vpn-foreign-exit`) - `www.microsoft.com` breaks clients |

Foreign exit must exist first (`vpn-foreign-exit` or equivalent). SSH: `~/.ssh/vpn_servers` — see `vpn-server-access`.

## Russian SNI donors (2026)

TSPU whitelists by IP+SNI — use **Russian domains only**:

| Domain | Notes |
|--------|-------|
| `www.wildberries.ru` | Recommended |
| `www.ozon.ru` | Good alternative |
| `2gis.ru` | Good option |
| `ok.ru`, `mail.ru` | Works |
| `vk.com` | Popular, more inspection |
| `yandex.ru` | Stable; Yandex analyzes traffic |

## Progress checklist

```
- [ ] Step 1: Server prep (apt, UFW 22/80/443/54321)
- [ ] Step 2: Install 3x-ui + panel password + HTTPS (vpn-3xui-common)
- [ ] Step 3: Generate Reality keys (Russian SNI)
- [ ] Phase 1: VLESS+Reality inbound on 443 + test users
- [ ] Phase 1 test: *-direct client shows RU IP
- [ ] Phase 2a: Xray wrapper (vpn-bridge)
- [ ] Phase 2b: extra.json outbound + per-user routing
- [ ] Phase 2c: Restart + journalctl verify
- [ ] Phase 2 test: bridge user → foreign IP; direct user → RU IP
```

## Workflow

### Step 1–3: Prep, 3x-ui, keys

Follow [reference.md](reference.md) § Steps 1–3. Use `vpn-server-access` for SSH and `vpn-3xui-common` for install/panel HTTPS.

**Reality key parsing** — xray output format is NOT `Private key:` / `Public key:`:

```python
priv_key, pub_key = "", ""
for line in out.splitlines():
    if line.startswith("PrivateKey:"):
        priv_key = line.split(":", 1)[1].strip()
    elif line.startswith("Password (PublicKey):"):
        pub_key = line.split(":", 1)[1].strip()
```

### Phase 1: Direct exit (standalone)

1. Create inbound `ru-relay-clients` on port 443 — full JSON in [reference.md](reference.md) § Phase 1.
2. Sniffing: `destOverride: ["http", "tls", "quic"]` — **no fakedns**.
3. Add users via `vpn-users`: e.g. `alice` (later bridged), `alice-direct` (stays direct).
4. **Test**: connect with `alice-direct` — visible IP must be **Russian server IP**.

### Phase 2: Bridge to foreign exit

Only after Phase 1 works.

1. Install Xray wrapper — `vpn-bridge` skill.
2. Write `extra.json` with `exit-foreign` outbound + routing rules — [reference.md](reference.md) § Phase 2.
3. `systemctl restart x-ui`; verify journal shows `Reading config: ...extra.json` and `prepend outbound with tag: exit-foreign`.
4. **Test**: `alice` → foreign IP; `alice-direct` → RU IP.

## Client link format

```
vless://<UUID>@<RELAY_IP>:443?type=tcp&security=reality&fp=chrome&sni=<RU_SNI>&pbk=<PUBLIC_KEY>&sid=<SHORT_ID>&flow=xtls-rprx-vision#<name>
```

## Adding users later

Use `vpn-users` → `add_client()` (the panel's `addClient` endpoint is gone in 3x-ui 3.4+).
Each new **bridge** user needs routing rules in `extra.json` and an x-ui restart; xray does
not pick up new clients without the restart. Put the user's name after `#` in the link.

## Debug: bypass foreign leg

Empty `extra.json` -> all traffic exits from Russian server (isolates Russian direct vs Russian->foreign). Restore when done. See `vpn-bridge`.

## If it breaks months later

| Symptom | Where to look |
|---------|---------------|
| Bridge users dead, `*-direct` fine, xray on relay OOM-killed, hundreds of stuck sockets to `<EXIT_IP>:443` | `vpn-bridge` § Troubleshooting → reverse SSH tunnel |
| All clients `handshake did not complete`, openssl probes pass | Donor cert too large → `vpn-foreign-exit` § Choosing the donor |
| New user's key does nothing | x-ui not restarted after the panel change → `vpn-users` |

## Related plugin skills

| Skill | Use for |
|-------|---------|
| `vpn-foreign-exit` | Prepare foreign VLESS+Reality exit node and relay credentials |
| `vpn-server-access` | SSH paramiko, file writes, diagnostics |
| `vpn-3xui-common` | Install, password, panel HTTPS |
| `vpn-users` | Add bridge/direct users + routing sync |
| `vpn-bridge` | Xray wrapper, extra.json, end-to-end test, filtered-path workaround |

## Repo docs

Optional context: `vpn/VPN_Setup_Guide.md`, `vpn/servers.md` (credentials — do not echo secrets into chat).
