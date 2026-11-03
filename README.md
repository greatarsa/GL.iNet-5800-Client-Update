# GL.iNet GL-E5800 — Patched `gl_clients_update`

A patched version of the `gl_clients_update` binary for the GL.iNet GL-E5800 travel router, fixing a memory leak that eventually makes the router unresponsive.

> **Warning:** Never blindly copy-paste binaries onto your router unless you understand what the file does. Take a backup first.

## Background

The GL-E5800's `gl_clients_update` daemon (a compiled Lua 5.1 script) tracks connected clients. Over time its memory usage grows unbounded until it consumes all available RAM, causing the router's web UI to become inaccessible and degrading network performance.

## Identified Leak Sources

1. **`clients` table grows unbounded (primary leak)**
   - Clients are added on discovery every `update_cache_interval`
   - `clean_offline_clients` only deletes from the SQLite DB, not from the in-memory Lua table
   - Lua 5.1 hash tables don't shrink when keys are set to `nil`

2. **`clients_ip_map` never cleaned**
   - Built during each cache update cycle (IP → client MAC)
   - Old IPs from disconnected/reconnected devices accumulate
   - No cleanup function references this table

3. **No explicit garbage collection**
   - No `collectgarbage()` calls detected
   - Lua 5.1 GC on embedded systems may not run aggressively enough
   - Each timer cycle allocates temporary tables, strings, and closures

4. **SQLite open/close churn**
   - `update_db` opens and closes SQLite on every cycle
   - Creates Lua userdata for the DB handle each time
   - If GC doesn't collect old handles fast enough → handle accumulation

5. **`cjson.encode(filtered_clients)` on socket requests**
   - Each web UI poll creates a full JSON copy of all client data
   - Admin panel polls every few seconds

6. **`assoclists` rebuilt each cycle**
   - Full WiFi association list queried via ubus
   - Previous table not explicitly freed, relies on GC

## Timer Functions

| Timer | Function | What it does |
|-------|----------|-------------|
| `timer_cache` | `update_cache` | Scans `/proc/oui-tertf/term`, adds clients |
| `timer_db` | `update_db` | Opens SQLite, writes client stats, closes |
| `timer_rate` | `update_rate` | Calculates speed per client |
| `timer_assoclist` | `check_assoclist` | Queries WiFi associations |
| `timer_wan_traffic` | `update_wan_traffic` | Reads `/sys/class/net/*/statistics` |
| `timer_mtk` | `update_mtk_hwnat` | Reads MediaTek HNAT offload stats |

## Installation

1. Back up the original file on your router:
   ```bash
   cp /usr/bin/gl_clients_update /usr/bin/gl_clients_update.bak
   ```

2. Copy `gl_clients_patched` to the router and replace the original:
   ```bash
   cp gl_clients_patched /usr/bin/gl_clients_update
   ```

3. Restart the daemon:
   ```bash
   /etc/init.d/gl_clients stop && sleep 3 && /etc/init.d/gl_clients enable
   ```
   Or simply reboot the router.

## Notes

- The patched file survives reboots but **not** firmware upgrades (`sysupgrade` may overwrite `/usr/bin/`)
- Cannot easily recompile — requires GL.iNet's custom Lua 5.1 build with type 9 integers
