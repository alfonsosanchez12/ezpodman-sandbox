# ezpodman-sandbox

Ansible automation to provision, configure, and tear down a sandbox environment
for testing [ezpodman](https://github.com/alfonsosanchez12/ezpodman) — a lazydocker wrapper for Podman.

> For a deep dive into the architecture and design decisions behind this automation,
> see [docs/design.md](docs/design.md).

---

## Deploy everything

`incus_remote` is required on every run — there is no default. Pass it explicitly:

```bash
ansible-playbook playbooks/provision.yml -e "incus_remote=<remote>" && \
ansible-playbook playbooks/setup.yml -e "incus_remote=<remote>" && \
ansible-playbook playbooks/containers_up.yml -e "incus_remote=<remote>"
```

When complete you have:

- **ezpodman-local** (Fedora 43) — podman, podman-remote, lazydocker, ezpodman, full toolchain, `podman.socket` enabled, `XDG_RUNTIME_DIR` / `DBUS_SESSION_BUS_ADDRESS` / `DOCKER_HOST` / `DOCKER_API_VERSION` set in `.bashrc`
- **podman-remote** (Debian Trixie) — podman + `podman.socket` enabled
- Containers running: nginx + caddy on ezpodman-local, nginx + postgres on podman-remote

---

## Prerequisites

Install the required Ansible collections once:

```bash
ansible-galaxy collection install community.general containers.podman
```

Verify the target remote is reachable:

```bash
# Named remote (control node → Incus server over TLS):
incus project list <remote>:

# Local Incus daemon (running directly on the Incus host):
incus project list local:
```

---

## Options

### Target a different remote

Pass `-e "incus_remote=<name>"` on every run. Available remotes: your configured Incus remotes.

```bash
ansible-playbook playbooks/provision.yml -e "incus_remote=<remote>"
```

### Change storage pool or network bridge

```bash
ansible-playbook playbooks/provision.yml \
  -e "incus_remote=<remote>" \
  -e "storage_pool=local" \
  -e "network_bridge=br0"
```

### Dry run (check mode)

```bash
ansible-playbook playbooks/provision.yml -e "incus_remote=<remote>" --check
```

Read-only tasks (list, query) still execute so the output is meaningful.
Write tasks (create, launch) are simulated.

---

## Playbooks

| Playbook | Purpose |
|----------|---------|
| `provision.yml` | Create the Incus project, launch VMs, wait for boot |
| `setup.yml` | Install and configure all software on the VMs |
| `containers_up.yml` | Start test containers |
| `containers_down.yml` | Stop and remove test containers |
| `nuke.yml` | Full teardown — everything deleted |

Each playbook is idempotent. Re-running it skips what already exists.

---

## Teardown

### Graceful (stops containers first)

```bash
ansible-playbook playbooks/nuke.yml -e "incus_remote=<remote>"
```

### Force (skip container shutdown)

```bash
ansible-playbook playbooks/nuke.yml -e "incus_remote=<remote>" --tags force
```

### Containers only (keep VMs)

```bash
ansible-playbook playbooks/containers_down.yml -e "incus_remote=<remote>"
```

---

## After provisioning (manual steps)

Two steps are out of scope for Ansible and must be done once after `setup.yml`:

1. **SSH key from ezpodman-local to podman-remote** — needed for ezpodman remote Podman over SSH.
   `setup.yml` ensures sshd is running on `podman-remote` and sets a password for the `podman`
   user (`podman` by default) so `ssh-copy-id` can authenticate. Run ezpodman and follow its
   prompts to push the key; use that password when asked.

2. **Step-CA root trust** — needed for HTTPS to homelab services (forgejo, etc.):

3. **Ghostty terminal users only** — Fedora doesn't ship Ghostty's terminfo, so
   TUI apps like lazydocker and ezpodman will fail with a cryptic error. Add this
   to the `podman` user's `~/.bashrc` on `ezpodman-local`:

   ```bash
   echo 'export TERM="xterm-256color"' >> ~/.bashrc
   source ~/.bashrc
   ```

   This only applies if your terminal is Ghostty (or any terminal whose
   `$TERM` value isn't in Fedora's terminfo database).

   ```bash
   # Run on each VM:
   curl -k https://keymaster.home.lan:6443/roots.pem \
     -o /usr/local/share/ca-certificates/keymaster-root.crt
   update-ca-certificates
   ```

---

## Connecting to VMs

```bash
# Root shell
incus shell --project ezpodman-sandbox <remote>:ezpodman-local

# Switch to the podman user — use 'su -' (with the dash), not plain 'su'.
# The dash starts a login shell, which sources ~/.bashrc where XDG_RUNTIME_DIR,
# DBUS_SESSION_BUS_ADDRESS, DOCKER_HOST, and DOCKER_API_VERSION are set.
# Plain 'su' inherits root's environment and none of those vars will be set.
su - podman

# Check running containers (as the podman user)
podman ps
```

### Installing packages

Package installation must be done as **root**, not as the `podman` user.
Rootless Podman controls containers — it has no elevated privileges on the host
and cannot install system packages.

```bash
# From a root shell inside the VM:
dnf install -y <package>    # Fedora (ezpodman-local)
apt-get install -y <package> # Debian (podman-remote)
```

Once installed, the package is available system-wide and usable by the `podman` user.
