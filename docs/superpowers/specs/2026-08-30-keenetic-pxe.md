# Spec: PXE / netboot server on Keenetic

Research artifact (full findings and sources):
<https://claude.ai/code/artifact/e38a5400-e3b7-41e4-be5d-6a76bd31b51a>

## Goal

Extend `flyoverhead.keenetic` so a Keenetic router can network-boot Linux
clients on its LAN: serve an iPXE loader over TFTP, serve kernels/initrds/ISOs
over HTTP from `/opt`, and point KeeneticOS's own DHCP server at both.

## Decisions

| Decision | Choice | Why |
| :--- | :--- | :--- |
| DHCP | KeeneticOS `ndhcps` keeps it | An Entware DHCP server on a USB stick owns LAN addressing; a dead stick means no LAN. Same blast-radius class as the xray tproxy gotcha, one layer lower. |
| Boot pointer | `next-server` + `bootfile`, **not** options 66/67 | Many PXE option ROMs read the BOOTP `siaddr`/`file` header fields and ignore option 66. KeeneticOS fills only what you ask for. Options 66/67 are set as well, for ROMs that read those instead. |
| TFTP daemon | `tftpd-hpa` | Ships only `/opt/sbin/tftpd-hpa` — no init script, no config, no DNS/DHCP surface. The role owns the service outright, as it already does for xray. |
| HTTP daemon | `lighttpd`, role-owned instance | Stock `S80lighttpd` is `ENABLED=yes` with no `server.port`, so it binds :80 — the router web UI's port. The role disables it and runs its own config. |
| Loader | upstream `ipxe.efi` + `autoexec.ipxe` | `autoexec.ipxe` in the TFTP root is fetched automatically by iPXE and breaks the chainload loop without a custom-built binary. |
| Images | staged locally under `/opt`, served over HTTP | Chosen over internet-pulled netboot.xyz: works with no WAN at boot. |
| Menu | role-templated `autoexec.ipxe` from a variable | Avoids mirroring netboot.xyz's menu layer; the boot menu becomes role data. |
| Architectures | one `bootfile` per pool | KeeneticOS cannot reliably split BIOS/UEFI (see open question below). Default is UEFI x64. |

## Client scope

Linux only. `tftpd-hpa`'s `-m` remap file — the one thing that makes Windows
/ WDS boot possible — stays available as an empty-by-default variable, but no
Windows support is built or tested.

## Non-goals

- Replacing or proxying the KeeneticOS DHCP server (Options B and C in the
  research artifact — both rejected).
- Legacy BIOS clients as a first-class target. `undionly.kpxe` is staged so a
  pool can be repointed by hand, but nothing auto-selects it.
- Mirroring netboot.xyz's hosted menu tree.
- Windows / WDS deployment.

## Constraints

- `ansible-lint` production profile must stay clean; `.ansible-lint` and
  `.yamllint` in the repo root are authoritative.
- Conventions follow the existing role exactly: `keenetic_*` variable prefix,
  `section | action` task names, `keenetic.<section>` tags, `apply:` on
  `include_tasks` so inner tasks do not inherit a wider tag set.
- Service management mirrors the xray pair: `keenetic_pxe_enabled` (does the
  role manage this at all) is distinct from `keenetic_pxe_service_state`
  (`started` / `stopped`), and the stopped intent is held in a flag file the
  init script honours, so it survives reboots.
- Init scripts are hand-written and self-contained, not `rc.func` wrappers —
  the same choice `S24xray` already documents.
- Every shell template is deployed with `validate: sh -n %s`.

## Open questions (verify on hardware; none block the build)

1. Is `S56dnsmasq` currently running on the fleet? `dnsmasq-full` is in
   `keenetic_packages` and auto-starts with an all-comments config, so it may
   already be answering DNS on :53 beside `ndnsproxy`. Out of scope for this
   change, but worth knowing.
2. Does `ip dhcp class` prefix-match option 60? If yes, a later change can
   serve BIOS and UEFI from one pool.
3. What does `GET /rci/ip/dhcp/pool` return? If it is usable, the
   `show running-config` parsing in Task 2 can be replaced with an RCI read.
