# Terraform

Module and example structure, requirements, the Ansible hand-off, running an
example and adding a cloud are documented once in
[Terraform layout](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/terraform-layout.md). This page covers the ingress-specific
modules (`ingress-node`, `bgp-anycast`) and examples.

## Generic example

`terraform/examples/generic/` provisions hosts you already have.

`terraform.tfvars.example`:

```hcl
hosts = [
  {
    name       = "node-01"
    public_ip  = "203.0.113.10"
    ssh_user   = "ubuntu"
    ssh_key    = "~/.ssh/id_ed25519"
  }
]

shard_bits   = 2
egress_mode  = "ethernet"
egress_iface = "eth1"
enable_bgp   = false
```

---

## AWS EC2 example

Provisions a VPC, security group, EC2 instance(s) running Ubuntu 24.04, and optionally an Elastic
IP for stable inbound addressing.

`terraform.tfvars.example`:

```hcl
aws_region       = "us-east-1"
instance_type    = "t3.medium"
instance_count   = 2
key_name         = "my-key-pair"
ssh_private_key  = "~/.ssh/id_ed25519"

shard_bits   = 2
egress_mode  = "gre"
gre_remote_ip6 = "2001:db8:feed::1"   # fabric is IPv6-only: ip6gre endpoints are IPv6

enable_bgp      = false
bgp_prefix  = []
```

> **GRE on EC2 requires IPv6.** The ip6gre local endpoint is the instance's
> primary IPv6 address (the example VPC assigns one at launch); Elastic IPs
> are IPv4-only and cannot be tunnel endpoints.

---

## Module: `ingress-node`

**Inputs** (`modules/ingress-node/variables.tf`):

| Variable | Type | Description |
|--------------------------|----------|--------------------------------------------------------------------|
| `host_ip` | string | Public IP of the target host (required) |
| `ssh_user` | string | SSH username (default `ubuntu`) |
| `ssh_private_key_path` | string | Path to SSH private key file (required) |
| `ansible_playbook_path` | string | Path to `site.yml` (default: bundled `ansible/site.yml`) |
| `ansible_inventory_path` | string | Where to write the generated inventory (default: per-host file) |
| `extra_ansible_vars` | map(any) | Additional Ansible extra-vars merged over the first-class inputs |

The common proxy/network/BGP knobs are **first-class inputs** with defaults —
`proxy_repo`, `proxy_version`, `listen_port` (passed to Ansible as
`udp_listen_port`), `egress_port`, `shard_bits`, `mc_scope`, `mc_group_id`,
`mc_route_prefix`, `metrics_addr`, `egress_mode`, `egress_iface`,
`gre_local_ip6` / `gre_remote_ip6` / `gre_inner_ipv6`, and the `bgp_*`
variables. Anything else goes through `extra_ansible_vars`.

### Version pin coupling

`proxy_version` in `modules/ingress-node/variables.tf` must equal
`proxy_version` in `ansible/group_vars/all.yml`, the single source of truth;
move both in one change. See
[version pin coupling](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/terraform-layout.md#version-pin-coupling).

**Outputs**: `host_ip`, `inventory_path`.

---

## Module: `bgp-anycast` (BGP)

A thin helper that constructs the BGP-related `extra_vars` map from structured inputs, for use with
the `ingress-node` module.

```hcl
module "bgp" {
  source         = "../../modules/bgp-anycast"
  enable_bgp     = true
  bgp_daemon     = "bird2"
  bgp_prefix = ["192.0.2.0/24"]
  bgp_vip    = "192.0.2.1"
  bgp_local_as   = 65001
  bgp_peer_as    = 65000
  bgp_peer_ip    = "203.0.113.254"
}
```
