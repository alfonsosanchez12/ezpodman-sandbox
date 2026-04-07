# ezpodman-sandbox — Architecture & Design

This document explains how the automation is structured, the Ansible concepts
it uses, and the reasoning behind every significant decision. It assumes no
prior Ansible knowledge.

---

## Table of Contents

1. [What this project does](#1-what-this-project-does)
2. [How the pieces fit together](#2-how-the-pieces-fit-together)
3. [Ansible core concepts used here](#3-ansible-core-concepts-used-here)
4. [Inventory — telling Ansible what exists](#4-inventory--telling-ansible-what-exists)
5. [The connection plugin — no SSH needed](#5-the-connection-plugin--no-ssh-needed)
6. [Variable hierarchy — where vars live and why](#6-variable-hierarchy--where-vars-live-and-why)
7. [Roles — reusable units of work](#7-roles--reusable-units-of-work)
8. [Playbooks — orchestrating the roles](#8-playbooks--orchestrating-the-roles)
9. [Idempotency — safe to re-run](#9-idempotency--safe-to-re-run)
10. [Rootless Podman — why a dedicated user](#10-rootless-podman--why-a-dedicated-user)
11. [Teardown design — nuke.yml](#11-teardown-design--nukeyml)

---

## 1. What this project does

The sandbox spins up two virtual machines inside an Incus project on a remote
homelab server, installs software on them, and starts test containers — all
with a single series of `ansible-playbook` commands from your control node (Mac or Linux).

```
Your control node (Mac or Linux)
│
│  ansible-playbook provision.yml   ← talks to Incus API via incus CLI
│  ansible-playbook setup.yml       ← execs into VMs via incus connection plugin
│  ansible-playbook containers_up.yml
│
▼
Incus server (your-server.home.lan)
└── project: ezpodman-sandbox
    ├── ezpodman-local (Fedora 43)
    │   ├── podman (rootless) + podman-remote
    │   ├── lazydocker, ezpodman, go, jq, fzf ...
    │   └── containers: nginx (:8080), caddy (:8081)
    └── podman-remote (Debian Trixie)
        ├── podman (rootless) + podman.socket + sshd
        └── containers: nginx (:8080), postgres (:5432)
```

The point of the sandbox is to test `ezpodman` — specifically its ability to
connect from `ezpodman-local` to `podman-remote` over SSH and manage containers
on the remote host, just like lazydocker does locally.

---

## 2. How the pieces fit together

```
ezpodman-sandbox/
├── ansible.cfg              ← tells Ansible where inventory and roles are
├── inventory/
│   ├── hosts.ini            ← what hosts exist and how to connect
│   └── group_vars/          ← variables scoped to groups of hosts
│       ├── all.yml          ← applies to every host (incus_remote, etc.)
│       ├── local.yml        ← applies to ezpodman-local only
│       └── remote.yml       ← applies to podman-remote only
├── playbooks/               ← the scripts you actually run
│   ├── provision.yml        ← creates VMs
│   ├── setup.yml            ← installs software
│   ├── containers_up.yml    ← starts containers
│   ├── containers_down.yml  ← stops containers
│   └── nuke.yml             ← destroys everything
└── roles/                   ← reusable task libraries
    ├── incus_project/       ← VM lifecycle
    ├── podman_setup/        ← rootless Podman
    └── test_containers/     ← container management
```

**The flow:** `ansible.cfg` sets the inventory and roles path so every
`ansible-playbook` command you run from the project root finds everything
automatically — no `-i` flags needed.

---

## 3. Ansible core concepts used here

### Control node vs managed nodes

- **Control node** — your Mac or Linux machine. Ansible runs here. Nothing is
  installed on it except Ansible and the `incus` CLI.
- **Managed nodes** — the VMs (`ezpodman-local`, `podman-remote`). Ansible
  reaches them and runs tasks on them.

### Tasks

A task is one unit of work: install a package, create a user, start a service.
Tasks call **modules** — built-in functions Ansible knows how to run remotely.

```yaml
- name: Install Podman          # human-readable description
  ansible.builtin.dnf:          # the module (dnf package manager)
    name: podman                # module parameter
    state: present              # desired state
```

### Modules

Modules are what actually do the work. Examples used in this project:

| Module | What it does |
|--------|-------------|
| `ansible.builtin.dnf` | Installs packages on Fedora/RedHat |
| `ansible.builtin.apt` | Installs packages on Debian |
| `ansible.builtin.user` | Creates/manages OS users |
| `ansible.builtin.command` | Runs a shell command |
| `ansible.builtin.raw` | Runs a command with no Python required |
| `ansible.builtin.get_url` | Downloads a file from a URL |
| `ansible.builtin.unarchive` | Extracts a tar/zip archive |
| `ansible.builtin.systemd` | Manages systemd services |
| `ansible.builtin.lineinfile` | Ensures a line exists in a file |
| `ansible.builtin.uri` | Makes HTTP requests (used for GitHub API) |
| `containers.podman.podman_container` | Manages Podman containers |

### Plays and playbooks

A **play** maps a set of tasks to a set of hosts. A **playbook** is a file
containing one or more plays.

```yaml
- name: Install software on all VMs   # this is a play
  hosts: all                           # which hosts to target
  tasks:
    - ...

- name: Extra setup on ezpodman-local  # second play in the same playbook
  hosts: local
  tasks:
    - ...
```

`setup.yml` uses exactly this pattern: Play 1 runs on all VMs (Podman setup),
Play 2 runs only on `ezpodman-local` (the Fedora toolchain).

### Roles

A role is a reusable, self-contained bundle of tasks with its own defaults and
structure. Instead of writing the same tasks in every playbook, you write a
role once and call it from multiple playbooks.

```yaml
roles:
  - role: podman_setup   # this calls roles/podman_setup/tasks/main.yml
```

---

## 4. Inventory — telling Ansible what exists

`inventory/hosts.ini` defines two groups:

```ini
[local]
ezpodman-local

[remote]
podman-remote

[all:vars]
ansible_connection=community.general.incus
```

**Groups matter** because `group_vars/local.yml` is automatically applied to
hosts in `[local]`, and `group_vars/remote.yml` to hosts in `[remote]`. This
is how Ansible knows to install `openssh-server` and set the user password on
`podman-remote` but not on `ezpodman-local` — the condition in `podman_setup` checks
`inventory_hostname in groups['remote']`.

`group_vars/all.yml` applies to every host, including `localhost`. That's how
`provision.yml` (which runs on localhost) picks up `incus_remote` and
`incus_project` without you having to repeat them.

---

## 5. The connection plugin — no SSH needed

By default, Ansible connects to managed nodes over SSH. This project uses a
different connection plugin: `community.general.incus`.

**Why:** The VMs live on the internal Incus bridge — they have no LAN IP and
are not reachable by SSH from your control node. The `incus` connection plugin bypasses
the network entirely by using `incus exec`, which is roughly equivalent to
`docker exec` — it runs commands inside the VM via the hypervisor.

Under the hood, every task Ansible runs translates to:

```bash
incus exec <remote>:ezpodman-local --project ezpodman-sandbox -- /bin/sh -c "<task>"
```

The plugin reads two variables to know which server and project to target:

```yaml
ansible_incus_remote: "{{ incus_remote }}"   # passed at runtime via -e "incus_remote=<remote>"
ansible_incus_project: ezpodman-sandbox
```

**Why named remotes over URLs:** The `incus` CLI already knows how to reach
`<remote>` (URL, TLS cert, key) from the `incus remote add` setup you did once.
The connection plugin reuses that config — no need to store cert paths in
Ansible variables.

**Local execution:** If you run Ansible directly on the Incus host (no separate
control node), pass `incus_remote=local`. `local` is a built-in Incus remote
that points to the local daemon — no TLS setup required. Every CLI command and
the connection plugin handle it identically to a named remote:

```bash
# Running ansible-playbook on the Incus host itself:
ansible-playbook playbooks/provision.yml -e "incus_remote=local"
```

**`provision.yml` is different:** It runs on `localhost` and uses the `incus`
CLI directly via `ansible.builtin.command`. This is because `provision.yml`
creates the VMs — the VMs don't exist yet, so the connection plugin has nothing
to connect to.

---

## 6. Variable hierarchy — where vars live and why

Ansible has a strict variable precedence order (higher = wins). This project
uses three levels:

### Role defaults (`roles/*/defaults/main.yml`) — lowest priority

Sensible fallbacks. Easy to override. Example:

```yaml
# roles/incus_project/defaults/main.yml
storage_pool: default
network_bridge: incusbr0
sandbox_user: podman
```

These are the values used unless something higher up overrides them. You can
change `storage_pool` for a single run with `-e "storage_pool=local"` without
touching any file.

### Group vars (`inventory/group_vars/`) — mid priority

Variables scoped to a group of hosts. This is where host-specific config
belongs:

- `all.yml` — things every host needs (`incus_project`, connection plugin
  vars). `incus_remote` has no default here — it must be passed at runtime.
- `local.yml` — container definitions for `ezpodman-local`
- `remote.yml` — container definitions for `podman-remote`

Container definitions live in `group_vars` (not in the role) because they
describe *what to run on which host*, which is inventory knowledge — not
something the role should decide.

### Extra vars (`-e` at runtime) — highest priority

Overrides everything. Used to change the target remote or storage pool without
editing any file:

```bash
# Named remote (control node → Incus server over TLS):
ansible-playbook provision.yml -e "incus_remote=<remote>"

# Local Incus daemon (running directly on the Incus host):
ansible-playbook provision.yml -e "incus_remote=local"
```

---

## 7. Roles — reusable units of work

### `incus_project` — VM lifecycle

**Used by:** `provision.yml`

**What it does:**
1. Verifies the target remote is reachable — either a named remote configured
   on your control node, or `local` when running directly on the Incus host
2. Creates the `ezpodman-sandbox` Incus project if it doesn't exist
3. Launches both VMs if they don't exist
4. Waits until both VMs respond to `incus exec`

**Key design decision — CLI over Ansible modules:**
The `community.general.incus_instance` module exists, but it requires you to
pass the remote server URL and TLS cert paths directly. Since you already have
named remotes configured (`incus remote add <remote> ...`), using the `incus`
CLI is simpler — it reads your local config and handles auth transparently.

**`check_mode: false` on read tasks:**
Ansible's `--check` (dry-run) mode skips all `command` tasks by default,
because they could be destructive. But some command tasks are purely
informational (list projects, list VMs). If those are skipped, subsequent
tasks that depend on their output fail. The fix is `check_mode: false` on
read-only tasks — it forces them to run even in dry-run mode.

```yaml
- name: List projects on remote
  ansible.builtin.command:
    cmd: "incus project list {{ incus_remote }}: --format csv"
  register: existing_projects
  changed_when: false    # listing never changes state
  check_mode: false      # always run, even during --check
```

### `podman_setup` — rootless Podman

**Used by:** `setup.yml`, `nuke.yml`

**What it does:**
1. Installs Podman (uses `dnf` on Fedora, `apt` on Debian — detected via
   `ansible_facts['os_family']`)
2. Installs and starts `openssh-server` on `podman-remote` only — the Debian
   Trixie minimal image does not ship it, and ezpodman needs sshd running to
   connect from `ezpodman-local` over SSH
3. Creates the `podman` user, with a password set on `podman-remote` only
   (`sandbox_user_password`, default: `podman`) — required so ezpodman's
   `ssh-copy-id` step can authenticate when pushing the SSH key
4. Enables lingering for that user
5. Enables `podman.socket` on both VMs — `ezpodman-local` needs it so ezpodman
   can find the local socket; `podman-remote` needs it for remote SSH access

**Why `gather_facts: false` + `pre_tasks` bootstrap:**
The Fedora 43 minimal image ships without Python. Ansible needs Python on
the managed node to run almost every module (it uploads a Python script and
executes it). The one exception is `ansible.builtin.raw` — it just runs a
shell command with no Python involved.

The bootstrap pattern:

```yaml
gather_facts: false        # don't try to gather facts yet — Python may not exist

pre_tasks:
  - name: Bootstrap Python3
    ansible.builtin.raw: |
      command -v python3 || dnf install -y python3   # shell conditional, no Python needed
    changed_when: false

  - name: Gather facts      # now Python exists, gather facts normally
    ansible.builtin.setup:
```

**Why lingering:**
When no user is logged in, systemd tears down that user's session — including
any user-scoped services like `podman.socket`. `loginctl enable-linger` tells
systemd to keep the user session alive permanently, even with no active login.
Without it, `podman.socket` would only work when someone is SSH'd in as
`podman`, which is useless for automation.

**Smoke test:**
The role ends with a real container run to confirm rootless Podman actually
works end-to-end:

```yaml
- name: Smoke-test rootless Podman
  ansible.builtin.command:
    cmd: podman run --rm --network none docker.io/hello-world
  become: true
  become_user: podman
```

If subuid/subgid is misconfigured, or user namespaces are disabled in the
kernel, this fails immediately — better than discovering it later.

### `test_containers` — container lifecycle

**Used by:** `containers_up.yml`, `containers_down.yml`, `nuke.yml`

**What it does:**
Starts or stops the containers defined in `group_vars`, depending on the
`container_state` variable passed in by the calling playbook.

**One role, two directions:**
Rather than writing separate "up" and "down" task files, the role accepts a
`container_state` variable:

```yaml
# containers_up.yml passes:
container_state: started

# containers_down.yml passes:
container_state: absent
```

The `containers.podman.podman_container` module is idempotent in both
directions — calling it with `state: started` on a running container is a
no-op; calling it with `state: absent` on a container that doesn't exist is
also a no-op.

**`become` and `XDG_RUNTIME_DIR`:**
Containers belong to the `podman` user, not root. To manage them, Ansible
must act as that user:

```yaml
become: true
become_user: podman
environment:
  XDG_RUNTIME_DIR: "/run/user/{{ uid }}"
```

`XDG_RUNTIME_DIR` is the path to the user's runtime directory, where
`podman.socket` lives. Systemd creates it at `/run/user/<UID>`. Without it,
Podman can't find its socket and the task fails.

The UID is looked up dynamically via `ansible.builtin.getent` rather than
hardcoded, so it works even if the user was created with a different UID on
different machines.

---

## 8. Playbooks — orchestrating the roles

### `provision.yml` — localhost only

Runs entirely on your control node. Uses the `incus_project` role to talk to the Incus
API via CLI. No connection plugin involved — the VMs don't exist yet.

### `setup.yml` — two-play structure

```
Play 1 (hosts: all)     → podman_setup role
Play 2 (hosts: local)   → Fedora toolchain tasks
```

Play 1 runs on both VMs because both need Podman. Play 2 runs only on
`ezpodman-local` because `podman-remote` needs nothing else.

The toolchain tasks in Play 2 are written directly in the playbook rather than
in a role — because they're only used once (one host, one play) and creating a
role for a one-time task adds structure without adding value.

**lazydocker download URL:**
GitHub release filenames embed the version number
(`lazydocker_0.25.0_Linux_x86_64.tar.gz`). The `/releases/latest/download/`
shortcut only works when the filename is version-agnostic. The fix is to query
the GitHub API first:

```yaml
- name: Get latest release version
  ansible.builtin.uri:
    url: https://api.github.com/repos/jesseduffield/lazydocker/releases/latest
  register: lazydocker_release

- name: Build download URL
  ansible.builtin.set_fact:
    lazydocker_url: "https://github.com/.../lazydocker_{{ version }}_Linux_{{ arch }}.tar.gz"
```

### `containers_up.yml` and `containers_down.yml`

Thin wrappers around the `test_containers` role. Their only job is to set
`container_state` and call the role:

```yaml
roles:
  - role: test_containers
    vars:
      container_state: started   # or absent
```

Container definitions (image, ports, env) live in `group_vars` — not in the
playbook or role — because they describe the desired state of specific hosts.

### `nuke.yml` — see next section

---

## 9. Idempotency — safe to re-run

Idempotency means running a playbook twice produces the same result as running
it once. It's one of the core design goals of Ansible.

Most Ansible modules are idempotent by default — `dnf: state=present` won't
reinstall a package that's already there. But `ansible.builtin.command` is
not — it runs every time regardless.

This project handles `command` idempotency with a check-then-act pattern:

```yaml
- name: List existing instances
  ansible.builtin.command:
    cmd: "incus list {{ incus_remote }}: --project ezpodman-sandbox --format csv --columns n"
  register: existing_instances   # store the output
  changed_when: false            # listing is never a change

- name: Launch VM
  ansible.builtin.command:
    cmd: "incus launch ..."
  when: item.name not in existing_instances.stdout   # only act if needed
```

The `when` condition is what makes the launch idempotent — it only runs if the
VM doesn't already exist. Re-running `provision.yml` against an environment
that's already provisioned will skip all the create/launch tasks.

`changed_when: false` is separate from idempotency — it tells Ansible "this
task never modifies state, don't count it as a change in the summary." Useful
for read-only commands so the play recap is accurate.

---

## 10. Rootless Podman — why a dedicated user

**Rootless Podman** means running containers as a normal (non-root) user.
The containers have no elevated privileges on the host, which is safer and
closer to how you'd run Podman in production.

This requires:

1. **A real user** — not root. This project creates a `podman` user.
2. **User namespace mapping** — the kernel must be able to map the container's
   internal root (UID 0) to an unprivileged UID on the host. This is configured
   in `/etc/subuid` and `/etc/subgid`. Modern Linux systems auto-populate these
   when a user is created via `useradd`.
3. **Lingering** — so the user's systemd session persists without an active
   login (see above).
4. **`XDG_RUNTIME_DIR`** — so tools know where to find the user's runtime
   socket.

When Ansible runs tasks as the `podman` user (via `become: true` +
`become_user: podman`), it must also set `XDG_RUNTIME_DIR` and
`DBUS_SESSION_BUS_ADDRESS` in the environment. Without them, `systemctl --user`
and the Podman socket cannot be reached.

**Why `.bashrc` instead of PAM:** `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS`
are normally set by `pam_systemd` at login. But `su - podman` inside an
`incus exec` session does not go through PAM, so they are never populated.
`setup.yml` writes them explicitly to `~/.bashrc` so they are always available
regardless of how the shell was started. The same applies to `DOCKER_HOST` and
`DOCKER_API_VERSION=1.41` (needed for lazydocker to talk to Podman's
Docker-compatible socket).

Always use `su -` (with the dash), never plain `su` — the dash starts a login
shell that sources `.bashrc`:

```bash
su - podman   # correct: sources ~/.bashrc, all env vars available
su podman     # wrong: inherits root's environment, env vars not set
```

**`TERM` and TUI apps:** `incus exec` propagates `$TERM` from the host into the
VM. If the host terminal is Ghostty (`TERM=xterm-ghostty`), TUI apps like
lazydocker will fail because Fedora's terminfo database doesn't include Ghostty.
Add `export TERM="xterm-256color"` to `~/.bashrc` manually on `ezpodman-local`
if using Ghostty on the control node.

**Package installation must be done as root.** The `podman` user is unprivileged
and cannot install system packages. Install via `dnf` or `apt` from a root
shell — the package becomes available system-wide and usable by the `podman`
user afterward.

---

## 11. Teardown design — nuke.yml

`nuke.yml` uses Ansible **tags** to create two execution paths from one file.

### How tags work

When you run `ansible-playbook nuke.yml` with no flags, every task runs.
When you pass `--tags force`, only tasks tagged `force` run — everything else
is skipped.

`nuke.yml` uses this at the **play** level:

```yaml
- name: Graceful container shutdown
  hosts: all
  tags: graceful        # skipped when --tags force is passed
  ...

- name: Destroy VMs and project
  hosts: localhost
  tags: force           # always runs; with --tags force, graceful play is skipped
  ...
```

### Teardown order matters

The destroy play lists what actually exists rather than relying on variables.
This makes it safe for partial deployments (e.g., if `provision.yml` failed
halfway through):

```yaml
- name: List VMs in project
  ansible.builtin.command:
    cmd: "incus list {{ incus_remote }}: --project ezpodman-sandbox --format csv --columns n"
  register: existing_vms

- name: Delete VMs
  ansible.builtin.command:
    cmd: "incus delete {{ incus_remote }}:{{ item }} --project ezpodman-sandbox --force"
  loop: "{{ existing_vms.stdout_lines }}"
```

The `--force` flag on `incus delete` stops the VM first if it's still running,
so the order of operations in a force teardown is: stop + delete VM → delete
images → delete project.

**Why images must be deleted before the project:**
Incus caches images inside projects after first use. The project deletion fails
if any images remain — Incus refuses to delete a non-empty project. This is why
image deletion is a dedicated step between VM deletion and project deletion.
