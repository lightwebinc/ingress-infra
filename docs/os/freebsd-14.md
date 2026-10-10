# FreeBSD 14

Packages, rc.d and rc.conf conventions, `gif0` tunnels, FRR differences and
diagnostics shared by all infra repositories are in the canonical
[FreeBSD 14 notes](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/os/freebsd-14.md). This page lists what is specific to
`shard-proxy`.

## Service

The rc.d script is rendered from `roles/shard-proxy/templates/shard_proxy.rc.j2`.

```bash
sudo service shard_proxy enable
sudo service shard_proxy start
sudo service shard_proxy status
sudo tail -f /var/log/shard_proxy.log
```

## Networking

- Ingress interface: dual-stack (DHCP + SLAAC) via `ifconfig_<iface>` and
  `ifconfig_<iface>_ipv6`.
- BGP VIPs: `ifconfig_lo0_alias0` (IPv4) and `ifconfig_lo0_alias1` (IPv6).
- Forwarding: `gateway_enable="YES"` and `ipv6_gateway_enable="YES"`.

## Ports

Same as [Ubuntu](ubuntu-24.04.md#ports); ingress-infra does not manage `pf`.

## File locations

| Path | Content |
|---|---|
| `/usr/local/bin/shard-proxy` | Compiled binary |
| `/usr/local/etc/shard-proxy.conf` | Environment config |
| `/usr/local/etc/rc.d/shard_proxy` | rc.d script |
| `/opt/shard-proxy/` | Source clone and build directory |
| `/etc/rc.conf` | Interface and service settings |
