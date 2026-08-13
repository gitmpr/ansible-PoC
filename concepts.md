# Concepts

What each moving part in this repo is, and why it's here. README.md says how
to run things; this explains what you're actually looking at.

## How the pieces fit together

```
                    control node (this laptop)
   ┌────────────────────────────────────────────────────┐
   │  Phase 1: .venv/bin/ansible-playbook  (bare metal)  │
   │  Phase 2: ansible-navigator → podman → EE container │
   │           (ansible-core + collections live in here) │
   └──────────────────────┬───────────────────────────────┘
                           │ SSH
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌─────────────────┐      ┌─────────────────┐
     │   ipa VM         │      │  keycloak VM     │
     │(Rocky 10, libvirt)│      │(Rocky 10, libvirt)│
     │                  │      │                  │
     │ ipa-server-install│     │ podman container:│
     │ runs directly on │      │ quay.io/keycloak │
     │ the VM (389-ds,  │      │ managed by a      │
     │ Kerberos, Dogtag)│      │ systemd unit      │
     └─────────────────┘      └─────────────────┘
```

Two independent axes worth keeping separate in your head:

- **Where the target services run** — real VMs (`ipa`, `keycloak`), because
  that's the layer Ansible is actually managing.
- **What runs `ansible-playbook` on the control side** — a plain venv at
  first, later swapped for a containerized Execution Environment. This axis
  doesn't change what the playbook does, only how/where it executes.

## Core Ansible vocabulary

- **Inventory** (`inventory/hosts.yml`) — the list of managed hosts and how
  to reach them, grouped (`ipa`, `keycloak`). Variables can be attached at
  the `all`, group, or individual host level; more specific wins. That's
  "variable precedence," and it's how `group_vars/ipa.yml` and
  `group_vars/keycloak.yml` each apply only to their own host without the
  playbook needing to know which is which.
- **Playbook** (`playbooks/site.yml`) — an ordered list of *plays*, each
  targeting a group of hosts and applying a list of roles/tasks to them.
- **Role** (`roles/*`) — a reusable, self-contained bundle of automation
  with a fixed directory shape: `tasks/` (what to do), `handlers/`
  (deferred actions, see below), `templates/` (files to render),
  `defaults/` (overridable variables). Splitting `common`/`ipaserver`/
  `keycloak` into separate roles is what makes each one independently
  reusable/testable instead of one giant script.
- **Module** — the actual unit of work a task calls (`ansible.builtin.dnf`,
  `ansible.builtin.template`, `community.general.keycloak_realm`, …).
  Modules are declarative and (ideally) idempotent: running one twice with
  the same inputs produces the same end state, and reports "no change" the
  second time. That idempotence is *the* property that makes Ansible safe
  to re-run instead of a one-shot script.
- **Collection** — a distributable package of modules/roles/plugins,
  installed from Ansible Galaxy via `requirements.yml` and
  `ansible-galaxy collection install`. `freeipa.ansible_freeipa` and
  `community.general` are collections; `ansible.builtin.*` modules ship
  with `ansible-core` itself and need no collection.
- **Handler** — a task that only runs when explicitly *notified* by another
  task that changed something (e.g. templating a new config file notifies
  "Restart keycloak"). This is why config-then-restart isn't two unconditional
  tasks: if the config didn't change, the restart doesn't fire either.
- **Template** (Jinja2, `.j2` files) — files rendered with variables
  substituted in, e.g. `roles/keycloak/templates/keycloak.service.j2` fills
  in the admin password and port from inventory vars.
- **`become`** — privilege escalation (sudo). Set per-task or per-play;
  needed here because installing packages/managing systemd units requires
  root on the VMs, while the SSH login itself is the unprivileged `vagrant`
  user.
- **Idempotency & check mode** — `ansible-playbook --check --diff` simulates
  a run without changing anything, showing what *would* change. Useful for
  confirming nothing unexpected would happen before actually applying.
- **Tags / `--limit`** — running a subset of a playbook: `--tags keycloak`
  runs only tasks tagged that way, `--limit ipa` runs only against that
  host group. Handy once a playbook covers more than you want to touch on a
  given run.
- **`ansible-vault`** — encrypts secrets at rest (`inventory/group_vars/
  all/vault.yml` holds the IPA and Keycloak admin passwords) so they can
  live in git without being plaintext. Decrypted transparently at run time
  using a vault password (kept in a gitignored `.vault-pass` file for this
  PoC — never commit that file).

## The execution model: ansible-core vs Execution Environments

This is the part worth being deliberate about, since it's genuinely a
different way of running Ansible than "install it and go."

**Phase 1 — plain `ansible-core`.** A Python venv (`requirements.txt`)
installs `ansible-core` directly on this laptop. `ansible-playbook` runs
natively here, SSHes to the VMs, done. Simple, fast feedback loop, but
"works on my machine" — the exact Python/collection versions depend on
whatever's installed on *this* laptop at *this* moment.

**Phase 2 — Execution Environment.** This is what Ansible Automation
Platform (and increasingly plain `ansible-core` users) actually run in
production now, and it's the reason `ansible-builder`/`ansible-navigator`
exist:

- **`ansible-builder`** — takes an `execution-environment.yml` (which base
  image to start from, which `requirements.yml` collections and
  `requirements.txt` Python packages to install) and *builds a container
  image* containing ansible-core, those collections, and their
  dependencies, all pinned and reproducible. Think of it as a Dockerfile
  generator purpose-built for Ansible's needs — the output is an ordinary
  container image, just one guaranteed to contain a consistent, versioned
  set of everything a playbook run needs.
- **`ansible-navigator`** — the tool you actually invoke instead of
  `ansible-playbook`. It starts a container from that built image (via
  Podman here), mounts the project directory in, and runs the playbook
  *inside* that container rather than on the bare host. It has two output
  modes: `stdout` (classic scrolling text, like `ansible-playbook`) and an
  interactive TUI (browse plays → tasks → individual results, inspect
  facts, replay a previous run's artifact).
- **Why this matters**: the "control node" stops being a fragile pile of
  whatever's pip-installed on someone's laptop and becomes a versioned,
  distributable artifact — the same EE image that ran this playbook on your
  laptop is the one CI or a teammate would use, byte-for-byte. That's the
  actual problem EEs solve.
- **Podman's role**: it's the container engine `ansible-builder` uses to
  build the image and `ansible-navigator` uses to run it — chosen over
  Docker here because it's daemonless/rootless and it's what the rest of
  this RHEL-adjacent stack (IPA, Keycloak, and the EE tooling itself) is
  built around.

Nothing about the roles/playbook/inventory changes between phase 1 and 2 —
only *what process executes `ansible-playbook`* changes. That's the point:
swapping the execution engine is an isolated, understandable step once the
automation itself is already proven to work.

## Infrastructure: Vagrant + libvirt

`Vagrant` (with the `vagrant-libvirt` plugin) is VM lifecycle management —
`vagrant up`/`halt`/`destroy` around `virsh`/QEMU/KVM, so the two target
VMs (`ipa`, `keycloak`) are defined declaratively in one `Vagrantfile`
rather than clicked together by hand. This exists purely to give Ansible
something real to manage; it's not itself an Ansible concept.

### Why Rocky Linux, not Ubuntu

FreeIPA's own downloads page officially lists Fedora, RHEL, and CentOS;
Debian gets a tracked package; **Ubuntu isn't on the list at all**. Rather
than fight that on the one VM that actually needs it, both VMs run Rocky
Linux 10 (RHEL-family, on the supported list) — Keycloak has no
distro-specific requirements either way, it only needs Podman, so there's
no cost to keeping both VMs the same OS.

## Why IPA and Keycloak are set up differently

Deliberately, so this PoC exercises two distinct Ansible techniques instead
of doing the same thing twice — this is about *how each service is
installed*, not about the underlying OS (both VMs run the same distro):

- **IPA**: a full OS-level install. `ipa-server-install` stands up 389-ds,
  MIT Kerberos, and a Dogtag CA as real systemd services on the VM — not
  something you'd sensibly containerize. Driven by the **official
  `freeipa.ansible_freeipa` collection**'s `ipaserver` role (a vendor
  collection, not hand-rolled), teaching "install and configure a full
  system service via someone else's well-tested role."
- **Keycloak**: a single containerized process, deployed via Podman managed
  by a hand-written systemd unit (teaching "run a container as a proper
  system service," a very common real-world pattern), then configured
  **through its REST API** using `community.general.keycloak_realm`/
  `keycloak_client`/`keycloak_user` — idempotent API-driven configuration,
  a different skill from installing packages.

## Deferred / excluded, and why

- **Harbor** — a real phase-2 target (register it as a Keycloak OIDC
  client, completing an identity chain), deferred until IPA + Keycloak are
  solid; it's ~9 containers via its own docker-compose installer.
- **GitLab, Satellite** — excluded: both exceed this host's resources
  (Satellite's floor alone is ~20GB+, more than this laptop has total).
- **Tenable** — excluded: no clean container/VM-native path for a free PoC
  (license activation required even for the free tier).
