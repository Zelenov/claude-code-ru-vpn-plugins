# Changelog

## 1.1.0 - 2026-09-13

Field fixes from running the chain in production since May 2026.

- **vpn-bridge:** troubleshooting section for a silently filtered relay → exit path
  (handshake passes, payload dropped, xray OOM-killed on the relay) and the reverse SSH
  tunnel workaround hosted by the exit. Client links stay unchanged.
- **vpn-bridge:** end-to-end test pattern - throwaway xray client on the relay, curl through
  it - instead of trusting `openssl s_client`.
- **vpn-users:** `add_client()` rewrites the inbound via `/panel/api/inbounds/update`;
  `/panel/api/inbounds/addClient` returns 404 on 3x-ui 3.4+. Restart x-ui after every client
  change. Put the user's name after `#` in the link and inside the QR.
- **vpn-foreign-exit / russian-vpn-setup:** donor SNI must have a small certificate chain.
  `www.microsoft.com` (~8 KB) makes every client fail with `handshake did not complete`;
  default example is now `gateway.icloud.com`, with a one-liner to measure a candidate.
- **vpn-server-access:** OOM / stuck-socket / tcpdump diagnostics; note on commands the
  auto-mode classifier refuses (SSH keys, authorized_keys, systemd units on remote hosts).
- **README (ru/en):** "when something breaks" section with copy-paste prompts.
- Fixed `README.en.md` and `vpn-foreign-exit/SKILL.md` being saved as UTF-16 (rendered as
  binary on GitHub, unreadable by the skill loader).

## 1.0.0 - 2026-05

Initial release.
