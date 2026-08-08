# Architecture

## Overview

`ingress-infra` deploys and configures `shard-proxy` nodes that form the ingress tier of a
Bitcoin SV multicast distribution fabric. Each node:

1. Receives BSV transaction frames (BRC-124/BRC-128 or legacy BRC-12) from senders on the public internet (UDP by default; TCP ingress is optional for reliable delivery).
2. Derives an IPv6 multicast group address from the transaction ID shard key.
3. Retransmits the datagram to the derived group over one or more egress interfaces connected to the
   multicast fabric.

Because the proxy is fully stateless and deterministic (same txid always maps to the same group), any
number of ingress nodes can run simultaneously without coordination. Nodes are horizontally scalable
and individually replaceable.

## Network tiers

```text
                        ┌─────────────────────────────────────────┐
                        │  BSV Senders (miners, services)         │
                        └────────────┬────────────────────────────┘
                                     │  UDP / TCP (BRC-124/BRC-128 or legacy BRC-12 frames)
                    ┌────────────────┼────────────────┐
                    │                │                │
              ┌─────▼──┐       ┌─────▼──┐       ┌─────▼──┐
              │ingress │       │ingress │       │ingress │   ← ingress-infra nodes
              │node A  │       │node B  │       │node C  │     (this repo)
              └─────┬──┘       └─────┬──┘       └─────┬──┘
                    │  IPv6 UDP multicast  FF05::<shard>
                    └────────────────┼────────────────┘
                                     │  (GRE tunnel or ethernet)
                        ┌────────────▼────────────────────────────┐
                        │         Multicast fabric                │
                        │  (site-scoped, FF05::/16)               │
                        └────┬──────────┬──────────┬──────────────┘
                             │          │          │
                        ┌────▼──┐  ┌────▼──┐  ┌────▼──┐
                        │miners │  │exch-  │  │other  │   ← direct multicast subscribers
                        │       │  │anges  │  │SVPs   │     (join shard groups)
                        └───────┘  └───────┘  └───┬───┘
                                                   │  shard-listener
                                              ┌────▼──────────────┐
                                              │ downstream unicast │   ← filtered + forwarded
                                              │ consumers          │     over UDP or TCP
                                              └───────────────────┘
```

## Protocol details

Deploys `shard-proxy`, which handles BRC-12, BRC-124/BRC-128 (tx frames),
BRC-130 (fragmentation), BRC-131 (block announce), BRC-132 (subtree data),
BRC-133 (coinbase), BRC-134 (anchor transactions), and BRC-142 (coalesced
bundle frames). Frame formats, shard derivation, subtree filtering,
and HashKey/SeqNum stamping are documented in the service and project repos:

- [shard-proxy — Architecture](https://github.com/lightwebinc/shard-proxy/blob/main/docs/architecture.md)
- [Wire Protocol Specification](https://github.com/lightwebinc/shard-common/blob/main/docs/protocol.md)
- [bsv-multicast — DESIGN.md](https://github.com/lightwebinc/bsv-multicast/blob/main/DESIGN.md)
- BRC specifications: `bsv-multicast/docs/brc-{124,126,127,128,129,130,131,132,133,134,135,139,142,143,144,148,149}-*.md`

## BGP ingress (optional)

When `enable_bgp: true`, each ingress node announces IPv4 or IPv6 prefixes via eBGP
to its upstream provider. All nodes can announce the same prefixes, so senders are routed to the
topologically nearest proxy by BGP best-path selection.

```text
Sender ──BGP──► nearest ingress node ──multicast──► fabric
```

See [bgp.md](bgp.md) for configuration details.

## Egress interface options

| Mode | When to use |
|----------------|-----------------------------------------------------------------------|
| Plain ethernet | Ingress node is directly layer-2 adjacent to multicast fabric |
| GRE tunnel | Ingress node connects to fabric over IP (cloud VM, remote colocation) |

See [networking.md](networking.md) for interface configuration.

## Deployment topology examples

### Minimal (single node, ethernet egress)

```text
internet ──[eth0]── proxy node ──[eth1]── multicast fabric
```

### Multi-node BGP pool (GRE egress)

```text
internet ──BGP──► node-A ──GRE──┐
                    ► node-B ──GRE──┼──► fabric router ──► fabric
                    ► node-C ──GRE──┘
```

## OS support

| OS | Service manager | Network config |
|--------------|-----------------|------------------------|
| Ubuntu 24.04 | systemd | Netplan / ip commands |
| Debian 13 | systemd | Netplan (installed by role) / ip commands |
| FreeBSD 14 | rc.d | rc.conf / ifconfig/gre |
