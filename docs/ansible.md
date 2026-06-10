# Ansible

## Requirements

- Ansible 2.15+
- Python 3.9+ on the control machine
- SSH access to target hosts with a user that has `sudo` / root privileges
- Supported target OS: Ubuntu 24.04 or FreeBSD 14

Install Ansible dependencies:

```bash
pip install ansible
ansible-galaxy collection install community.general ansible.posix
```

---

## Quick start

```bash
cd ansible/

# 1. Copy and edit the inventory
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml

# 2. Review and override variables
$EDITOR group_vars/all.yml

# 3. Run the full playbook
ansible-playbook -i inventory/hosts.yml site.yml
```

---

## Inventory

`inventory/hosts.example.yml` shows the expected structure:

```yaml
all:
  children:
    ingress_nodes:
      hosts:
        node-01:
          ansible_host: 203.0.113.10
          ansible_user: ubuntu
          egress_iface: eth1
          bgp_peer_ip: 203.0.113.254
          bgp_router_id: 203.0.113.10
        node-02:
          ansible_host: 198.51.100.20
          ansible_user: ubuntu
          egress_iface: eth1
          bgp_peer_ip: 198.51.100.254
          bgp_router_id: 198.51.100.20
```

Host-level variables override `group_vars/all.yml`.

> **Important — `egress_iface` precedence**: `group_vars/all.yml` defines `egress_iface: eth1` as a default. Because Ansible gives `group_vars/all` higher precedence than inventory group `vars:` blocks, `egress_iface` **must be set per-host** (under `hosts: <name>:`) to take effect. Setting it only in the inventory `vars:` block will silently use the `group_vars/all.yml` default instead.

---

## Variables reference

All variables with defaults live in `group_vars/all.yml`.

### Proxy

| Variable | Default | Description |
|---------------------|----------------------------|-------------------------------------------------------------------------------------------------------|
| `proxy_repo` | (GitHub URL) | git URL of shard-proxy source |
| `proxy_version` | `main` | git ref (branch, tag, or SHA) to check out |
| `proxy_install_dir` | `/opt/shard-proxy` | Clone and build destination |
| `proxy_bin_dir` | `/usr/local/bin` | Where to install the compiled binary |
| `listen_addr` | `[::]` | Bind address for incoming frames |
| `udp_listen_port` | `9000` | UDP ingress port |
| `tcp_listen_port` | `0` | TCP ingress port for reliable delivery (0 = disabled) |
| `egress_port` | `9001` | UDP egress port for multicast groups |
| `shard_bits` | `2` | Bit width of shard key (0–12; must match the rest of the fabric) |
| `mc_scope` | `site` | Multicast scope: link / site / org / global |
| `mc_group_id` | `"0x000B"` | IANA group-id (bytes 12–13); default = IANA Bitcoin allocation `FF0X::B` |
| `num_workers` | `0` | Worker count (0 = runtime.NumCPU) |
| `metrics_addr` | `:9100` | HTTP address for /metrics /healthz /readyz |
| `otlp_endpoint` | `""` | OTLP gRPC endpoint (empty = disabled) |
| `drain_timeout` | `"0s"` | Pre-drain delay before closing sockets on shutdown; set to `≥` LB health-check interval in production |

> **`TimeoutStopSec` relationship:** `systemd` sends `SIGKILL` after `TimeoutStopSec` if the service has not exited. Ensure `TimeoutStopSec > drain_timeout + 15s` (OTLP flush + drain buffer). The default service unit sets `TimeoutStopSec=30`, which is sufficient for `drain_timeout ≤ 15s`.

### Networking

| Variable | Default | Description |
|------------------|------------|-------------------------------------------------------|
| `egress_mode` | `ethernet` | `ethernet` or `gre` |
| `egress_iface` | `eth1` | Interface name(s) — list or comma string |
| `gre_local_ip6` | `""` | Local IPv6 address for the GRE tunnel (gre mode only) |
| `gre_remote_ip6` | `""` | Remote IPv6 GRE endpoint address |
| `gre_iface` | `gre6-bsp` | GRE tunnel interface name |
| `gre_inner_ipv6` | `""` | IPv6 address/prefix for the GRE interface |

### BGP

| Variable | Default | Description |
|-----------------|---------|-------------------------------------------------|
| `enable_bgp` | `false` | Enable eBGP |
| `bgp_daemon` | `bird2` | `bird2` or `frr` |
| `bgp_prefix` | `[]` | IPv4 BGP prefixes announced by all nodes (list) |
| `bgp_vip` | `""` | IPv4 loopback BGP VIP |
| `bgp_prefix6` | `[]` | IPv6 BGP prefixes announced by all nodes (list) |
| `bgp_vip6` | `""` | IPv6 loopback BGP VIP |
| `bgp_local_as` | `65001` | Local ASN |
| `bgp_peer_as` | `65000` | Upstream provider ASN |
| `bgp_peer_ip` | `""` | Upstream BGP peer IP |
| `bgp_router_id` | `""` | BGP router ID (defaults to primary IPv4) |
| `bgp_hold_time` | `90` | BGP hold time (seconds) |
| `bgp_keepalive` | `30` | BGP keepalive interval (seconds) |
| `bgp_password` | `""` | Optional MD5 session password |

---

## Roles

| Role | Purpose |
|-----------------------|-----------------------------------------------------------|
| `common` | OS packages, Go toolchain install, build dependencies |
| `perf-tuning` | High-PPS host tuning: UDP buffers, busy-poll, txqueuelen, deep C-state disable, irqbalance off |
| `shard-proxy` | Clone, build, install binary, configure service unit |
| `networking` | Ethernet or GRE egress interface, IPv6 multicast routing |
| `bgp` | BIRD2 or FRR install, config template, health-check timer |

Roles are applied in the order listed by `site.yml`. The `bgp` role is skipped when `enable_bgp: false`.

### perf-tuning role

Applies the host-level network/CPU tunings measured to raise small-packet
(200–256 B Bitcoin P2PKH) proxy throughput. All knobs live in
`roles/perf-tuning/defaults/main.yml`; the role is self-documenting via
inline comments. Highlights:

| Variable | Default | Effect |
|----------|---------|--------|
| `perf_tuning_enabled` | `true` | Master switch; set `false` for stock OS behaviour |
| `perf_tuning_sysctls` | UDP rmem/wmem 256 MiB, `busy_poll`/`busy_read` 50 µs, backlog 1 M | `/etc/sysctl.d/65-perf-tuning.conf` |
| `perf_tuning_txqueuelen` | `10000` | systemd-networkd `.link` drop-in on the egress NIC |
| `perf_tuning_disable_cstates` | `true` | Disables C3–C10 at runtime + boot (systemd oneshot) |
| `perf_tuning_grub_cmdline` | `true` | Adds `intel_idle.max_cstate=1` to GRUB (reboot required) |
| `perf_tuning_disable_irqbalance` | `true` | Stops + masks `irqbalance` (conflicts with manual IRQ affinity) |

The same role ships identically in `listener-infra` and
`retransmission-infra`.

---

## Tags

Run only specific roles using Ansible tags:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags proxy
ansible-playbook -i inventory/hosts.yml site.yml --tags networking
ansible-playbook -i inventory/hosts.yml site.yml --tags bgp
ansible-playbook -i inventory/hosts.yml site.yml --tags common
ansible-playbook -i inventory/hosts.yml site.yml --tags perf-tuning
```

---

## Upgrading the proxy

To pull a new version and rebuild:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags proxy -e proxy_version=v1.2.0
```

The role will git-fetch, check out the new ref, run `go build`, copy the binary, and restart the service.

---

## Idempotency

All roles are fully idempotent. Re-running the playbook is safe and will only apply changes when the
system state differs from the declared configuration.

---

## Lab and Kubernetes deployment

For container-based local labs and CI testing, use the Go Docker harness in
[multicast-test](https://github.com/lightwebinc/multicast-test). For
Kubernetes deployment, see
[multicast-kube-infra](https://github.com/lightwebinc/multicast-kube-infra)
and the [shard-proxy-helm](https://github.com/lightwebinc/shard-proxy-helm)
chart.
