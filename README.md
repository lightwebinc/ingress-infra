# ingress-infra

[![Lint](https://github.com/lightwebinc/ingress-infra/actions/workflows/lint.yml/badge.svg)](https://github.com/lightwebinc/ingress-infra/actions/workflows/lint.yml)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

> Part of the [**BSV Layered Multicast**](https://github.com/lightwebinc/bsv-multicast) open-source project — see the main repository for the full architecture, design docs, and BRC specifications.

Ansible and Terraform automation for deploying
[`shard-proxy`](https://github.com/lightwebinc/shard-proxy)
nodes — the stateless ingress tier of the BSV multicast pipeline.

```text
BSV senders ──UDP/TCP──▶  shard-proxy  ──multicast──▶  FF05::<shard>:9001
                          (this repo deploys)                   (subscriber fabric)
```

(ASM group form shown; SSM deployments use `FF35`/`FF3E` — this repo's `group_vars` default is still `asm`.)

## Platforms

Ubuntu 24.04, Debian 13 and FreeBSD 14 via Ansible; AWS EC2 or any SSH host via
Terraform. See [supported platforms](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/platforms.md).

## Quick Start

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml
ansible-playbook -i inventory/hosts.yml site.yml
```

## Documentation

- [Shared host-deployment docs](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/README.md) (platforms, Ansible operations, Terraform layout, OS notes)
- [Architecture](docs/architecture.md)
- [Ansible usage](docs/ansible.md)
- [Networking (GRE / ethernet)](docs/networking.md)
- [BGP](docs/bgp.md)
- [Terraform](docs/terraform.md)
- OS notes: [Ubuntu 24.04](docs/os/ubuntu-24.04.md), [Debian 13](docs/os/debian-13.md), [FreeBSD 14](docs/os/freebsd-14.md)

## Repository Layout

```text
ansible/     Roles and playbooks
terraform/   Modules and cloud examples
docs/        Per-topic documentation
```

## License

Apache 2.0 — see [LICENSE](LICENSE).
