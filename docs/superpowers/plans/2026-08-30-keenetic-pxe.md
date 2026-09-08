# Keenetic PXE Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the `keenetic` role so a Keenetic router serves an iPXE loader over TFTP and boot images over HTTP from `/opt`, with KeeneticOS's own DHCP server pointing clients at both.

**Architecture:** KeeneticOS keeps DHCP; the role adds two role-owned Entware daemons (`tftpd-hpa` on UDP/69, `lighttpd` on a non-80 port) with hand-written self-contained init scripts modelled on the existing `S24xray`, then writes `next-server` and `bootfile` into the router's DHCP pool over `ndmc`. iPXE is chainloaded from TFTP and picks up a role-templated `autoexec.ipxe`, which presents a menu pointing at the local HTTP server.

**Tech Stack:** Ansible (core ≥ 2.16), Entware `opkg` (`tftpd-hpa` 5.2, `lighttpd` 1.4.82), iPXE, KeeneticOS `ndmc` CLI, POSIX `sh` init scripts.

**Spec:** `docs/superpowers/specs/2026-08-30-keenetic-pxe.md`

## Global Constraints

- Role directory: `/Users/aletunov/git/ansible/roles/keenetic`. All repo-relative paths below are relative to it.
- Scratchpad (test harness lives here, **never committed**): `/private/tmp/claude-501/-Users-aletunov-git-ansible-roles-keenetic/5cbd75cf-c4e8-49d3-963c-6e6ba49a819e/scratchpad`. Referred to below as `$SCRATCH`. Export it once per shell: `export SCRATCH=/private/tmp/claude-501/-Users-aletunov-git-ansible-roles-keenetic/5cbd75cf-c4e8-49d3-963c-6e6ba49a819e/scratchpad`
- `ansible-lint` runs the **production** profile. It must be clean at every commit. Run from the role root with no arguments: `ansible-lint`.
- `yamllint` config is `.yamllint` in the role root. Run `yamllint .` from the role root.
- Variable prefix is `keenetic_`. Task names are `section | action`, lowercase after the pipe. Tags are `keenetic.<section>` — this feature's tag is `keenetic.pxe`.
- Every `include_tasks` in `tasks/main.yml` uses `apply:` with the same tag list it carries, so inner tasks do not inherit a wider set. This is deliberate and documented in the existing file — follow it.
- Shell templates are deployed with `validate: sh -n %s`.
- Init scripts are hand-written, **not** `rc.func` wrappers. `tftpd-hpa` ships no init script at all; `S24xray` documents why the role does not use `rc.func`.
- Do not modify anything under `tasks/xray.yml`, `tasks/preflight.yml`, `tasks/connect.yml`, or `templates/S24xray.j2`. This feature is additive.
- Commit messages use Conventional Commits (`feat:`, `docs:`, `fix:`), matching the repo history.
- The role has no test framework and this plan does not add one. The red/green cycle is a controller-side **render test** kept in `$SCRATCH`, which renders each template with the role's own defaults and validates the output. It never touches a router.

---

## File Structure

| File | Responsibility | Task |
| :--- | :--- | :--- |
| `defaults/main.yml` | PXE variables, appended after the existing xray/user/package blocks | 1, 3, 4 |
| `tasks/pxe.yml` | Everything PXE: asserts, packages, directories, templates, DHCP fields, service convergence | 1–4 |
| `templates/S59tftpd.j2` | Self-contained init script for `tftpd-hpa` | 1 |
| `templates/autoexec.ipxe.j2` | iPXE boot menu, rendered from `keenetic_pxe_menu` | 3 |
| `templates/pxe-httpd.conf.j2` | lighttpd config for the role-owned image server | 4 |
| `templates/S82pxehttpd.j2` | Self-contained init script for that lighttpd instance | 4 |
| `handlers/main.yml` | `pxe | restart tftpd`, `pxe | restart httpd`, `pxe | save router configuration` | 1, 2, 4 |
| `tasks/main.yml` | Gated `pxe` include, `keenetic.pxe` added to the `detect` tag list | 1 |
| `README.md` | Variable rows, facts rows, tag row, gotchas | 1–5 |
| `CHANGELOG.md` | 1.1.0 entry | 5 |
| `$SCRATCH/pxe-render.yml` | Controller-side render test (uncommitted) | 1–4 |

`tasks/pxe.yml` stays one file rather than splitting into preflight/install/config: it is roughly the size of `tasks/xray.yml`, which the role already keeps whole, and splitting it would scatter the service-state reasoning that has to be read in order.

---

### Task 1: TFTP service

Deliverable: the router runs a role-managed `tftpd-hpa` bound to its LAN address, serving `/opt/srv/tftp`, with `started`/`stopped` honoured across reboots.

**Files:**
- Create: `templates/S59tftpd.j2`
- Create: `tasks/pxe.yml`
- Create: `$SCRATCH/pxe-render.yml`
- Modify: `defaults/main.yml` (append)
- Modify: `handlers/main.yml` (append)
- Modify: `tasks/main.yml` (add `keenetic.pxe` to the `detect` tag list; add the `pxe` include after `xray`)
- Modify: `README.md` (variable rows, tag row)

**Interfaces:**
- Produces: variables `keenetic_pxe_enabled`, `keenetic_pxe_service_state`, `keenetic_pxe` (dict with keys `tftp_root`, `http_root`, `conf_dir`, `lan_iface`, `http_port`, `bootfile`, `remap_file`, `log_file`), `keenetic_pxe_next_server`; handler name `pxe | restart tftpd`; init script path `/opt/etc/init.d/S59tftpd` with verbs `start|stop|restart|status` and a `status` line containing the literal string `tftpd running`; disable flag at `{{ keenetic_pxe.conf_dir }}/disabled`.
- Consumes: `ansible_facts` from the existing `detect.yml`.

- [ ] **Step 1: Write the failing render test**

Create `$SCRATCH/pxe-render.yml`:

```yaml
---

- name: render keenetic pxe templates
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    role_dir: /Users/aletunov/git/ansible/roles/keenetic
    out_dir: /tmp/keenetic-pxe-render
    # Stands in for the fact-derived default, which needs a real router.
    keenetic_pxe_next_server: 192.168.1.1

  tasks:

    - name: render | load the role defaults
      ansible.builtin.include_vars:
        file: '{{ role_dir }}/defaults/main.yml'

    - name: render | create the output directory
      ansible.builtin.file:
        path: '{{ out_dir }}'
        state: directory
        mode: '0755'

    - name: render | render the init script
      ansible.builtin.template:
        src: '{{ role_dir }}/templates/S59tftpd.j2'
        dest: '{{ out_dir }}/S59tftpd'
        mode: '0755'

    - name: render | the init script is valid posix sh
      ansible.builtin.command:
        cmd: sh -n {{ out_dir }}/S59tftpd
      changed_when: false

    - name: render | the init script carries the required settings
      ansible.builtin.assert:
        that:
          - "'--secure' in script"
          - "'--user root' in script"
          - "'/opt/srv/tftp' in script"
          - "'192.168.1.1:69' in script"
          - "'tftpd running' in script"
          - "'/opt/etc/pxe/disabled' in script"
        fail_msg: 'S59tftpd rendered without a required setting'
      vars:
        script: "{{ lookup('file', out_dir ~ '/S59tftpd') }}"
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
export SCRATCH=/private/tmp/claude-501/-Users-aletunov-git-ansible-roles-keenetic/5cbd75cf-c4e8-49d3-963c-6e6ba49a819e/scratchpad
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: FAIL on `render | load the role defaults` or on the template task — `keenetic_pxe` is undefined and `templates/S59tftpd.j2` does not exist.

- [ ] **Step 3: Add the variables**

Append to `defaults/main.yml`:

```yaml
keenetic_pxe_enabled: false

# started | stopped -- whether the PXE daemons run, as distinct from
# keenetic_pxe_enabled, which is whether the role manages them at all. Same
# split as keenetic_xray_service_state, and for the same reason: `stopped`
# writes a flag the init scripts honour, so a deliberate stop survives a reboot
# and a later deploy rather than being undone by the boot auto-start.
keenetic_pxe_service_state: started

keenetic_pxe:
  # /opt is the external drive. The TFTP root and the image root live there
  # because the router's own rootfs is a small read-only squashfs.
  tftp_root: /opt/srv/tftp
  http_root: /opt/srv/http
  conf_dir: /opt/etc/pxe
  lan_iface: br0
  # Not 80: the KeeneticOS web UI holds it. Not 8080 blindly either -- check
  # nothing else on the router has taken it before overriding.
  http_port: 8081
  # What the router's DHCP pool hands out as the BOOTP `file` field. UEFI x64 by
  # default; KeeneticOS serves one bootfile per pool and cannot reliably split
  # BIOS from UEFI, so legacy clients need this repointed at undionly.kpxe.
  bootfile: ipxe.efi
  # tftpd-hpa's -m map file. Empty by default: the only thing it is needed for
  # is Windows/WDS boot, which requests paths with backslashes and mixed case.
  # Left as a hook rather than removed, because it is the single capability
  # dnsmasq's built-in TFTP server cannot provide.
  remap_file: ''
  log_file: /opt/var/log/pxe-httpd.log

# The address clients are told to fetch the loader from, and the address both
# daemons bind. Derived from the LAN bridge rather than hardcoded so a router on
# a non-default subnet works untouched. Override with a literal if the bridge
# carries more than one address.
keenetic_pxe_next_server: >-
  {{ ansible_facts[keenetic_pxe.lan_iface].ipv4.address }}
```

- [ ] **Step 4: Write the init script template**

Create `templates/S59tftpd.j2`:

```sh
#!/bin/sh
# tftpd-hpa service for Entware on KeeneticOS. MANAGED BY ANSIBLE -- DO NOT EDIT.
#
# Hand-written rather than an rc.func wrapper, as S24xray is: the tftpd-hpa
# package ships nothing but /opt/sbin/tftpd-hpa -- no init script, no config --
# so there is nothing to wrap, and rc.func's ENABLED flag lives in the script
# itself, which an opkg upgrade would reset.

PATH=/opt/sbin:/opt/bin:/usr/sbin:/usr/bin:/sbin:/bin
export PATH

BIN=/opt/sbin/tftpd-hpa
ROOT={{ keenetic_pxe.tftp_root }}
# Bound to the LAN address, never 0.0.0.0. TFTP is unauthenticated and read-only
# here, but there is no reason for it to answer on the WAN side even behind the
# firewall.
ADDRESS={{ keenetic_pxe_next_server }}:69
# Administrative "stay stopped" marker, from keenetic_pxe_service_state. `stop`
# alone cannot express intent: a stopped daemon is indistinguishable from one
# not yet started, so the boot auto-start and the deploy handler would each undo
# a deliberate stop.
DISABLED={{ keenetic_pxe.conf_dir }}/disabled
PIDFILE=/var/run/tftpd-hpa.pid
{% if keenetic_pxe.remap_file %}
REMAP="-m {{ keenetic_pxe.remap_file }}"
{% else %}
REMAP=""
{% endif %}

log() { logger -t tftpd "$*"; }

running() {
    [ -f "${PIDFILE}" ] || return 1
    kill -0 "$(cat "${PIDFILE}")" 2>/dev/null
}

start() {
    if [ -f "${DISABLED}" ]; then
        log "start refused: ${DISABLED} exists"
        echo "tftpd is administratively disabled (${DISABLED}); not starting"
        return 0
    fi

    if running; then
        echo "tftpd already running (pid $(cat "${PIDFILE}"))"
        return 0
    fi

    [ -x "${BIN}" ] || { log "missing binary ${BIN}"; return 1; }
    # --secure chroots into ROOT, so a missing root is a hard failure rather
    # than a daemon quietly serving nothing. ROOT is on the external drive: this
    # is the unplugged-USB case, and it should be loud.
    [ -d "${ROOT}" ] || { log "missing tftp root ${ROOT}"; return 1; }

    # --user root because tftpd-hpa defaults to "nobody", which is not
    # guaranteed to exist on KeeneticOS. --secure chroots and, with no -c, the
    # daemon is read-only. --foreground so this script owns the pid: tftpd-hpa
    # writes no pidfile of its own.
    #
    # trap '' HUP before exec reimplements nohup, which busybox here does not
    # build -- without it, sshd signalling the process group when the ansible
    # handler's channel closes kills the daemon right after a "successful"
    # deploy. Same mechanism as S24xray.
    # shellcheck disable=SC2086
    ( trap '' HUP; exec "${BIN}" --listen --foreground --secure --user root \
        --address "${ADDRESS}" ${REMAP} "${ROOT}" >/dev/null 2>&1 ) &
    echo $! > "${PIDFILE}"
    sleep 1

    if ! running; then
        log "tftpd failed to stay up"
        rm -f "${PIDFILE}"
        return 1
    fi

    log "started (pid $(cat "${PIDFILE}"))"
    echo "tftpd started"
}

stop() {
    if running; then
        pid=$(cat "${PIDFILE}")
        kill "${pid}" 2>/dev/null
        i=0
        while kill -0 "${pid}" 2>/dev/null && [ "${i}" -lt 10 ]; do
            i=$((i + 1))
            sleep 1
        done
        kill -9 "${pid}" 2>/dev/null
    else
        killall tftpd-hpa 2>/dev/null
    fi
    rm -f "${PIDFILE}"
    log "stopped"
    echo "tftpd stopped"
}

case "$1" in
    start) start ;;
    stop) stop ;;
    restart) stop; start ;;
    status)
        if running; then
            echo "tftpd running (pid $(cat "${PIDFILE}"))"
        else
            echo "tftpd stopped"
        fi
        ;;
    *) echo "Usage: $0 {start|stop|restart|status}"; exit 1 ;;
esac
# No trailing `exit 0`: it would mask start's exit status and always report
# success to the ansible handler. Same reasoning as S24xray.
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: PASS, all tasks `ok`, `render | the init script is valid posix sh` succeeds.

- [ ] **Step 6: Write the task file**

Create `tasks/pxe.yml`:

```yaml
---

- name: pxe | require a valid service state
  ansible.builtin.assert:
    that:
      - keenetic_pxe_service_state in ['started', 'stopped']
    fail_msg: >-
      keenetic_pxe_service_state must be 'started' or 'stopped', got
      {{ keenetic_pxe_service_state | default('undefined') }}. A typo would
      otherwise be read as "not started" and silently disable network boot.

- name: pxe | require a LAN address to advertise
  ansible.builtin.assert:
    that:
      - keenetic_pxe_next_server | default('') | length > 0
      - keenetic_pxe_next_server is match('^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$')
    fail_msg: >-
      keenetic_pxe_next_server resolved to
      '{{ keenetic_pxe_next_server | default("undefined") }}', not an IPv4
      address. It defaults to the address of
      {{ keenetic_pxe.lan_iface }}, so either that bridge is named differently
      on this router or facts were not gathered. Set keenetic_pxe_next_server
      to a literal address for this host.

- name: pxe | install packages
  community.general.opkg:
    name:
      - tftpd-hpa
    state: present

- name: pxe | create directories
  ansible.builtin.file:
    path: '{{ item }}'
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - '{{ keenetic_pxe.conf_dir }}'
    - '{{ keenetic_pxe.tftp_root }}'

- name: pxe | install the tftpd init script
  ansible.builtin.template:
    src: S59tftpd.j2
    dest: /opt/etc/init.d/S59tftpd
    owner: root
    group: root
    mode: '0755'
    validate: sh -n %s
  notify:
    - pxe | restart tftpd

# Ordering matters, as it does for xray: after the init script, so the script
# honouring this flag is in place, and before main.yml's flush_handlers, so a
# restart notified above also sees it. Otherwise a deploy restarts a daemon that
# was stopped on purpose.
- name: pxe | set the administrative service state
  ansible.builtin.file:
    path: '{{ keenetic_pxe.conf_dir }}/disabled'
    state: '{{ "absent" if keenetic_pxe_service_state == "started" else "touch" }}'
    owner: root
    group: root
    mode: '0644'
  # `touch` always reports changed; only absent->present is a real change, and
  # the flag's content is never read.
  changed_when: false

# check_mode: false -- command has no check mode, so --check fabricates empty
# stdout, which reads as "not running" and would make the converge task below
# claim a change on every dry run. Read-only.
- name: pxe | read the current tftpd state
  ansible.builtin.command:
    cmd: /opt/etc/init.d/S59tftpd status
  register: keenetic_pxe_tftpd_status
  changed_when: false
  failed_when: false
  check_mode: false

- name: pxe | converge tftpd to its declared state
  ansible.builtin.command:
    cmd: >-
      /opt/etc/init.d/S59tftpd
      {{ 'start' if keenetic_pxe_service_state == 'started' else 'stop' }}
  changed_when: true
  when: >-
    (keenetic_pxe_service_state == 'started') !=
    ('tftpd running' in keenetic_pxe_tftpd_status.stdout | default(''))
```

- [ ] **Step 7: Add the handler**

Append to `handlers/main.yml`:

```yaml
- name: pxe | restart tftpd
  become: true
  ansible.builtin.command:
    cmd: /opt/etc/init.d/S59tftpd restart
  changed_when: true
```

- [ ] **Step 8: Wire it into main.yml**

In `tasks/main.yml`, add `- keenetic.pxe` to **both** the `apply.tags` list and the outer `tags` list of the `detect` include (it needs `ansible_facts` for the LAN address). Then insert this block after the `xray` include and before the `config` include:

```yaml
- name: pxe
  ansible.builtin.include_tasks:
    file: pxe.yml
    apply:
      tags:
        - keenetic.pxe
  when:
    - keenetic_pxe_enabled | default(false)
  tags:
    - keenetic.pxe
```

- [ ] **Step 9: Lint**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
yamllint . && ansible-lint
```

Expected: both clean, no output beyond the lint summary.

- [ ] **Step 10: Syntax-check the role as a play**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
cat > "$SCRATCH/syntax.yml" <<'EOF'
---
- name: syntax check
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: /Users/aletunov/git/ansible/roles/keenetic
EOF
ansible-playbook --syntax-check "$SCRATCH/syntax.yml"
```

Expected: `playbook: .../syntax.yml` with no errors.

- [ ] **Step 11: Document the variables**

In `README.md`, add these rows to the "Role Variables" table, keeping the existing alphabetical-by-variable ordering within the `keenetic_pxe*` group:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `keenetic_pxe_enabled` | Whether this role manages the PXE daemons at all | `false` |
| `keenetic_pxe_service_state` | `started` \| `stopped` — whether the PXE daemons should be running, distinct from `keenetic_pxe_enabled` | `started` |
| `keenetic_pxe` | Paths, LAN interface, HTTP port, bootfile name and the optional tftpd remap file | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_pxe_next_server` | Address handed to clients and bound by both daemons; defaults to the `keenetic_pxe.lan_iface` address | `192.168.1.1` |

Add to the "Tags" table:

| Tag | Purpose |
| :--- | :--- |
| `keenetic.pxe` | TFTP and HTTP boot services, and the router's DHCP boot fields |

And extend the sentence under the Tags table that lists which tags `detect.yml` carries — it now carries seven, not six.

- [ ] **Step 12: Commit**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
git add defaults/main.yml handlers/main.yml tasks/main.yml tasks/pxe.yml templates/S59tftpd.j2 README.md docs/
git commit -m "feat: add role-managed tftpd-hpa service for network boot"
```

---

### Task 2: Router DHCP boot fields

Deliverable: the router's DHCP pool advertises `next-server` and `bootfile` (plus options 66/67), idempotently, persisted across reboot.

**Files:**
- Modify: `tasks/pxe.yml` (append after the tftpd convergence block)
- Modify: `defaults/main.yml` (append `keenetic_pxe_dhcp_pool`)
- Modify: `handlers/main.yml` (append the save handler)
- Modify: `README.md` (variable row, facts rows, gotcha)

**Interfaces:**
- Consumes: `keenetic_pxe.bootfile` and `keenetic_pxe_next_server` from Task 1.
- Produces: variable `keenetic_pxe_dhcp_pool`; facts `keenetic_pxe_running_config`, `keenetic_pxe_pool_block`; handler name `pxe | save router configuration`.

- [ ] **Step 1: Add the variable**

Append to `defaults/main.yml`:

```yaml
# The KeeneticOS DHCP pool that gets the boot fields. `_WEBADMIN` on a flat
# config; segmented routers use names like `_WEBADMIN_HOME`. Read the real name
# out of `ndmc -c "show running-config"` -- the assertion in pxe.yml fails
# loudly rather than writing into a pool that does not exist. Set to an empty
# string to have the role leave the router's DHCP configuration alone.
keenetic_pxe_dhcp_pool: _WEBADMIN
```

- [ ] **Step 2: Write the failing check**

There is no render test for this task — it produces no template. The red state is the assertion in Step 3 firing against a config that has no such pool. Verify the guard works by rendering the regex against a fixture:

```bash
export SCRATCH=/private/tmp/claude-501/-Users-aletunov-git-ansible-roles-keenetic/5cbd75cf-c4e8-49d3-963c-6e6ba49a819e/scratchpad
cat > "$SCRATCH/pool-regex.yml" <<'EOF'
---
- name: pool block extraction
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    keenetic_pxe_dhcp_pool: _WEBADMIN
    fixture: |
      ip dhcp pool _GUEST
          range 192.168.2.33 192.168.2.154
          bootfile wrong.efi
      !
      ip dhcp pool _WEBADMIN
          range 192.168.1.33 192.168.1.154
          next-server 192.168.1.1
          bootfile ipxe.efi
      !
  tasks:
    - name: extract the pool block
      ansible.builtin.set_fact:
        block_text: >-
          {{ fixture | regex_search('(?ms)^ip dhcp pool ' ~
          keenetic_pxe_dhcp_pool ~ '$.*?^!$') | default('', true) }}

    - name: the block is the right one
      ansible.builtin.assert:
        that:
          - "'192.168.1.33' in block_text"
          - "'bootfile ipxe.efi' in block_text"
          - "'wrong.efi' not in block_text"
        fail_msg: 'pool block regex matched the wrong pool'
EOF
ansible-playbook "$SCRATCH/pool-regex.yml"
```

Expected: PASS. If it fails, the regex is wrong and must be fixed before Step 3 uses it. (This is the guard against the naive substring check, which would see `bootfile wrong.efi` from the guest pool and skip the write.)

- [ ] **Step 3: Append the tasks**

Append to `tasks/pxe.yml`:

```yaml
- name: pxe | configure the router DHCP boot fields
  when: keenetic_pxe_dhcp_pool | default('') | length > 0
  block:

    # check_mode: false -- read-only, and a fabricated empty stdout under
    # --check would make the pool assertion below fail on every dry run.
    - name: pxe | read the router running configuration
      ansible.builtin.command:
        cmd: ndmc -c "show running-config"
      register: keenetic_pxe_running_config
      changed_when: false
      check_mode: false

    # Scoped to the one pool, not a substring search of the whole config: a
    # segmented router has several pools, and `bootfile ipxe.efi` sitting in the
    # guest pool would otherwise read as "already configured" and the real pool
    # would silently never be written.
    - name: pxe | extract the dhcp pool block
      ansible.builtin.set_fact:
        keenetic_pxe_pool_block: >-
          {{ keenetic_pxe_running_config.stdout |
          regex_search('(?ms)^ip dhcp pool ' ~ keenetic_pxe_dhcp_pool ~
          '$.*?^!$') | default('', true) }}

    - name: pxe | require the dhcp pool to exist
      ansible.builtin.assert:
        that:
          - keenetic_pxe_pool_block | length > 0
        fail_msg: >-
          No `ip dhcp pool {{ keenetic_pxe_dhcp_pool }}` block in this router's
          running configuration. The pool is named `_WEBADMIN` on a flat config
          and something like `_WEBADMIN_HOME` once network segments are in use.
          Run `ndmc -c "show running-config"` on the router, find the real name,
          and set keenetic_pxe_dhcp_pool for this host. Writing into a
          non-existent pool would be accepted by ndmc and quietly do nothing.

    # next-server and bootfile are the BOOTP siaddr/file header fields, and they
    # are the ones that actually work: a large share of PXE option ROMs read
    # those and ignore option 66 entirely. Options 66/67 are set as well, for
    # the ROMs that do the reverse. KeeneticOS populates only what it is asked
    # for -- unlike most DHCP servers, it does not mirror one into the other.
    - name: pxe | set the dhcp boot fields
      ansible.builtin.command:
        cmd: >-
          ndmc -c "ip dhcp pool {{ keenetic_pxe_dhcp_pool }}
          {{ item.verb }} {{ item.value }}"
      loop:
        - verb: next-server
          value: '{{ keenetic_pxe_next_server }}'
        - verb: bootfile
          value: '{{ keenetic_pxe.bootfile }}'
        - verb: option 66 ascii
          value: '{{ keenetic_pxe_next_server }}'
        - verb: option 67 ascii
          value: '{{ keenetic_pxe.bootfile }}'
      loop_control:
        label: '{{ item.verb }}'
      when: >-
        (item.verb ~ ' ' ~ item.value) not in keenetic_pxe_pool_block
      changed_when: true
      notify:
        - pxe | save router configuration
```

- [ ] **Step 4: Add the handler**

Append to `handlers/main.yml`:

```yaml
# Without this the boot fields live only in the running configuration and are
# gone after the next reboot or power cut -- network boot then stops working
# with nothing in the role's output to explain it.
- name: pxe | save router configuration
  become: true
  ansible.builtin.command:
    cmd: ndmc -c "system configuration save"
  changed_when: true
```

- [ ] **Step 5: Lint and syntax-check**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
yamllint . && ansible-lint && ansible-playbook --syntax-check "$SCRATCH/syntax.yml"
```

Expected: all clean.

- [ ] **Step 6: Document**

Add to the README "Role Variables" table:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `keenetic_pxe_dhcp_pool` | KeeneticOS DHCP pool that receives the boot fields; empty string leaves the router's DHCP configuration untouched | `_WEBADMIN` |

Add to the "Facts Set by This Role" table:

| Fact | Description |
| :--- | :--- |
| `keenetic_pxe_running_config` | `ndmc -c "show running-config"` output, the boot fields are diffed against it |
| `keenetic_pxe_pool_block` | The single `ip dhcp pool` block extracted from the above, so a sibling pool's settings cannot be mistaken for this one's |
| `keenetic_pxe_tftpd_status` | `S59tftpd status` output, compared against `keenetic_pxe_service_state` |

- [ ] **Step 7: Commit**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
git add defaults/main.yml handlers/main.yml tasks/pxe.yml README.md
git commit -m "feat: set keenetic dhcp next-server and bootfile for pxe"
```

---

### Task 3: iPXE loader and boot menu

Deliverable: `ipxe.efi`, `undionly.kpxe` and a rendered `autoexec.ipxe` sit in the TFTP root, so a client chainloads iPXE and lands on a menu instead of looping.

**Files:**
- Create: `templates/autoexec.ipxe.j2`
- Modify: `defaults/main.yml` (append `keenetic_pxe_loaders`, `keenetic_pxe_menu`)
- Modify: `tasks/pxe.yml` (append)
- Modify: `$SCRATCH/pxe-render.yml` (extend)
- Modify: `README.md`

**Interfaces:**
- Consumes: `keenetic_pxe.tftp_root`, `keenetic_pxe.http_port`, `keenetic_pxe_next_server` from Task 1.
- Produces: variables `keenetic_pxe_loaders` (list of `{name, url, checksum}` where `checksum` is optional) and `keenetic_pxe_menu` (list of `{id, label, key, kernel, initrd, args}`); file `{{ keenetic_pxe.tftp_root }}/autoexec.ipxe`.

- [ ] **Step 1: Extend the render test (red)**

Append these tasks to `$SCRATCH/pxe-render.yml`, and add `keenetic_pxe_menu` to its `vars:` block so the test has content to render:

```yaml
    keenetic_pxe_menu:
      - id: debian
        label: Debian 12 netinstall
        key: d
        kernel: debian-12/linux
        initrd: debian-12/initrd.gz
        args: 'vga=788 --- quiet'
      - id: alpine
        label: Alpine Linux 3.20
        key: a
        kernel: alpine-3.20/vmlinuz-lts
        initrd: alpine-3.20/initramfs-lts
        args: 'modloop=http://192.168.1.1:8081/alpine-3.20/modloop-lts'
```

```yaml
    - name: render | render the ipxe menu
      ansible.builtin.template:
        src: '{{ role_dir }}/templates/autoexec.ipxe.j2'
        dest: '{{ out_dir }}/autoexec.ipxe'
        mode: '0644'

    - name: render | the menu is a valid ipxe script
      ansible.builtin.assert:
        that:
          - menu.startswith('#!ipxe')
          - "'http://192.168.1.1:8081' in menu"
          - "'item --key d debian' in menu"
          - "'item --key a alpine' in menu"
          - "':debian' in menu"
          - "'debian-12/initrd.gz' in menu"
          - "'--- quiet' in menu"
          - "'goto' in menu"
        fail_msg: 'autoexec.ipxe rendered without a required directive'
      vars:
        menu: "{{ lookup('file', out_dir ~ '/autoexec.ipxe') }}"
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: FAIL at `render | render the ipxe menu` — `templates/autoexec.ipxe.j2` does not exist.

- [ ] **Step 3: Add the variables**

Append to `defaults/main.yml`:

```yaml
# Loaders staged into the TFTP root. ipxe.efi is what keenetic_pxe.bootfile
# points at by default; undionly.kpxe is staged alongside so a legacy-BIOS
# client can be served by repointing the bootfile, without a second deploy.
#
# boot.ipxe.org publishes no stable checksums -- upstream rebuilds the binaries
# in place -- so `checksum` is optional per entry. Set it if you mirror the
# files somewhere you control.
keenetic_pxe_loaders:
  - name: ipxe.efi
    url: https://boot.ipxe.org/ipxe.efi
  - name: undionly.kpxe
    url: https://boot.ipxe.org/undionly.kpxe

# Entries rendered into autoexec.ipxe. `kernel` and `initrd` are paths relative
# to keenetic_pxe.http_root; the menu prefixes them with the HTTP base URL.
keenetic_pxe_menu: []
  # - id: debian
  #   label: Debian 12 netinstall
  #   key: d
  #   kernel: debian-12/linux
  #   initrd: debian-12/initrd.gz
  #   args: 'vga=788 --- quiet'
```

- [ ] **Step 4: Write the menu template**

Create `templates/autoexec.ipxe.j2`:

```jinja
#!ipxe
# MANAGED BY ANSIBLE -- DO NOT EDIT.
#
# iPXE fetches autoexec.ipxe from the TFTP server it booted from, before falling
# back to a fresh DHCP request. That is what breaks the chainload loop: without
# it, the DHCP pool hands out ipxe.efi, iPXE re-requests DHCP, and is handed
# ipxe.efi again, forever. No custom-built binary needed.

set base http://{{ keenetic_pxe_next_server }}:{{ keenetic_pxe.http_port }}
set timeout 30000

:start
menu Network boot -- {{ inventory_hostname }}
{% for entry in keenetic_pxe_menu %}
item --key {{ entry.key }} {{ entry.id }} {{ entry.label }}
{% endfor %}
item --gap
item shell iPXE shell
item local Boot from local disk
choose --default local --timeout ${timeout} target || goto local
goto ${target}

{% for entry in keenetic_pxe_menu %}
:{{ entry.id }}
kernel ${base}/{{ entry.kernel }}{% if entry.args is defined and entry.args %} {{ entry.args }}{% endif %}

initrd ${base}/{{ entry.initrd }}
boot || goto failed

{% endfor %}
:shell
shell
goto start

:local
# Hands control back to the next BIOS/UEFI boot device. `exit 1` rather than
# `exit`: exit 0 tells the firmware the boot succeeded and some implementations
# then stop trying anything else.
exit 1

:failed
echo Boot failed, returning to the menu in 5 seconds
sleep 5
goto start
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: PASS. Inspect the output once by hand — `cat /tmp/keenetic-pxe-render/autoexec.ipxe` — and confirm each `kernel` line and its `initrd` line are on separate lines with no blank line between the `kernel` and `initrd` of the same entry.

- [ ] **Step 6: Append the staging tasks**

Append to `tasks/pxe.yml`:

```yaml
- name: pxe | stage the ipxe loaders
  ansible.builtin.get_url:
    url: '{{ item.url }}'
    dest: '{{ keenetic_pxe.tftp_root }}/{{ item.name }}'
    checksum: '{{ item.checksum | default(omit) }}'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ keenetic_pxe_loaders }}'
  loop_control:
    label: '{{ item.name }}'
  register: keenetic_pxe_loader_download
  until: keenetic_pxe_loader_download is succeeded
  retries: 3

- name: pxe | install the ipxe boot menu
  ansible.builtin.template:
    src: autoexec.ipxe.j2
    dest: '{{ keenetic_pxe.tftp_root }}/autoexec.ipxe'
    owner: root
    group: root
    mode: '0644'
```

Note there is deliberately no `notify` on either task: TFTP reads from disk on every request, so new files are live immediately and restarting the daemon would only interrupt a transfer in flight.

- [ ] **Step 7: Lint and syntax-check**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
yamllint . && ansible-lint && ansible-playbook --syntax-check "$SCRATCH/syntax.yml"
```

Expected: all clean.

- [ ] **Step 8: Document**

Add to the README "Role Variables" table:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `keenetic_pxe_loaders` | iPXE binaries staged into the TFTP root: `name`, `url`, optional `checksum` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `keenetic_pxe_menu` | Boot menu entries rendered into `autoexec.ipxe`: `id`, `label`, `key`, `kernel`, `initrd`, optional `args` | Definition example in [defaults/main.yml](defaults/main.yml) |

Add to the "Facts Set by This Role" table:

| Fact | Description |
| :--- | :--- |
| `keenetic_pxe_loader_download` | Result of the iPXE loader downloads, used for its `until` retry |

- [ ] **Step 9: Commit**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
git add defaults/main.yml tasks/pxe.yml templates/autoexec.ipxe.j2 README.md
git commit -m "feat: stage ipxe loaders and render the autoexec boot menu"
```

---

### Task 4: HTTP image service

Deliverable: a role-owned lighttpd serves `/opt/srv/http` on the LAN address and a non-80 port, the stock `S80lighttpd` is disabled, and configured images are staged.

**Files:**
- Create: `templates/pxe-httpd.conf.j2`
- Create: `templates/S82pxehttpd.j2`
- Modify: `defaults/main.yml` (append `keenetic_pxe_images`)
- Modify: `tasks/pxe.yml` (append)
- Modify: `handlers/main.yml` (append)
- Modify: `$SCRATCH/pxe-render.yml` (extend)
- Modify: `README.md`

**Interfaces:**
- Consumes: `keenetic_pxe.http_root`, `keenetic_pxe.http_port`, `keenetic_pxe.log_file`, `keenetic_pxe.conf_dir`, `keenetic_pxe_next_server` from Task 1.
- Produces: variable `keenetic_pxe_images` (list of `{name, url, checksum}`); config `{{ keenetic_pxe.conf_dir }}/httpd.conf`; init script `/opt/etc/init.d/S82pxehttpd` with verbs `start|stop|restart|status` and a `status` line containing `httpd running`; handler `pxe | restart httpd`; fact `keenetic_pxe_httpd_status`.

- [ ] **Step 1: Extend the render test (red)**

Append to `$SCRATCH/pxe-render.yml`:

```yaml
    - name: render | render the httpd config
      ansible.builtin.template:
        src: '{{ role_dir }}/templates/pxe-httpd.conf.j2'
        dest: '{{ out_dir }}/httpd.conf'
        mode: '0644'

    - name: render | the httpd config binds the right address and port
      ansible.builtin.assert:
        that:
          - "'server.port                 = 8081' in conf"
          - "'server.bind                 = \"192.168.1.1\"' in conf"
          - "'/opt/srv/http' in conf"
          - "'mod_dirlisting' in conf"
        fail_msg: 'pxe-httpd.conf rendered without a required setting'
      vars:
        conf: "{{ lookup('file', out_dir ~ '/httpd.conf') }}"

    - name: render | render the httpd init script
      ansible.builtin.template:
        src: '{{ role_dir }}/templates/S82pxehttpd.j2'
        dest: '{{ out_dir }}/S82pxehttpd'
        mode: '0755'

    - name: render | the httpd init script is valid posix sh
      ansible.builtin.command:
        cmd: sh -n {{ out_dir }}/S82pxehttpd
      changed_when: false

    - name: render | the httpd init script carries the required settings
      ansible.builtin.assert:
        that:
          - "'/opt/etc/pxe/httpd.conf' in script"
          - "'httpd running' in script"
          - "'/opt/etc/pxe/disabled' in script"
        fail_msg: 'S82pxehttpd rendered without a required setting'
      vars:
        script: "{{ lookup('file', out_dir ~ '/S82pxehttpd') }}"
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: FAIL at `render | render the httpd config` — the template does not exist.

- [ ] **Step 3: Write the lighttpd config template**

Create `templates/pxe-httpd.conf.j2`:

```jinja
# {{ ansible_managed }}
#
# A role-owned lighttpd instance, deliberately NOT /opt/etc/lighttpd/lighttpd.conf:
# that path is owned by the lighttpd package and an opkg upgrade would restore
# the stock file, which sets no server.port and therefore binds :80 -- the
# KeeneticOS web UI's port.

server.document-root        = "{{ keenetic_pxe.http_root }}"
server.bind                 = "{{ keenetic_pxe_next_server }}"
server.port                 = {{ keenetic_pxe.http_port }}
server.pid-file             = "/var/run/pxe-httpd.pid"
server.errorlog             = "{{ keenetic_pxe.log_file }}"
server.modules              = ( "mod_dirlisting" )

# Directory listings are for debugging: "is the image actually on the router"
# is the first question every failed boot raises, and answering it from a
# browser beats an ssh session.
dir-listing.activate        = "enable"
index-file.names            = ( "index.html" )

include "/opt/etc/lighttpd/mime.conf"

# Anything the mime map does not recognise -- kernels, initrds, squashfs images
# -- is served as a byte stream rather than guessed at.
mimetype.assign            += ( "" => "application/octet-stream" )
```

- [ ] **Step 4: Write the httpd init script template**

Create `templates/S82pxehttpd.j2`:

```sh
#!/bin/sh
# PXE image HTTP service for Entware on KeeneticOS. MANAGED BY ANSIBLE -- DO NOT EDIT.
#
# A second lighttpd instance, separate from the package's S80lighttpd, which the
# role disables: the stock init script is ENABLED=yes and its config sets no
# server.port, so it binds :80 and collides with the router web UI.
#
# Unlike S59tftpd this does not background the daemon itself -- lighttpd
# daemonises and writes its own pidfile, named in the config above.

PATH=/opt/sbin:/opt/bin:/usr/sbin:/usr/bin:/sbin:/bin
export PATH

BIN=/opt/sbin/lighttpd
CONF={{ keenetic_pxe.conf_dir }}/httpd.conf
ROOT={{ keenetic_pxe.http_root }}
# Shared with S59tftpd: one keenetic_pxe_service_state governs both daemons, so
# they cannot drift into a half-running state where clients get a loader but no
# images.
DISABLED={{ keenetic_pxe.conf_dir }}/disabled
PIDFILE=/var/run/pxe-httpd.pid
LOGFILE={{ keenetic_pxe.log_file }}

log() { logger -t pxe-httpd "$*"; }

running() {
    [ -f "${PIDFILE}" ] || return 1
    kill -0 "$(cat "${PIDFILE}")" 2>/dev/null
}

start() {
    if [ -f "${DISABLED}" ]; then
        log "start refused: ${DISABLED} exists"
        echo "pxe httpd is administratively disabled (${DISABLED}); not starting"
        return 0
    fi

    if running; then
        echo "httpd already running (pid $(cat "${PIDFILE}"))"
        return 0
    fi

    [ -x "${BIN}" ] || { log "missing binary ${BIN}"; return 1; }
    [ -f "${CONF}" ] || { log "missing config ${CONF}"; return 1; }
    [ -d "${ROOT}" ] || { log "missing document root ${ROOT}"; return 1; }

    # -tt parses the config and exits. Without it a bad port or a missing
    # include leaves lighttpd dead with the failure only in syslog, and this
    # script would report a successful start.
    if ! "${BIN}" -tt -f "${CONF}" >/dev/null 2>&1; then
        log "config test failed, refusing to start"
        "${BIN}" -tt -f "${CONF}"
        return 1
    fi

    mkdir -p "$(dirname "${LOGFILE}")" 2>/dev/null
    "${BIN}" -f "${CONF}"
    sleep 1

    if ! running; then
        log "httpd failed to stay up"
        return 1
    fi

    log "started (pid $(cat "${PIDFILE}"))"
    echo "httpd started"
}

stop() {
    if running; then
        pid=$(cat "${PIDFILE}")
        kill "${pid}" 2>/dev/null
        i=0
        while kill -0 "${pid}" 2>/dev/null && [ "${i}" -lt 10 ]; do
            i=$((i + 1))
            sleep 1
        done
        kill -9 "${pid}" 2>/dev/null
    fi
    rm -f "${PIDFILE}"
    log "stopped"
    echo "httpd stopped"
}

case "$1" in
    start) start ;;
    stop) stop ;;
    restart) stop; start ;;
    status)
        if running; then
            echo "httpd running (pid $(cat "${PIDFILE}"))"
        else
            echo "httpd stopped"
        fi
        ;;
    *) echo "Usage: $0 {start|stop|restart|status}"; exit 1 ;;
esac
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
ansible-playbook "$SCRATCH/pxe-render.yml"
```

Expected: PASS, including both `sh -n` checks.

- [ ] **Step 6: Add the images variable**

Append to `defaults/main.yml`:

```yaml
# Boot payloads staged into keenetic_pxe.http_root and referenced by
# keenetic_pxe_menu. Downloaded by the router over its own WAN link, not
# proxied through the controller -- these are large files and /opt is where
# they land. `name` may contain slashes to nest them, e.g. debian-12/linux.
#
# Give every entry a checksum: distributions rebuild netboot artefacts in place
# under the same URL, and a silently updated kernel beside a stale initrd fails
# at boot in a way that looks like a network problem.
keenetic_pxe_images: []
  # - name: debian-12/linux
  #   url: https://deb.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux
  #   checksum: sha256:0000000000000000000000000000000000000000000000000000000000000000
```

- [ ] **Step 7: Append the tasks**

Append to `tasks/pxe.yml`:

```yaml
- name: pxe | install the http server package
  community.general.opkg:
    name:
      - lighttpd
    state: present

# The package ships S80lighttpd with ENABLED=yes and a config that sets no
# server.port, so a stock instance binds :80 and fights the KeeneticOS web UI.
# Disabling it here rather than removing the file: opkg owns it, and a removal
# would come back on the next upgrade with ENABLED=yes again.
- name: pxe | disable the packaged lighttpd instance
  ansible.builtin.lineinfile:
    path: /opt/etc/init.d/S80lighttpd
    regexp: '^ENABLED='
    line: ENABLED=no
    owner: root
    group: root
    mode: '0755'
  register: keenetic_pxe_stock_httpd

- name: pxe | stop the packaged lighttpd instance
  ansible.builtin.command:
    cmd: /opt/etc/init.d/S80lighttpd stop
  changed_when: true
  failed_when: false
  when: keenetic_pxe_stock_httpd is changed

- name: pxe | create the image directory
  ansible.builtin.file:
    path: '{{ keenetic_pxe.http_root }}'
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: pxe | install the http server config
  ansible.builtin.template:
    src: pxe-httpd.conf.j2
    dest: '{{ keenetic_pxe.conf_dir }}/httpd.conf'
    owner: root
    group: root
    mode: '0644'
  notify:
    - pxe | restart httpd

- name: pxe | install the http server init script
  ansible.builtin.template:
    src: S82pxehttpd.j2
    dest: /opt/etc/init.d/S82pxehttpd
    owner: root
    group: root
    mode: '0755'
    validate: sh -n %s
  notify:
    - pxe | restart httpd

- name: pxe | stage the boot images
  ansible.builtin.get_url:
    url: '{{ item.url }}'
    dest: '{{ keenetic_pxe.http_root }}/{{ item.name }}'
    checksum: '{{ item.checksum | default(omit) }}'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ keenetic_pxe_images }}'
  loop_control:
    label: '{{ item.name }}'
  register: keenetic_pxe_image_download
  until: keenetic_pxe_image_download is succeeded
  retries: 3

# check_mode: false for the same reason as the tftpd read above.
- name: pxe | read the current httpd state
  ansible.builtin.command:
    cmd: /opt/etc/init.d/S82pxehttpd status
  register: keenetic_pxe_httpd_status
  changed_when: false
  failed_when: false
  check_mode: false

- name: pxe | converge httpd to its declared state
  ansible.builtin.command:
    cmd: >-
      /opt/etc/init.d/S82pxehttpd
      {{ 'start' if keenetic_pxe_service_state == 'started' else 'stop' }}
  changed_when: true
  when: >-
    (keenetic_pxe_service_state == 'started') !=
    ('httpd running' in keenetic_pxe_httpd_status.stdout | default(''))
```

`pxe | stage the boot images` uses `get_url` with nested `name` values, so the parent directory must exist. If any `keenetic_pxe_images` entry contains a slash, add this task immediately before it:

```yaml
- name: pxe | create the image subdirectories
  ansible.builtin.file:
    path: '{{ keenetic_pxe.http_root }}/{{ item.name | dirname }}'
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop: '{{ keenetic_pxe_images }}'
  loop_control:
    label: '{{ item.name }}'
  when: item.name is search('/')
```

- [ ] **Step 8: Add the handler**

Append to `handlers/main.yml`:

```yaml
- name: pxe | restart httpd
  become: true
  ansible.builtin.command:
    cmd: /opt/etc/init.d/S82pxehttpd restart
  changed_when: true
```

- [ ] **Step 9: Lint and syntax-check**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
yamllint . && ansible-lint && ansible-playbook --syntax-check "$SCRATCH/syntax.yml"
```

Expected: all clean.

- [ ] **Step 10: Document**

Add to the README "Role Variables" table:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `keenetic_pxe_images` | Boot payloads staged into `keenetic_pxe.http_root`: `name` (may nest), `url`, `checksum` | Definition example in [defaults/main.yml](defaults/main.yml) |

Add to the "Facts Set by This Role" table:

| Fact | Description |
| :--- | :--- |
| `keenetic_pxe_stock_httpd` | Whether the packaged `S80lighttpd` had to be disabled, used to decide whether to stop it |
| `keenetic_pxe_httpd_status` | `S82pxehttpd status` output, compared against `keenetic_pxe_service_state` |
| `keenetic_pxe_image_download` | Result of the boot image downloads, used for its `until` retry |

- [ ] **Step 11: Commit**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
git add defaults/main.yml handlers/main.yml tasks/pxe.yml templates/pxe-httpd.conf.j2 templates/S82pxehttpd.j2 README.md
git commit -m "feat: serve pxe boot images over a role-owned lighttpd"
```

---

### Task 5: Gotchas, changelog and end-to-end verification

Deliverable: the feature is documented in the same voice as the existing xray entries, the changelog records 1.1.0, and a dry run against a real router is clean.

**Files:**
- Modify: `README.md` (Gotchas section, quick-start note)
- Modify: `CHANGELOG.md`

**Interfaces:**
- Consumes: everything from Tasks 1–4.

- [ ] **Step 1: Write the gotchas**

Add to the README "⚠️ Gotchas" section, after the existing xray bullets:

```markdown
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
```

- [ ] **Step 2: Write the changelog entry**

Insert above the `## 1.0.0` heading in `CHANGELOG.md`:

```markdown
## 1.1.0

### Added

- **Network boot** (`pxe.yml`), opt-in via `keenetic_pxe_enabled`. A
  role-managed `tftpd-hpa` serving `keenetic_pxe.tftp_root` and a role-managed
  `lighttpd` serving `keenetic_pxe.http_root`, both bound to the LAN address and
  both hand-written self-contained init scripts rather than `rc.func` wrappers
  -- `tftpd-hpa` ships no init script at all, and the `lighttpd` package's is
  `ENABLED=yes` on port 80, which collides with the web UI. The packaged
  `S80lighttpd` is disabled rather than removed, because opkg owns the file.
- **Router DHCP boot fields.** `next-server` and `bootfile` written into
  `keenetic_pxe_dhcp_pool` over `ndmc`, alongside options 66/67. Both are set
  because PXE option ROMs are split on which they read, and KeeneticOS -- unlike
  most DHCP servers -- does not mirror one into the other. Idempotency comes
  from extracting the single pool block out of `show running-config`, so a
  sibling pool's settings cannot be mistaken for this one's.
- **iPXE chainloading** with a role-rendered `autoexec.ipxe` menu from
  `keenetic_pxe_menu`. iPXE fetches that file from the TFTP server it booted
  from before re-requesting DHCP, which is what breaks the loop where the pool
  hands out `ipxe.efi` to an iPXE that is already running.
- **A service state separate from role management.**
  `keenetic_pxe_service_state` (`started` / `stopped`) is distinct from
  `keenetic_pxe_enabled`, held in a `disabled` flag file under
  `keenetic_pxe.conf_dir` that both init scripts honour, so a deliberate stop
  survives a reboot and a later deploy. Same shape as the xray pair.
- **The `keenetic.pxe` tag**, added to `detect.yml`'s tag list so a tagged run
  still gathers the facts the LAN address is derived from.
```

- [ ] **Step 3: Lint**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
yamllint . && ansible-lint
```

Expected: clean.

- [ ] **Step 4: Dry run against a real router**

This is the first step that needs hardware. From the inventory that has a Keenetic host:

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe --check --diff \
  -e keenetic_pxe_enabled=true
```

Expected: the play reaches `pxe | require the dhcp pool to exist`. If it **fails** there, that is the useful outcome — read the real pool name out of the failure message's suggested command and set `keenetic_pxe_dhcp_pool` for that host before continuing. Note that `--check` will report the `opkg` and `get_url` tasks as changes without making them.

- [ ] **Step 5: Real run and manual verification**

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe \
  -e keenetic_pxe_enabled=true
```

Then, on the router over ssh:

```bash
/opt/etc/init.d/S59tftpd status          # expect: tftpd running (pid N)
/opt/etc/init.d/S82pxehttpd status       # expect: httpd running (pid N)
ndmc -c "show running-config" | grep -A6 'ip dhcp pool'   # expect next-server + bootfile
opkg install tftp-hpa                    # the client, for the loopback test
tftp -g -r ipxe.efi -l /tmp/ipxe.efi <lan-ip> && ls -l /tmp/ipxe.efi
tftp -g -r autoexec.ipxe -l /tmp/autoexec.ipxe <lan-ip> && cat /tmp/autoexec.ipxe
curl -sI http://<lan-ip>:8081/           # expect: HTTP/1.1 200
```

Then boot one physical client from the network and confirm it reaches the iPXE menu. Record the outcome — if the client gets an address but never starts TFTP, the boot fields are not reaching it and the next thing to check is whether that ROM needs the hex-with-trailing-null form of option 67.

- [ ] **Step 6: Re-run for idempotency**

```bash
ansible-playbook <your-playbook>.yml --limit <router> --tags keenetic.pxe \
  -e keenetic_pxe_enabled=true
```

Expected: `changed=0`. The tasks most likely to report a false change are `pxe | set the dhcp boot fields` (regex not matching the running-config's actual indentation) and `pxe | converge tftpd to its declared state` (status string mismatch). Both are fixable in place; do not accept a permanently-changed run.

- [ ] **Step 7: Commit**

```bash
cd /Users/aletunov/git/ansible/roles/keenetic
git add README.md CHANGELOG.md
git commit -m "docs: document the pxe feature and record 1.1.0"
```

---

## Self-Review Notes

**Spec coverage.** Every spec decision maps to a task: DHCP retained (Task 2, no DHCP daemon anywhere in the plan); `next-server`/`bootfile` (Task 2 Step 3); `tftpd-hpa` (Task 1); role-owned lighttpd with the stock instance disabled (Task 4 Steps 3, 4, 7); `ipxe.efi` + `autoexec.ipxe` (Task 3); local images over HTTP (Task 4 Step 7); templated menu (Task 3 Step 4); one bootfile per pool documented as a limitation (Task 5 Step 1). Non-goals stay absent — no dnsmasq configuration, no BIOS auto-selection, no netboot.xyz mirroring, no Windows remap file beyond the empty variable hook. The three open questions are in the spec, and Task 5 Step 5 records the one that produces evidence.

**Type consistency.** The `status` strings are the load-bearing cross-task contract: `S59tftpd` prints `tftpd running` (Task 1 Step 4) and `pxe | converge tftpd` greps exactly that (Task 1 Step 6); `S82pxehttpd` prints `httpd running` (Task 4 Step 4) and its converge task greps exactly that (Task 4 Step 7). The `disabled` flag path `{{ keenetic_pxe.conf_dir }}/disabled` is written once (Task 1 Step 6) and read by both init scripts. `keenetic_pxe.http_port` appears in the lighttpd config, the iPXE menu's `${base}`, and the render test's fixture strings — all three resolve to 8081.

**Known soft spot.** Task 2's idempotency depends on `ndmc -c "show running-config"` rendering pool settings as `    next-server 192.168.1.1`, with the verb and value adjacent. The `in` test tolerates any leading whitespace, but not a different rendering (say `next-server: 192.168.1.1`). Task 5 Step 6 is where that surfaces, and it surfaces as a permanently-changed run rather than as breakage.
