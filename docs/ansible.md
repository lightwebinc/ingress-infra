# Ansible

Shared workflow, requirements, variable precedence, version pins, the `common`
and `perf-tuning` roles, upgrades and known issues are documented once in
[Ansible operations](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/ansible-operations.md). This page covers what is
specific to `shard-proxy`.

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

## Roles

| Role | Purpose |
|-----------------------|-----------------------------------------------------------|
| `common` | OS packages, Go toolchain install, build dependencies; journald cap + scheduled disk reclaim (Linux); opt-in OS patching (`--tags os_update`) |
| `perf-tuning` | High-PPS host tuning: UDP buffers, busy-poll, txqueuelen, deep C-state disable, irqbalance off |
| `shard-proxy` | Clone, build, install binary, configure service unit |
| `networking` | Ethernet or GRE egress interface, IPv6 multicast routing |
| `bgp` | BIRD2 or FRR install, config template, health-check timer |
| `bgp-ibgp` | iBGP daemon on upstream peer nodes (separate playbook: `bgp-ibgp.yml`) |

Roles are applied in the order listed by `site.yml`. The `bgp` role is skipped when `enable_bgp: false`. The `bgp-ibgp` role runs via its own playbook (`ansible-playbook -i inventory/hosts.yml bgp-ibgp.yml`), not `site.yml`.

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

Bump `proxy_version` in `group_vars/all.yml` (and the Terraform default, see
[terraform.md](terraform.md)) and run `--tags proxy`. Forced rebuilds
(`proxy_force_build`) and pre-built binaries (`proxy_local_binary`) are
described in [Ansible operations](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/ansible-operations.md#build-upgrade-and-pre-built-binaries).

---

## Service variables

Every variable in `group_vars/all.yml`, with its default. Topic guides:
egress interfaces, GRE, multicast routing, push-frame ingest and dedup backend in
[networking.md](networking.md); eBGP / iBGP and `bgp_health_path` in
[bgp.md](bgp.md).

### shard-proxy source and build

| Variable | Default | Notes |
|---|---|---|
| `proxy_repo` | `https://github.com/lightwebinc/shard-proxy.git` | Git source of the service |
| `proxy_version` | pinned in `group_vars/all.yml` | Release tag to build; see [version pins](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/infra/ansible-operations.md#version-pins) |
| `proxy_install_dir` | `/opt/shard-proxy` | Clone and build directory |
| `proxy_bin_dir` | `/usr/local/bin` | Binary install directory |
| `proxy_user` | `shard-proxy` | Service user |
| `proxy_group` | `shard-proxy` | Service group |
| `proxy_local_binary` | `""` | Pre-built local binary to push; empty = clone and build on the host |
| `proxy_force_build` | `false` | Rebuild even when a binary already exists |
| `go_version` | pinned in `group_vars/all.yml` | Go toolchain; must be at or above the `go` directive in the service go.mod at the pinned tag |
| `go_install_dir` | `/usr/local/go` | Go toolchain install directory |

### Proxy runtime configuration

| Variable | Default | Notes |
|---|---|---|
| `listen_addr` | `[::]` |  |
| `udp_listen_port` | `8725` |  |
| `tcp_listen_port` | `0` |  |
| `subtree_listen_port` | `0` | Push-frame ingest (replaces the deprecated miner multicast port, 2026-07-07). The user ports above are transaction-only; blocks and subtrees enter as header-stripped BRC-144 / BRC-143 PUSH frames on dedicated TCP ... |
| `block_listen_port` | `0` |  |
| `beef_listen_port` | `0` | BRC-148 BEEF object plane. BEEF is an OPEN class: submission records (0xBEEF-tagged) and framed FrameVer 0x09 ride the public tx port 8725 regardless of beef_listen_port, which only opens an OPTIONAL dedicated lane ... |
| `beef_shard_bits` | `0` |  |
| `beef_max_object_bytes` | `1048576` |  |
| `require_block_pow` | `true` | Permissionless PoW gate on BRC-131 block announces (validates work, not identity). DEFAULT ON — matches the shard-proxy binary default (-require-block-pow / REQUIRE_BLOCK_POW = true). Set false only for lab fixtures ... |
| `min_pow_bits` | `0` | Difficulty floor for the gate, in Bitcoin compact nBits form: "0x1d00ffff"  mainnet / testnet "0x207fffff"  devnet / regtest "0"           no floor — header self-consistency only (still gated) |
| `egress_port` | `9001` |  |
| `proxy_egress_hoplimit` | `1` | IPV6_MULTICAST_HOPS on egress. 1 = single L2 segment (binary default, non-breaking). Set to 64 for a routed / ip6gre-mesh fabric so egress multicast frames survive past the first hop (cross-tunnel). |
| `egress_multicast_loop` | `false` | IPV6_MULTICAST_LOOP on egress. Only needed on collapsed/mesh router nodes so locally-originated multicast is forwarded by the kernel MFC. |
| `shard_bits` | `2` |  |
| `mc_scope` | `site` |  |
| `mc_group_id` | `0x000B` |  |
| `source_mode` | `asm` | Multicast addressing model: asm (default) \| ssm (FF3x::/32 per RFC 4607; requires PIM-SSM in the fabric). bind_source is required and MUST be a unique IPv6 literal per replica when source_mode is ssm. |
| `bind_source` | `""` |  |
| `stamp_source` | `true` | Authoritatively stamp the BRC HashKey from the observed packet source IP. Set false only behind a source-rewriting load balancer. |
| `num_workers` | `0` |  |
| `recv_batch` | `32` | Datagrams per recvmmsg syscall (1 = per-packet legacy path). |
| `metrics_addr` | `:9100` |  |
| `otlp_endpoint` | `""` |  |
| `otlp_interval` | `30s` |  |
| `log_format` | `json` | text \| json (json for fleet aggregation; collector phase is deferred) |
| `log_level` | `info` | debug\|info\|warn\|error; runtime-togglable via POST /loglevel + SIGHUP |
| `trace_sampling` | `0` | 0..1 trace head sampling (0 = off; exports via otlp_endpoint; control-plane only) |
| `drain_timeout` | `0s` | Pre-drain delay on shutdown; set to ≥ LB health-check interval in production |
| `frag_mtu` | `1400` | BRC-130 fragmentation MTU. ON by default — 0 does NOT mean "send frames unfragmented", it means every payload above (mtu-140) becomes a single oversize datagram the fabric cannot carry: a silent 1360-byte ceiling at ... |
| `coalesce` | `false` | opt-in (binary default off) |
| `coalesce_max_bytes` | `1500` | max bundle datagram size (1500 Ethernet, 9000 jumbo) |
| `coalesce_max_members` | `0` | max member txs per bundle (0 = MTU-bound) |
| `coalesce_carry_txid` | `false` | carry per-member TxID (dedup/billing) vs recompute |
| `proxy_debug` | `false` |  |
| `txid_dedup_local_cap` | `1048576` | tier-1 LRU capacity; 0 = disable |
| `txid_dedup_backend` | `""` | redis\|aerospike\|memory\|none; empty infers redis when addr set, else none |
| `txid_dedup_redis_addr` | `""` | Redis/Valkey/Dragonfly addr; empty = local-only |
| `txid_dedup_aerospike_hosts` | `""` | comma-separated host:port (required when backend=aerospike) |
| `txid_dedup_aerospike_namespace` | `cache` |  |
| `txid_dedup_aerospike_set` | `bsp` |  |
| `txid_dedup_prefix` | `bsp:tx:` | key prefix — must match listener's ingress_set_prefix |
| `txid_dedup_ttl` | `10m` | tier-2 entry TTL |
| `proxy_instance_id` | `""` | INSTANCE_ID: OTel service.instance.id (empty = hostname) |
| `proxy_retry_tee` | `""` | BSP_RETRY_TEE: mirror egress DATA to a co-located retry endpoint's tee, e.g. "[::1]:9001" (collapsed edge) |
| `verify_subtree_root` | `true` | VERIFY_SUBTREE_ROOT: drop BRC-132 subtrees whose hashes miss the root |
| `verify_payload_hash` | `false` | VERIFY_PAYLOAD_HASH: check framed TxID against payload |
| `require_ef` | `false` | REQUIRE_EF: admit Extended Format (BRC-30) submissions only |
| `allow_stamped_ingress` | `false` | ALLOW_STAMPED_INGRESS: accept already-sequenced frames (relay / spine collect lane) |
| `ingress_dedup` | `true` | INGRESS_DEDUP: false bypasses ingress TxID dedup entirely |
| `recv_buf_bytes` | `0` | BSP_RECV_BUF_BYTES: per-worker SO_RCVBUF (0 = worker default) |

### Networking

| Variable | Default | Notes |
|---|---|---|
| `egress_mode` | `ethernet` | ethernet \| gre |
| `egress_iface` | `eth1` | interface name or list of names |
| `mc_route_prefix` | `""` | Multicast route prefix for the egress interface. Defaults to "" which means auto-derive from mc_scope: link   -> ff02::/16 site   -> ff05::/16 org    -> ff08::/16 global -> ff0e::/16 Override this when using a ... |
| `gre_local_ip6` | `""` | IPv6 address of the local tunnel endpoint |
| `gre_remote_ip6` | `""` | IPv6 address of the remote tunnel endpoint |
| `gre_iface` | `gre6-bsp` | interface name for the tunnel |
| `gre_inner_ipv6` | `""` | IPv6 address/prefix assigned inside the tunnel |

### BGP (optional)

| Variable | Default | Notes |
|---|---|---|
| `enable_bgp` | `false` |  |
| `bgp_daemon` | `bird2` | bird2 \| frr |
| `bgp_prefix` | `[]` | IPv4 BGP prefixes announced by all nodes, e.g. ["192.0.2.0/24"] |
| `bgp_vip` | `""` | IPv4 loopback VIP, e.g. "192.0.2.1" |
| `bgp_prefix6` | `[]` | IPv6 BGP prefixes announced by all nodes, e.g. ["2001:db8::/48"] |
| `bgp_vip6` | `""` | IPv6 loopback VIP, e.g. "2001:db8::1" |
| `bgp_local_as` | `65001` |  |
| `bgp_peer_as` | `65000` |  |
| `bgp_peer_ip` | `""` | IPv4 BGP peer address (leave empty if peer is v6-only) |
| `bgp_peer_ip6` | `""` | IPv6 BGP peer address |
| `bgp_router_id` | (templated) |  |
| `bgp_hold_time` | `90` |  |
| `bgp_keepalive` | `30` |  |
| `bgp_password` | `""` |  |
| `bgp_health_path` | `/healthz` | Health path the bsp-bgp-check timer probes to enable/withdraw the anycast VIP. /healthz (liveness) = withdraw when the proxy is dead; /readyz = withdraw at drain start (graceful anycast shed for a hostNetwork proxy ... |
| `bgp_ibgp_peers` | `[]` | iBGP peers (used by the bgp-ibgp role, playbook: bgp-ibgp.yml) Each entry: { peer_ip: "", peer_ip6: "", description: "" } At least one of peer_ip or peer_ip6 is required per entry. Include both for dual-stack peers. |

