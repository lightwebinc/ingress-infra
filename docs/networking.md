# Networking

## Egress interface modes

`shard-proxy` sends IPv6 UDP multicast datagrams out of one or more named network interfaces.
The `networking` Ansible role configures the egress interface in one of two modes:

| Mode | Variable | Description |
|----------------|-------------------------|-------------------------------------------------|
| Plain ethernet | `egress_mode: ethernet` | Use an existing physical/VLAN interface as-is |
| GRE tunnel | `egress_mode: gre` | Create a GRE tunnel to a remote fabric endpoint |

Set `egress_mode` in `ansible/group_vars/all.yml` or per-host in the inventory.

---

## Plain ethernet

The simplest configuration. The proxy node has a physical or logical interface directly on (or routed
to) the multicast fabric L2 segment. No additional setup is required beyond ensuring IPv6 is enabled
on the interface and that multicast routing/MLD snooping is configured on the fabric switches.

```yaml
egress_mode: ethernet
egress_iface: eth1        # interface name on target host
```

### Ubuntu 24.04 — Netplan snippet

The `networking` role writes `/etc/netplan/60-ingress-infra.yaml`:

```yaml
network:
  version: 2
  ethernets:
    eth1:
      dhcp4: false
      dhcp6: false
      addresses:
        - "2001:db8:1::1/64"
```

### FreeBSD 14 — rc.conf snippet

The role appends to `/etc/rc.conf`:

```text
ifconfig_vtnet1_ipv6="inet6 2001:db8:1::1 prefixlen 64"
```

---

## GRE tunnel

Use GRE when the ingress node connects to the multicast fabric over IP (e.g., a cloud VM reaching a
colocation fabric router). The role creates a GRE interface, assigns an IPv6 address to it, and
configures the routing table so multicast traffic uses the tunnel.

The fabric is **IPv6-only**. The GRE tunnel runs over IPv6 (`ip6gre` on Linux, `gif` on FreeBSD).
The local and remote endpoints are IPv6 addresses.

```yaml
egress_mode: gre
gre_local_ip6: "2001:db8:a::1"    # IPv6 address of this node (tunnel source)
gre_remote_ip6: "2001:db8:a::254" # IPv6 address of the fabric router (tunnel destination)
gre_iface: gre6-bsp               # tunnel interface name
gre_inner_ipv6: "2001:db8:2::2/64"
```

### Ubuntu 24.04

The role creates `/etc/netplan/61-ingress-infra-gre.yaml`:

```yaml
# /etc/netplan/61-ingress-infra-gre.yaml
network:
  version: 2
  tunnels:
    gre6-bsp:
      mode: ip6gre
      local: "2001:db8:a::1"
      remote: "2001:db8:a::254"
      addresses:
        - "2001:db8:2::2/64"
      routes:
        - to: "ff05::/16"
          scope: link
```

### FreeBSD 14

The role appends to `/etc/rc.conf`:

```text
cloned_interfaces="gif0"
ifconfig_gif0="tunnel 2001:db8:a::1 2001:db8:a::254"
ifconfig_gif0_ipv6="inet6 2001:db8:2::2 prefixlen 64"
ipv6_route_bsp_mcast="ff05::/16 -interface gif0"
```

---

## IPv6 multicast routing

The proxy sends datagrams to addresses in the `FF<scope>::/16` range determined by `mc_scope`.
The OS must have a route for that prefix pointing at the egress interface.

### Route prefix derivation

The route prefix is derived automatically from `mc_scope`:

| `mc_scope` | Derived `mc_route_prefix` |
|------------|---------------------------|
| `link` | `ff02::/16` |
| `site` | `ff05::/16` |
| `org` | `ff08::/16` |
| `global` | `ff0e::/16` |

To override (e.g. when `mc_group_id` narrows the address space further):

```yaml
mc_route_prefix: "ff05:0:0:1234::/64"   # explicit prefix, skips auto-derivation
```

Leave `mc_route_prefix: ""` (the default) to use the auto-derived scope prefix.

### Egress hop limit (routed / ip6gre mesh)

The proxy sets `IPV6_MULTICAST_HOPS` on its egress socket from `proxy_egress_hoplimit`
(rendered as `EGRESS_HOPLIMIT`). The default is `1` (a single L2 segment, matching the
binary default); on a routed or `ip6gre`-mesh fabric raise it to `64` so egress multicast
frames survive past the first hop and reach cross-tunnel receivers:

```yaml
proxy_egress_hoplimit: 64   # routed / ip6gre mesh; default 1 dies at hop 1
```

### Multicast route — Ubuntu 24.04

The route is injected as a `routes:` stanza in the egress interface's netplan file
(`/etc/netplan/60-ingress-infra.yaml` or `61-ingress-infra-gre.yaml`) and applied
immediately via `ip -6 route replace`.

```yaml
routes:
  - to: "ff05::/16"   # matches mc_scope: site
    scope: link
```

### Multicast route — FreeBSD 14

The route is persisted via an `ipv6_route_bsp_mcast` entry in `/etc/rc.conf` and applied
immediately via `route add -inet6`:

```text
ipv6_route_bsp_mcast="ff05::/16 -interface eth1"
```

IPv4/IPv6 forwarding is also enabled:

```text
gateway_enable="YES"
ipv6_gateway_enable="YES"
```

---

## Multiple egress interfaces

`shard-proxy` supports comma-separated `-iface` values and fans out each datagram to all
listed interfaces. To use multiple egress interfaces, set `egress_iface` as a list:

```yaml
egress_iface:
  - eth1
  - gre0
```

The role joins the list into a comma-separated string and passes it to the `-iface` flag.

---

## Ingress interface

The ingress (sender-facing) interface is where `shard-proxy` listens for BRC-124/BRC-128 (or legacy BRC-12) frames.
This is typically the default route interface and requires no special configuration beyond reachability
from senders.

```yaml
listen_addr: "[::]"       # bind all interfaces
udp_listen_port: 8725     # UDP ingress (always active)
tcp_listen_port: 0        # TCP ingress for reliable delivery (0 = disabled)
```

Both transports share the same forwarding pipeline; they can run simultaneously. Enable TCP by setting
`tcp_listen_port` to a non-zero port value.

If eBGP is enabled, the ingress interface IP (or BGP VIP) is announced via BGP.
See [bgp.md](bgp.md).

### Miner ingress (privileged frames)

The user ports above are transaction-only. Block announcements (BRC-131),
coinbase (BRC-133), and subtree data (BRC-132) are privileged control-plane
frames that egress to a broadcast group every subscriber receives, so they may
originate only from miner-tier peers. They are no longer submitted as multicast
frames (the miner multicast port is deprecated): blocks and subtrees arrive as
header-stripped **BRC-144** / **BRC-143** push frames on dedicated TCP ports,
disabled by default:

```yaml
subtree_listen_port: 8726     # TCP BRC-143 subtree push ingest; 0 = disabled
block_listen_port: 8727       # TCP BRC-144 block push ingest; 0 = disabled
```

The proxy reframes each into the fabric internally. These push ports **must** be
reachable only by miner-tier sources — enforce this at the host firewall /
provider security group (allow them only from the miner tunnel CIDRs, tunnel-
bound), since `ingress-infra` does not manage a host firewall itself. Only the
transaction port (8725) stays open to all senders. See
[bsv-multicast architecture.md § Teranode Relationship](https://github.com/lightwebinc/bsv-multicast/blob/main/multicast-skills/architecture.md).

## Dedup backend connectivity

The tier-2 ingress dedup backend (`txid_dedup_backend`) is an out-of-band TCP
service on the management network: Redis/Valkey/Dragonfly on 6379, or Aerospike
Community Edition on 3000 (client). It is independent of the multicast fabric.
Backend errors fail open — the proxy forwards the frame and records a metric —
so a backend outage never stops ingress. See
[shard-common cache backend](https://github.com/lightwebinc/shard-common/blob/main/docs/cache-backend.md).
