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

**`group_vars/all.yml` is the canonical variables reference** — every variable
ships there with a default and an inline comment (build source, ports,
push-frame ingest (subtree/block), PoW gate, BRC-142 coalescing, SSM source mode, dedup backend,
networking, BGP). Override per-host in the inventory. Topic guides:

- Egress interfaces, GRE, multicast routing, hop limit, push-frame ingest (subtree/block), dedup backend — [networking.md](networking.md)
- eBGP / iBGP, health-gated announce, `bgp_health_path` — [bgp.md](bgp.md)

> **`TimeoutStopSec` relationship:** `systemd` sends `SIGKILL` after `TimeoutStopSec` if the service has not exited. Ensure `TimeoutStopSec > drain_timeout + 15s` (OTLP flush + drain buffer). The default service unit sets `TimeoutStopSec=30`, which is sufficient for `drain_timeout ≤ 15s`.

---

## Roles

| Role | Purpose |
|-----------------------|-----------------------------------------------------------|
| `common` | OS packages, Go toolchain install, build dependencies |
| `perf-tuning` | High-PPS host tuning: UDP buffers, busy-poll, txqueuelen, deep C-state disable, irqbalance off |
| `shard-proxy` | Clone, build, install binary, configure service unit |
| `networking` | Ethernet or GRE egress interface, IPv6 multicast routing |
| `bgp` | BIRD2 or FRR install, config template, health-check timer |
| `bgp-ibgp` | iBGP daemon on upstream peer nodes (separate playbook: `bgp-ibgp.yml`) |

Roles are applied in the order listed by `site.yml`. The `bgp` role is skipped when `enable_bgp: false`. The `bgp-ibgp` role runs via its own playbook (`ansible-playbook -i inventory/hosts.yml bgp-ibgp.yml`), not `site.yml`.

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
ansible-playbook -i inventory/hosts.yml site.yml --tags proxy -e proxy_version=v1.25.1
```

The role will git-fetch, check out the new ref, run `go build`, copy the binary, and restart the service.
The build is stat-guarded: if a binary already exists in `proxy_install_dir` it is not rebuilt unless
`proxy_force_build: true` is set. To skip clone+build entirely and push a locally pre-built binary,
set `proxy_local_binary` to its local path.

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
