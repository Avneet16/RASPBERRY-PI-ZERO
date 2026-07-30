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

## Phase 8h — Dashboard: service-down, kill-switch, Proton-stale, SD-health alerts

Extended Phase 8g's alerting to four more transitions, same seed-at-
startup / alert-on-change-only pattern as everything already there:
- **Service down/up** - any monitored unit (AdGuard, Unbound, Vaultwarden,
  Tailscale, nginx, UFW, fail2ban) going inactive or recovering, batched
  into one message if several change in the same poll. The single most
  obviously-missing signal before this - previously you'd only find out
  a service crashed by opening the dashboard.
- **Proton kill-switch** - the same condition already shown as a passive
  NET tab warning (Phase 8f) now also fires a Telegram alert, checked in
  `_sample_net` right where both underlying values are already available
- **Proton handshake staleness** - alerts if a nominally-"up" Proton
  connection's last handshake exceeds 5 min (`STALE_HANDSHAKE_S`) -
  `wg-quick` doesn't tear the interface down on peer unreachability, so
  `up` can stay true long after the tunnel actually died
- **SD card health** - extended the existing `system-events` grep
  pattern (already used for OOM/undervoltage/failover) to also catch
  mmcblk/Buffer-I/O/EXT4-fs kernel log lines - no new dependency, SD
  cards don't expose SMART data, and a read-only remount is the classic
  early-failure symptom. Reuses the same fresh-line diffing already
  computed for OOM detection, just a second disjoint filter - no new
  helper action needed.

Verified with the same deterministic test harness as Phase 8g (patches
`time.sleep` to break out after one loop iteration, mocks the underlying
data functions) - onset, no-duplicate-while-ongoing, and recovery for
all four conditions, all passed.

**Kill-switch confirmed on the real device** the same session it shipped -
got the exact expected Telegram alert ("exit node active, Proton is OFF -
peer traffic is going direct") the first time the condition occurred
naturally. Service-down/up, SD-health, and Proton-staleness not yet
confirmed on hardware - service-down/up is easy to trigger deliberately
(`sudo systemctl stop <unit>` on something safe) whenever that's worth
doing; SD-health and Proton-staleness are harder to safely force the same
way the OOM/power gap was in 8g.

## Phase 8i — Dashboard: ALERTS, LOGS, and SECURITY tabs

Three new tabs after TERM, each big enough to deserve its own space
rather than another Overview card.

**ALERTS tab:**
- Alert history - every Telegram alert ever sent is now persisted (same
  trends.db sqlite file, new `alert_history` table, capped at the 200
  most recent rows) and browsable, not just the single most-recent one
  on Overview's TELEGRAM ALERTS card
- Live-tunable thresholds - `TEMP_ALERT_C`/`DISK_ALERT_PCT`/
  `STALE_HANDSHAKE_S` are no longer hardcoded constants. They're now
  `ALERT_THRESHOLDS`, loaded from a persisted `settings` table at startup
  (same db file again), editable straight from the dashboard, effective
  on the very next sampling poll - no code edit or redeploy required.
  Directly closes the friction hit testing Phase 8g, where tuning a
  threshold meant hand-editing `app.py` and restarting twice (once to
  set a test value, once to revert)

**LOGS tab:** general-purpose log viewer for any monitored service (or
pi-control itself) - pick a unit + line count, tap load. New whitelisted
`service-logs` helper action, explicit case-match on allowed units, no
arbitrary `journalctl -u` access.

**SECURITY tab:**
- Fail2ban summary - total/currently failed and banned counts per jail,
  complementing (not duplicating) the currently-banned IP list already
  on Overview
- UFW rule list (`ufw status verbose`) - previously only active/inactive
  was visible, not what's actually allowed through
- SSH auth log, last 24h (accepted/failed/invalid-user) - relevant given
  SSH is still password-auth by deliberate choice (Phase 1)

**Frontend note:** added an `AUTO_POLL_TABS` list so the 5s auto-refresh
only applies to the original 4 tabs (over/sys/net/disk) - alerts/logs/
security load once per tab switch instead, so a log view's scroll
position or an in-progress threshold edit never gets silently reset by a
background poll. Tabbar also switched from equal 1/8-width flex shares
(illegible at 8 tabs on a phone) to horizontal scroll with a sane
per-button minimum width.

Verified with a Flask test-client suite covering all new endpoints
(threshold get/set with bounds validation and sqlite persistence, alert
history round-trip, logs unit whitelist + line clamping, security
endpoint shape) plus a full regression pass confirming the
constant-to-dict refactor didn't break the existing alert transition
logic. Not yet exercised on the real device.

## Phase 8j — Fixed real-device issues from 8i, UI polish

Two bugs found immediately on first real use of the new tabs:

- **"-- No entries --" showing up as fake events.** journalctl prints
  this exact literal line when a `-g` grep query matches nothing (older
  systemd versions) - SYSTEM EVENTS and the SECURITY tab's SSH auth log
  were both passing this straight through as if it were a real event.
  Filtered out at the source (`grep -v -- '^-- No entries --$'`) in both
  `system-events` and `ssh-auth-log`.
- **LOGS tab empty for AdGuard/Unbound/nginx/UFW/Fail2Ban** (only
  Tailscale and pi-control itself showed anything). Root cause: most of
  these services don't route their real logs through the systemd journal
  at all - nginx and fail2ban write their own log files, AdGuard Home has
  its own log file, unbound can log under a bare `unbound` syslog tag
  rather than its unit name depending on how it forks. `journalctl -u X`
  only ever sees stdout/stderr, so there was frequently nothing there to
  find - not a dashboard bug so much as a wrong assumption about where
  each service actually logs. `service-logs` now falls back in order:
  `journalctl -u unit` -> (unbound only) `journalctl -t unbound` -> known
  log file per service (`AdGuardHome.log`, `nginx/error.log`,
  `fail2ban.log`) -> a clear "nothing found" message instead of a silent
  blank box. Verified the full fallback chain with a mocked-journalctl
  test harness (5 scenarios: journalctl empty->file fallback, journalctl
  empty->syslog-tag also empty->generic message, real journalctl data,
  no-fallback-defined unit, rejected unit).
- Vaultwarden logs are expected to stay empty until it's actually
  installed - not a bug, per the standing note from earlier in this doc.

**UI polish**, all requested together after the first real look at the
new tabs:
- Proton kill-switch warning now renders as a bordered `alert-banner`
  (background tint + border), hidden via `display:none` when inactive
  instead of leaving an empty line - previously it was plain colored text
  that blended into the card
- ALERTS tab's SAVE button now uses the same `toast()` notification every
  other action in the app already uses, instead of a one-off inline
  status line (removed the now-dead element)
- Added manual REFRESH buttons to ALERT HISTORY and the SECURITY tab's
  fail2ban summary, since neither auto-polls by design - previously the
  only way to refresh was switching tabs away and back

## Deploy — pi-control dashboard (standing reference)

**Preferred: one command.** `pi-control/deploy-pi-control.sh` now ships
inside the tarball and runs the whole sequence below in order:
```
bash ~/pi-control/deploy-pi-control.sh
```
It re-extracts `~/pi-control.tar.gz` (or `.tar`) itself, so nothing needs
to be extracted by hand first - just make sure the fresh archive is at
`~/pi-control.tar.gz` before running it. Since the script itself lives
inside the tarball, the very first time it's used requires one manual
extraction to bootstrap it onto the Pi (see the manual sequence below);
every deploy after that just re-runs the same script path, which carries
itself forward automatically with each new tarball drop.

**Manual sequence** (what the script above actually runs - useful as a
fallback for debugging, or for the one-time bootstrap extraction):
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
last needed for Phase 8f's `--threads=8`; not handled by the deploy
script, since it's rare enough not to be worth the daemon-reload
complexity in an otherwise-idempotent script):
```
sudo cp /opt/pi-control/picontrol.service /etc/systemd/system/picontrol.service
sudo systemctl daemon-reload
```

## Phase 8k — Dashboard: de-duplicate Overview, peer IPs, egress fix, real log paths

**Consolidated cards that had ended up duplicated across tabs** once
ALERTS/LOGS/SECURITY existed:
- TELEGRAM ALERTS removed from Overview - SEND TEST ALERT moved to a new
  TELEGRAM TEST card on the ALERTS tab (which already has history +
  thresholds). `testTelegram()` now uses `toast()` like every other
  action instead of a dedicated status element.
- FAIL2BAN BANS removed from Overview - the banned-IP list + unban button
  moved to the SECURITY tab, next to the jail summary already there.
  `/api/security` now also returns `fail2ban_banned` (reused from the
  existing CACHE value, no new subprocess call). `_overview_snapshot()`
  no longer carries either field, since nothing reads them from Overview
  anymore.

**Tailscale peers** now show their Tailscale IP next to hostname
(`TailscaleIPs[0]` from `tailscale status --json`).

**Egress check fixed** - it never worked. Root cause: `--interface
tailscale0` uses SO_BINDTODEVICE for curl's *entire* request, including
its own DNS lookup - but that lookup goes to 127.0.0.1:53 (AdGuard),
unreachable once the socket is bound to tailscale0 instead of loopback,
so the request died before ever reaching the HTTPS part. Fixed by
resolving via the normal unrestricted resolver first (`getent ahostsv4`),
then handing curl the IP directly via `--resolve` so it never does its
own DNS lookup over the bound interface. Also forces `-4` on both the
direct and exit-node checks - Proton is IPv4-only, so an IPv6 "direct"
result was never a meaningful comparison. Caught a second real bug while
*testing* this fix: the `getent` pipeline needed `|| true` (same pipefail
class as the Phase 7d lesson) - without it, `set -e` killed the whole
action silently whenever DNS resolution failed, never reaching the
fallback message. Found by tracing with `bash -x` against a mocked
`getent`/`curl`, not by inspection alone.

**LOGS tab fallback paths corrected against the real Pi**, replacing
guesses from Phase 8j with confirmed values (systemctl cat, config
greps, file listings, journalctl per service):
- AdGuardHome: unit sets `StandardOutput`/`StandardError=journal`
  explicitly, no separate log file exists - the wrong file-fallback
  guess was removed; an empty journal here is genuinely quiet operation,
  not a wrong-path bug
- unbound: no `logfile:`/`use-syslog:` configured, so default verbosity
  only logs startup/errors - quiet, error-free operation is expected to
  produce nothing; message now explains this and how to opt into a log
  file if wanted
- nginx: confirmed real files at `/var/log/nginx/{error,access}.log`,
  but `error.log` was 0 bytes (healthy = no errors) while `access.log`
  had real traffic - fallback now tries multiple files in priority order
  instead of reporting "nothing found" when the service is demonstrably
  working
- ufw: confirmed purely the oneshot rule loader as already documented -
  added a kernel-log fallback (`journalctl -k -g '\[UFW'`) since real
  firewall activity, if `ufw logging on` is enabled, lands there instead
  of on the unit itself
- fail2ban: confirmed `/var/log/fail2ban.log` is the real logtarget with
  real content - original fallback was already correct, no change needed

No deploy changes for this batch - standard `bash
~/pi-control/deploy-pi-control.sh`.

## Phase 8l — Normal-update button, Proton data-usage fix, egress timeout race

**Added a plain "UPDATE" button** next to CHECK/FULL UPGRADE on the
Overview SYSTEM UPDATES card - runs `apt-get upgrade` (conservative: never
removes an installed package or pulls in a new one to satisfy a
dependency change, no autoremove either) as distinct from the existing
FULL UPGRADE (`apt-get full-upgrade` + `autoremove`). Backend threads the
choice through as a `kind` ("normal"/"full") parameter: new
`normal-upgrade` helper action, `_run_upgrade(kind)`, `UPDATE_STATE`
gains a `kind` field so `/api/updates/status` can report which one is
running, `/api/updates/upgrade` validates `kind` and 400s on anything
else. Both buttons now disable/re-enable together in `app.js` via a
shared `upgradeButtons()` helper instead of the old single `upgrade-btn`
id.

**DATA USAGE card fixed** - it showed "proton: not active" even with
Proton genuinely connected (confirmed active on the PROTON VPN card,
same tab, same load). Root cause: `_net_iface_stats()` was matching a
hardcoded literal `"proton"` interface name, but per the Phase 7d
multi-server design `wg-quick` names the interface after the config file
basename (`proton-us.conf` -> interface `proton-us`), so a literal
`"proton"` never exists in `psutil.net_io_counters()` regardless of
connection state. Fixed by searching the counters for any key starting
with `"proton-"` and reporting whichever one is actually up; falls back
to the literal `"proton"`/`None` stats if no server is connected, same
as before.

**Egress check still showing "failed" even after the Phase 8k DNS fix**
- root cause this time was a timeout race, not the DNS bug from before.
The helper's internal `curl --max-time 8` had zero buffer against the
Python-side `subprocess.run(..., timeout=8)` wrapping it, and `getent`
itself has no timeout of its own - so on a slow real-world DNS lookup or
handshake over the bound `tailscale0` interface, the *outer* Python
timeout could fire first and raise `TimeoutExpired`, discarding whatever
diagnostic string the helper would otherwise have echoed. Fixed on both
sides: helper now bounds `getent` with `timeout 5` and adds `-S` (with
stderr merged via `2>&1`) so curl's own error text comes through instead
of just the generic fallback marker; Python side bumps the outer
timeout to 15s (5s getent + 8s curl = 13s worst case, now with real
headroom) and captures stderr as well as stdout so a real curl error
message surfaces in the UI instead of collapsing to "failed".

**7-day performance trends still empty** - two candidate root causes were
identified but not yet confirmed; see Phase 8m below for the resolution.

No `picontrol.service` changes this batch - standard `bash
~/pi-control/deploy-pi-control.sh` is enough.

## Phase 8m — Real root cause of empty 7-day trends: /opt/pi-control ownership

Diagnostic from Phase 8l came back: `trends.db` doesn't exist at all,
despite `picontrol` having been up for hours (`ActiveEnterTimestamp` many
hours in the past) - which rules out "not enough continuous uptime yet"
outright. That pointed straight at the other candidate: a writability
problem.

Root cause, confirmed by reading `deploy-pi-control.sh` alongside
`picontrol.service`: the deploy script installs app files with `sudo cp
-r "$HOME/pi-control/"* /opt/pi-control/`, which leaves everything under
`/opt/pi-control` **owned by root** (plain `cp` doesn't preserve the
source owner, and it's invoked via `sudo`). But `picontrol.service` runs
as `User=anon` - an unprivileged user that can read root-owned files
fine (which is exactly why the rest of the app has always worked: it
only ever reads `app.py`/`templates/`/`static/`), but can never *create a
new file* inside a directory it doesn't own. Every call to
`_init_trends_db()`, on every single startup since this feature was
introduced, has been hitting a permission-denied opening `trends.db` for
the first time, caught by the deliberate crash-avoidance `except
Exception: TRENDS_DB_AVAILABLE = False` guard - so the failure was
silent by design, never crashed anything, and never left an error
anywhere obvious to spot.

This is bigger than just the 7-day chart: `_load_setting`/`_save_setting`
(ALERT_THRESHOLDS persistence across restarts) and `_record_alert_history`
(the ALERTS tab's alert history) both depend on the exact same file being
writable, so both have also been silently no-ops this whole time -
threshold edits on the ALERTS tab were taking effect live but reverting
to defaults on every restart, and alert history was never actually being
recorded to survive a restart either.

**Fix**: `deploy-pi-control.sh` now runs `sudo chown -R anon:anon
/opt/pi-control` immediately after the `cp -r` step, every deploy - fully
self-healing on the very next redeploy, no manual one-off command needed
on the device itself.

No `picontrol.service` changes - standard `bash
~/pi-control/deploy-pi-control.sh` picks up the fix automatically.

## Phase 8n — False-positive DOWN/stale Telegram alerts (UFW, Tailscale, Proton)

Real device Telegram logs showed repeated, isolated flapping - "service(s)
DOWN: UFW" immediately followed by "service(s) back UP: UFW" (four times),
one isolated Tailscale DOWN/UP pair, and Proton handshake "stale" (301s,
then 302s - right at the 300s default threshold) immediately followed by
"fresh again", each time with nothing else changing at the same moment.

Root cause: `_service_statuses()` and `_proton_status()` each spawn one or
more `sudo` subprocesses (systemctl, the ufw-status helper action, wg) on
every ~3s tick of `_sample_fast()`. On a Pi Zero 2W, any transient hiccup
(disk I/O from the trends write every 5 min, journald contention, a slow
sudo lookup) can occasionally push one of those calls past its 5s timeout;
the code already handles that failure safely (defaults to "down"/falls
back), but the *alerting* logic was treating a single bad poll exactly the
same as a real state change - so one missed poll was enough to fire a
false DOWN alert, followed by a false recovery UP alert 3s later once the
next poll came back clean. This is a different bug from the two fixed
earlier in this same investigation (Phase 8l/8m) - those were "the data
is missing/wrong"; this one is "occasionally slow to read, and reading
'down' once was wrongly treated as certain."

**Fix**: added a small consecutive-confirmation counter
(`ALERT_CONFIRM_POLLS = 2`, `_ALERT_PENDING`) for both the per-service
down/up check and the Proton stale-handshake check - a reading now has to
show up on 2 consecutive ~3s polls before it's treated as a real
transition and alerted on. A single-poll blip that self-corrects on the
very next poll (exactly what the logs showed) no longer alerts at all;
this adds at most ~3-6s of detection latency for a genuine outage, which
is immaterial for these checks. The live status shown on the dashboard
itself (Overview's ACTIVE SERVICES, NET's PROTON VPN card) is unaffected -
only the Telegram alert *transition* detection is debounced, so the UI
still always reflects the latest real poll.

Verified with a mocked `_service_statuses()`/`_proton_status()`/`time.sleep`
harness feeding `_sample_fast()` a scripted sequence: a single-poll UFW
blip (down then immediately back up) fires zero alerts, while a
genuinely-sustained down (confirmed on 2 consecutive polls) fires exactly
one DOWN alert.

If Proton handshake-stale alerts keep recurring right around 300-310s
even with the debounce, that's likely genuine boundary-hovering (idle
traffic gaps naturally push the handshake age close to the default
300s threshold) rather than a bug - `stale_handshake_s` is already
live-tunable from the ALERTS tab without a redeploy, so bumping it up a
bit (e.g. to 450-600s) is the right knob to reach for if it's still
noisy after this fix.

No `picontrol.service` changes - standard `bash
~/pi-control/deploy-pi-control.sh` is enough.

## Phase 8o — 7-day trends confirmed fixed; UPDATE/FULL UPGRADE silently doing nothing

The Phase 8m ownership fix confirmed working: 7-day trends now populate
on the real device.

**UPDATE/FULL UPGRADE reported as "not working"** - diagnosed by hitting
the endpoint directly with curl (bypassing the browser/JS/nginx
entirely), using a token freshly scraped from the current page:
```
CSRF=$(curl -s http://127.0.0.1:8088/ | grep -oP 'data-csrf-token="\K[^"]+')
curl -sv -X POST http://127.0.0.1:8088/api/updates/upgrade \
  -H "Content-Type: application/json" -H "X-CSRF-Token: $CSRF" -d '{"kind":"normal"}'
```
This came back `200 {"ok":true,"started":true,"kind":"normal"}`, and
`journalctl -u picontrol` showed the `normal-upgrade` helper action
actually ran to completion (~19s) with no errors - so the backend has
been fine the whole time.

Root cause was in the browser, not the server: `CSRF_TOKEN =
secrets.token_hex(32)` (`app.py`) is generated once per process start
and baked into the page HTML at load time. Any dashboard tab left open
from *before* the most recent `picontrol` restart (extremely likely
after this session's string of redeploys) is holding a stale token that
no longer matches the current process - every POST from that tab 403s
silently. This wasn't unique to the new UPDATE button; it would affect
any POST action from a stale tab, but UPDATE/FULL UPGRADE happened to be
the first POST tried since the last restart, so it's what surfaced it.

That alone explains "not working," but reading `runUpgrade()` in
`app.js` turned up a real, independent bug worth fixing regardless of
root cause: it only special-cased HTTP 409 ("already running") - any
*other* non-2xx response (403 from a stale CSRF token, 400, 500, ...)
fell through silently into `pollUpgradeStatus()`, which would just see
`running: false` / `done_ok: null` (nothing ever actually started) and
quietly re-enable the buttons with zero feedback. From the user's
perspective that's indistinguishable from "button does nothing" -
exactly what got reported. Fixed: a non-OK response now surfaces the
server's actual `error` message (or the raw HTTP status if the body
isn't JSON) in an error toast instead of failing silently.

**Action for you**: hard-refresh (force-reload) the dashboard tab before
retrying UPDATE - that alone should resolve it even before redeploying
this fix, since it's a client-side stale-token issue. After redeploying
this batch, any future CSRF failure will show up as a clear toast
instead of silent nothing.

No `picontrol.service` changes - standard `bash
~/pi-control/deploy-pi-control.sh` is enough.

## Phase 8p — Replace nginx Basic Auth with a real login page (Bitwarden autofill)

**Motivation**: nginx's `auth_basic` triggers Chrome's *native* HTTP-auth
dialog, which lives outside the page DOM - Bitwarden's extension (and
most extension-based password managers) can't autofill it, since content
scripts only see the actual page. Clicking the Bitwarden icon shifts
focus away from the dialog, and Chrome frequently cancels/resends the
request with no credentials, surfacing as "Auth required" for no visible
reason.

**Investigated the actual routing first**, since guessing wrong here
risks something much worse than an annoying dialog: `sudo cat
/etc/nginx/sites-available/pi-control` showed `auth_basic` set once at
the *server* level, covering two completely different upstreams -
`location /` (the Flask dashboard, :8088) and `location /terminal/` (a
separate `ttyd` process, :7681, giving full shell access). The terminal
was only ever protected because it happened to share the same
`auth_basic` block - a plain per-app session cookie checked only inside
the Flask app would have left `/terminal/` (full shell, same as SSH)
completely unauthenticated the moment `auth_basic` was removed, since
ttyd is a different process nginx proxies to directly and never routes
through Flask at all.

**Fix uses nginx's `auth_request` module** (confirmed present via `nginx
-V 2>&1 | grep auth_request` before committing to this design) so *both*
upstreams get gated by the same check, in one place, instead of only the
one Flask actually fronts:

Flask side (`app.py`):
- `_load_setting()` generalized to take an optional `cast` (was
  hardcoded to `float` for the alert-threshold use case) so it can also
  load/store strings - reused for the session secret key and the
  username/password hash, in the same `settings` table Phase 8m already
  fixed the write-permission for.
- `app.secret_key` is generated once and persisted (same pattern as
  `flask_secret_key` in `settings`) - regenerating it every restart
  like `CSRF_TOKEN` would silently log everyone out on every redeploy,
  making "remember me for 30 days" a lie given how often this app gets
  redeployed during development.
- `/login` (GET/POST): first-run experience *is* the setup form - if no
  password is configured yet, it asks you to create a username/password
  (min 8 chars, confirm match) instead of showing a login form; once
  set, later visits show a normal login. No env vars or manual hash
  generation needed to bootstrap it. Password hashed with
  `werkzeug.security.generate_password_hash` (already a Flask
  dependency, PBKDF2-SHA256 - no new pip package).
  `login.html` uses plain `<form>` fields with standard
  `autocomplete="username"/"current-password"/"new-password"` attributes
  specifically so Bitwarden (or any password manager) recognizes and
  autofills it like any other website login - the entire point of this
  change.
- `/logout` clears the session.
- `/api/auth/check`: the endpoint nginx's `auth_request` calls
  internally on every request to check the session cookie - 200 if
  logged in, 401 (via the existing generic `/api/` 401 handling in
  `_require_login`) if not. Not reachable directly from outside once
  nginx marks it `internal`.
- New `_require_login` `before_request` hook (runs before the existing
  CSRF hook) exempts only `/login` and `/static/*` (so the login page
  and its CSS can load while logged out); everything else redirects to
  `/login` (or 401s for `/api/*`) without a valid session.
- Session cookie: `SESSION_COOKIE_SECURE=True` (nginx already terminates
  TLS - confirmed via `listen 9448 ssl;` in the site config),
  `HTTPONLY=True`, `SAMESITE=Lax`, 30-day lifetime per your answer.

Verified with a Flask test-client script driving the full flow: logged
out redirects to `/login` (401 for `/api/*`, `/login` and `/static/`
stay reachable); first POST to `/login` with no password configured
creates the account, logs in, and persists across a simulated restart;
wrong password rejected with a message; correct password logs in;
`/logout` clears the session and protected routes redirect again;
`/api/auth/check` returns exactly the 401/200 nginx's `auth_request`
needs to see in each state.

**nginx side (NOT yet applied - deliberately last)**: replace
`auth_basic`/`auth_basic_user_file` with `auth_request /api/auth/check`
on *both* `location /` and `location /terminal/`, an internal probe
location proxying to the Flask app with `proxy_method GET` (an
`auth_request` subrequest reuses the original request's method by
default, and Flask's `/api/auth/check` route only accepts GET - without
forcing GET here, every POST /api/action/* would 405 the probe and nginx
would 500 instead of proxying through), and a named `@login_redirect`
location (`return 302 /login`) wired via `error_page 401 = @login_redirect`
so an unauthenticated visitor lands on the new login page instead of
seeing a raw 401. `location = /login` and `location /static/` are
carved out with no `auth_request` (avoids a redirect loop and lets the
login page's own CSS load). AdGuard's separate `auth_basic` in
`adguard.conf` is untouched - it has no login page of its own, so Basic
Auth stays there by design.

**Deploy order, deliberately staged so there's no lockout window and no
gap with zero auth**:
1. Redeploy this batch (`bash ~/pi-control/deploy-pi-control.sh`) -
   nginx's `auth_basic` stays in place throughout, so access doesn't
   change yet.
2. Visit the dashboard - you'll hit the existing Basic Auth prompt first
   (unchanged), then land on the new `/login` page underneath it in
   setup mode. Create the account, confirm you land on the dashboard,
   confirm `/logout` and logging back in both work.
3. Only once that's confirmed working: swap
   `/etc/nginx/sites-available/pi-control` to the `auth_request` version,
   `sudo nginx -t`, then `sudo systemctl reload nginx` (reload, not
   restart - keeps `adguard.conf`/`default` sites up the whole time).
   From that point on, Basic Auth is gone for pi-control and Bitwarden
   autofill works normally on `/login`.
