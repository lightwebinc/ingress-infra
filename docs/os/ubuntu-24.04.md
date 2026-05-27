# Ubuntu 24.04

## System requirements

- Ubuntu 24.04 LTS (Noble Numbat)
- IPv6 enabled on the egress interface
- Internet access for package installation and cloning `shard-proxy`
- `sudo` access for the Ansible user

## What the Ansible roles install

| Package / component | Source | Notes |
|-----------------------|----------------------|--------------------------------------|
| `build-essential` | apt | gcc, make, etc. for Go CGO |
| `git` | apt | clone shard-proxy |
| `curl` | apt | health-check script |
| Go toolchain | go.dev tarball | version set by `go_version` variable |
| `shard-proxy` | built from source | binary in `/usr/local/bin/` |
| `bird2` or `frr` | apt (if BGP enabled) | BGP daemon |

## Service management

The proxy runs as a **systemd service** (`shard-proxy.service`). The service unit is
templated from `roles/shard-proxy/templates/shard-proxy.service.j2`.

```bash
# Status
sudo systemctl status shard-proxy

# Logs
sudo journalctl -u shard-proxy -f

# Restart
sudo systemctl restart shard-proxy
```

## Networking

- Egress interface configuration is written to `/etc/netplan/60-ingress-infra.yaml`.
- GRE tunnels use `/etc/netplan/61-ingress-infra-gre.yaml`.
- BGP VIP is written to `/etc/netplan/62-ingress-infra-vip.yaml`.
- IPv6 forwarding is enabled via `/etc/sysctl.d/60-ingress-infra.conf`.

Apply netplan changes manually if needed:

```bash
sudo netplan apply
```

## BGP (BIRD2)

```bash
sudo systemctl status bird
sudo birdc show protocols
sudo birdc show route
```

## BGP (FRR)

```bash
sudo systemctl status frr
sudo vtysh -c 'show bgp summary'
sudo vtysh -c 'show ip bgp'
```

## Firewall

The Ansible `common` role does not manage `ufw` rules — add rules for your site policy. Ports that
must be reachable:

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------------------------------------|
| 9000 | UDP | inbound | shard-proxy ingress |
| 179 | TCP | in+out | BGP (if `enable_bgp: true`) |
| 9100 | TCP | inbound | Prometheus metrics / health endpoints |

## File locations

| Path | Content |
|---------------------------------------------------|----------------------------------|
| `/usr/local/bin/shard-proxy` | Compiled binary |
| `/etc/shard-proxy/config.env` | Environment variable config file |
| `/etc/systemd/system/shard-proxy.service` | systemd unit |
| `/opt/shard-proxy/` | Source clone and build directory |
| `/etc/bird/bird.conf` | BIRD2 config (if enabled) |
| `/etc/frr/frr.conf` | FRR config (if enabled) |
| `/etc/netplan/60-ingress-infra.yaml` | Egress interface netplan |
| `/etc/sysctl.d/60-ingress-infra.conf` | IPv6 sysctl settings |
