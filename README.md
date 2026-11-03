# GL.iNet-5800-Client-Update
A patched version of client update binary for GL.iNet 5800

Not sure if this would be useful for anyone , also never just copy paste binaries in your router unless you know what you do and what IT does!Recently I bought a GL.iNet 5800 travel router, my FTTH is really bad and sometimes it disconnects all together to my goal was to have ethernet connected to this router with a sim card with data so it would switch automatically when ethernet (FTTH) is down, happily I did the setup , tested , all works but after a few days I see that I can't login to the 5800 router anymore , internet does work but super slow. I understood something is wrong and I did a bit of search and saw other people had the same issue. ssh'd to the router and checked a few things and immediately I noticed that the gl_clients keeps growing in ram, set a script to output the ram usage and in the morning I saw that it took over the whole memory and again it's not accessible, took the binary out and analyzed it a bit and saw that it's a lua script , I do dabble in programming but I have no clue about LUA so I gave it to claude to analyze it and once he found the source of the mem leak I asked him to patch it, I will attach the patched file here which goes to /etc/init.d/ and replaces the original file (take a backup first if you want to) once you have replaced the binary better restart the router once to make sure the new binary gets loaded properly although you can use " etc/init.d/gl_clients stop && sleep 3 && /etc/init.d/gl_clients enable"here is also the claude notes: 

Identified Leak Sources

1. clients table grows unbounded (PRIMARY LEAK)

Clients are added on discovery (update_cache, every update_cache_interval)

clean_offline_clients (proto#2) only deletes from SQLite DB

No evidence the clients[mac] = nil cleanup happens in the Lua table

Lua 5.1 hash tables don't shrink when keys are set to nil

2. clients_ip_map never cleaned

Built during each cache update cycle

Maps IP → client MAC

Old IPs from disconnected/reconnected devices accumulate

No cleanup function references this table

3. No explicit garbage collection

No collectgarbage() calls detected in string constants

Lua 5.1 GC on embedded systems may not run aggressively enough

Each timer cycle allocates temporary tables, strings, closures

4. SQLite open/close churn

update_db opens and closes SQLite on every cycle

Creates Lua userdata for DB handle each time

If GC doesn't collect old handles fast enough → handle accumulation

5. cjson.encode(filtered_clients) on socket requests

Each web UI poll creates a full JSON copy of all client data

Large string allocation freed only by GC

Admin panel polls every few seconds

6. assoclists rebuilt each cycle

Full WiFi association list queried via ubus

Previous table not explicitly freed, relies on GC

Timer Functions (all running concurrently)

Timer

Function

What it does

timer_cache

update_cache

Scans /proc/oui-tertf/term, adds clients

timer_db

update_db

Opens SQLite, writes client stats, closes

timer_rate

update_rate

Calculates speed per client

timer_assoclist

check_assoclist

Queries WiFi associations

timer_wan_traffic

update_wan_traffic

Reads /sys/class/net/*/statistics

timer_mtk

update_mtk_hwnat

Reads MediaTek HNAT offload stats

Fix Feasibility

Cannot easily recompile — needs GL.iNet's custom Lua 5.1 build with type 9 integers

Could write a wrapper that restarts the daemon every N hours as a crude fix

Best approach: keep disabled, write lightweight on-demand replacement if client list is needed

Upload method: netcat via screen session (same as download), target /usr/bin/gl_clients_update

Persistence: file survives reboots but NOT firmware upgrades (sysupgrade may overwrite /usr/bin/)
