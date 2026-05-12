# bitcoin-ingress

[![Lint](https://github.com/lightwebinc/bitcoin-ingress/actions/workflows/lint.yml/badge.svg)](https://github.com/lightwebinc/bitcoin-ingress/actions/workflows/lint.yml)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

Ansible and Terraform automation for deploying
[`bitcoin-shard-proxy`](https://github.com/lightwebinc/bitcoin-shard-proxy)
nodes — the stateless ingress tier of the BSV multicast pipeline.

```text
BSV senders ──UDP/TCP──▶  bitcoin-shard-proxy  ──multicast──▶  FF05::<shard>:9001
                          (this repo deploys)                   (subscriber fabric)
```

## Supported Platforms

| OS           | Automation | Service Manager |
| ------------ | ---------- | --------------- |
| Ubuntu 24.04 | Ansible    | systemd         |
| FreeBSD 14   | Ansible    | rc.d            |
| AWS EC2      | Terraform  | systemd         |
| Any SSH host | Terraform  | generic         |

## Quick Start

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml
ansible-playbook -i inventory/hosts.yml site.yml
```

## Documentation

- [Architecture](docs/architecture.md)
- [Ansible usage](docs/ansible.md)
- [Networking (GRE / ethernet)](docs/networking.md)
- [BGP](docs/bgp.md)
- [Terraform](docs/terraform.md)
- [LXD lab](docs/lxd-lab.md)
- OS notes: [Ubuntu 24.04](docs/os/ubuntu-24.04.md), [FreeBSD 14](docs/os/freebsd-14.md)

## Repository Layout

```text
ansible/     Roles and playbooks
terraform/   Modules and cloud examples
docs/        Per-topic documentation
```

## License

Apache 2.0 — see [LICENSE](LICENSE).
