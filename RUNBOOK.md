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

**Confirmed live on the device**: all three steps completed - account
created via the setup form, `/login`/`/logout` round-tripped correctly,
`auth_basic`/`auth_basic_user_file` swapped for the `auth_request`
config above (old file kept as
`/etc/nginx/sites-available/pi-control.bak-<timestamp>` in case a
rollback is ever needed), `nginx -t` passed, reloaded clean. Both the
dashboard and the TERMINAL tab now go straight to `/login` with no
Basic Auth prompt, a single session covers both, and Bitwarden autofills
the login form correctly - the original problem this phase set out to
fix. AdGuard's own site/`auth_basic` was untouched throughout, as
intended.

## Phase 8q — Security review triage: fixed a real stored XSS + 7 others, rejected 7

A large external security-review dump (15 numbered findings plus a
separate "critical" list, evidently AI/tool-generated) came in. Rather
than implementing it wholesale, each finding was checked against the
actual code before acting - several turned out to be flatly wrong about
what's in this codebase, and a couple of the suggested "fixes" would
have actively broken something or reversed an explicit earlier decision.

**Fixed (confirmed real):**

1. **Stored XSS via WiFi SSID (was rated CRITICAL, and rightly so)** -
   `app.js`'s NET tab rendered `n.ssid`/`n.security` from a live `nmcli`
   scan straight into `innerHTML` with zero escaping. Any device
   broadcasting an SSID like `<img src=x onerror=fetch(...)>` within
   WiFi range would get that executed in the dashboard's own page
   context on the next poll - and since the CSRF token sits in
   `document.body.dataset.csrfToken`, script running via this XSS could
   read it directly and fire any authenticated POST (reboot, upgrade,
   disconnect Proton, ...) with zero further access needed. This is a
   physical-proximity attack requiring no credentials at all. Fixed by
   switching to `textContent`/`createTextNode` (saved-networks select
   and available-networks list). Added a small `esc()` helper and
   applied it to three other spots carrying real external/attacker-
   influenced text that had the same unescaped-`innerHTML` pattern:
   system-events lines (journalctl, which can embed content from
   network-failover events), Tailscale peer hostnames (settable by
   anyone else on the same tailnet), SSH auth log lines (a failed SSH
   login attempt can embed an arbitrary "attempted username" string in
   the log, reachable with no auth at all over the network), and alert
   history messages. Left fail2ban jail names/IPs and process names
   (top CPU/mem) un-escaped - lower-value targets (fixed config names,
   dotted-quad IPs, or already requires local code execution to abuse)
   where the diff cost didn't seem worth it given time spent elsewhere
   in this batch.
2. **WiFi active-scan radio contention** - `_sample_net()` ran a full
   active `nmcli device wifi list` scan every 8s, which takes the radio
   off-channel and can briefly interrupt current wlan0 traffic (client
   mode or this Pi's own AP-failover mode). Now only forces a real scan
   (`--rescan yes`) once every ~64s (every 8th poll); the other 7 polls
   read NetworkManager's existing cached results via `--rescan no`. The
   cheap per-poll checks (device status, saved networks, Tailscale peers)
   stay on the fast 8s cycle since they don't touch the radio.
3. **CSRF token was a single process-wide constant** - regenerated once
   at process start and shared by every visitor. Now generated per
   session (`session["csrf_token"]`, lazily created on first render of
   `/`) now that real sessions exist from Phase 8p - a leak of one
   session's token no longer matters to any other session, and a
   redeploy/restart no longer invalidates every open tab's token
   independent of whether their login session itself is still valid.
4. **No rate limiting on `/login`** - added a small in-memory sliding-
   window limiter (5 failed attempts / 5 minutes, keyed by
   `X-Real-IP` since nginx sets that header) instead of pulling in the
   `flask-limiter` package - a plain dict + lock matches this app's
   existing lightweight style and avoids a new dependency on a Pi Zero.
   Only guards the real login path; setup_mode isn't brute-forceable
   (no password exists yet to guess).
5. **`session.clear()` before setting the new session on login** - cheap
   defensive hygiene, added to both the setup and normal-login success
   paths.
6. **Aggressive `clean-logs`** - swapped the hard `--vacuum-size=20M` +
   unconditional delete-every-rotated-log for `--vacuum-time=7d` +
   `-mtime +30` on the `find` deletes. This Pi has 26GB free; there's no
   real pressure to keep the journal that small, and destroying all
   forensic history on every click of CLEAN LOGS was a bad trade for the
   disk space it saved.
7. **Silent background-thread failures** - added `logging.basicConfig`
   (journald already captures stderr from this systemd-run process, so
   this needed zero new infrastructure) and replaced every bare
   `except Exception: pass` in the six `_sample_*`/`_sample_loop`
   background threads with `logger.exception(...)`. Also found and fixed
   a real related gap while doing this: `_sample_slow` and `_sample_disk`
   had **no outer catch-all at all** - an exception anywhere outside
   their inner per-item try/excepts (e.g. inside the `CACHE_LOCK` block)
   would have silently killed that daemon thread forever, with nothing
   but a bare stderr traceback and no periodic retry - worse than the
   "silent but at least still running" issue the review actually
   flagged. Both now wrap the whole loop body and log-and-continue like
   the other four sampling loops.
8. **No confirmation before FULL UPGRADE** - added a `confirm()` dialog,
   matching the existing pattern already used for REBOOT PI. Does NOT
   touch the conservative UPDATE button.

**Explicitly rejected or downgraded, with reasoning:**

- **"Remove FULL UPGRADE / require SSH for upgrades"** - directly
  contradicts what was explicitly asked for and built in Phase 8l
  (adding easier access to upgrades, not less). A confirmation dialog
  (above) is the right-sized safety improvement here, not removing the
  feature.
- **"CACHE reads aren't lock-protected in `_overview_snapshot()`"** -
  checked every `CACHE[...]` read site in the file; all of them,
  including `_overview_snapshot()`, already sit inside `with
  CACHE_LOCK:`. This finding doesn't match the actual code - no change
  made.
- **"UPDATE_LOCK stays held for up to 30 minutes if the browser closes"**
  - checked `_run_upgrade()`: the lock is only held for the instant
    state-flip at the start and end, never across the `subprocess.run(...)`
  call itself, and the upgrade runs in a fully detached background
  thread that doesn't care whether a browser is still connected. The
  suggested fix code (`with UPDATE_LOCK(timeout=30):`) also isn't valid
  Python - `threading.Lock` doesn't support that call signature. No
  change made.
- **"Hide command stderr from the user, return generic errors instead"**
  - wrong context fit: this is a single-owner admin tool where the
    "user" and the person debugging their own Pi are the same person.
  Multiple earlier phases in this exact runbook (8n, 8o) deliberately
  added *more* raw stderr surfacing to fix real debugging pain -
  reversing that for a generic-audit "don't leak internals" rule would
  directly undo those fixes for no real benefit here.
- **"Add an HTTPS-enforcing `before_request` redirect"** - as written,
  this would have created an infinite redirect loop: nginx doesn't
  currently forward `X-Forwarded-Proto` to the Flask app, and the
  suggested check depends on that header being present. Flask is only
  ever reachable from nginx over loopback, which itself only accepts
  Tailscale/loopback traffic per the `allow`/`deny` rules already in
  place - the described bypass scenario needs nginx itself to already
  be misconfigured, at which point a lot more than this would be
  broken. Skipped as unneeded complexity for the actual risk.
- **iframe `sandbox` attribute on the terminal** - added the harmless
  part (`referrerpolicy="no-referrer"`) but deliberately skipped
  `sandbox="allow-scripts allow-same-origin"`. This iframe is same-
  origin, fully-trusted content this app itself serves (ttyd) - sandbox
  protects a parent page from an *untrusted* child, which doesn't apply
  here, and there was real, untestable risk of silently breaking the
  embedded terminal (a feature relied on for actual system
  administration) for a benefit that's mostly theoretical in this
  specific same-origin case.
- **"Session fixation - regenerate the session ID after login"** -
  category mismatch: Flask's default sessions are signed client-side
  cookies (itsdangerous), not an opaque server-side session ID an
  attacker could pre-fixate the way classic session-fixation attacks
  work. `session.clear()` (above) is added as cheap hygiene regardless,
  but the severity as originally framed doesn't apply to this
  architecture.
- **Chart.js load size "DoS"** - mischaracterized as a vulnerability;
  it's normal script-loading latency, not something an attacker can
  trigger, and it's already mitigated for repeat visits by the existing
  1-year cache header on `/static/vendor/*` from an earlier phase.

Verified with a Flask test-client script: per-session CSRF tokens
differ between two independent sessions and both correctly reject a
wrong token; the login rate limiter blocks the 6th attempt within the
window and is scoped per-`X-Real-IP` (a different IP is unaffected); the
WiFi rescan cadence alternates `yes` on polls 1 and 9 and `no` on the 7
in between, exactly as designed. All `app.py`/`app.js`/`pi-control-
helper.sh`/`index.html` changes diff-reviewed clean against the last
committed tarball.

No `picontrol.service` changes - standard `bash
~/pi-control/deploy-pi-control.sh` is enough. After redeploying, note
that everyone's existing CSRF token becomes invalid at once (it's now
per-session and freshly generated on the first page load after this
change) - a normal page load handles this transparently, nothing to do
manually.

## Phase 8r — BentoPDF/OmniTools static self-hosting, RAM investigation, enable/disable toggle

**Deployed BentoPDF** as pure static files, no Docker: confirmed this Pi
runs 32-bit Raspberry Pi OS (`armv7l` via `uname -a`), and neither
BentoPDF nor OmniTools publish an `armv7` Docker image (only `amd64`/
`arm64`) - Docker was a dead end here regardless of RAM concerns.
BentoPDF's `STATIC-HOSTING.md` documents an official pre-built
`dist-{version}.zip` on GitHub Releases, so no build tooling is needed
at all - `curl` the zip straight from the Pi, unzip to `/opt/bentopdf`.
Stripped `*.br` files (need an nginx module this Pi's stock nginx
doesn't have) and 20 non-English locale directories (pure page
duplication, ~85MB, not needed for a single-user personal deployment) -
162M final footprint. New nginx site on port 9449 (`sites-available/
bentopdf`), same TLS cert and Tailscale/loopback `allow`/`deny` rules as
the other sites, no `auth_basic` (matches the reasoning from Phase 8p -
personal utility tool, no persistent server-side data since processing
is 100% client-side, network layer already gates access).

**OmniTools deployment stalled on tooling, not architecture** - this one
needs an actual build (TypeScript + Vite, no pre-built static release
exists). Hit three separate walls before landing on the right approach:
1. My own sandbox's network policy blocks `registry.npmmirror.com` and
   `cdn.sheetjs.com` (transitive dependencies resolve through these,
   confirmed via direct `curl` - genuine CONNECT-tunnel 403s, not an
   npm config issue) - dead end for building it there.
2. Termux (Android) failed differently: `@swc/core`'s prebuilt native
   binding can't load under Android's Bionic libc (a known Termux
   limitation for native Node addons, unrelated to phone specs).
3. The repo's own CI (`.github/workflows/ci.yml`) doesn't upload a
   static `dist` artifact anywhere downloadable - it deploys straight to
   Netlify/Docker, no plain zip to grab.

Resolution: build on any normal Linux/Mac/Windows machine (a laptop -
static output doesn't care what built it, only that the machine has a
working native-addon-capable Node, which the Pi's own weak hardware and
architecture wouldn't handle for a build this heavy anyway even
setting the tooling issues aside). Pending - user has a laptop but
needs time to get to it. Once built, needs an SPA-style nginx site
(`try_files ... /index.html`) since OmniTools is a single-page app with
client-side routing, unlike BentoPDF's separate-static-page-per-tool
build.

**RAM investigation - thorough, and the conclusion is "nothing is
wrong"**: user suspected BentoPDF was consuming significant RAM after
seeing overall usage climb. Traced through several layers before
concluding correctly:
- `ps aux --sort=-%mem` showed nginx workers at 2-5Mi RSS each - static
  file serving via `sendfile` costs nginx nothing regardless of the
  size of files on disk, ruling out BentoPDF as a resident-memory
  consumer immediately.
- Per-process `VmSwap` breakdown (`/proc/$pid/status`) identified
  AdGuardHome as the largest single contributor to swap (~60Mi logical)
  - matches its own startup log exactly (`dnsproxy: cache enabled
    size=41943040`, a ~40MB DNS response cache) - expected caching
  behavior for a DNS server, not a leak. pi-control itself: ~15Mi,
  unremarkable for a Flask app.
- Confirmed swap is zram-backed (`swapon --show` -> `/dev/zram0`), and
  `zram-stats` showed a real ~1.55-1.61x compression ratio - the ~260Mi
  "in swap" figure everyone was staring at only costs ~165-190Mi of
  *actual* RAM once compression is accounted for, and that compressed
  cost is already what `free -h`'s "used" reflects.
- Zero OOM-kills ever, confirmed via `journalctl -k | grep -i oom`.
- **Empirically proved the null hypothesis**: disabled BentoPDF's nginx
  site entirely and took `free -h` immediately before/after. Memory
  usage went *up* slightly (342Mi -> 359Mi used), not down - conclusive,
  measured evidence that stopping BentoPDF returns zero RAM, because
  there was never a dedicated process consuming it to begin with. The
  small uptick is just the normal worker-respawn cost of any nginx
  reload, unrelated to which site was removed.

Declined to add a low-`MemAvailable` alert - user was satisfied with the
above and didn't want additional monitoring for this.

**Added an enable/disable toggle for self-hosted static apps** anyway -
not for RAM (proven above to be pointless for that), but for reducing
exposed attack surface: one less live listener when a tool isn't in
active use. New generic mechanism, not hardcoded to just BentoPDF, so
adding OmniTools later is just one more entry in two lists:
- `pi-control-helper.sh`: `ALLOWED_NGINX_SITES` whitelist (mirrors the
  `ALLOWED_SERVICES` pattern already used for `restart`) plus
  `site-status`/`site-enable`/`site-disable` actions - `ln -sf`/`rm` on
  the `sites-enabled` symlink, `nginx -t` before `systemctl reload`.
- `app.py`: `NGINX_SITES` list (name/label/port), `/api/sites/status`
  (polled every 5s alongside the rest of the NET tab - cheap, it's just
  a symlink existence check) and `/api/action/sites/<site>/<action>`,
  validated against the whitelist server-side too (defense in depth,
  same as every other action route in this app).
- New "SELF-HOSTED APPS" card on the NET tab: status dot, an "Open"
  link when enabled, and an ENABLE/DISABLE button per app.

Verified with a Flask test-client script: status/enable/disable all
invoke the correct helper action with the correct arguments; a site
name outside the whitelist (tried `omni-tools` before it's actually
configured, and a path-traversal-style value) is rejected with 400
*before* ever reaching the helper subprocess call; an invalid action
name is rejected the same way.

**Action for you**: redeploy (`bash ~/pi-control/deploy-pi-control.sh`),
then re-run the two BentoPDF nginx commands from earlier in this session
(`ln -sf .../sites-available/bentopdf .../sites-enabled/bentopdf`,
`nginx -t && systemctl reload nginx`) to re-enable it, since it's
currently disabled from the live RAM test above - or just use the new
toggle in the dashboard once redeployed.

## Phase 8s — OmniTools deployed via Windows build, added to the toggle

The Termux/sandbox build blockers from Phase 8r turned out not to apply
on a real Windows machine with Node installed normally: `npm install`
and `npm run build` both worked without issue there (the `@swc/core`
native binding that failed under Termux's Bionic libc loads fine from
its normal prebuilt Windows binary). Confirms the diagnosis from 8r -
those were tooling/environment problems, not anything about the app
itself.

Transferred the built `dist/` (zipped via `Compress-Archive`) straight
from Windows to the Pi over `scp`, since the Windows machine is already
on the same Tailscale network - no phone/Termux hop needed this time.

Deployed as a new nginx site on port 9450 (next free port after
9442/9448/9449), same TLS cert and `allow`/`deny` rules as the other
sites, `root /opt/omni-tools`. Needs `try_files $uri $uri/ /index.html;`
for the SPA fallback - unlike BentoPDF's separate-static-page-per-tool
build, OmniTools is a single-page app with client-side routing, so a
direct hit on any non-root path needs to fall through to `index.html`
or it 404s.

Added `omni-tools` to both whitelists (`ALLOWED_NGINX_SITES` in
`pi-control-helper.sh`, `NGINX_SITES` in `app.py`) so it shows up in the
SELF-HOSTED APPS card next to BentoPDF automatically - deliberately did
NOT symlink it into `sites-enabled` manually this time, since the whole
point of the Phase 8r toggle work was to make that unnecessary going
forward. First enable is via the dashboard, not a manual `ln -sf`.

No `picontrol.service` changes - standard `bash
~/pi-control/deploy-pi-control.sh`. After redeploying, both BentoPDF and
OmniTools should appear in the SELF-HOSTED APPS card - enable OmniTools
from there and confirm a couple of tools actually run (file conversion,
image tools, whatever) to verify the WASM-based client-side processing
works end to end, same as the BentoPDF verification.

## Phase 8t — Stale "Upgrade complete" toast on every reload, swipe reliability

**"Upgrade complete" replaying on every page reload, and stale package
counts after a real upgrade** - both traced to the same root cause,
confirmed via a real device screenshot showing the toast stuck on-screen
overlapping the ACTIVE SERVICES grid. `UPDATE_STATE["done_ok"]` on the
server is a *persistent* "what happened last time" flag - it stays `true`
(or `false`) forever after an upgrade finishes, only resetting when the
*next* upgrade starts. `pollUpgradeStatus()` runs unconditionally on
every page load (to correctly resume showing "Updating..." if one is
genuinely still in flight after a reload mid-upgrade) - but it was also
unconditionally replaying the completion toast every time it saw
`done_ok: true`, which after any successful upgrade is *every single
reload from then on*, not just the one that actually just finished.
Fixed by tracking whether this poll chain has actually observed a
running state before treating a "not running" response as something
worth announcing (`sawUpgradeRunning` flag) - a fresh page load
discovering an already-settled result now stays silent, while reloading
mid-upgrade still correctly resumes polling and announces the result
once it actually completes.

**Second half of the same report** - "takes a while to update but
dashboard still shows there's an update available" - the completion
handler was calling `checkUpdates()`, which reads `UPDATE_CHECK_CACHE`
(only refreshed every 30 minutes by the background loop). The moment
right after an upgrade finishes is exactly when that cache is guaranteed
to be stale - it still reflects the pre-upgrade package count. Switched
to `recheckUpdates()`, which does a live `apt-get update` and also
refreshes the shared cache, so the count is accurate immediately instead
of up to 30 minutes late.

**Tab-switching feeling unreliable / dashboard feeling "not smooth"** -
one root cause again, not two separate problems. `.tab` sets
`touch-action: pan-y` specifically so our own JS handles horizontal
swipe gestures instead of fighting the browser's native scroll - but
that also means a swipe that fails our detection thresholds doesn't fall
back to *any* native behavior, it just does nothing. The original
thresholds (70px minimum, and horizontal movement needing to be 1.5x
vertical movement) were strict enough that a normal hand's natural arc
during a swipe - some vertical drift is unavoidable - regularly failed
to register, which reads as "the dashboard is unresponsive" rather than
"that gesture wasn't recognized." Loosened to 45px / 1.1x, and added a
600ms max-duration guard (using timestamps captured on touchstart/
touchend) so a slow drag still doesn't get misread as a swipe in the
other direction now that the ratio is more forgiving.

No backend changes this batch - `static/app.js` only. Standard `bash
~/pi-control/deploy-pi-control.sh`.

## Phase 8u — RSS reader + article-to-EPUB pipeline, new READ tab

Built the personal-article-aggregator pipeline the user described (fetch
RSS -> extract clean text -> convert to EPUB -> grab it on an e-reader),
as a full web UI in the dashboard rather than a terminal/newsboat-driven
workflow - by explicit choice, confirmed up front since it's a
significantly bigger build than keeping RSS browsing in the TERMINAL tab.

**Why this was a good fit for the Pi Zero, unlike the recent Node.js/
Docker attempts**: `pandoc`, `feedparser` (pip), `trafilatura` (pip), and
`samba` are all long-established packages with solid 32-bit ARM support
via Raspberry Pi OS's own apt repo and piwheels (the Pi Foundation's own
prebuilt-wheel mirror specifically so packages like `lxml` - a
trafilatura dependency - don't need a slow from-source compile on weak
hardware). None of them run as a heavy persistent service either -
`pandoc`/`trafilatura` only do work for the few seconds it takes to
convert one article, not continuously. `newsboat` itself ended up not
needed at all - since RSS browsing lives in the web UI now, Python's
`feedparser` replaces its job entirely.

**Pipeline, verified end-to-end before shipping** (real `trafilatura`
extraction + real `pandoc` conversion tested against hand-built fixture
HTML/RSS, only the network fetch mocked, since this sandbox's own egress
policy blocks general web fetching entirely - unrelated to the Pi, which
has normal internet access):
1. `trafilatura.fetch_url(url)` downloads the page, `trafilatura.extract(
   ..., output_format="markdown")` strips nav/ads/sidebars/footers down to
   clean article text, `trafilatural.extract_metadata()` pulls title/
   author.
2. A small markdown file (`# Title` + `*by Author*` + the extracted body)
   gets written, then `pandoc file.md -o file.epub --metadata title=...`
   converts it - confirmed via `file` and by inspecting the zip contents
   that the output is a genuine, correctly-structured EPUB (`content.opf`,
   `nav.xhtml`, chapter files), not just a renamed text file.
3. The `.md` intermediate is deleted immediately after conversion - only
   the `.epub` sticks around in `~/Articles` (`/home/anon/Articles`).

**New sqlite tables** (same `trends.db`, same write-guard as everything
else): `feeds` (subscribed feed URLs) and `articles` (fetched items,
deduplicated by URL via `INSERT OR IGNORE`, with a `saved` flag).

**New background loop** `_sample_feeds()` - every ~20min, polls every
subscribed feed via `feedparser.parse()` and inserts any new entries.
Wrapped in the same outer-try/log-and-continue pattern as the other five
sampling loops (Phase 8q), so one broken feed can't kill the whole loop
or go silently unnoticed.

**New routes**, all running as the unprivileged `anon` user - no sudo
helper action needed anywhere in this feature, unlike almost everything
else in this app:
- `GET/POST /api/feeds`, `POST /api/feeds/<id>/delete` - subscribe/list/
  remove feeds. (Delete is POST, not the more RESTful DELETE method -
  deliberately, since the existing CSRF middleware only checks POST
  requests under `/api/`; a DELETE route would have silently bypassed
  it. Caught by checking for existing precedent before adding a new HTTP
  method to this codebase, rather than assumed safe.)
- `GET /api/articles`, `POST /api/articles/<id>/save` - list fetched
  articles, convert one to EPUB on demand.
- `POST /api/save-url` - the same conversion pipeline for any arbitrary
  URL, bypassing feeds entirely - lets you save an article straight from
  your phone's browser without needing to have subscribed to its feed.
- `GET /api/epubs`, `GET /epubs/<filename>` - list and download generated
  EPUBs (`send_from_directory` handles path-traversal rejection).

**New READ tab**: SAVE ARTICLE FROM URL (paste any link), RSS FEEDS
(subscribe/manage), LATEST ARTICLES (per-article SAVE AS EPUB button),
SAVED EPUBS (download links). Feed titles, article titles/summaries, and
URLs are all external, attacker-influenceable text (same reasoning as
the WiFi SSID stored-XSS fix, Phase 8q) - run through `esc()` before
touching `innerHTML`, not trusted directly. Not added to `AUTO_POLL_TABS`
- re-rendering these lists every 5s would fight anything mid-typed in
the add-feed/save-url inputs.

**Manual one-time setup required on the Pi** (not automated by
`deploy-pi-control.sh`, same reasoning as the nginx site work - these
are rare, one-shot infrastructure changes, not something to script into
every redeploy):
```
sudo apt-get install -y pandoc samba
/opt/pi-control/venv/bin/pip install trafilatura feedparser

# Samba share so an e-reader can grab EPUBs over the LAN
sudo tee -a /etc/samba/smb.conf > /dev/null << 'EOF'

[Articles]
path = /home/anon/Articles
read only = yes
browseable = yes
guest ok = no
valid users = anon
EOF
sudo smbpasswd -a anon
sudo systemctl restart smbd
```
After the pip install, redeploy as usual
(`bash ~/pi-control/deploy-pi-control.sh`) and restart `picontrol` so the
new imports actually load. Verify pandoc landed a real armv7 build
before relying on it (`pandoc --version`) - every other Docker/Node
attempt this session hit an architecture wall, so confirm rather than
assume this one is actually clean, even though apt/piwheels have a much
better track record here.

No `picontrol.service` changes - the systemd unit itself doesn't change,
only what's installed underneath it.

## Phase 8v — In-browser EPUB reader (epub.js), no phone app needed

**Why**: after Phase 8u shipped, the user had to install a separate EPUB
app on their phone just to open a saved article - defeating the "way
easier" point of the whole pipeline. Asked for an open-source, Pi-
installable way to open an EPUB straight from the dashboard.

**Tool choice**: `epub.js` (npm `epubjs`) + its zip dependency `jszip` -
a client-side-only EPUB renderer, ships pre-built minified bundles with
no build step. Deliberately not `calibre-web` (a full self-hosted
ebook-library service with its own persistent backend process) - same
"prefer zero-backend-cost static tools over a running service" call
already made for BentoPDF/OmniTools (Phase 8r/8s) and against Actual
Budget (parked for this same Pi Zero). Nothing runs server-side for this
feature beyond serving two static JS files and one template - the actual
EPUB parsing/pagination happens entirely in the visiting browser.

**Vendoring**: this agent's own sandbox blocks fetching from CDNs
(`cdn.jsdelivr.net` etc, same recurring restriction hit all session for
various domains) but allows `registry.npmjs.org`, so `npm pack epubjs
jszip` pulled the real published tarballs, which were then extracted
locally to grab the pre-built `dist/epub.min.js` (224KB) and
`dist/jszip.min.js` (98KB) - spot-checked as genuine, non-truncated
minified bundles before vendoring. Dropped into `static/vendor/`
alongside the existing Chart.js bundle, same pattern, no npm/build
tooling needed on the Pi itself.

**New route** `GET /reader` - returns `templates/reader.html`, a
standalone full-page view (not part of the tabbed dashboard shell).
Automatically login-gated like every other route (no exemption added),
confirmed via a test client hit while unauthenticated (302 to
`/login`).

**`reader.html`**: loads `jszip.min.js` then `epub.min.js`, reads the
`file` query param client-side, does
`ePub('/epubs/' + encodeURIComponent(file))` -> `.renderTo('viewer',
{width:'100%', height:'100%'})` -> `.display()`. Title bar populates
from `book.loaded.metadata`. PREV/NEXT buttons plus ArrowLeft/ArrowRight
keyboard handlers call `rendition.prev()/.next()`. No new backend
surface - `/epubs/<filename>` already existed and is already
path-traversal-safe via `send_from_directory` (Phase 8u), so epub.js is
just another client of a route that was already safe to expose.

**`loadEpubs()` update** (`app.js`): each saved EPUB now shows a `Read
→` link (`/reader?file=...`, opens in a new tab) next to the existing
`Download →` link.

**Polish**: registered `application/epub+zip` for `.epub` via
`mimetypes.add_type()` near app startup, since Python's default
`mimetypes` table doesn't necessarily know that extension - affects the
`Content-Type` header on `/epubs/<filename>`, though epub.js's own
JSZip-based fetch-and-parse doesn't actually depend on it.

**Verification**: `python3 -m py_compile app.py` and `node --check
static/app.js` both clean. Ran a real Flask test-client smoke test (in a
throwaway venv with flask/feedparser/trafilatura/psutil installed) -
confirmed authenticated `GET /reader` returns 200 with both vendor
`<script>` tags and the `ePub(` call present in the response body,
`GET /reader?file=test.epub` also returns 200 (file existence isn't
checked server-side, by design - epub.js handles a missing/bad file
client-side via the `#reader-error` fallback), and unauthenticated
`GET /reader` redirects to `/login` exactly like every other route.
Diffed the full working tree against the last committed tarball before
repackaging - confirmed the only changes were the two new `import`/
route lines and mimetype registration in `app.py`, the `loadEpubs()` Read
link in `app.js`, and the three new vendor/template files, nothing else
moved.

No `picontrol.service` or Samba changes needed - this feature is pure
addition on top of the Phase 8u pipeline.

## Phase 8w — Fixed "could not download the page" for Medium (and similar) articles

**Report**: SAVE ARTICLE FROM URL failed on a Medium article with "could
not download the page." Separately, pasting that same article URL into
ADD FEED gave "could not parse as an RSS/Atom feed" - a *different,
correct* error, not a bug: an article page is HTML, not a feed. (Medium
does publish real feeds, just at a different URL -
`https://medium.com/feed/@username`, `/feed/<publication>`, or
`/feed/tag/<tag>` - pasting the article's own URL into ADD FEED will
never work.)

**Root cause of the actual bug**: read trafilatura's source
(`downloads.py`) rather than guess - `fetch_url()` sends every request
with the literal header `User-Agent: trafilatura/2.2.0
(+https://github.com/adbar/trafilatura)`, and `_is_suitable_response()`
treats any non-200 response as a failed download with no further
detail. Medium (and plenty of other sites) block that self-identifying
scraper UA outright, so every fetch attempt was failing before
extraction even started - nothing wrong with the article page itself.

**Fix**: build one module-level `ConfigParser` via
`trafilatura.settings.use_config()`, override its `USER_AGENTS` entry
with a handful of real desktop-browser UA strings (Chrome/Windows,
Safari/Mac, Chrome/Linux), and pass it as `config=` on the `fetch_url()`
call in `_generate_epub()` - `trafilatura`'s own header logic picks one
at random per request. `extract()`/`extract_metadata()` don't touch the
network so they're untouched.

**Verification**: didn't trust this against real Medium (the agent
sandbox blocks fetching arbitrary domains, same restriction hit all
session) - instead spun up a local `http.server` that mimics the exact
failure: 403s any request whose User-Agent contains `"trafilatura"`,
200s everything else with real article HTML. Confirmed
`trafilatura.fetch_url()` returns `None` against it with the default
config (reproducing the bug) and returns the full page with the
browser-UA config (confirming the fix), then ran the real
`_generate_epub()` end-to-end against the same server and confirmed a
valid non-empty `.epub` came out the other end.

No frontend changes - same `/api/save-url` and `/api/articles/<id>/save`
routes, same error contract, just a fetch that now actually succeeds
against UA-sniffing sites.

## Phase 8x — Same Medium URL still failed after 8w; made the error diagnostic instead of guessing again

**Report**: the exact same "could not download the page" on a specific
Medium article, even after the browser-UA fix (8w). Tried to reproduce
directly against the real URL first, via both `curl` and the `WebFetch`
tool - both come back `EGRESS_BLOCKED`/403 for `medium.com` specifically
from this agent's sandbox (not the Pi - the Pi has always had normal
internet access; this is the same sandbox-network restriction hit
repeatedly all session for other domains). No way to get a real response
from Medium from inside this environment to confirm what's actually
happening.

Rather than guess at a second blind fix (Medium's real bot-defense is
almost certainly more than a User-Agent string - TLS/HTTP fingerprinting
and JS challenges are common on sites at that scale, and a plain
`urllib3` GET can't clear those regardless of headers; separately, some
Medium posts are genuinely member-paywalled and would fail for *any*
unauthenticated fetch, browser included), changed `_generate_epub` to
surface the *actual* HTTP status instead of a single flat string, so the
next failure is diagnosable instead of another guess:
- non-200 response -> `"could not download the page (site returned HTTP
  <code>)"` - tells a bot-block (403/429) apart from a dead link (404)
  apart from a server error (5xx).
- 200 but suspiciously short/empty body -> `"downloaded the page but it
  looked empty - likely a login wall, paywall, or JS-only page"` -
  covers the case where the site returns 200 with a login/JS-shell page
  instead of a real block, which a bare "could not download" would have
  hidden.

Implementation: switched from the high-level `trafilatura.fetch_url()`
(collapses every failure into a bare `None`) to the lower-level
`trafilatura.fetch_response()`, which keeps `.status` and `.html`
around.

**Verification**: extended the same local `http.server` fixture from
8w with three response modes (403, 200-with-login-wall-body,
200-with-real-article) and ran `_generate_epub()` against each,
confirming each one now produces its own distinct, correct message
(`HTTP 403` / `"looked empty"` / a real `.epub`) rather than the same
generic string for all three. Still could not test against the real
`medium.com` URL itself from this sandbox - that's the actual gap here.

**Next step, not yet done**: the real fix (if the cause is bot
fingerprinting rather than a paywall) would most likely need a
JS-capable fetch path or a service specifically built to get past this
(e.g. a headless-browser render), which is a meaningfully bigger and
heavier addition than anything else in this pipeline and wasn't added
speculatively without confirming it's actually needed. Asked the user to
run one `curl` command directly on the Pi (real internet access, unlike
this sandbox) to get the real status/response, so the next fix is
grounded in an actual observed cause instead of another guess.

## Phase 8y — Confirmed Cloudflare JS challenge; added a Jina Reader fallback

**Confirmed cause**: the user ran the requested `curl -I` on the Pi
against the real Medium URL. Response: `HTTP/2 403` with
`cf-mitigated: challenge`, a Cloudflare-managed JS challenge header
(Turnstile), plus a CSP naming `challenges.cloudflare.com`. Not a simple
User-Agent block - this requires executing JavaScript and clearing a
browser-fingerprint check, which no HTTP client (curl, urllib3,
trafilatura) can do by sending different headers. The Phase 8w fix
(rotating browser UAs) genuinely helps against simpler UA-sniffing
sites, but was never going to be enough for Medium specifically.

**Options weighed with the user** (via AskUserQuestion, since this is a
real architectural tradeoff, not something to silently pick):
1. **Add a Jina Reader fallback** - r.jina.ai (Jina AI Reader,
   open-source: github.com/jina-ai/reader) runs a real browser
   server-side and returns rendered markdown, free, no API key. Only
   real drawback: the article URL gets sent to that third party instead
   of the Pi talking to the site directly.
2. Leave it as-is and accept Cloudflare-challenge sites don't work.
3. Run a real headless browser on the Pi itself - rejected without even
   trying: Chromium alone needs 300-500MB+ RAM and the Pi Zero 2W only
   has 512MB total, so this would almost certainly starve or crash
   everything else running on the device.

User picked option 1.

**Implementation**: split the old single fetch+extract block in
`_generate_epub` into `_fetch_direct()` (unchanged trafilatura pipeline,
same diagnostic status-aware errors from Phase 8x) and a new
`_fetch_via_jina_reader()`, tried only when `_fetch_direct()` comes back
empty. Jina's plain-GET response format is `Title: ...` / `URL Source:
...` / `Markdown Content:\n<body>` - parsed out with a regex + a
`str.find()` split, falling back to using the raw response verbatim if
that marker's ever missing so a format change degrades gracefully
instead of breaking outright. On success the extracted title/markdown
feed into the exact same pandoc conversion path as the direct-fetch
case - no separate code path downstream of the fetch.

**Verification, and its limits**: this agent's sandbox blocks
`r.jina.ai` too (confirmed via both `curl` and the `WebFetch` tool
returning `EGRESS_BLOCKED`), so the Jina Reader response format itself
could only be sanity-checked from documented behavior/training
knowledge, not fetched and inspected directly - flagging this honestly
rather than presenting it as fully verified. What *was* tested directly:
a local `http.server` returning 403 (simulating the Cloudflare block)
confirmed `_generate_epub` still falls through to the Phase 8x
diagnostic error when Jina is also unreachable (this test's own call to
the real `r.jina.ai` failed exactly as expected, given the sandbox
block, and was caught cleanly by the `try/except` in
`_fetch_via_jina_reader` rather than propagating), and a mocked
`urllib.request.urlopen` returning a synthetic Jina-format response
confirmed the title/markdown-parsing and full pandoc conversion produce
a valid, non-empty `.epub`. What's still unverified is Jina's *actual*
live response shape and whether it successfully renders a
Cloudflare-challenged page end to end - ask the user to try the same
Medium URL again after redeploying and report back what they get.

No new dependencies, no new routes - `_generate_epub` is called from the
same two existing routes (`/api/save-url`, `/api/articles/<id>/save`) as
before.

## Phase 8z — Medium "cutoff" was the member paywall, not a bug; closing the loop

**Report**: after 8y, that Medium URL now converted successfully (past
the Cloudflare challenge) but only produced a small top portion of the
article, not the full text.

Two very different causes could produce that same symptom - Jina's
headless render not waiting for lazy-loaded content, or the article
genuinely being a Medium Partner Program (member-only) post where even
a real logged-out browser only ever sees a free preview. These need
opposite responses (a rendering tweak vs. not touching it at all), so
rather than guess a third time, asked the user to open the same URL in
an ordinary logged-out browser tab and check whether it also cuts off at
a "Member-only story" banner. Confirmed: yes, same cutoff.

**Conclusion**: not a bug. The pipeline is extracting exactly what's
actually publicly served - the free preview - because that's genuinely
all a non-member gets, full stop. No code change made, and none should
be: getting the rest would require an authenticated Medium session
(login credentials/cookies), which is exactly the paywall circumvention
this whole pipeline was designed from the start to avoid in favor of
RSS/legitimate access (see the original pipeline rationale, Phase 8u).
Scope stays as-is: full extraction works for non-paywalled Medium posts
(the Cloudflare-challenge fix from 8y covers those) and for any other
site a direct fetch or the Jina fallback can reach; Partner-Program
Medium posts will only ever yield their free preview through this tool,
by design.

## Phase 9 — Removed the RSS reader / article-to-EPUB pipeline entirely

**Why**: the feature (Phases 8u-8z) was built around reading saved
articles, but the one concrete article the user wanted to read on it hit
Medium's member paywall (8z) - genuinely unfixable without paywall
circumvention, which was explicitly declined (including a follow-up ask
about a Freedium-style Googlebot-spoofing bypass - also declined, same
reasoning). With that closed off, the user asked to remove the tab and
its code rather than keep unused surface area around.

**Removed**:
- `app.py`: the entire RSS/EPUB pipeline block (`ARTICLES_DIR`,
  `_slugify`, `_TRAFILATURA_CONFIG`, `_fetch_direct`,
  `_fetch_via_jina_reader`, `_generate_epub`, `_fetch_all_feeds`,
  `_sample_feeds`, and the `/api/feeds` (GET/POST/delete),
  `/api/articles` (GET/save), `/api/save-url`, `/api/epubs`,
  `/epubs/<filename>`, and `/reader` routes), the `_sample_feeds` thread
  start, the `feeds`/`articles` `CREATE TABLE` statements in
  `_init_trends_db`, the `mimetypes.add_type` EPUB registration, and the
  now-unused `calendar`/`mimetypes`/`urllib.request`/`feedparser`/
  `trafilatura`/`send_from_directory` imports (confirmed via grep each
  had no other use in the file before removing).
- `templates/reader.html`, `static/vendor/epub.min.js`,
  `static/vendor/jszip.min.js`.
- `templates/index.html`: the `#tab-reading` section and the READ
  `#tabbar` button.
- `static/app.js`: `loadReading`/`loadFeeds`/`addFeed`/`deleteFeed`/
  `loadArticles`/`saveArticle`/`saveArticleUrl`/`loadEpubs`, the
  `reading` entry in `titles`/`TAB_ORDER`, and the `loadTab` dispatch
  line.

**Left alone, deliberately** (pre-existing manual one-time setup from
Phase 8u, not something a code change should silently undo): the Samba
`[Articles]` share config in `/etc/samba/smb.conf`, the `anon` Samba
user, `pandoc`/`samba` apt packages, the Python `trafilatura`/
`feedparser` pip packages, any `.epub` files already sitting in
`~/Articles`, and any existing `feeds`/`articles` rows already in
`trends.db` (the tables just stop being created on a fresh install now -
existing ones aren't dropped). None of this is harmful left in place;
if you want it fully gone, that's a manual cleanup on the Pi
(`sudo apt remove pandoc samba`, remove the `[Articles]` block from
`smb.conf` and `sudo systemctl restart smbd`, `rm -rf ~/Articles`) -
not doing this automatically since it's a real infrastructure change on
your device, not just a code revert.

**Verification**: `python3 -m py_compile app.py` and `node --check
static/app.js` both clean. Grepped the whole codebase afterward for
`reader`/`reading`/`epub`/`feedparser`/`trafilatura`/`/api/feeds`/
`/api/articles`/`/api/save-url`/`ARTICLES_DIR` and confirmed zero
remaining references outside unrelated matches (e.g. "reading the
process directly"). Ran a real Flask test-client smoke test:
authenticated `GET /` returns 200 with no `tab-reading` in the body and
no READ button in the tabbar, and `GET /reader`, `/api/epubs`,
`/api/feeds`, `POST /api/save-url` (with a valid CSRF token) all
correctly 404 now. Diffed the full tree against the last committed
tarball and confirmed the diff was exactly these five files with
nothing else touched.

## Phase 10 — Installable PWA, so the home-screen icon opens app-like (no widget)

**Ask**: opening the dashboard "always through a shortcut again and
again" was annoying - wanted an Android home-screen widget. Clarified
first (real fork in scope, not a detail): a true glanceable widget
(numbers on the home screen with no tap at all) needs either a native
Android app - a completely different tech stack from this Python
project - or a third-party widget app like KWGT/Tasker polling the
JSON API. The one-tap option - make the existing home-screen shortcut
open the dashboard full-screen, app-like, no browser address bar/tabs -
is a small addition on top of what's already here. User picked the
one-tap PWA option.

**What makes a home-screen shortcut open "app-like" instead of just a
bookmarked browser tab**: a Web App Manifest (`manifest.json`) with
`display: "standalone"` plus icons, linked from the page's `<head>`. A
registered service worker isn't required for Chrome's manual "Add to
Home Screen" action specifically, but it removes any doubt about
Chrome's installability checks being satisfied, costs nothing
meaningful, and is close to free to add - so added one that does
nothing but pass every request straight to the network (this dashboard
is live system state, not something to cache offline).

**Added**:
- `static/manifest.json` - name, `start_url: "/"`, `scope: "/"`,
  `display: "standalone"`, background/theme color matching the
  dashboard's own `--bg` (`#0a0a12`), three icon entries (192, 512,
  and a maskable 512 for Android's adaptive-icon cropping).
- `static/icons/icon-192.png`, `icon-512.png`, `icon-512-maskable.png`
  - generated with Pillow, not sourced from anywhere: a glowing cyan
    &pi; glyph on the dashboard's own dark background, echoing the
    existing neon `.dot.on { box-shadow: 0 0 6px cyan }` indicator
    styling already used throughout the UI. The maskable version uses a
    smaller safe-zone ratio (glyph sized to fit inside a centered 72%-
    diameter circle vs. 85% for the regular icons) so Android cropping
    it into a circle/squircle/whatever the launcher uses doesn't clip
    the glyph.
- `static/sw.js` - a pass-through service worker (install/activate/fetch
  handlers only, no caching).
- `app.py`: `GET /sw.js` (served via `app.send_static_file` from the
  site root, not `/static/sw.js`, so its default scope covers the whole
  app rather than just `/static/` - a service worker's scope is bounded
  by its own script path unless the server sends a
  `Service-Worker-Allowed` header, and serving from the root sidesteps
  needing that). Added `/sw.js` to the `_require_login` exemption list
  alongside `/login` and `/static/*` - the browser refreshes a
  registered service worker on its own schedule regardless of whether
  the dashboard session is currently logged in, and getting an HTML
  login-redirect back instead of the real script on that background
  refresh would break it.
- `templates/index.html` and `templates/login.html`: manifest link,
  `theme-color` meta, `apple-touch-icon` (harmless bonus for iOS
  Safari's own Add to Home Screen, not the ask, but free to include),
  and a `navigator.serviceWorker.register('/sw.js')` call. Added to
  *both* templates, not just `index.html` - the manifest's `start_url`
  is `/`, which redirects to `/login` if the 30-day session cookie has
  expired, and the login page should feel like the same app rather than
  reverting to a bare browser tab the moment you're logged out.

**Real limitation, stated honestly rather than assumed away**: this
dashboard is reached over a self-signed TLS cert on a raw Tailscale IP
(`listen ... ssl` per Phase 8p, no real hostname/CA involved), not a
publicly-trusted certificate. Chrome's *automatic* PWA install
banner/mini-infobar does require full installability criteria on a
trusted origin, which this may not satisfy. What this doesn't depend
on: the manual "Add to Home Screen" action from Chrome's menu, which in
testing across Chrome versions reads the manifest and honors
`display: standalone` for the resulting shortcut's launch behavior
independent of that stricter automatic-install path. Any home-screen
icon added *before* this change was created without a manifest present
and won't retroactively pick up standalone mode - it has to be removed
and re-added after redeploying for the new behavior to take effect.

**Verification**: `python3 -m py_compile app.py`, manifest.json parsed
with `json.load`, `node --check static/sw.js` all clean. Rendered both
generated icons and visually confirmed the glyph sits fully inside the
maskable safe zone and reads clearly at small size. Real Flask
test-client smoke test: logged-out `GET /sw.js`, `/static/manifest.json`,
and `/static/icons/icon-192.png` all return 200 (confirming the
login-exemption works), `/login`'s body contains the manifest link and
SW registration call, logged-out `GET /` still redirects to `/login` as
before (exemption didn't leak), and logged-in `GET /` contains the
manifest link, `apple-touch-icon`, and SW registration. Diffed the full
tree against the last committed tarball - confirmed exactly `app.py`,
both templates, and the three new `static/` files changed, nothing
else.

**Not verified, and can't be from here**: whether Android actually
renders the resulting shortcut as standalone/app-like on the user's
specific device and Chrome version - ask them to remove the existing
shortcut, redeploy, re-add it from Chrome's menu, and confirm it opens
without the address bar.

**Confirmed working**: the self-signed cert was in fact the whole
blocker. Chrome's manual "Add to Home Screen" flow doesn't need full
installability criteria, but "Install" does, and it specifically
requires a validly-trusted certificate chain - a self-signed one fails
that check even after the user clicks through the browser's warning.
Fixed on the real device (MagicDNS + HTTPS Certificates were both
already enabled in the Tailscale admin console, nothing to change
there):
```
tailscale cert --cert-file /tmp/monitor.crt --key-file /tmp/monitor.key anon.tail8dd783.ts.net
sudo install -o root -g root -m 644 /tmp/monitor.crt /etc/nginx/ssl/monitor.crt
sudo install -o root -g root -m 600 /tmp/monitor.key /etc/nginx/ssl/monitor.key
sudo nginx -t && sudo systemctl reload nginx
```
(needed `sudo` on the `tailscale cert` call itself too - plain user
access got "Access denied: cert access denied".) Since every nginx site
on this Pi already pointed at this same shared `monitor.crt`/`.key` pair
(per Phase 8s), this one file swap upgraded pi-control, BentoPDF, and
OmniTools all at once, with zero nginx config edits. Dashboard now lives
at `https://anon.tail8dd783.ts.net:9448/` instead of the raw Tailscale
IP - PWA installs are scoped per-origin, so the old IP-based shortcut
doesn't inherit this; confirmed on-device that Chrome now offers a real
"Install" (not falling back to a plain shortcut) and no longer shows a
cert warning.

## Phase 11 — Automatic renewal for the new Tailscale cert

**Why**: `tailscale cert`-issued certs are real Let's Encrypt certs -
they expire (~90 days) and don't renew themselves. Left alone, the cert
Phase 10 just set up quietly breaks HTTPS (and the PWA install) again
in a few months.

**Added**:
- `renew-monitor-cert.sh` (deploys to `/usr/local/bin/`) - runs
  `tailscale cert` to a temp location, diffs the result against the
  live `/etc/nginx/ssl/monitor.crt` via `cmp`, and only if it actually
  changed: installs with the same ownership/permissions as the Phase 10
  manual steps (`root:root`, 644 for the cert / 600 for the key), runs
  `nginx -t`, and only reloads nginx if that test passes. `tailscale
  cert` itself only actually reissues near expiry regardless of how
  often it's called, so running this far more often than strictly
  necessary is safe - most runs are a same-bytes no-op.
- `pi-control-cert-renew.cron` (deploys to
  `/etc/cron.d/pi-control-cert-renew`) - runs it weekly, Sunday 3:17am.
- `deploy-pi-control.sh`: installs both, same pattern as the existing
  `pi-control-helper.sh`/`telegram-send.sh` install steps.
- On any real failure (`tailscale cert` itself failing, or a newly
  fetched cert somehow failing `nginx -t`), sends a Telegram alert via
  the existing `telegram-send.sh` in addition to a syslog line via
  `logger -t pi-control-cert-renew` - reusing the same proactive
  alerting pipeline the rest of this dashboard already has (Phase 8g),
  rather than a failure sitting silent in syslog until the cert
  actually expires months later. A nginx-test failure specifically
  leaves the new cert files written to disk but does **not** reload
  nginx - the live process keeps running on its still-valid old cert
  either way, so this failure mode has zero production impact beyond
  needing someone to go look at why.
- Runs entirely as root via cron (root crontab entry) rather than as
  `anon` - both `tailscale cert` and writing into `/etc/nginx/ssl/`
  need root regardless, so there's no reason to fight that with the
  `tailscale set --operator=` escape hatch the CLI suggested during
  Phase 10's manual run.

**Verification**: since this only touches real infrastructure on the
Pi (root cron, `/etc/nginx/ssl/`, `tailscale`, `nginx`, `telegram-send.sh`)
none of which exists in this sandbox, verified the *script's own logic*
against mocked versions of every external command (`tailscale`,
`nginx`, `systemctl`, `logger`, `telegram-send.sh`) standing in for the
real ones, run from a copy of the script pointed at a scratch
`CERT_DIR` instead of the real `/etc/nginx/ssl/`. Confirmed all four
paths behave correctly: (1) cert unchanged -> logged as a no-op, no
install, no reload; (2) cert changed + `nginx -t` passes -> installed
with correct ownership/perms, nginx reloaded exactly once; (3)
`tailscale cert` itself fails -> non-zero exit, alerted via both
`logger` and the Telegram mock, existing cert left untouched; (4) cert
changed but `nginx -t` fails -> new (bad) cert written to disk but
reload correctly skipped, alerted via both channels. `bash -n` clean on
both `renew-monitor-cert.sh` and the updated `deploy-pi-control.sh`.
Diffed the full tree against the last committed tarball and confirmed
the only changes were the two new files and the four added lines in
`deploy-pi-control.sh`.

**Not verified, and can't be from here**: the real `tailscale cert`
CLI's actual output/exit-code shape (mocked from what was observed
during the Phase 10 manual run, not independently confirmed against
live Tailscale infrastructure), and whether cron on this specific Pi OS
picks up a new `/etc/cron.d/` file without a service restart (standard
Debian cron does this automatically by checking file mtimes - not
something to just assume works identically on every install).
Redeploy, then a soft way to sanity-check without waiting for Sunday:
`sudo /usr/local/bin/renew-monitor-cert.sh` by hand once, and confirm
via `journalctl -t pi-control-cert-renew` that it logged a normal
"cert unchanged, nothing to do" (expected, since Phase 10 just renewed
it) rather than an error.

## Phase 12 — Fixed deploy-pi-control.sh silently skipping its own new steps

**Report**: ran the redeploy for Phase 11 - it completed clean, service
started fine, but `renew-monitor-cert.sh` was never actually installed
(`sudo /usr/local/bin/renew-monitor-cert.sh` came back "command not
found", `journalctl -t pi-control-cert-renew` had nothing at all).

**Root cause**: `deploy-pi-control.sh` overwrites itself mid-run. It's
invoked as `bash ~/pi-control/deploy-pi-control.sh` - a stale on-disk
copy from the *previous* deploy - and partway through, it does `tar
xzf ~/pi-control.tar.gz -C ~` which extracts a fresh
`deploy-pi-control.sh` over that exact same path. bash reads a script's
content before executing it, so the already-running process kept going
on its old in-memory copy of the script and finished normally -
completely skipping the newly-added cert-renewal install step, since
that process never saw it existed. This isn't specific to that one
step - *any* step added to this script will silently not run on the
first deploy after being added, forever, until this is fixed.

**Reproduced deliberately before trusting the diagnosis**: wrote a
throwaway script that prints a step, overwrites its own `$0` mid-run via
a heredoc, then tries to print more old-content steps - confirmed the
already-running process doesn't pick up the new content (it didn't even
finish printing its own remaining old-content lines, consistent with
bash having only buffered part of the file before the overwrite
disrupted the read) - and confirmed a *second*, separate invocation
does correctly see the new content. Matches exactly what was observed
on the real Pi: the run completed using full old-script behavior,
simply missing the one step that didn't exist in the version this
process had already read.

**Fix**: added a guard right after `cd "$HOME"` - extract the tarball,
then `exec bash "$HOME/pi-control/deploy-pi-control.sh" "$@"` to
relaunch from the just-extracted (now current) copy, gated by a
`PI_CONTROL_DEPLOY_REEXECED` env var so it only happens once. Every step
after that point runs from a freshly-read, guaranteed-current file, in
the same single `bash ~/pi-control/deploy-pi-control.sh` invocation the
user already runs - no workflow change needed, no "run it twice" advice
to remember for every future update.

**Verification**: reproduced the underlying bug first (above), then
verified the fix against the *exact same* reproduction shape (old
content overwriting itself, guarded re-exec added) and confirmed a
single invocation now correctly runs the new content instead of the
old. Beyond that isolated mechanism check, ran the real, unmodified
`deploy-pi-control.sh` end-to-end against a fully mocked environment
(fake `$HOME`, a real tarball built from the current source tree, mocked
`sudo`/`systemctl`/`chown` so nothing touched real system state
destructively) and confirmed: "Extracting pi-control archive" prints
exactly once (no re-exec loop), every step through "Installing
renew-monitor-cert.sh + its weekly cron job" actually runs, the files
land where expected, and `systemctl stop/start picontrol` each fire
exactly once (no duplicate stop/start from the re-exec). `bash -n`
clean. Diffed the full tree against the last committed tarball -
confirmed `deploy-pi-control.sh` was the only file that changed.

Redeploy once more (`bash ~/pi-control/deploy-pi-control.sh`) - this run
should finally show the "Installing renew-monitor-cert.sh" step and
actually install it, since the fix itself needs one old-script run to
get onto the Pi before it can start protecting future updates.

## Phase 13 — IP geolocation on EGRESS CHECK, and a WEATHER card

**Where this came from**: built a searchable directory ("Patch Panel",
a separate artifact, not part of this repo) of every API in
public-apis/public-apis, then a standalone bulk-prober script
(`probe-apis.py`, sent directly to the user, not part of this repo
either - a one-off tool, not a persistent feature) that hit all 786
"no key needed" entries from the real Pi (this agent's own sandbox
blocks nearly all outbound domains, confirmed again by testing a
handful of these APIs directly before concluding the probe had to run
on the Pi instead). Of the 23 that came back as genuinely live JSON
APIs, most were either irrelevant to this project (currency rates,
foreign postcodes, academic data) or outright suspicious (several
recently-added "crypto agent payment" entries explicitly describing
pay-per-call pricing despite being listed as "no auth needed" - flagged
as likely spam, not used). Two were real, directly relevant wins:
`ipinfo.io` (IP geolocation) and Open-Meteo (confirmed live in the
probe, though under its marketing homepage URL, not the real API
endpoint the directory should have listed).

**IP geolocation on EGRESS CHECK**: `_geolocate_ip(ip)` (`app.py`) calls
`ipinfo.io/<ip>/json` (free, no key) and returns a city/country label
plus ISP org. `/api/net/egress` now calls it for both `direct_ip` and
`exit_path_ip` and returns `direct_geo`/`exit_geo` alongside the
existing bare IPs. This isn't just decoration - geolocating *both* IPs
turns "the number changed" into "the country and ISP actually changed",
a real confirmation that the exit-node/Proton path is doing what it's
supposed to (verified in testing: a mock direct IP resolved to Noida,
India while a mock exit-path IP resolved to Amsterdam under a Proton
ASN - exactly the kind of contrast this is meant to surface). The
existing `curl ifconfig.me` direct-IP lookup was pulled out into
`_direct_public_ip()` so the weather feature below can reuse it. Drive-by
fix while touching this code: `direct_ip`/`exit_path_ip` (both external,
attacker-influenceable strings, technically, even if in practice always
just a dotted IP) now go through `esc()` before hitting `innerHTML` in
`checkEgress()`, matching the rest of the codebase's standing rule -
they weren't before.

**WEATHER card** (new, on the OVERVIEW tab): condition/temp/humidity/wind
from Open-Meteo's real forecast endpoint
(`api.open-meteo.com/v1/forecast`), free and keyless. Location is a
one-time lat/lon setting (`weather_lat`/`weather_lon` in the existing
generic `settings` table, same `_load_setting`/`_save_setting` helpers
alert thresholds already use) rather than auto-detected fresh on every
load - IP geolocation is city-level at best and occasionally wrong
outright, and weather specifically is worth getting right once rather
than silently drifting if the Pi's apparent location shifts (VPN
reconnects, ISP reassigns the exit IP, etc). A **DETECT FROM IP** button
reuses `_geolocate_ip()`/`_direct_public_ip()` to prefill the lat/lon
inputs as a convenience, but doesn't auto-save - shown back to the user
to glance at before committing. Numeric WMO weather codes translated to
plain text (`_WEATHER_CODES`) since Open-Meteo returns a code, not a
description.

**Deliberately not on the 5s auto-poll cycle**: the OVERVIEW tab already
refreshes every 5s via `AUTO_POLL_TABS`, which is exactly wrong for this
- weather doesn't change that fast, and hammering a free/no-key API
every 5s just because its card happens to live on the auto-polled tab
would be poor citizenship for zero benefit. `loadWeather()` isn't
wired into `loadTab()` at all; it runs once at page load and on its own
independent 15-minute `setInterval`, plus a manual REFRESH button.

**Verification**: `python3 -m py_compile app.py`, `node --check
static/app.js` both clean. Since every new/changed route shells out to
`curl` for a real external domain (ifconfig.me, ipinfo.io,
open-meteo.com) that this sandbox can't reach, verified the full
request/response/validation logic with a mocked `curl` (and mocked
`sudo` for the exit-node path) standing in for the real ones: confirmed
`/api/net/egress` returns correctly-shaped `direct_geo`/`exit_geo` (the
Noida-vs-Amsterdam case above), `/api/weather` correctly 400s with no
location set and returns the right condition/temp/humidity/wind once
one is, `/api/weather/location` GET/POST round-trips correctly and
rejects both non-numeric and out-of-range lat/lon with 400s, and
`/api/weather/detect-location` returns the mocked coordinates without
saving them. Rendered `GET /` while authenticated and confirmed all the
new WEATHER markup (display card, setup form, both buttons) is present
in the page. Diffed the full tree against the last committed tarball -
confirmed exactly `app.py`, `static/app.js`, and `templates/index.html`
changed, matching what was actually intended.

**Not verified, and can't be from here**: the real `ipinfo.io` and
Open-Meteo response shapes and rate-limit behavior under actual live
traffic - the mocked responses were built from the exact JSON fields
the earlier bulk-probe (Phase 13's own predecessor step) captured from
these same two services, not independently re-confirmed live. Ask the
user to redeploy, tap CHECK MY IP to see real geolocation on both
addresses, and use DETECT FROM IP + SAVE (or type coordinates directly)
to get the WEATHER card showing real local conditions.

## Phase 14 — Weather follows the phone, not the Pi; egress shows a real bug

**Two reports after Phase 13 went live**: (1) weather correctly showed
real data, but for wherever the *Pi* is - not useful, since a wall-
mounted/plugged-in Pi never moves but the person looking at the
dashboard does; (2) EGRESS CHECK showed `Exit-node path: curl: (28)
Connection timed out after 8002 milliseconds curl_failed_via_tailscale0`
- the exit-node route itself timing out (a real Tailscale/Proton
connectivity issue on the device, not something fixed here), but also
exposing that this diagnostic text was being displayed and geolocated
as if it were a real IP.

**Weather now prefers the browser's own location.** `_browserLocation()`
(`app.js`) wraps `navigator.geolocation.getCurrentPosition()` -
`enableHighAccuracy: false` (city-level is plenty for weather, and
faster/cheaper than a precise GPS fix), `maximumAge: 10min` (lets the
browser reuse a recent fix instead of forcing a fresh one on every
15-minute auto-refresh). `loadWeather()` tries this first and passes the
coordinates as `?lat=&lon=` on `/api/weather`; only falls back to the
saved default location (Phase 13's `weather_lat`/`weather_lon` setting)
if permission is denied, the browser doesn't support it, or it times
out. The response now includes `"source": "live"` vs `"saved"`, shown
under the card so it's clear which one is actually in effect. This only
works at all because of the real Tailscale cert (Phase 10) - the
Geolocation API is flatly unavailable on an insecure origin, no silent
fallback, so this literally could not have worked before that fix
landed. Relabeled the setup form and its DETECT button ("DETECT FROM
PI'S IP") to make clear that flow is now a fallback, not the primary
path.

**The diagnostic-text-treated-as-an-IP bug**: `egress-exit` (the sudo
helper action) deliberately returns text like
`curl_failed_via_tailscale0` or the raw `curl` stderr instead of an IP
when the exit-node path is down - by design, so the dashboard can show
*why* it failed rather than a bare "failed". `_geolocate_ip()` and
`checkEgress()`'s rendering didn't know the difference and would
happily try to geolocate that text and display it as if it were a
value. Added `_looks_like_ip()` (`ipaddress.ip_address()`, already
imported) as a real validity check - `_geolocate_ip()` now returns
`None` immediately for anything that doesn't parse as an IP, and
`/api/net/egress` exposes `direct_ok`/`exit_ok` booleans so the frontend
can render diagnostic text in the existing pink error color instead of
presenting it as a normal value.

**Verification**: `python3 -m py_compile app.py`, `node --check
static/app.js` both clean. Extended the mocked-`curl`/mocked-`sudo` test
harness from Phase 13 with the exact real-world failure reported -
`sudo` mock returns the literal
`curl: (28) Connection timed out after 8002 milliseconds
curl_failed_via_tailscale0` string for `egress-exit` - confirmed
`exit_ok` comes back `False`, `exit_geo` comes back `None` (no wasted
geolocation attempt against garbage text), while the still-good
`direct_ip` path is unaffected. Confirmed `/api/weather?lat=&lon=`
returns `source: "live"` with the right data, omitting both params
falls back to `source: "saved"`, no location at all still 400s, and
garbage (non-numeric) lat/lon query params correctly fall through to
the saved default rather than crashing. Rendered `GET /` and confirmed
the new `weather-source` element and relabeled buttons are present;
confirmed `app.js` contains the new geolocation code (checked the
static file directly, not the rendered page - `navigator.geolocation`
lives in the separate `app.js`, not inline in `index.html`, an actual
mistake in this verification step's first draft that was caught before
calling it done). Diffed the full tree against the last committed
tarball - confirmed exactly `app.py`, `static/app.js`, and
`templates/index.html` changed.

**Not verified, and can't be from here**: the real browser Geolocation
permission-prompt flow, and whether the real `egress-exit` timeout is
transient or an ongoing Tailscale/Proton connectivity problem worth
separately investigating on the actual device - ask the user to
redeploy, confirm the WEATHER card now shows their phone's location
(the browser should prompt for permission on first load) with the
`weather-source` line correctly reading "Your current location", and
separately let them know whether the exit-node path is expected to be
down right now (Proton off? exit-node not currently selected on the
phone's Tailscale?) or if it's a surprise worth digging into.

## Phase 15 — Bottom tab bar replaced with a left side drawer

**Ask**: now that this installs as a real standalone PWA (Phase 10), it
should feel like an app, not a mobile website - specifically wanted the
old bottom tab bar reworked. Talked through the tradeoffs first (native
bottom bars usually cap at ~5 icons; this dashboard has 8 tabs, already
needing horizontal scroll to fit) - user's own call once it came up:
skip the bottom bar entirely, go with a side drawer opened by a 3-dot
menu button instead. That sidesteps the tab-count problem outright - a
vertical list has room for any number of tabs without scrolling to find
one, which a bottom bar never would.

**What changed**: `#tabbar` (bottom, `position:fixed`, horizontally
scrolling) is gone, replaced by `#nav-drawer` (left, `position:fixed`,
`transform: translateX(-100%)` sliding to `translateX(0)` when
`.open`), a `#nav-scrim` dimming overlay behind it, and a small 3-dot
(`&#8942;`, the actual Unicode vertical-ellipsis character - no icon
asset needed) menu button added to the header. Drawer items reuse the
exact same `data-tab` attributes and `activateTab()` logic as the old
bottom buttons - only the container and layout changed, not the
tab-switching mechanism itself. `activateTab()` now also closes the
drawer, so picking a destination feels like a single action instead of
"pick, then separately dismiss the menu." Closing works three ways -
tapping a nav item, tapping the scrim, and Escape - all verified.
Labels expanded from the bottom bar's cramped abbreviations (SYS, NET,
DISK, SEC) to the full names already used elsewhere in the app (the
existing `titles` map's values), since a drawer has the width to spare
and abbreviations only existed to fight the old bar's cramped space.

Drive-by cleanup now that nothing sits fixed at the bottom of the
screen: `body`'s `padding-bottom` (previously reserving space for the
tab bar) and `#toast`'s `bottom` offset (previously positioned just
above it) both now just use `env(safe-area-inset-bottom, 0px)`-aware
minimal spacing instead. Header and drawer both add
`env(safe-area-inset-top/bottom)` padding too, so none of this collides
with a phone's notch or gesture-nav bar - relevant now specifically
because Phase 10 made this an installed, chrome-less PWA where those
safe areas are actually visible, unlike a normal browser tab.

**Swipe-to-switch-tabs (Phase 8t) still works**, updated to query
`#nav-drawer button.active` instead of the removed `#tabbar`, plus a
new guard that ignores swipes entirely while the drawer is open (it's
its own interaction surface, not the underlying tab content).

**Verification**: `node --check static/app.js` clean. Grepped all three
changed files for any leftover `tabbar` reference after the rename -
found and fixed one in the 5s auto-poll interval's active-tab lookup
that a first pass missed (would have thrown on every poll tick once
`#tabbar` no longer existed - caught before shipping, not after).
Beyond static checks, actually ran the real Flask app under Playwright
against headless Chromium (both pre-installed in this environment) on a
390&times;844 mobile viewport, logged in for real, and screenshotted
the closed state, the open drawer, and the state right after tapping a
nav item - confirmed visually that the drawer slides in correctly, the
active tab is highlighted, the scrim dims the background, and selecting
LIVE NETWORK both closes the drawer and swaps the content. Separately
scripted and confirmed all three close paths (nav-item tap, scrim tap,
Escape key) actually flip `#nav-drawer`'s `open` class and restore
`document.body.style.overflow`, not just that they don't throw. Diffed
the full tree against the last committed tarball - confirmed only
`static/app.js`, `static/style.css`, and `templates/index.html`
changed; `app.py` untouched, as expected for a pure frontend nav
change.

**Not verified, and can't be from here**: real-device feel (transition
smoothness on the Pi Zero's target hardware class of phone, whether
78vw/280px drawer width feels right on the user's actual screen size) -
ask them to redeploy and try it.

## Phase 16 — UNBOUND INTERNALS panel, from researching ar51an/unbound-dashboard

**Where this came from**: user pointed at
github.com/ar51an/unbound-dashboard and asked if it could be built into
this dashboard. Cloned and read it - it's not one app, it's a full
observability stack (Grafana + Prometheus + Loki + Promtail + a custom
Go exporter) built for a Pi 4 with several GB of RAM. Recommended
against building that literally: Grafana alone typically wants
150-250MB, and this Pi has ~425MB total already running AdGuard,
Unbound, Vaultwarden, Tailscale, nginx, fail2ban, BentoPDF, OmniTools,
and picontrol - it would not fit. Also flagged that their per-client/
per-domain/blocked-query panels get their data from Unbound's own
logs, because *their* setup has Unbound doing the blocking - on this
Pi, AdGuard Home does the blocking and already exposes that same kind
of data through its own UI, so replicating it here would just be a
weaker copy of a tool the user already has, not new value. What
doesn't overlap with AdGuard at all: Unbound's own internal resolver
performance (cache efficiency, recursion latency, request-list health)
- AdGuard has no visibility into any of that. Scoped down to just that,
with the user's explicit agreement.

**Real output checked before writing any parsing code**: asked the user
to run `sudo unbound-control stats_noreset` (the exact command the
existing hit-rate card already calls) and paste the real result, rather
than build against assumed field names from general Unbound docs and
risk silently-wrong output. Turned out this Unbound doesn't have
`extended-statistics: yes` set, so the richer breakdowns (query types,
response codes, the response-time histogram, cache memory) genuinely
aren't available yet - only the base counters are. Scoped the panel to
exactly what's actually there rather than build for fields that don't
exist on this install: total queries, cache hits/misses (already used),
prefetch, expired, recursive replies, rate-limited/timed-out counts,
recursion time avg/median, request-list avg/max/exceeded, resolver
uptime, and a live thread count (counted from however many distinct
`threadN.` prefixes are actually present, not hardcoded to a specific
number),

**Added**:
- `_parse_unbound_stats(raw)` (`app.py`) - replaces the old 6-line inline
  hits/misses-only parse in `_sample_slow` with a real parser pulling
  all of the above, using a small `g(key, cast, default)` helper so a
  missing/renamed field degrades to a default instead of crashing (the
  existing `hit_rate`/`hits`/`misses` keys are unchanged, so the OVER
  tab's existing UNBOUND DNS card keeps working with zero changes).
  Verified field-by-field against the user's actual pasted output before
  moving on, not just eyeballed.
- New `unbound_trends` table (same `trends.db`, same write-guard
  pattern as the existing `trends`/`settings`/`alert_history` tables) +
  `_persist_unbound_trend_sample()`, written every 20th `_sample_slow`
  tick (~5min, matching the existing long-trends table's resolution) so
  recursion latency has a real history to chart, not just a live
  number.
- `/api/unbound/trends?hours=N` (capped at `TRENDS_RETENTION_DAYS`&times;24,
  same bounds-clamping pattern as `/api/trends/long`) and
  `unbound: CACHE["unbound"]` added to the existing `/api/sys` response
  - no new polling loop needed, SYS already auto-polls every 5s.
- **New UNBOUND INTERNALS card** (SYS tab, after TOP MEMORY HOGS) - a
  stat-tile grid for everything above, plus a note that query-type/
  response-code breakdowns need `extended-statistics: yes` turned on
  (not done - a separate ask, not assumed).
- **New RECURSION LATENCY (6HR) chart** - a Chart.js line chart
  (avg + median, cyan/orange to match the existing trend charts) fed by
  the new trends endpoint, built with the exact same
  create-once/update-in-place pattern and 60s refetch throttle as the
  existing `loadLongTrends()`, just pointed at the new endpoint instead
  of duplicating the pattern differently.

**Verification**: `python3 -m py_compile app.py`, `node --check
static/app.js` both clean. Ran `_parse_unbound_stats()` directly against
the user's real pasted `stats_noreset` output (not a hand-built fixture)
and asserted every single output field against hand-computed expected
values, including the uptime formatting (240442.889029s &rarr; "2d 18h
47m") and the thread count correctly coming back `1` (this install only
showed `thread0.*`, unlike ar51an's 4-thread example - confirms the
count is genuinely derived, not copied from their example). Full Flask
test-client pass: seeded `CACHE["unbound"]` and a persisted trend row,
confirmed `/api/sys` includes the new `unbound` block with correct
values, `/api/unbound/trends` returns the seeded row with correct
fields, and a garbage `?hours=` value falls back to the 6h default
instead of 500ing. Rendered `GET /` and confirmed the new card, all
twelve stat-tile IDs, and the new canvas are present. Diffed the full
tree against the last committed tarball - confirmed exactly `app.py`,
`static/app.js`, and `templates/index.html` changed.

**Not done, deliberately, pending a separate answer**: enabling
`extended-statistics: yes` in `/etc/unbound/unbound.conf` (needed for
the query-type/response-code/histogram breakdowns this panel doesn't
show yet) - that's a real edit to a production DNS resolver's config
plus a restart, so it gets asked about on its own rather than folded
into this change.

**Not verified, and can't be from here**: whether `_sample_slow`'s
existing 5s sudo-helper timeout comfortably covers `unbound-control
stats_noreset` on the real device under real load (the parse itself is
cheap; the risk, if any, is entirely in the existing subprocess call
this change didn't touch), and how the new chart/tiles actually look
against real historical data once the 5-minute trend table has enough
rows to plot a real line - ask the user to redeploy and check back in a
few hours once there's more than one data point.

## Phase 17 — UNBOUND INTERNALS redesign after real-device feedback

**Report**: real screenshot from the Pi after Phase 16 - functionally
correct, but "not that good" visually. Three concrete problems visible
in the screenshot, not vague:
1. The recursion-latency chart's two lines sat pinned flat near the top
   of the chart with barely any visible shape - `scales: { y: { min: 0
   } }` forced a 0 baseline onto a metric (latency in ms) with no
   natural 0-100 range, unlike the CPU/RAM percentage charts that
   pattern was copied from, so real ~250-350ms movement got squashed
   into the top 30% of the chart.
2. All 12 stats sat in one flat 2-column grid, every number the same
   size and color - nothing told you which 2-3 numbers actually matter
   at a glance.
3. The "extended-statistics: yes" note rendered visibly larger than the
   text around it - wrapped in a bare `<code>` tag with no styling
   anywhere in style.css, so it fell back to the browser's default
   monospace size against the `.sub` class's deliberately-shrunk
   0.75rem text.

**Fixes, in the same three spots**:
- Chart: dropped the `min: 0` constraint (`beginAtZero: false` instead,
  letting Chart.js auto-scale to the actual data range), added a subtle
  area fill under each line (`fill: true` + a low-alpha version of each
  line's own color), and a scriptable `pointRadius` that only draws a
  marker on the single most recent point per line - draws the eye to
  "where things stand right now" without cluttering the rest of the
  line. Legend switched to `usePointStyle: true` (small circles instead
  of Chart.js's default rectangle swatches) - scoped to just this
  chart's own options object, not the shared CPU/RAM charts, so it
  doesn't second-guess an established pattern used elsewhere without
  being asked to.
- Stat grid: restructured into a hero row (Cache Hit Rate, Recursion
  Avg - full `.big` size) plus three labeled secondary groups (CACHE /
  LATENCY / HEALTH, using a new small-caps `.group-label` divider) with
  smaller `.big.small` tiles. Combined naturally-paired stats into
  single tiles (`3.83 / 63` for request-list avg/max, `0 / 0` for
  rate-limited/timed-out) to cut 12 separate numbers down to a scannable
  8. Total queries and resolver uptime moved into a single caption line
  under the hero row instead of taking two more full tiles.
- Color now encodes health, not just decoration: cache hit rate turns
  green at &ge;85%, pink below 60% (a new `.big.pink` class, mirroring
  the existing `.big.green`). Rate-limited/timed-out stays neutral at
  zero (the boring, expected case) and only turns pink when either is
  actually non-zero - the opposite convention from hit rate, deliberately,
  since for these two "high number = good" is backwards.
- `<code>` replaced with `<strong>` plus a new `.sub strong { color:
  var(--text); font-weight: 700; }` rule, so emphasis reads as brighter/
  bolder text at the *same* size as its sentence instead of an
  oversized, unstyled tag.

**Verification**: `python3 -m py_compile app.py`, `node --check
static/app.js` both clean (app.py itself is untouched - this was a pure
CSS/HTML/JS visual pass, no backend logic changed). Rather than trust
the fix from reading the diff, actually re-ran the same
Playwright-against-a-live-Flask-instance approach from Phase 15,
this time seeding *realistic* data instead of empty defaults: a fake
`subprocess.run` stand-in returned real-looking `unbound-control`
output so the actual `_sample_slow` background thread populated
`CACHE["unbound"]` the normal way (an earlier attempt that just poked
`CACHE["unbound"]` directly from outside the running process got
overwritten by that same thread within seconds - a real, if
sandbox-only, race worth noting for the technique itself), and 72 rows
of synthetic recursion-latency history (6 hours at 5-minute resolution,
randomized within a realistic 300-400ms band) were inserted directly
into `unbound_trends` before launch. The resulting screenshot confirmed
all three fixes visually: the chart shows real jagged movement with a
visible fill and endpoint dots instead of a flat line, the hero stats
(87.2% in green, 348.4ms) are clearly the first thing the eye lands on
against the smaller grouped tiles below, and the extended-statistics
note now reads as one consistently-sized sentence with bold emphasis
rather than a jarring size jump. Diffed the full tree against the last
committed tarball - confirmed only `static/app.js`, `static/style.css`,
and `templates/index.html` changed.

**Not verified, and can't be from here**: how this reads on the user's
actual phone screen/brightness rather than a synthetic 390&times;1100
headless render - ask them to redeploy and confirm it actually reads
better this time.

## Phase 18 — extended-statistics enabled; query types, response codes, histogram

**User enabled `extended-statistics: yes`** in `/etc/unbound/unbound.conf`
themselves - checked for an existing directive first (none), backed up
the config, inserted the line with `sed -i '/^server:/a\    extended-
statistics: yes'`, validated with `unbound-checkconf` before touching
the running service (came back clean), restarted, and confirmed
`num.query.type.*` fields appeared. Then pasted the full real
`stats_noreset` dump with extended stats on, so this phase's parser
could be built and verified against actual field names/values rather
than general Unbound documentation, same discipline as every prior
phase touching this panel.

**New fields surfaced**, all parsed dynamically (no hardcoded type/code
lists, so whatever query types or response codes actually occur just
show up - confirmed by testing against the real dump, which only had
A/AAAA/HTTPS and NOERROR/nodata present, not the full standard set):
- `query_types` / `response_codes` - built from any `num.query.type.*` /
  `num.answer.rcode.*` key present with a non-zero count.
- `histogram` - Unbound's 52 fine-grained buckets (microsecond
  resolution at the low end) collapsed into 6 human-scaled ranges
  (`<1ms` through `>10s`) by parsing each bucket key's upper bound and
  summing into whichever range it falls under.
- `cache_mem_kb` (sum of all `mem.*` byte counters) and `cache_entries`
  (msg/rrset/infra/key cache entry counts).
- `unwanted_queries`/`unwanted_replies` (dropped as unwanted - a real
  security-relevant counter) and `secure_answers`/`bogus_answers`
  (DNSSEC validation outcomes).
- `extended: bool` - `True` only when `mem.cache.rrset` is present in
  the output, so the frontend can tell a genuinely-extended dump apart
  from a base one without needing its own separate config check.

**New UI**: two donut charts (QUERY TYPES, RESPONSE CODES) and a bar
chart (RESPONSE TIME DISTRIBUTION), plus two more stat groups (CACHE
MEMORY, SECURITY) inside the existing UNBOUND INTERNALS card. All of it
- the three new cards and the two new stat groups - stays hidden
(`display: none`) unless `ub.extended` is true, rather than showing
permanent `--` placeholders when it isn't; the old "enable extended-
statistics" note does the reverse (hidden once it's on, shown when it
isn't) instead of both being visible/stale at once.

**Real bug caught by actually rendering the chart, not just reading the
diff**: the histogram bars came out in alphabetical order
(`1-10ms, 1-10s, 10-100ms, 100ms-1s, <1ms, >10s`) instead of the
intended small-to-large latency order. Root cause: Flask 3.1.3 sorts
JSON object keys alphabetically by default (confirmed directly -
`jsonify({"<1ms":0,...})` came back with keys resorted), so returning
the histogram as a Python dict threw away the deliberate insertion
order the moment it crossed the JSON boundary, even though the dict
itself was correctly ordered right up until serialization. Fixed by
returning `histogram` as a list of `{"label", "count"}` objects instead
of a dict - a JSON array has no keys to sort, so order survives
regardless of Flask's dict-sorting default. `query_types`/
`response_codes` were left as dicts since (unlike the histogram) there's
no inherently "correct" order for those to begin with - alphabetical is
merely arbitrary there, not wrong.

**Verification**: `python3 -m py_compile app.py`, `node --check
static/app.js` both clean. Verified the parser field-by-field against
the user's real extended-stats dump (query types, response codes,
histogram bucket sums, cache memory/entries, unwanted/DNSSEC counters)
with hand-computed expected values, and separately re-confirmed the
Phase-16 non-extended sample still parses with `extended: False` and
none of the new keys present - this change doesn't regress the base
case. The histogram ordering bug was caught by actually rendering the
chart in a real (headless) browser via Playwright and looking at it -
not something a unit test on the Python function alone would have
caught, since the bug only exists at the Flask JSON-serialization
boundary. After the list-based fix, re-verified three ways: the
Python-level object has correct order, a real `flask.jsonify()` call
round-tripped through `json.loads()` preserves that order, and the live
rendered Chart.js instance's `.data.labels` (read directly out of the
running page via Playwright, not just asserted from the source) matches
`["<1ms","1-10ms","10-100ms","100ms-1s","1-10s",">10s"]` exactly.
Screenshotted all three new cards and confirmed the donuts, histogram,
and new stat groups all render correctly with real data, and that the
old "not enabled yet" note is gone now that `extended` is true. Diffed
the full tree against the last committed tarball - confirmed exactly
`app.py`, `static/app.js`, and `templates/index.html` changed.

**Not verified, and can't be from here**: how this looks on the user's
real phone once real (not synthetic/seeded) query-type and response-
code diversity accumulates over normal usage - ask them to redeploy and
check back once there's more than 13 total queries' worth of data to
look at.

## Phase 19 — OSINT TOOLS tab, running the personal-use toolkit from the dashboard

**Ask**: after manually installing a personal-use OSINT toolkit
directly on the Pi (sherlock, holehe, maigret, socialscan, h8mail,
ghunt, subfinder, assetfinder, gau, waybackurls, httprobe, httpx,
theHarvester, Photon, recon-ng, sn0int - a separate, lengthy install
effort not tracked in this file since it's Pi-side tooling, not
dashboard code), user asked to run these from a new dashboard page
instead of SSHing in from a phone every time, with failures shown
directly on the page instead of needing to go find them in a terminal.

**What changed**: new OSINT TOOLS tab (`#tab-osint`), added to the nav
drawer after SECURITY. A `<select>` lists the available tools (fetched
from `/api/osint/tools`), a text input takes the target (its
placeholder swaps to match the selected tool's expected input - a
username, an email, a domain, or a URL), and a RUN button starts it.
Output renders in a `.log-box` matching the SERVICE LOGS/SECURITY tabs'
existing style; a failed run additionally gets a new `.log-box.error`
style (pink border/text, matching `#toast.error`) so a failure is
visually obvious at a glance, not just readable in the text.

Backend mirrors the existing system-update background-job pattern
(`UPDATE_STATE`/`UPDATE_LOCK`) rather than inventing a new one:
`OSINT_STATE`/`OSINT_LOCK` track one run at a time (a 409 blocks a
second run while one is in flight - deliberately serialized, not
queued, given how little RAM headroom this Pi has to spare for two
concurrent scans), `/api/osint/run` starts a background thread and
returns immediately, `/api/osint/status` is polled every 3s from the
frontend while running and once more on tab load to pick up whatever
the last run left behind. Each tool's invocation lives in
`_OSINT_TOOLS` as a `(argv, stdin)` builder - `_osint_arg` for tools
that take the target as a trailing argv element, `_osint_stdin` for the
tomnomnom-style tools (waybackurls, httprobe, httpx) that read it from
stdin instead. All subprocess calls use list-form argv (no
`shell=True`), so the target text can't reach a shell regardless of
what characters it contains. Output is captured (stdout + stderr),
truncated to 20,000 characters if a tool is especially chatty, and a
180s timeout guards against a hung/slow lookup (maigret and
theHarvester in particular can take a while against a real target).

`recon-ng` and `sn0int` are deliberately **not** on this page - both
are interactive consoles/module frameworks, not oneshot CLIs, and
wiring either one up would mean scripting their own module-invocation
syntax rather than just running a command and capturing output. Still
usable over SSH as before; a scripted variant is a separate future
task if wanted.

**Verification**: `python3 -m py_compile app.py` and `node --check
static/app.js` both clean. Built an isolated venv (this sandbox has no
Flask installed globally) and exercised every new endpoint through
Flask's real test client: tool list returns all 14 oneshot tools,
an unknown tool and an empty target both correctly 400, starting a run
returns 200, a second run attempted while the first is still in flight
correctly 409s, and - since the actual tool binaries don't exist in
this sandbox - confirmed the `FileNotFoundError` path resolves cleanly
into a readable "Tool binary not found: ..." message with
`returncode: -1` rather than crashing the request or leaving the lock
stuck on `running: true`. Beyond that, ran the real Flask app under
Playwright against headless Chromium on a real browser session: logged
in, opened the drawer, clicked into OSINT TOOLS, confirmed all 14 tools
populate the dropdown, confirmed the target placeholder updates when
the tool selection changes, ran a tool end-to-end and watched the
status line go from "Starting..." to "Running sherlock on
testuser123..." to "Finished with errors (exit -1)", with the output
box showing the real captured message and picking up the `.log-box
error` class exactly as designed. Diffed the full tree against the
last committed tarball - confirmed exactly `app.py`, `static/app.js`,
`static/style.css`, and `templates/index.html` changed.

**Not verified, and can't be from here**: real output from any of the
14 tools against a real target (this sandbox can't reach the actual
binaries or the network they'd hit) - ask the user to redeploy, pick
sherlock or holehe against their own username/email first (fastest,
clearest output), and confirm the RUN button, polling, and both the
success and failure display paths look right on their real phone.

## Phase 19a — Per-tool description + example on the OSINT TOOLS tab

**Ask**: after Phase 19 shipped, user asked for a short description of
each OSINT tool right on the page (what it checks/shows) plus an
example of what to type into the target field - so picking the right
tool and knowing its input format doesn't require going back to chat
or guessing from the tool name alone.

**What changed**: `_OSINT_TOOLS` (`app.py`) gained `description` and
`example` strings per tool - what it does/what it returns for
`description`, a concrete sample value (a username, an email, a bare
domain, or a full URL depending on the tool) for `example`, both
included in `/api/osint/tools`'s response. A new `.sub` block
(`#osint-tool-desc`) sits between the tool `<select>` and the target
`<input>`, updated by the same `onchange` handler that already sets
the input's placeholder - picking a tool now shows both what it does
and what to type in it, not just an empty input with a one-word hint.

**Verification**: `python3 -m py_compile app.py` and `node --check
static/app.js` both clean. Confirmed via Flask's real test client that
all 14 tools carry non-empty `description`/`example` fields in the API
response. Ran the real Flask app under Playwright again: opened the
OSINT TOOLS tab, confirmed Sherlock's description/example render by
default, switched the dropdown to httpx and confirmed both the
description text and the placeholder update together, and
screenshotted the result to eyeball the layout - description text
wraps cleanly under the dropdown, the bolded "Example:" line reads
clearly against the dark theme, nothing overlaps the input or button
below it. Diffed the full tree against the last committed tarball -
confirmed exactly `app.py`, `static/app.js`, and `templates/index.html`
changed (`static/style.css` untouched this time, no new CSS needed).

**Not verified, and can't be from here**: whether the chosen wording
for each tool's description is exactly what the user wants once they
see all 14 side by side on their real phone - easy to tweak in
`_OSINT_TOOLS` if any read oddly in practice.

## Phase 20 — OSINT toolkit removed (RAM pressure on other services)

**Ask**: after testing the toolkit through the Phase 19/19a dashboard
tab, user asked to uninstall it - the venvs, Go binaries, and Rust
toolchain installed for it were adding to memory pressure that the
Pi's actually-important services (AdGuard, Unbound, Tailscale,
Vaultwarden) need. Asked whether to also remove the now-dead dashboard
tab rather than leave it pointing at binaries that no longer exist -
user chose to remove it.

**What changed (dashboard)**: `app.py`, `static/app.js`,
`static/style.css`, and `templates/index.html` all reverted to their
exact Phase 18 state - the entire `_OSINT_TOOLS`/`OSINT_STATE`/
`OSINT_LOCK`/`_run_osint_tool`/`/api/osint/*` block removed from
`app.py` (along with the now-unused `import traceback`), the OSINT
TOOLS nav button and `#tab-osint` markup removed from `index.html`,
the `loadOsintTab`/`runOsintTool`/`pollOsintStatus` functions and
`osint` tab-order/title entries removed from `app.js`, and the
`.log-box.error` rule removed from `style.css` (nothing else uses it).

**What to remove on the Pi itself** (not part of this repo/tarball -
these were installed directly, per the Phase 19 ask):
```bash
rm -rf ~/osint-tools ~/theharvester-venv ~/go
rm -rf ~/recon-ng ~/theHarvester ~/Photon
rm -rf ~/.cargo ~/.rustup
sudo apt remove -y golang nmap exiftool tor libsodium23 libopenblas0 libopenjp2-7
sudo apt autoremove -y
```
A reboot afterward is worth doing too - the elevated swap usage seen
while testing (51.2%, 553/1080MB) is very likely leftover pages from
the earlier Go/Rust compiles that Linux has no reason to reclaim on its
own until something else needs that memory; a reboot clears it
outright rather than waiting for pressure to force it back down
naturally.

**Verification**: `python3 -m py_compile app.py` and `node --check
static/app.js` both clean. `grep -rn "osint\|OSINT"` across all four
files returned nothing, confirming a complete removal, not just the
UI-visible parts. Extracted the Phase 18 tarball straight from that
commit (`git show 6b2392f:pi-control.tar.gz`) and diffed it against
the reverted working tree directly (not just against my own memory of
what Phase 18 looked like) - byte-for-byte identical aside from a
stray `__pycache__` directory from the compile check, which was
deleted before repackaging. This is a stronger check than the usual
"diff against last commit" - it confirms the revert didn't just remove
the OSINT code but landed on the exact same file contents Phase 18
shipped, not some close-but-not-identical approximation.

**Not verified, and can't be from here**: that the Pi-side `rm`/`apt
remove` commands above actually ran and that swap/RAM pressure
actually dropped afterward - ask the user to run them, reboot, and
check the SYSTEM DETAILS tab's swap/RAM numbers again once back up.

## Phase 20 — Vaultwarden restored (native binary), added to ACTIVE SERVICES, and a real layout bug fixed

**Context**: separately from dashboard work, Vaultwarden was reinstalled
on the Pi as a native binary (not Docker) at `/opt/vaultwarden`,
systemd-managed (`vaultwarden.service`), fronted by nginx on port 9443
(Tailscale-only, no basic auth - deliberate, so official Bitwarden
clients don't need extra config to handle a Basic Auth challenge before
reaching Vaultwarden's own login). After a long detour through two
empty/abandoned test accounts and a stale `config.json` silently
overriding a freshly-set `ADMIN_TOKEN` environment variable (Vaultwarden
persists `DOMAIN`/`SIGNUPS_ALLOWED`/`ADMIN_TOKEN` into `data/config.json`
once ever changed via the admin panel, and that file wins over the env
var from then on), the real vault (119 entries) was found in an
encrypted `.tar.gz.enc` Google Drive backup the user had separately, per
their own written restore guide - decrypted via `openssl enc -d
-aes-256-cbc -pbkdf2`, verified (`PRAGMA integrity_check`, real cipher
count) before being copied into place. `vaultwarden.service` was already
in the app's `SERVICES` list (`app.py`), so once the systemd unit
existed, it started showing up in the ACTIVE SERVICES grid automatically
- no dashboard code needed for basic status monitoring.

**What changed here**: two things, both in the frontend only (no Python
changes this phase).

1. **A real pre-existing layout bug, unrelated to Vaultwarden**: the
   RESTART button on each ACTIVE SERVICES row used `style="float:right"`
   (`static/app.js`). A float's wrap position depends on how much room
   the preceding inline text leaves on that line - for short labels
   (Nginx, UFW) the button landed beside the name; for longer ones
   (AdGuard, Unbound, Vaultwarden, Tailscale, Fail2Ban) it wrapped below
   instead, next to the mem-usage line. Caught from a real screenshot the
   user sent showing the inconsistency directly. Fixed by replacing the
   float with an explicit flex row (`.service-box-top`, `.service-box-actions`
   in `static/style.css`) - label+dot pinned left, action button(s)
   pinned right, on the same line regardless of label length.
2. **A Vaultwarden-specific OPEN button**, added into that same flex row
   only for `s.unit === 'vaultwarden'`, opening `VAULTWARDEN_URL`
   (`https://anon.tail8dd783.ts.net:9443`, a new top-of-file constant in
   `static/app.js`) in a new tab via `window.open` - not embedded via
   iframe, a deliberate choice from earlier in this phase (a password
   vault relaxing its `frame-ancestors`/`X-Frame-Options` to allow
   framing is a real, if modest, security tradeoff the user chose to
   avoid). The user separately installed Vaultwarden's own web-vault as
   its own PWA icon for day-to-day access; this button is a secondary
   quick-link/fallback, not the primary access path.

**Verification**: `node --check static/app.js` clean. Since the real
`SERVICES` list renders unconditionally regardless of whether
`systemctl`/`sudo` succeed (this sandbox has neither), verified the
layout fix directly by running the real Flask app under Playwright on a
390&times;900 mobile viewport and screenshotting the ACTIVE SERVICES
card: all seven rows (AdGuard, Unbound, Vaultwarden, Tailscale, Nginx,
UFW, Fail2Ban) now show label+dot flush left and button(s) flush right
on one consistent line regardless of label length, and Vaultwarden's row
shows both OPEN and RESTART side by side as designed. Diffed the full
tree against the last committed tarball - confirmed exactly
`static/app.js` and `static/style.css` changed, nothing else.

**Not verified, and can't be from here**: that `window.open(...)` to a
different-origin Tailscale HTTPS URL actually hands off to the installed
Vaultwarden PWA on the user's phone rather than opening a plain browser
tab (OS/browser-dependent behavior, can't observe from this sandbox) -
ask the user to tap OPEN from their phone and see which one happens; either
is a fine outcome, just worth knowing which.
