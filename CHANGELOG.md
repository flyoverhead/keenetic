# Changelog

All notable changes to `keenetic`.

## 1.0.0

Initial release. A standalone role rather than a `flyoverhead.*` collection,
because it targets one appliance family and has no siblings to group with, but
it follows the same conventions as those collections: prefixed variables,
`section | action` task names, `<role>.<section>` tags and a production-profile
`ansible-lint` clean.

Targets Entware on KeeneticOS. `ansible_python_interpreter` must point at
`/opt/bin/python3`, and the role bootstraps that interpreter over `raw` on first
contact if it is missing.

### Added

- **Connection bootstrap** (`connect.yml`). Probes port 22 and
  `keenetic_ssh_port`, moves `ansible_port` onto the custom port once dropbear
  has been reconfigured, falls back to the stock password when the first
  connection is refused, sets the root password, and installs `authorized_keys`
  from public keys read off the controller. `keenetic_user.authorized_ssh_keys`
  names keys under `~/.ssh` **without** the `.pub` suffix -- the role appends it.
- **Packages and repositories** (`install.yml`). Unconditional opkg packages
  from `keenetic_packages`, plus extra repositories from `keenetic_repos`
  written as `.conf` files under `keenetic_repo_path`. The `repos | update
  cache` handler tolerates `opkg update` returning rc 1.
- **Cron jobs** (`config.yml`) via `ansible.builtin.cron`, from
  `keenetic_cron_jobs`.
- **Architecture detection** (`preflight.yml`). `opkg print-architecture` is
  authoritative rather than `uname -m`, because busybox `uname` cannot
  distinguish mips from mipsel. The result drives the xray download URL.
- **xray over TProxy** (`xray.yml`), opt-in via `keenetic_xray_enabled`.
  Version-pinned binary compared against the installed one for equality rather
  than substring, a netfilter hook, and preflight assertions for `xt_TPROXY.ko`
  and for port 443 being free of the web UI.
- **A service state separate from role management.**
  `keenetic_xray_service_state` (`started` / `stopped`) is distinct from
  `keenetic_xray_enabled`, and is held in a `disabled` flag file under
  `keenetic_xray.conf_dir` -- present means stopped -- which survives reboots,
  deploys and the geofile cron. Stop xray this way rather than with a bare
  `S24xray stop`, which leaves the flag out of sync and lets a later restart
  handler bring xray back up.
- **Check mode support.** `--check --diff` reports drift in `dropbear.conf`,
  `authorized_keys`, the opkg repo files, the cron jobs, the xray config, the
  netfilter hook and `S24xray`. Six read-only probes carry `check_mode: false`
  so that a skipped module cannot fabricate a result that reads as a definite
  answer -- the two `wait_for` port checks, `opkg print-architecture`, the
  `/rci/ip/http` GET, `xray version` and `S24xray status`.
- **Tags.** `keenetic.cron`, `keenetic.packages`, `keenetic.repo`,
  `keenetic.ssh`, `keenetic.user` and `keenetic.xray`. `detect.yml` carries all
  six so any single tag still runs the fact gathering it depends on;
  `connect.yml` carries only `keenetic.ssh` and `keenetic.user`, so no other
  tagged run implies the credential and dropbear rewrite.
- **README** with status badges, the role variables, the facts the role sets,
  the tag semantics, the check-mode analysis and the operational warnings.

### Known issues

- **Check mode does not work against a router that has not been bootstrapped
  yet.** The python3 bootstrap is `ansible.builtin.raw`, which is skipped under
  `--check`, and every module task after it needs the interpreter that task
  installs. Run the role for real once first.
- **The xray service state is reported, not converged, in a check run.** `xray |
  set the administrative service state` uses `touch` and carries
  `changed_when: false`, so the flag file appears unchanged whatever the
  declared state is. `xray | converge the service to its declared state` is the
  task that would act.
- **Tags do not discriminate within `install.yml`.** `Taggable.tags` is
  `extend=True`, so its tasks inherit the include's tags on top of their own and
  `--tags keenetic.packages` and `--tags keenetic.repo` each run the whole file.
  Left as-is for consistency with `flyoverhead.server`'s `packages` include; the
  per-task tags there document intent rather than gate execution.
- **`xray.yml` will not render a client profile.** It asserts
  `keenetic_xray_config_src` exists and prints the command that generates it,
  which needs `--tags xray.clients,xray.tuning` against the xray server host.
