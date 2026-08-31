# `keenetic`

[![ansible-core](https://img.shields.io/badge/ansible--core-%E2%89%A52.16-black?logo=ansible&logoColor=white)](https://docs.ansible.com/ansible-core/devel/index.html)
[![License](https://img.shields.io/badge/license-GPL--3.0--only-green)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Entware%20%2F%20KeeneticOS-0A6EBD)](#-quick-start)

Configures a Keenetic router: SSH access, package installation, cron jobs,
and an optional xray/TProxy deployment.

## 🚀 Quick Start

Targets Entware on KeeneticOS. `ansible_python_interpreter` must point at
`/opt/bin/python3`, and the role bootstraps that interpreter over `raw` on first
contact if it is missing.

```yaml
- name: keenetic
  hosts: keenetic
  ignore_unreachable: true
  gather_facts: true

  roles:
    - role: keenetic
      tags:
        - keenetic
```

## ⚙️ Role Variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `keenetic_user` | Root user config: `name`, `password`, `authorized_ssh_keys` (public key basenames under `~/.ssh/` on the controller) | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_cron_jobs` | Cron jobs to install via `ansible.builtin.cron` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_packages` | opkg packages installed unconditionally | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_repo_path` | Directory the custom opkg repo `.conf` files are written to | `/opt/etc/opkg` |
| `keenetic_repos` | Extra opkg repos to add: `name`, `src`, `packages` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_xray_enabled` | Whether this role manages xray at all (installs/updates config and binary) | `false` |
| `keenetic_xray_service_state` | `started` \| `stopped` -- whether xray should actually be running, distinct from `keenetic_xray_enabled` | `started` |
| `keenetic_xray_version` | xray version pinned for this router; compared against the installed binary for equality, not substring | `26.7.28` |
| `keenetic_xray` | Paths, TProxy port, fwmark, routing table, LAN interface, log file, corp dummy range and download URL | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_xray_config_src` | Path on the controller to the rendered client profile copied to the router | `{{ inventory_dir }}/files/clients/{{ keenetic_xray_server_name }}-{{ inventory_hostname }}.json` |
| `keenetic_xray_server_name` | Name of the xray server host this router's client profile was generated against | `my-vps` |
| `keenetic_pxe_enabled` | Whether this role manages the PXE daemons at all | `false` |
| `keenetic_pxe_service_state` | `started` \| `stopped` — whether the PXE daemons should be running, distinct from `keenetic_pxe_enabled` | `started` |
| `keenetic_pxe` | Paths, LAN interface, HTTP port, bootfile name and the optional tftpd remap file | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_pxe_next_server` | Address handed to clients and bound by both daemons; defaults to the `keenetic_pxe.lan_iface` address | `192.168.1.1` |
| `keenetic_pxe_dhcp_pool` | KeeneticOS DHCP pool that receives the boot fields; empty string leaves the router's DHCP configuration untouched | `_WEBADMIN` |
| `keenetic_pxe_loaders` | iPXE binaries staged into the TFTP root: `name`, `url`, optional `checksum` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_pxe_menu` | Boot menu entries rendered into `autoexec.ipxe`: `id`, `label`, `key`, `kernel`, `initrd`, optional `args` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_pxe_images` | Boot payloads staged into `keenetic_pxe.http_root`: `name` (may nest), `url`, `checksum` | Definition example in [defaults/main.yml](defaults/main.yml) |

`keenetic_user.authorized_ssh_keys` names public keys under `~/.ssh` **on the
controller**, without the `.pub` suffix — [tasks/connect.yml](tasks/connect.yml)
appends it. `id_ed25519` reads `~/.ssh/id_ed25519.pub`.

## 🔍 Facts Set by This Role

<details>
<summary><b>All 25 rows</b> — including the two <code>ansible_*</code> connection variables the role rewrites</summary>

| Fact | Description |
| :--- | :--- |
| `keenetic_ssh_port` | Set by `connect.yml` when the router is reached on a non-default port; used to render `dropbear.j2` and to move `ansible_port` |
| `keenetic_default_ssh_port` | Result of the port-22 probe, used to decide whether the router is still on the stock port |
| `keenetic_custom_ssh_port` | Result of the `keenetic_ssh_port` probe, used to decide whether `ansible_port` may move onto it |
| `keenetic_check_connection_result` | Result of the initial `ping`, used to pick the bootstrap credentials |
| `keenetic_controller_ssh_keys` | Public key material read off the controller, rendered into `authorized_keys.j2` |
| `keenetic_python3_bootstrap` | Result of the `raw` python3 bootstrap, used only for its `changed_when` |
| `keenetic_root_password` | `keenetic_user.password`, held `no_log` for the `user` module to hash |
| `keenetic_update_result` | Result of `opkg update` in the `repos \| update cache` handler, which tolerates rc 1 |
| `keenetic_opkg_architecture` | Raw `opkg print-architecture` output; authoritative because busybox `uname -m` cannot tell mips from mipsel |
| `keenetic_xray_arch` | Derived from the above; used to build the xray download URL in `keenetic_xray.repo` |
| `keenetic_xt_tproxy` | Whether the firmware ships `xt_TPROXY.ko` for the running kernel |
| `keenetic_rci_http` | The `/rci/ip/http` response the port-443 precondition is read from |
| `keenetic_https_on_443` | Whether the web UI still holds 443, which TProxy needs free |
| `keenetic_xray_installed` / `keenetic_xray_installed_version` | `xray version` output and the version token extracted from it |
| `keenetic_xray_stage` / `keenetic_xray_download` | Controller-side staging directory and release download |
| `keenetic_xray_profile` | Whether `keenetic_xray_config_src` has been generated yet |
| `keenetic_xray_status` | `S24xray status` output, compared against `keenetic_xray_service_state` |
| `keenetic_pxe_running_config` | `ndmc -c "show running-config"` output, the boot fields are diffed against it |
| `keenetic_pxe_pool_block` | The single `ip dhcp pool` block extracted from the above, so a sibling pool's settings cannot be mistaken for this one's |
| `keenetic_pxe_tftpd_status` | `S59tftpd status` output, compared against `keenetic_pxe_service_state` |
| `keenetic_pxe_loader_download` | Result of the iPXE loader downloads, used for its `until` retry |
| `keenetic_pxe_httpd_status` | `S82pxehttpd status` output, compared against `keenetic_pxe_service_state` |
| `keenetic_pxe_image_download` | Result of the boot image downloads, used for its `until` retry |
| `ansible_port` | Rewritten to 22 while bootstrapping, then to `keenetic_ssh_port` once dropbear has moved |
| `ansible_password` | Rewritten to the stock password when the first connection is refused |

</details>

## 🏷 Tags

| Tag | Purpose |
| :--- | :--- |
| `keenetic.cron` | Cron job management |
| `keenetic.packages` | Base opkg package installation |
| `keenetic.repo` | Custom opkg repo setup |
| `keenetic.ssh` | Dropbear config, authorized_keys, ssh port detection/change |
| `keenetic.user` | Root password and connection bootstrap |
| `keenetic.xray` | Preflight checks and xray install/config/service state |
| `keenetic.pxe` | TFTP and HTTP boot services, and the router's DHCP boot fields |

`detect.yml` carries all seven tags, so any single tag still runs the fact
gathering it depends on — `preflight.yml` reads `ansible_facts.kernel` for the
`xt_TPROXY` path, which is why `keenetic.xray` is in that list too.

`connect.yml` carries only `keenetic.ssh` and `keenetic.user`. A `keenetic.cron`,
`keenetic.packages`, `keenetic.repo`, `keenetic.xray` or `keenetic.pxe` run
therefore does **not** re-run the connection bootstrap, and relies on
`ansible_port` and the credentials in inventory already being correct for the
router as it stands.
That is deliberate: the bootstrap changes the root password and rewrites
dropbear's config, which no other tag should imply.

`keenetic.xray` additionally requires `keenetic_xray_enabled: true` — the
`preflight` and `xray` includes are gated on it, so a tagged run against a host
with it `false` correctly does nothing.

Tags do not discriminate *within* `install.yml`. `Taggable.tags` is
`extend=True`, so its tasks inherit both of the include's tags on top of their
own, and `--tags keenetic.packages` and `--tags keenetic.repo` each run the whole
file. This is the same shape as `flyoverhead.server`'s `packages` include and is
left alone for consistency with it; the per-task tags there document intent
rather than gate execution.

## ⚠️ Gotchas

- Stop xray with `keenetic_xray_service_state: stopped`, never a bare
  `S24xray stop`. The service state is written to a flag file that survives
  reboots, deploys and the geofile cron; stopping it out-of-band leaves that
  flag out of sync and a later restart handler can bring xray back up
  unexpectedly.
- While xray is running, the router proxies its **entire LAN** via tproxy, so
  taking it down (or leaving it down when it should be up) affects every
  client on the network, not just this host.
- `preflight.yml` must run before `xray.yml` -- the xray download URL depends
  on the `keenetic_xray_arch` fact `preflight.yml` sets -- and both must run
  after `install.yml`, which supplies `ca-certificates` (needed by xray's
  `get_url`), `iptables` and `ip` (needed by the netfilter hook).
- `xray.yml` will not render a client profile for you. It asserts
  `keenetic_xray_config_src` exists and tells you the exact command to generate
  it, which needs **both** `--tags xray.clients,xray.tuning` against the xray
  server host.
- Stop the PXE daemons with `keenetic_pxe_service_state: stopped`, never a bare
  `S59tftpd stop`. Both init scripts read one flag file under
  `keenetic_pxe.conf_dir`; stopping either out-of-band leaves that flag out of
  sync and the next deploy or reboot brings the daemon back.
- `dnsmasq-full` is in `keenetic_packages` on every host, it ships
  `/opt/etc/init.d/S56dnsmasq` with `ENABLED=yes`, and its packaged
  `dnsmasq.conf` is entirely comments -- so a running instance answers DNS on
  :53 beside `ndnsproxy`. This role does not manage it. Check
  `/opt/etc/init.d/S56dnsmasq check` before assuming the router's DNS path is
  what you think it is, and do not build PXE on that daemon: without `port=0`
  and with any `dhcp-range`, it competes with the router's own DHCP server for
  the whole LAN.
- The `lighttpd` package likewise ships `S80lighttpd` with `ENABLED=yes` and a
  config that sets no `server.port`, so a stock instance binds :80 -- the web
  UI's port. `pxe.yml` sets `ENABLED=no` there and runs its own instance from
  `keenetic_pxe.conf_dir/httpd.conf`. Do not re-enable it.
- `next-server` and `bootfile` are the fields that make network boot work, not
  options 66/67. Many PXE option ROMs read the BOOTP `siaddr`/`file` header
  fields and ignore option 66 entirely, and KeeneticOS populates only what it is
  asked for. The role sets all four.
- Router CLI changes made through `ndmc` live in the running configuration only.
  The `pxe | save router configuration` handler runs
  `system configuration save`; if it is skipped, the boot fields survive until
  the next reboot and then silently vanish.
- One `bootfile` per DHCP pool means one client architecture. The default is
  UEFI x64 (`ipxe.efi`). `undionly.kpxe` is staged alongside it, but serving
  both from one pool needs DHCP class matching on option 60, which is untested
  on KeeneticOS.
- TFTP and the image HTTP server bind `keenetic_pxe_next_server`, which defaults
  to the `br0` address. Neither is authenticated. On a router whose LAN bridge
  carries more than one address, set the variable explicitly rather than letting
  it pick.
- Both roots live under `/opt`, the external drive. An unplugged stick means
  `S59tftpd` and `S82pxehttpd` refuse to start rather than serving an empty
  tree -- deliberately loud.
- `pxe.yml`'s DHCP boot-field write is only as idempotent as `ndmc`'s own
  rendering. It compares each `next-server`/`bootfile`/option 66/option 67
  value against `show running-config` assuming that output is unquoted and
  sits directly after the verb (`option 66 ascii 192.168.1.1`); if a firmware
  version quotes option values instead, those two loop items will report
  `changed` on every run even though the router already has them set.

<details>
<summary><b>Check mode</b> — what <code>--check --diff</code> covers, the six probes that opt out of it, and two things it cannot tell you</summary>

`--check --diff` reports drift in `dropbear.conf`, `authorized_keys`, the opkg
repo files, the cron jobs, the xray config, the netfilter hook and `S24xray`.

Six probes carry `check_mode: false`, because they only read and later tasks
branch on their output. Left to be skipped, each would fabricate a result that
reads as a definite answer rather than "unknown":

- The two `wait_for` port checks in [tasks/connect.yml](tasks/connect.yml).
  `wait_for` declares no check mode support, and the `when` beneath each reads
  `.msg | default("")`, so a skip resolves to "the port answered" — forcing
  `ansible_port` to 22 on a router whose sshd is elsewhere.
- `opkg print-architecture` and the `/rci/ip/http` GET in
  [tasks/preflight.yml](tasks/preflight.yml). `command` fabricates rc 0 with
  empty stdout under `--check`, which makes `keenetic_xray_arch` resolve to
  `unknown`; `uri` is skipped outright before the module runs, which would
  silently `when`-skip the port-443 assertion and report a green play with its
  single most consequential precondition unchecked.
- `xray version` and `S24xray status` in [tasks/xray.yml](tasks/xray.yml),
  which otherwise read as "nothing installed" and "not running" and make every
  dry run claim a binary install and a service change.

Two things a check run cannot tell you:

- **It does not work against a router that has not been bootstrapped yet.**
  The python3 bootstrap is `ansible.builtin.raw`, which is skipped under
  `--check`, and every module task after it needs the interpreter that task
  installs. Run the role for real once first.
- **The service state is reported, not converged.** `xray | set the
  administrative service state` uses `touch`, which reports `changed` on every
  real run by design and so carries `changed_when: false`; the flag file
  therefore appears unchanged in a check run whatever the declared state is.
  Read `xray | converge the service to its declared state` instead — that is
  the task that would act.

The preflight assertions do run, so a check against a bootstrapped aarch64
router is a genuine way to verify the `xt_TPROXY` and port-443 preconditions
without touching anything.

</details>

## 📄 License

GPL-3.0-only

## 👤 Author Information

fLy0v3rH34d
