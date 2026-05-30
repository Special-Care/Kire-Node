# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

KireNode is a VPN proxy gateway for Linux VPS (Ubuntu only). The Python service auto-fetches VPNGate nodes, dials one over OpenVPN onto a `tun0` adapter under a private policy-routing table, and exposes a local SOCKS5/HTTP proxy (port 7928) plus a web admin UI (port 8787). Designed to be the egress for an upstream Xray/3x-ui.

No build, no tests, no lint. Source is plain Python 3 executed by `systemd` on the target VPS.

## Runtime layout (production)

Installed at `/opt/kire-node` by `install.sh`. Runs as the `kire-node.service` systemd unit (`ExecStart=/usr/bin/python3 vpngate_manager.py`, `WorkingDirectory=/opt/kire-node`). All mutable state lives in `/opt/kire-node/vpngate_data/` (gitignored):

- `nodes.json` — full pool of probed nodes (also the IPC channel to the `kire` CLI).
- `state.json` — current status snapshot (active node id, connecting flag, proxy_ok, latencies).
- `ui_auth.json` — web UI host/port/secret_path/username/password (auto-generated on first run).
- `vpngate_auth.txt` — OpenVPN user/pass (default `vpn`/`vpn`), 0600.
- `ip_cache.json` — 7-day cache of ip-api.com enrichment.
- `configs/*.ovpn` — per-node OpenVPN profiles, written on demand and unlinked after disconnect.
- `vpngate.log`, `logs/YYYY-MM-DD.json` — tee'd stdout + structured logs (3-day rotation).

## Common commands

On the target VPS (everything below requires root):

- `kire` / `kire status` — interactive status panel (reads `state.json` / `nodes.json` in a loop).
- `kire start` / `kire stop` / `kire restart` — wrap `systemctl ... kire-node.service`.
- `kire logs` — `tail -f` on `vpngate_data/vpngate.log`.
- `kire update` — `git fetch && git reset --hard origin/{main|master}` then re-runs `install.sh`. **Wipes local edits** unless `/opt/kire-node/.local_dev` exists.
- `kire web` / `kire port` / `kire password` — edit `ui_auth.json` (and prompt to restart).
- `kire uninstall` — disable+remove the unit, delete `/opt/kire-node` and `/usr/bin/kire`.

The `kire` binary is a self-contained Python script embedded inside `install.sh` (cat'd into `/usr/bin/kire`). It is **not** in this repo — when changing `kire` behavior, edit the heredoc in `install.sh`.

Local development on the VPS:

```bash
touch /opt/kire-node/.local_dev          # stop install.sh from clobbering your edits
systemctl restart kire-node.service       # pick up changes to *.py
journalctl -u kire-node -f                # service logs (also tee'd to vpngate.log)
```

Direct run for debugging (bypasses systemd; still needs root for `ip rule` / `openvpn`):

```bash
sudo python3 vpngate_manager.py
```

Env knobs read at startup (see top of `vpngate_manager.py`): `FETCH_INTERVAL_SECONDS`, `CHECK_INTERVAL_SECONDS`, `TARGET_VALID_NODES`, `MAX_SCAN_ROWS`, `OPENVPN_TEST_TIMEOUT_SECONDS`, `OPENVPN_CMD`, `OPENVPN_AUTH_USER/PASS`, `LOCAL_PROXY_HOST/PORT`, `UI_HOST/PORT`, `VPNGATE_DATA_DIR`, `OPENVPN_UPSTREAM_SOCKS`, `OPENVPN_UPSTREAM_HTTP`.

## Architecture

Three Python files. There is no module-style package; `vpngate_manager.py` `import`s the other two directly.

### `vpngate_manager.py` (~3600 lines, single file)

`main()` boots, in this order:
1. `kill_existing_openvpn_processes()` — `pkill -f "openvpn.*tun0|openvpn.*vpngate_data"` to clear stale OpenVPN from a previous crash.
2. Redirects stdout/stderr through `Tee` to `vpngate.log`.
3. Starts `proxy_server.start_proxy_server(127.0.0.1, 7928)` in a thread, waits up to 15 s for the port to listen.
4. Starts three daemon threads:
   - `collector_loop` — every `CHECK_INTERVAL_SECONDS` (~16 min) calls `maintain_valid_nodes`: fetch VPNGate API → merge into `nodes.json` → concurrently probe the top 10 with `test_multiple_nodes` (each gets a unique `tunN` device via `get_free_test_index`) → if no VPN is up, `auto_switch_node`.
   - `background_proxy_checker` — every 30 s runs `check_proxy_health` (curl through SOCKS5 to `ip.sb`, fallback `api.ipify.org`). On failure with an `active_openvpn_node_id`, marks the node unavailable and calls `auto_switch_node`.
   - `active_node_pinger` — every 10 s pings the active node's IP, updates `active_node_latency` in state.
5. Foreground `ThreadingHTTPServer` on `UI_HOST:UI_PORT` with the `Handler` class.

`Handler` (the web UI):
- All requests go through `validate_path()`: rewrites `/<secret_path>/foo` → `/foo`; bare `/<secret_path>` 302's to `/<secret_path>/`; everything else is 404. The secret path comes from `ui_auth.json` and is part of the URL — it is **the** access barrier alongside the password.
- `is_authorized()` checks a `session=<uuid>` cookie against the in-memory `active_sessions` dict (30-day expiry). Sessions are not persisted — restarting the service logs everyone out.
- GET serves: `/` (LOGIN_HTML if unauth, otherwise INDEX_HTML), `/api/nodes`, `/configs/<file>.ovpn` (served from `node["config_text"]`, not disk).
- POST endpoints: `/api/login`, `/api/logout`, `/api/update_settings` (re-verifies current creds, writes `ui_auth.json`, then `os._exit(0)` to force systemd to restart with the new port/host/secret), `/api/check`, `/api/refresh_nodes`, `/api/test_nodes`, `/api/test_node`, `/api/connect`, `/api/disconnect`, `/api/test_proxy`.
- `LOGIN_HTML` and `INDEX_HTML` are giant Python raw-string literals (~2000 lines, roughly lines 1031–3060). Frontend is vanilla JS/CSS — no build pipeline. Edit them in place; they ship as-is to the browser.

Connection lifecycle:
- `connect_node(id)` is guarded by the `is_connecting` global and the `lock` RLock. Sequence: `stop_active_openvpn()` → write `.ovpn` → `run_openvpn_until_ready(keep_alive=True, route_nopull=True, dev="tun0")` → `setup_policy_routing("tun0")`. Sets `active_openvpn_node_id` and updates `state.json`.
- `run_openvpn_until_ready` spawns OpenVPN with `--pull-filter ignore route-ipv6/ifconfig-ipv6 --route-delay 2 --connect-retry-max 1`. It reads OpenVPN's stdout line-by-line, scans for `"Initialization Sequence Completed"`, and on the keep-alive path updates UI status via `update_handshake_status` (Chinese phase labels).
- `stop_active_openvpn()` cleans the policy routing table, kills the process, then `pkill`s any orphans, and unlinks the `.ovpn`.

### `proxy_server.py` — the leak-safe proxy

This is the security-critical file. Two invariants protect against VPS-IP leaks:

1. **Every outbound socket binds to `tun0`** via `setsockopt(SOL_SOCKET, SO_BINDTODEVICE, b"tun0")` in `create_connection`. If `tun0` doesn't exist or the VPN is down, `connect()` fails and the proxy returns `502 Bad Gateway` (HTTP) or SOCKS error `0x04`. It never falls back to the physical NIC.
2. **DNS is resolved manually through `tun0`**, not via the system resolver. `resolve_dns_over_tun0` constructs a raw DNS query, sends it from a UDP socket bound to `tun0` (default `8.8.8.8`), parses A records by hand. Only if that returns an IP do we proceed with `getaddrinfo` on the IP.

Do not "simplify" by removing the `SO_BINDTODEVICE` or by switching to `socket.create_connection` / `socket.getaddrinfo(host, …)` directly — that reintroduces the leak the README's "Fail-Safe Leak Protection" guarantee depends on.

Accept loop handles both HTTP CONNECT/absolute-form and SOCKS5 (version-byte sniff on first byte) in a per-connection thread. No threadpool, no auth.

### `vpn_utils.py`

Stateless helpers used by both files above: `parse_remote` (extracts host/port/proto from .ovpn), `is_config_tcp`, `get_physical_interface` (reads `ip route` and skips `tun/tap/wg/ppp` defaults to find the real NIC for ping-via-physical-interface), `ping_latency_ms` (ping → ping-no-bind → TCP-connect → fallback), `check_and_fix_dns` (WSL heal: appends `1.1.1.1`/`8.8.8.8` to `/etc/resolv.conf` if name resolution fails but raw IP connect works), `load/save_ip_cache`, `enrich_ip_info` (batch POST to `ip-api.com/batch`, 100 at a time, 7-day cache), and the `COUNTRY_TRANSLATIONS` dict (English country name → 中文).

## Non-obvious behaviors to remember

- **Policy routing, not default-route hijacking.** Routing table 100 has `default dev tun0`; rule `ip rule add oif tun0 table 100` is the only thing steering traffic into it. SSH, the admin UI, and any process that doesn't explicitly bind to `tun0` keep using the original NIC. This is what makes the gateway safe to enable over an SSH session.
- **Parallel probes need separate TUNs.** `test_multiple_nodes` calls `get_free_test_index()` to pick a unique `tunN` (2–99) per worker; reusing `tun0` for probes would tear down the live connection. The `keep_alive=False, route_nopull=True` probe path is throw-away — it only confirms initialization completes.
- **IPv4-only resolver monkeypatch** at the top of `vpngate_manager.py` overrides `socket.getaddrinfo` to force `AF_INET`. This is to avoid 10+ s AAAA timeouts on WSL/IPv6-broken hosts; don't remove it unless you're ready to rework all the DNS in the file.
- **Active-node detection is process-based, not state-based.** `kire status` and `Handler` both scan `/proc/*/cmdline` for `vpngate_manager.py` and `openvpn` rather than trusting any file — so a crashed service won't lie about being up.
- **`/api/update_settings` restarts the process** by calling `os._exit(0)` 2 s after writing `ui_auth.json`. This relies on `Restart=always` in the systemd unit. The same mechanism is how port/host changes take effect.
- **`install.sh` is BOTH installer and bundler.** It writes `/lib/systemd/system/kire-node.service` AND emits `/usr/bin/kire` from an inlined Python heredoc. Bumping the `kire` CLI means editing `install.sh`, not a separate file.
- **Debian is technically supported** but `install.sh` hard-rejects non-Ubuntu. The README documents a `sed 's/"${ID:-}"/"ubuntu"/g'` workaround — keep this string stable when refactoring the OS check.
- **Blacklist is currently a no-op.** `load_blacklist()` returns `{}` and `mark_blacklisted()` is a `pass`. Failed nodes are demoted via `probe_status="unavailable"` and `auto_switch_node` skipping them, not via a persistent blacklist. Don't assume a blacklist file exists.
- **No tests, no CI, no formatter config.** Validation is by running the service on a VPS and watching logs.
