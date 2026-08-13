# ansible-PoC

A hands-on Ansible learning project: use Ansible to stand up FreeIPA
(identity/LDAP/Kerberos) and Keycloak (SSO/OIDC, federated against IPA) on
two Rocky Linux 10 VMs. See `concepts.md` for what every moving part is and why
it's here — this file is just the "how to run it."

## Phase 1: plain ansible-core

Get the playbook itself working with the simplest possible setup before
introducing an Execution Environment (see `concepts.md`).

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

vagrant up

# One-time: create the vault holding the IPA/Keycloak admin passwords.
openssl rand -base64 32 > .vault-pass   # local-dev only, gitignored, never commit
ansible-vault create inventory/group_vars/all/vault.yml
# contents:
#   vault_ipaadmin_password: <password>
#   vault_ipadm_password: <password>
#   vault_keycloak_admin_password: <password>

ansible -i inventory/hosts.yml all -m ping
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### Verify

```bash
# IPA: from the ipa VM
vagrant ssh ipa -c 'echo <vault_ipaadmin_password> | kinit admin && ipa ping'

# Keycloak: from the control node
curl -sf http://192.168.56.12:8080/health/ready
curl -s http://192.168.56.12:8080/realms/poc-demo | jq .realm
```

## Phase 2: Execution Environment (not built yet)

Once the playbook above is verified working, swap the execution engine to
`ansible-navigator` running a Podman-built Execution Environment instead of
this venv's `ansible-core` directly — see `concepts.md` for what that means
and why. Nothing about the roles/playbook/inventory changes; only what
process runs them does.

## Layout

```
Vagrantfile              # the two target VMs (ipa, keycloak)
requirements.txt         # phase 1 control-node venv (ansible-core, ansible-lint)
requirements.yml         # Ansible Galaxy collections
ansible.cfg
inventory/
  hosts.yml
  group_vars/
    all.yml               # shared vars (domain, timezone)
    all/vault.yml          # ansible-vault encrypted secrets (created above, gitignored contents are still committed encrypted)
    ipa.yml
    keycloak.yml
roles/
  common/                  # base packages, timezone
  ipaserver/                # wraps freeipa.ansible_freeipa's ipaserver role
  keycloak/                  # podman + systemd unit, then API-driven config
playbooks/
  site.yml
concepts.md               # what everything is and why
```

## Deferred

- **Harbor** — phase-2 stretch goal, registered as a Keycloak OIDC client
  once IPA + Keycloak are solid.
- **GitLab, Satellite, Tenable** — out of scope for this PoC; see
  `concepts.md` for why.
