# k3s-server-bootstrap

Reusable Ansible bootstrap for Debian servers running K3s, ingress-nginx and ArgoCD.

## Scope

A single playbook takes a fresh Debian server from its provider bootstrap account to a reusable K3s host:

- Connect with the provider's initial user and password
- Generate or use a dedicated local SSH key
- Create/configure an infrastructure admin user
- Install the SSH public key
- Configure passwordless sudo for automation
- Reconnect automatically as the infrastructure user
- Install base packages
- Install K3s with bundled Traefik disabled
- Install ingress-nginx
- Install ArgoCD

K3s ServiceLB remains enabled. ingress-nginx is deployed with a LoadBalancer Service so K3s can expose HTTP/HTTPS through the node ports 80 and 443 without requiring MetalLB on a single VPS.

Everything application-specific after the bootstrap should be managed through GitOps in application repositories.

## Requirements

On the control machine:

- Ansible
- `ssh-keygen`
- SSH access to the target Debian server
- Initial server user with `sudo`

## Create the infrastructure SSH key

Create one dedicated Ed25519 key per server or environment. For example:

```bash
ssh-keygen -t ed25519 \
  -f ~/.ssh/id_ed25519_infra \
  -C "k3s-infra"
```

This creates:

```text
~/.ssh/id_ed25519_infra
~/.ssh/id_ed25519_infra.pub
```

The private key must remain only on the control machine and must never be committed to Git.

Configure the inventory to use that key:

```yaml
infra_ssh_private_key: "{{ lookup('env', 'HOME') + '/.ssh/id_ed25519_infra' }}"
infra_ssh_public_key: "{{ infra_ssh_private_key }}.pub"
```

The public key does not need to be copied manually to the server. During the first playbook run, Ansible connects with the provider bootstrap user and password, creates the infrastructure user, installs the public key and then reconnects automatically using the private key.

If the configured SSH key does not exist, the playbook can generate it automatically using the configured `infra_ssh_private_key` path.

## Usage

Copy the example inventory:

```bash
cp -R inventories/example inventories/server
```

Real inventories are ignored by Git. Only `inventories/example` is versioned.

Set the server address and provider bootstrap user in:

```text
inventories/server/hosts.yml
```

Example for a fresh OVH Debian VPS:

```yaml
all:
  children:
    bootstrap_servers:
      hosts:
        server:
          ansible_host: 203.0.113.10
          ansible_user: debian
```

Set environment-specific bootstrap values in:

```text
inventories/server/group_vars/all.yml
```

Then run:

```bash
ansible-playbook \
  -i inventories/server/hosts.yml \
  site.yml \
  --ask-pass \
  --ask-become-pass
```

The password prompts are only used for the provider bootstrap account. During the same run Ansible creates the infrastructure user, installs the SSH public key, reconnects with the dedicated key, and continues automatically.

Subsequent runs can use the same playbook. If the provider bootstrap account remains available, the command above is still valid and idempotent.

## Security

Passwords, private keys, tokens, real server inventories and application secrets must never be committed to this repository.

The infrastructure user uses passwordless sudo by default to allow unattended Ansible automation. This can be disabled with:

```yaml
infra_passwordless_sudo: false
```

## Design

The playbook is intended to be idempotent and reusable across servers. Environment-specific values live in local inventories; roles stay generic.
