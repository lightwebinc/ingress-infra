# Ubuntu 24.04

Packages, systemd and netplan conventions, BGP daemon paths and diagnostics
shared by all infra repositories are in the canonical
[Ubuntu 24.04 notes](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/os/ubuntu-24.04.md). This page lists what is specific to
`shard-proxy`.

## Service

```bash
sudo systemctl status shard-proxy
sudo journalctl -u shard-proxy -f
sudo systemctl restart shard-proxy
```

## Ports

ingress-infra ships no firewall role (the `common` role does not manage
`ufw`); apply your own site policy. Ports that must be reachable:

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 8725 | UDP | inbound | shard-proxy ingress |
| `tcp_listen_port` (if set) | TCP | inbound | Optional TCP ingress (0 = disabled) |
| `subtree_listen_port` / `block_listen_port` (if set) | TCP | inbound | Push ingest (BRC-143 subtree / BRC-144 block); **tunnel-bound, allowlist miner-tier source CIDRs only** |
| 179 | TCP | in+out | BGP (if `enable_bgp: true`) |
| 9100 | TCP | inbound | Prometheus metrics / health endpoints |

## File locations

| Path | Content |
|---|---|
| `/usr/local/bin/shard-proxy` | Compiled binary |
| `/etc/shard-proxy/config.env` | Environment config |
| `/etc/systemd/system/shard-proxy.service` | systemd unit |
| `/opt/shard-proxy/` | Source clone and build directory |
| `/etc/netplan/60-ingress-infra.yaml` | Egress interface |
| `/etc/netplan/61-ingress-infra-gre.yaml` | GRE tunnel |
| `/etc/netplan/62-ingress-infra-vip.yaml` | BGP VIP |
| `/etc/sysctl.d/60-ingress-infra.conf` | IPv6 sysctls (forwarding) |
