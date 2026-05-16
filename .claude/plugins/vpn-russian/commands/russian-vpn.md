---
description: Set up a Russian VPS as VLESS+Reality relay (phase 1 direct, phase 2 bridge to foreign exit)
argument-hint: [relay-ip]
---

Set up a Russian VPN relay using the **vpn-russian** plugin workflow.

$ARGUMENTS

Follow `/vpn-russian:russian-vpn-setup` end to end:

1. Gather missing inputs (relay IP, panel URL/password, foreign exit keys if phase 2).
   - If foreign exit is not ready, run `vpn-foreign-exit` first.
2. Complete Phase 1 (standalone direct exit) and verify with a `*-direct` user.
3. Only then add the bridge (Xray wrapper + `extra.json`) for Phase 2.

Use sibling skills as needed: `vpn-foreign-exit`, `vpn-server-access`, `vpn-3xui-common`, `vpn-users`, `vpn-bridge`.
