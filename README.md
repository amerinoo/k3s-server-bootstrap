# k3s-server-bootstrap

Reusable Ansible bootstrap for Debian servers running K3s and ArgoCD.

## Scope

This repository intentionally handles only the host bootstrap layer:

- Create/configure an infrastructure admin user
- Install base packages
- Install K3s
- Disable bundled Traefik
- Install ArgoCD

Everything deployed inside the cluster after ArgoCD should be managed through GitOps in application-specific repositories.

## Requirements

On the control machine:

- Ansible
- SSH access to the target server
- A public SSH key to install for the infrastructure user

The target host is expected to be Debian with an initial user that can run `sudo`.

## Usage

Copy the example inventory:

```bash
cp -R inventories/example inventories/deca
```

Edit:

```text
inventories/deca/hosts.yml
inventories/deca/group_vars/all.yml
```

Then run:

```bash
ansible-playbook -i inventories/deca/hosts.yml site.yml
```

If the infrastructure user already exists, the playbook keeps it and converges the rest of the configuration.

## Design

The playbook is intended to be idempotent and reusable across servers. Environment-specific values live in inventories; roles stay generic.

No passwords, private keys, tokens or application secrets should be committed to this repository.
