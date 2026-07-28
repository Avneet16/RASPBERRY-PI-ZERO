# Pi Runbook

## Phase 0 — Backup
Full backup of prior system taken before wipe: pi-backup-full.tar.gz
Contains: custom scripts, systemd units, nginx configs+certs, AdGuard config,
Unbound config, Vaultwarden data (db.sqlite3 verified via PRAGMA integrity_check = ok),
Redis dump, crontabs, ufw rules, Tailscale state.
Stored off-device on phone + additional backup location.
Old 16GB SD card kept as physical fallback (untouched, labeled).

## Phase 1 — Fresh OS
Raspberry Pi OS Lite (32-bit/armhf), Debian 13 "trixie"
Flashed via Raspberry Pi Imager to 32GB SanDisk card.
SSH enabled, password authentication (deliberate choice — easier to
manage during rebuild; revisit key-only auth once stable).
WiFi credentials preloaded (for AP-failover use later, NOT as a second
LAN client — see Phase 3 notes).
Raspberry Pi Connect: explicitly declined/masked (redundant with Tailscale,
extra attack surface for no new capability here).

## Phase 2 — Base hardening
- fail2ban installed, sshd jail active
- ufw installed: default deny incoming / allow outgoing, port 22/tcp allowed
- unattended-upgrades configured
- rpi-connect-lite (auto-installed via apt) masked

## Phase 3 — Networking (in progress)
[to be filled in]

## Services & ports (running log)
| Service | Port | Interface scope | Notes |
|---|---|---|---|
| SSH | 22 | all | password auth for now |

## Credentials & secrets (locations, not values)
- Telegram bot token: pi-tele.sh / telegram-send.sh (restore from backup)
- Vaultwarden data: /opt/vaultwarden (restore from backup, integrity-verified)
- nginx basic auth: .htpasswd, .htpasswd-adguard (restore or regenerate)

## Phase 3 — Networking (verified)
- systemd-networkd not present/conflicting on fresh image; NetworkManager (netplan renderer) is sole network manager
- eth0: primary, DHCP, single default route
- wlan0: no longer dual-active client on same subnet as eth0
- 3-tier failover script: /usr/local/bin/network-failover.sh
  1. eth0 (primary)
  2. wlan0 client, known networks (auto via NetworkManager)
  3. wlan0 AP "Pi-Failover" (192.168.50.1/24, connection name wlan0-ap) - last resort, for
     emergency management access AND as a portal to add new WiFi creds when relocating
- Tested: eth0-down -> wlan0 client reconnect (pass); eth0-up -> wlan0 disconnect (pending verification)
- Note: standalone hostapd.conf / dnsmasq.d/wlan0-ap.conf are unused leftovers -
  AP is driven entirely via the wlan0-ap NetworkManager connection profile instead
- Not yet wired to a timer - currently manual-only via network-failover.sh

## Phase 3 — Networking (fully verified)
- Forward path tested: eth0 down -> wlan0 auto-reconnects to known network (confirmed 10:03)
- Reverse path tested: eth0 up -> wlan0 client disconnects, single default route restored (confirmed 10:05)
- Both directions produce clean logger output via `logger -t network-failover`

## Phase 4a — Unbound (recursive resolver)
- Restored tuned config from backup (QNAME minimization, DNSSEC hardening,
  Pi-tuned cache sizes) - config verified sound, not clutter as initially assumed
- Restored remote-control cert pair for unbound-control on :8953
- Root hints + DNSSEC trust anchor regenerated fresh (not restored - these are
  meant to be system-generated, not carried over)
- Decision: stayed with root hints (full recursion) over Quad9-over-TLS forwarding
  - Benchmarked both: root hints cold ~300-670ms, Quad9-TLS cold ~260-270ms but
    failed on 2/4 test queries (TLS session drops)
  - Warm cache: Unbound repeat query = 0ms; Quad9 test showed no improvement
    (artifact of kdig opening a fresh TLS session each call, not representative
    of Unbound-as-client behavior in production)
  - Chose root hints: matches the original privacy goal (no single third party
    sees full DNS history), latency cost is one-time per cache TTL window
- Listening on 127.0.0.1:5335 only - not exposed to network, AdGuard will forward here

## Correction — Phase 3 dnsmasq cleanup
- Standalone dnsmasq (installed Phase 3 for the original hostapd+dnsmasq AP plan,
  later superseded by driving AP purely via NetworkManager's ipv4.method=shared)
  was left RUNNING despite being `disable`d - disable only prevents future boot
  starts, doesn't stop an already-running process
- Was silently squatting on 0.0.0.0:53, conflicting with AdGuard Home setup
- Fixed: explicitly `stop` + `mask` both dnsmasq and hostapd
- Lesson: after `systemctl disable` on a service you don't want, always also
  check `systemctl status` / `ss -tulnp` to confirm it's not still live from
  its initial apt-install auto-start

## Phase 4b — AdGuard Home
- Fresh install via official script, /opt/AdGuardHome
- Admin panel locked to 127.0.0.1:3000 (loopback only) - wizard defaults to
  0.0.0.0:80 which would have exposed unauthenticated admin panel to LAN;
  caught and corrected before any real exposure
- DNS listening 0.0.0.0:53 (intentional - needs to serve the whole LAN)
- Upstream set to 127.0.0.1:5335 (Unbound) - verified via dashboard: 100% of
  queries routed there, ~146ms average response time
- Fixed system-level DNS leak: Pi's own /etc/resolv.conf was still pointing at
  router/ISP DNS via NetworkManager auto-config, bypassing AdGuard entirely -
  corrected via `nmcli ... ipv4.dns 127.0.0.1 ipv4.ignore-auto-dns yes` (+ ipv6
  equivalent to stop IPv6 leak too)
- Blocklists not yet re-added (declined restore of old config, rebuilding fresh)
- ufw: port 53 opened for eth0 + wlan0 only, not global
- Access to admin panel currently via SSH tunnel only (ssh -L 3000:localhost:3000) -
  temporary until nginx reverse proxy is rebuilt in a later phase

## Phase 6 — Proton WireGuard geo-exit
- Config built correctly first time with table-52 fix already in place (from
  Phase 5/earlier debugging): carve-out rule (to 100.64.0.0/10 -> table 52,
  priority 50) sits above the catch-all (from 100.64.0.0/10 -> table 200,
  priority 100) - SSH/dashboards over Tailscale unaffected by Proton state
- Caught and fixed: IPv6 leak. Proton tunnel is IPv4-only (AllowedIPs = 0.0.0.0/0
  only), but Tailscale's exit-node feature routes both v4 and v6 by default with
  no per-family toggle. Without a fix, IPv6 traffic from exit-node clients was
  bypassing Proton entirely and leaking real ISP/location via eth0's IPv6.
- Fix: net.ipv6.conf.all.forwarding set to 0 in /etc/sysctl.d/99-tailscale.conf -
  IPv6 now fails closed for exit-node clients instead of leaking; IPv4 traffic
  unaffected and continues through Proton correctly
- Verified: curl -4 ifconfig.me (via phone, exit-node=Pi) shows Proton IP;
  curl -6 ifconfig.me fails/times out as expected
- Toggle workflow unchanged: `sudo wg-quick up/down proton`, manual only,
  not enabled at boot

## Correction — Phase 7a, AdGuard's own auth removed
- Decision: removed AdGuard Home's built-in user auth entirely, relying on
  nginx basic auth as the sole gate (justified: AdGuard binds 127.0.0.1:3000
  only, unreachable except through nginx; adding AdGuard's own login was
  redundant and its 15-min lockout-on-disk was actively counterproductive
  during setup/testing)
- If defense-in-depth is wanted later, re-add via the validated process
  above (htpasswd -B -n, edit yaml, validate with python3 yaml.safe_load
  BEFORE restarting - this exact sequence broke twice from manual edits)

## Correction — Phase 7a, actual root cause of the auth loop
- Root cause: `sudo htpasswd -c ...` recreate command was suggested multiple
  times during troubleshooting but not actually executed until the final
  attempt - .htpasswd-adguard still had the OLD "admin:" entry the whole
  time, so every "anon" login attempt correctly failed (user genuinely
  didn't exist in the file yet)
- Lesson: when a fix doesn't take effect, verify the file's actual current
  contents directly (`cat` the htpasswd/config file) before troubleshooting
  further upstream - don't assume a previously-given command was run
- Final working state: nginx basic auth username "anon", AdGuard's own
  login removed (nginx is sole auth gate - see prior correction)
- Verified: phone -> Tailscale (100.97.48.99:9442) -> nginx auth -> AdGuard
  dashboard, working end-to-end

## Phase 8 (partial) — Swap via zram
- Installed zram-tools, /etc/default/zramswap: ALGO=lz4, PERCENT=50 (~211MB
  swap on a 424MB Pi)
- Gotcha: first `systemctl enable --now` failed with "mkswap: /dev/zram0 is
  mounted" - a stale/partially-initialized zram0 device was already active
  as swap when the service tried to configure it fresh
- Fix: `sudo swapoff /dev/zram0` + `echo 1 | sudo tee /sys/block/zram0/reset`
  to force a clean slate, then `systemctl restart zramswap.service`
- Verified survives a real reboot, not just this session
- Dashboard now surfaces swap % and MB used/total on the Overview tab

## Incident resolved — remote lockout root cause (full writeup)

Two independent bugs combined to cause the lockout:

**Bug 1 — AdGuard silently broke AP mode.**
AdGuard's `bind_hosts: 0.0.0.0` wildcard-bound port 53 on every address,
including ones assigned later. When wlan0-ap activated and got 192.168.50.1,
NetworkManager's internal dnsmasq tried to bind DNS on that same address and
failed ("Address already in use"), causing wlan0-ap to activate then die on
~30s loop, over and over. This is why AP mode was so unreliable/hard to
connect to during the actual incident.
Fix: static IP for eth0 (192.168.1.8), AdGuard bind_hosts restricted to
127.0.0.1 / 192.168.1.8 / 100.97.48.99 instead of 0.0.0.0.

**Bug 2 — the actual root cause of why wlan0-client reconnect kept failing.**
`nmcli device connect wlan0` (by device, not connection name) picks "the best
available connection" using internal heuristics that favor whichever
connection was most recently activated. Because wlan0-ap kept getting
activated repeatedly (testing, then the real incident), its timestamp stayed
newer than the home WiFi profile's — so "connect the device" kept silently
choosing the AP profile over the real home network, even with strong signal
and correct credentials. This was likely the root cause going back to the
very first test.
Fix: always activate by explicit connection name
(`nmcli connection up netplan-wlan0-Airtel_nish_4886`), never by device alone.
Updated in network-failover.sh at both call sites.

**Bug 3 (design gap, also fixed this session) — no retry once AP latched.**
Original script only rechecked for eth0 returning once in AP mode; never
retried the wifi-client path on its own. Fixed with a periodic retry
(every 5 min while in AP mode) using a timestamp file at
/run/network-failover-last-wifi-retry.

**Also fixed this session:** Telegram alerting wired into the watchdog's
log() function - confirmed working on a real failover event this time.

**Verified end-to-end**, real eth0-down test, with the actual fix in place:
eth0 down -> wlan0 correctly connects to real home WiFi (not AP) -> Telegram
alert fires -> eth0 back up -> wlan0 disconnects, single default route restored.

**Process lesson:** should have armed an `at` auto-revert before the FIRST
live network test while remote, not just later ones. New standing rule:
no live network changes while remote, full stop, until a smart WiFi plug is
in place for guaranteed remote power-cycle capability.

## Phase 7d — Multi-server Proton switching
- Each server gets its own config: /etc/wireguard/proton-<name>.conf, same
  table-52/IPv6-leak fixes baked into every one (us, romania, swiss configured
  so far)
- Helper actions: proton-list, proton-status (now reports which server, not
  just up/down), proton-switch <name>, proton-off
- Dashboard Proton card redesigned: per-server CONNECT/SWITCH TO/ACTIVE
  buttons instead of a single toggle, since only one server can hold table
  200's route at a time - switching means clean teardown of whichever is
  active before bringing up the new one

**Bug found and fixed:** the helper script has `set -euo pipefail`. Several
proton-* actions did `... | grep '^proton-' | ...` to detect whether
anything was currently up. When NOTHING is up (the normal "off" state),
grep finding no match returns exit 1 - which pipefail propagates and
set -e treats as fatal, silently killing the whole script before it ever
reached the `wg-quick up` line. Result: switching between two already-active
servers worked fine (grep always matched something), but turning Proton ON
from a fully-off state silently failed every time - exit code 1, zero output,
nothing useful to debug from the dashboard side alone.
Fix: append `|| true` after every grep-based detection in this script, so
"found nothing" is treated as a valid empty result, not a fatal error.
Lesson: under `set -e -o pipefail`, any grep/pipeline that can legitimately
return "no match" as a normal expected state needs an explicit `|| true`,
or set -e will silently abort mid-script with no error output at all.

## Phase 8d — Dashboard: undervoltage/throttle flag + OOM-kill watch
- New helper actions: `throttled` (vcgencmd get_throttled) and `oom-events`
  (journalctl -k grep for OOM-killer lines, last 24h)
- Overview tab: POWER tile next to SWAP - separates "active right now" from
  "occurred since boot" (vcgencmd bit layout: bits 0-3 current state, bits
  16-19 since-boot), so a brownout that already cleared doesn't get lost
- MEMORY EVENTS card lists any kernel OOM kills in the last 24h - swap
  already running ~30% used at idle on this Zero 2 W (425MB RAM / 424MB
  zram swap) made this worth having ahead of RAM pressure actually causing
  a kill, not just after
- oom-events journalctl call gets `|| true`, per the Phase 7d pipefail
  lesson above - a clean "no matches" run must not abort the script
- Polled: throttled every 3s (transient brownouts are quick), oom-events
  every 15s (not time-critical)

## Phase 8e — Dashboard: refresh-latency fix + fail2ban/zram/validate/events/long-trends

**Performance root cause found:** dashboard refresh "taking a few seconds"
traced to `/api/updates/check` running `apt-get update` synchronously on
every single page load - a multi-second network call that, with waitress's
small thread pool, could stall other API calls on the same page load.
Fix: served from a cache refreshed by a background thread every 30min; the
CHECK button now hits a new `/api/updates/recheck` endpoint for an
explicit live check instead. Same root-cause class as the proton-off
pipefail bug in Phase 7d - a slow/blocking call sitting where the rest of
the app follows a strict cache-then-serve pattern.

**Second latency contributor:** Chart.js (204KB, the single biggest static
asset) was a render-blocking `<script>` in `<head>` on every page load
regardless of which tab was open. Now lazy-loaded only on first visit to
the System tab; the vendored file also gets a 1-year cache header (app.js/
style.css keep default revalidation since those still change often).

**New features, same session:**
- Fail2ban ban-list card + unban button (helper: fail2ban-banned,
  fail2ban-unban) - SSH is still password-auth (Phase 1), so fail2ban is
  the real front line, not just a status light
- Config-validate-before-restart for AdGuard/Unbound (helper:
  validate-config): restarting either now runs its config check first and
  blocks the restart on failure - directly closes the Phase 7a gap where a
  bad manual YAML edit broke auth twice because nothing validated before
  restarting. Needs PyYAML present on the Pi for the AdGuard check
  (`python3 -c "import yaml"`) - not yet confirmed installed on this image
- Zram compression ratio (helper: zram-stats) folded into the existing
  SWAP tile instead of a new card
- oom-events broadened into a unified `system-events` helper action -
  supersedes Phase 8d's standalone MEMORY EVENTS card. SYSTEM EVENTS now
  also covers undervoltage/voltage-normalised transitions and
  network-failover watchdog log lines, merged and time-sorted in one place
- Persistent long-term trends: 5min-downsampled temp/cpu/ram/swap history
  in a local sqlite file (30-day retention - negligible size against this
  card's 26GB free), new PERFORMANCE TRENDS (7 DAYS) chart on System tab

Verified with a Flask test-client smoke test (CSRF gate, cache-header
override, input validation, cached vs. live update-check, new endpoint
shapes) - no real Pi available in the dev environment for this batch.

## Phase 8f — Refresh-latency round 2, Proton kill-switch, Telegram fix

**Confirmed on-device:** refresh is now noticeably faster after Phase 8e's
apt-get/Chart.js fixes, but still had room - traced to the SPA pattern
itself: every tab load starts blank ("--" placeholders) until its first
fetch round-trips over Tailscale, even though the server answers those
fetches instantly from cache.

- Overview's data is now embedded directly into the page HTML at render
  time (`window.__INITIAL_OVERVIEW__`, via a shared `_overview_snapshot()`
  helper used by both `/` and `/api/overview`) - the Overview tab paints
  with real numbers on first load with zero API round-trips. Every load
  after the first still does a normal fetch.
- System tab's `/api/sys` fetch no longer waits behind Chart.js + the
  long-trends chart in a serial await chain - it now runs concurrently,
  which matters most the first time System is opened and Chart.js's
  204KB still needs downloading.
- waitress-serve bumped from its default 4 threads to 8
  (`picontrol.service` - needs `sudo cp .../picontrol.service
  /etc/systemd/system/` + `daemon-reload` to actually take effect, unlike
  every other file in this tarball).
- Fixed a related latent gap: CACHE's "live" default dict was missing
  swap_pct/swap_used_mb/swap_total_mb, which could have thrown a
  client-side error in the ~3s window right after service start.

**Proton kill-switch warning:** NET tab now shows an explicit warning if
this Pi is offering itself as a Tailscale exit node while Proton is off -
peer traffic goes direct instead of through the geo-exit in that state,
easy to miss since the two systems don't know about each other.

**telegram-send.sh bug found and fixed:** the script only checked curl's
own exit code, which reflects transport-level success (DNS, connection) -
not whether Telegram actually accepted the message. A bad bot token or
chat ID comes back as a normal HTTP 200 with `"ok":false` in the response
body, so the script always reported success even when nothing was
delivered - a silent-failure class of bug, same family as the Phase 7d
pipefail one, just at the application layer instead of the shell layer.
Fixed to parse the actual API response; same invocation signature, so
nothing else that calls it needs to change.
Brought the script into this repo (`pi-control/telegram-send.sh`, deploys
to `/usr/local/bin/telegram-send.sh`, needs `chmod +x` after copying, same
as the helper script) since the dashboard now depends on its exit code
being meaningful. New TELEGRAM ALERTS card + SEND TEST ALERT button on
Overview, wired to a new fixed-message-only `telegram-test` helper action
(never accepts arbitrary text, so it can't become a message-sending
oracle) - confirmed working end-to-end on the real device.

**Deploy sequence updated** to also copy+chmod `telegram-send.sh` on every
deploy (same pattern as `pi-control-helper.sh`), plus the one-time
`picontrol.service`/`daemon-reload` step above for this batch's thread
change. See "Deploy — pi-control dashboard" below for the standing
command block to use going forward.

## Phase 8g — Dashboard: proactive Telegram alerting on health transitions

Turned five previously passive dashboard cards into an actual monitoring
system - the background sampling threads now fire a Telegram alert the
moment a condition newly becomes true (or clears), never on every poll,
so a flapping issue or an ongoing one doesn't spam:
- Undervoltage / frequency-cap / throttling - onset and recovery, off the
  structured `_power_status()` flags (not text-parsed)
- CPU temp crossing 75°C in either direction (`TEMP_ALERT_C`)
- Disk usage crossing 90% on any mount, either direction (`DISK_ALERT_PCT`)
- New fail2ban bans, batched into one message if several land in the same
  poll window - avoids a message storm during a real credential-stuffing
  attempt
- New OOM kills, pulled from the existing system-events stream but
  explicitly excluding network-failover-tagged lines (already alerted by
  the watchdog itself) and undervoltage lines (already covered by the
  structured power check) - avoids double-alerting the same real event
  through two paths

`ALERT_STATE` is seeded from real current conditions at startup
(`_init_alert_state`), before the sampling loops start - a service
restart never re-alerts for conditions that already existed before the
dashboard came up. New internal-only `telegram-alert` helper action,
never wired to an HTTP route (unlike `telegram-test`), so it can't be
reached with arbitrary user input the way an API action could. TELEGRAM
ALERTS card now shows the last alert that actually fired, separate from
the manual test button's result.

Verified with a deterministic test harness (patches `time.sleep` to break
out after exactly one loop iteration per sampling function, walks through
9 scenarios: no alert on steady state, alert on transition, no duplicate
while ongoing, alert on recovery, batched fail2ban alert, dedup across
polls, OOM-vs-failover-line exclusion, disk threshold, last_alert
exposure) - no real Pi available in the dev environment for this batch,
confirmed correctness of the transition logic itself instead.

**Confirmed on-device, end to end:**
- Fail2ban: deliberately failed several SSH logins from an external
  device, got banned, Telegram alert fired and matched the dashboard's
  FAIL2BAN BANS card
- Disk + temp: temporarily raised both thresholds just above idle
  (`TEMP_ALERT_C`/`DISK_ALERT_PCT`) so a restart seeds baseline as
  "not yet high", then crossed them live with a CPU stress loop
  (`yes > /dev/null &` x4) and a scratch file (`fallocate`) - both fired
  correctly on crossing *and* on recovery, restored to 75.0/90.0
  afterward
- Gotcha hit during testing: setting a threshold *below* current
  live values before restarting seeds `_init_alert_state()` as
  already-high immediately, so no alert fires - by design (matches the
  no-re-alert-on-restart behavior), but easy to trip over when testing;
  the threshold has to start above the live value for a restart to seed
  a clean "not yet high" baseline to cross
- OOM and power/undervoltage intentionally left untested on the real
  device - inducing a real kernel OOM kill risks the kernel picking an
  unintended process to kill on a 425MB device, and there's no safe way
  to force a genuine electrical undervoltage condition on demand. Both
  covered by the same unit-tested transition logic as everything else.

## Deploy — pi-control dashboard (standing reference)

Run every time a new `pi-control.tar.gz` is pulled onto the Pi:
```
sudo systemctl stop picontrol
tar xzf ~/pi-control.tar.gz -C ~/ 2>/dev/null || tar xf ~/pi-control.tar -C ~/
sudo cp -r ~/pi-control/* /opt/pi-control/
sudo cp /opt/pi-control/pi-control-helper.sh /usr/local/bin/
sudo chmod 755 /usr/local/bin/pi-control-helper.sh
sudo cp /opt/pi-control/telegram-send.sh /usr/local/bin/
sudo chmod 755 /usr/local/bin/telegram-send.sh
sudo systemctl start picontrol
```
`picontrol-helper.sh` and `telegram-send.sh` both need the explicit
copy+chmod step every time because `/opt/pi-control/*` isn't where either
actually runs from - only `app.py`/`templates/`/`static/` are read
in-place from `/opt/pi-control`.

One-off extra step, only when a change specifically touches
`picontrol.service` itself (called out explicitly whenever that happens -
last needed for Phase 8f's `--threads=8`):
```
sudo cp /opt/pi-control/picontrol.service /etc/systemd/system/picontrol.service
sudo systemctl daemon-reload
```
