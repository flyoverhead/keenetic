# PXE hardware acceptance checklist

**Status as of 2026-09-08: steps 1-4 and 6 pass on a live router. Steps 5 and 7
are still outstanding.** Run against gigamsk with Debian 13 and Ubuntu 26.04
amd64 staged. What was found, because none of it was visible from the code:

- **Step 1 (dry run) cannot fully pass on a first deployment, and that is not a
  defect to chase.** In check mode `pxe | create directories` reports `changed`
  without creating anything, so `get_url` then fails with
  `Destination /opt/srv/tftp does not exist`. The dry run is still worth doing --
  it validated every assertion, the pool name, the LAN-address derivation and
  both rendered templates, and it is what surfaced the 404 below -- but expect
  the loader staging to fail until the directories exist for real.
- **The 404.** `keenetic_pxe_loaders` shipped pointing at
  `https://boot.ipxe.org/ipxe.efi`, which upstream has retired in favour of
  per-architecture directories. Fixed in 1.1.1; if a fresh deploy fails on
  `HTTP Error 404` staging a loader, upstream has moved the files again.
- **Step 6 passes cleanly** -- `changed=0`, no handlers -- and **none of the
  three predicted false-positives fired.** The reason is specific to this
  firmware: it renders pool options unquoted (`option 66 ascii 172.16.1.1`),
  exactly as the substring test assumes. Treat that as confirmed for this fleet,
  not as proof the test is robust in general.
- **Port 8081 was a false alarm.** KeenDNS holds 8081 on `198.51.100.11` and the
  IPv6 address, not on the br0 address the role's daemons bind, so both listen
  simultaneously. Check `netstat -ltn` *per address*.

What remains is the part that matters most and is easiest to skip: **step 7's
power-cycle checks and step 5's physical client boot.** Every check that passed
above would pass identically on a router that loses PXE entirely at the next
power cut, so nothing so far exercises the reboot claims at all.

The original checklist follows. `ansible-playbook` was blocked in the
environment this feature was built in, so template verification went through
ad-hoc `ansible` module calls; the runs above were driven by the operator
instead.


Run these against a real, already-bootstrapped Keenetic router, in order,
from a session where `ansible-playbook` is not blocked.

**1. Dry run (`--check --diff`)**

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe --check --diff \
  -e keenetic_pxe_enabled=true
```

- Expect the play to reach `pxe | require the dhcp pool to exist`.
- **If it fails there**, that is the *useful* outcome, not a bug: the assert's
  `fail_msg` names the real pool (`_WEBADMIN` on a flat config,
  `_WEBADMIN_HOME`-style on a segmented one) and tells you to run
  `ndmc -c "show running-config"` yourself and read it off. Set
  `keenetic_pxe_dhcp_pool` for that host and re-run.
- `--check` will report the `opkg` and `get_url` tasks (packages, ipxe
  loaders, boot images) as `changed` without actually installing/downloading
  anything — expected, not a defect.
- If check mode reports the router is not reachable/bootstrapped at all,
  that's the known "check mode does not work pre-bootstrap" limitation (same
  as the xray path) — run the role for real once first.

**2. Real run**

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe \
  -e keenetic_pxe_enabled=true
```

Expect it to complete with the pool assertion passing and `changed` on the
DHCP-field, package, download, and service-converge tasks.

**3. Manual verification over ssh on the router**

```bash
/opt/etc/init.d/S59tftpd status          # expect: tftpd running (pid N)
/opt/etc/init.d/S82pxehttpd status       # expect: httpd running (pid N)
ndmc -c "show running-config" | grep -A6 'ip dhcp pool'   # expect next-server + bootfile
opkg install tftp-hpa                    # the client, for the loopback test
tftp -g -r ipxe.efi -l /tmp/ipxe.efi <lan-ip> && ls -l /tmp/ipxe.efi
tftp -g -r autoexec.ipxe -l /tmp/autoexec.ipxe <lan-ip> && cat /tmp/autoexec.ipxe
curl -sI http://<lan-ip>:8081/           # expect: HTTP/1.1 200
```

**4. HTTP smoke test against a staged image (added per controller instruction)**

Static inspection can confirm `pxe-httpd.conf.j2` sets
`server.modules = ( "mod_dirlisting" )` and relies on lighttpd's documented
default-loaded core modules (`mod_staticfile`, `mod_indexfile`,
`mod_dirlisting` itself) to actually serve files, but it cannot confirm the
running binary behaves that way. After staging at least one
`keenetic_pxe_images` entry (e.g. `debian-12/linux`), from the router or any
LAN host:

```bash
curl -sI http://<lan-ip>:8081/debian-12/linux
```

- Expect `HTTP/1.1 200 OK` with a `Content-Length` matching the staged file's
  size, and `Content-Type: application/octet-stream` (the config's catch-all
  mimetype for anything `mime.conf` doesn't recognize — kernels, initrds,
  squashfs images all fall here).
- If this 404s despite the file existing on disk under `keenetic_pxe.http_root`,
  the failure is not `mod_staticfile`/`mod_indexfile` being unloaded (both are
  lighttpd core modules loaded by default and need not appear in
  `server.modules`) — look instead at `server.document-root`, file
  permissions, or the directory-nesting logic in
  `pxe | create the image subdirectories` for a `name` containing a slash.
- Also hit `curl -sI http://<lan-ip>:8081/` (no path) and expect `200` with
  `dir-listing.activate = "enable"` producing an HTML index — this is the
  intended debugging aid ("is the image actually on the router") and its
  absence would point at a config-load failure the init script's `-tt`
  self-test should already have caught at start time.

**5. Physical client boot test**

Boot one physical client from the network. Expect it to reach the iPXE menu
(`autoexec.ipxe`'s `Network boot -- <hostname>` screen).

- If the client gets a DHCP lease but never starts a TFTP transfer, the boot
  fields are not reaching it. Check first whether that ROM needs the
  hex-with-trailing-null encoding of option 67 rather than the ascii form the
  role writes (`ndmc -c "ip dhcp pool ... option 67 ascii ..."`).

**6. Idempotency re-run**

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe \
  -e keenetic_pxe_enabled=true
```

- Expect `changed=0`.
- **`pxe | set the dhcp boot fields`** is the most likely false-positive: its
  `when` does a plain substring test,
  `(item.verb ~ ' ' ~ item.value) not in keenetic_pxe_pool_block`, against the
  raw `ndmc -c "show running-config"` text. If this KeeneticOS build renders
  pool options quoted (e.g. `option 66 ascii "192.168.1.1"` instead of
  `option 66 ascii 192.168.1.1`), the `in` test never matches and all four
  loop items report `changed` every run even though the router already has
  them set. This is also called out as an existing README gotcha.
- **`pxe | converge tftpd to its declared state`** and
  **`pxe | converge httpd to its declared state`** are the other two: each
  compares a literal substring (`'tftpd running'` / `'httpd running'`) against
  the corresponding `status` command's stdout. If a firmware/BusyBox variant
  changes that wording, or the process briefly fails a `kill -0` liveness
  check between the `status` read and the converge task, these will report a
  spurious `changed`.
- None of these three should be accepted as a permanently-changed run — fix
  the regex/substring to match the router's actual rendering before treating
  the feature as done.

**7. Power-cycle verification (added per final whole-branch review)**

Steps 1-6 never power-cycle the router, yet three of this feature's central
claims are reboot claims: that `keenetic_pxe_service_state: stopped` (the
`disabled` flag) survives a reboot, that `system configuration save` makes
the DHCP boot fields persist rather than reverting to the pre-deploy running
config, and that `rc.unslung` auto-starts both hand-written init scripts on
its own. Every step above would pass identically on a router that loses PXE
entirely at the next power cut, so none of them exercises any of the three.

*7a. Reboot with the service left `started`*

```bash
ndmc -c "system reboot"
# wait for the router to come back and ssh to answer again
/opt/etc/init.d/S59tftpd status          # expect: tftpd running (pid N) -- a NEW pid, proving rc.unslung started it
/opt/etc/init.d/S82pxehttpd status       # expect: httpd running (pid N) -- likewise a new pid
ndmc -c "show running-config" | grep -A6 'ip dhcp pool'   # expect next-server + bootfile still present
```

Then repeat the physical client boot test from step 5. If the client no
longer reaches the iPXE menu after a reboot that "passed" steps 1-6, the
boot fields did not survive `system configuration save`, or `rc.unslung`
did not start one of the two init scripts — either way this is the failure
mode steps 1-6 cannot see.

*7b. Reboot with the service left `stopped`*

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe \
  -e keenetic_pxe_enabled=true -e keenetic_pxe_service_state=stopped
ndmc -c "system reboot"
# wait for the router to come back
/opt/etc/init.d/S59tftpd status          # expect: tftpd stopped
/opt/etc/init.d/S82pxehttpd status       # expect: httpd stopped
```

This is the only step in the whole checklist that actually proves the
`disabled`-flag mechanism rather than assuming it: `rc.unslung` runs both
scripts' `start` unconditionally on every boot, and `start` is the one place
each script checks `${DISABLED}` before doing anything. If either daemon
comes back running after this reboot, the flag file did not survive, or the
init script's own `[ -f "${DISABLED}" ]` check is not being reached before
the wait/bind logic.
