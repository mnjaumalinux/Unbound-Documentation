# resolve01 — Anycast DNS Resolver

## Overview

`resolve01` is a high-performance, security-hardened anycast DNS resolver running Unbound,
advertised via iBGP to the upstream network. DNS queries are served on the anycast address
`102.219.211.211`, enabling transparent failover and geographic load distribution.

---

## System Details

| Property         | Value                    |
|------------------|--------------------------|
| Hostname         | resolve01                |
| Physical IP      | 102.219.210.213/31 (eno1)|
| Anycast DNS IP   | 102.219.211.211/32 (lo1) |
| BGP AS           | 328856                   |
| BGP Peer         | 102.219.210.212          |
| OS               | Ubuntu 24.04 LTS         |
| CPUs             | 24                       |
| RAM              | 16 GB                    |

---

## Architecture

```
Clients
  │
  ▼ DNS queries to 102.219.211.211:53
┌─────────────────────────────────┐
│           resolve01             │
│                                 │
│  lo1: 102.219.211.211/32        │  ← anycast dummy interface (systemd-networkd)
│  eno1: 102.219.210.213/31       │  ← physical uplink / BGP session source
│                                 │
│  Unbound (port 53)              │  ← recursive resolver
│  FRR BGP (AS 328856)            │  ← advertises 102.219.211.211/32 upstream
└─────────────────────────────────┘
  │
  ▼ iBGP to 102.219.210.212 (AS 328856)
Upstream Router
```

---

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/systemd/network/10-lo1.netdev` | Defines `lo1` dummy interface |
| `/etc/systemd/network/10-lo1.network` | Assigns `102.219.211.211/32` to `lo1` |
| `/etc/frr/frr.conf` | FRR BGP configuration |
| `/etc/frr/daemons` | FRR daemon enablement (`bgpd=yes`) |
| `/etc/unbound/unbound.conf.d/recursive-resolver.conf` | Unbound resolver configuration |
| `/etc/ssh/sshd_config` | SSH restricted to `102.219.210.213` |
| `/etc/systemd/system/ssh.socket.d/override.conf` | SSH socket binding override |

---

## Loopback Interface (lo1)

The anycast prefix is hosted on a persistent dummy interface managed by `systemd-networkd`.

**`/etc/systemd/network/10-lo1.netdev`**
```ini
[NetDev]
Name=lo1
Kind=dummy
```

**`/etc/systemd/network/10-lo1.network`**
```ini
[Match]
Name=lo1

[Network]
Address=102.219.211.211/32
```

---

## BGP Configuration (FRR 8.4.4)

iBGP session to upstream peer `102.219.210.212`. The anycast prefix `102.219.211.211/32`
is originated locally and advertised upstream. Session is sourced from the physical interface
IP `102.219.210.213`.

```
router bgp 328856
 bgp router-id 102.219.211.211
 bgp log-neighbor-changes
 neighbor 102.219.210.212 remote-as 328856
 neighbor 102.219.210.212 description anycast-dns-upstream
 neighbor 102.219.210.212 update-source 102.219.210.213
 !
 address-family ipv4 unicast
  network 102.219.211.211/32
  neighbor 102.219.210.212 next-hop-self
  neighbor 102.219.210.212 soft-reconfiguration inbound
 exit-address-family
```

**Upstream peer must configure:**
```
neighbor 102.219.210.213 remote-as 328856
```

---

## Unbound Configuration

Unbound listens on `102.219.211.211:53` (anycast) and `127.0.0.1:53` (local resolver).

### Performance Tuning (24 CPU / 16 GB RAM)

| Parameter | Value | Reason |
|-----------|-------|--------|
| `num-threads` | 24 | Match CPU count |
| `so-reuseport` | yes | Per-thread socket distribution |
| `so-rcvbuf / so-sndbuf` | 4m | Large socket buffers for burst traffic |
| `outgoing-range` | 8192 | Concurrent outbound queries per thread |
| `num-queries-per-thread` | 4096 | In-flight query capacity |
| `msg-cache-size` | 512m | Large answer cache |
| `rrset-cache-size` | 1024m | Large RRset store |
| `key-cache-size` | 256m | DNSSEC key cache |
| `neg-cache-size` | 128m | Negative answer cache |
| `*-cache-slabs` | 32 | Reduce lock contention (power of 2 ≥ threads) |
| `infra-cache-numhosts` | 100000 | Track more upstream nameservers |
| `serve-expired` | yes | Serve stale cache during upstream refresh |
| `prefetch` | yes | Refresh popular records before TTL expiry |
| `minimal-responses` | yes | Strip unnecessary DNS sections |
| `cache-min-ttl` | 300s | Prevent excessive re-querying |
| `cache-max-ttl` | 86400s | Cap TTL to 24h |

### Security Hardening

| Feature | Setting | Description |
|---------|---------|-------------|
| DNSSEC | `harden-dnssec-stripped: yes` | Reject stripped DNSSEC |
| DNSSEC | `harden-algo-downgrade: yes` | Prevent algorithm downgrade |
| DNSSEC | `harden-below-nxdomain: yes` | Enforce NXDOMAIN below cut |
| DNSSEC | `aggressive-nsec: yes` | Synthesise NXDOMAIN from NSEC |
| DNSSEC | `val-clean-additional: yes` | Strip unvalidated additional data |
| Privacy | `qname-minimisation: yes` | Minimise labels sent to upstreams |
| Privacy | `hide-identity / hide-version: yes` | Suppress server info |
| Privacy | `use-caps-for-id: yes` | 0x20 encoding against spoofing |
| Abuse | `deny-any: yes` | Block ANY queries (DDoS amplifier) |
| Abuse | `ratelimit: 1000` | Limit queries/s per authoritative zone |
| Abuse | `ip-ratelimit: 1000` | Limit queries/s per source IP |
| Rebinding | `private-address` blocks | Prevent DNS rebinding to RFC-1918/link-local |
| Glue | `harden-glue: yes` | Reject out-of-bailiwick glue |

### Access Control

Only the following subnets may query the resolver:

| Network | Description |
|---------|-------------|
| `127.0.0.0/8` | Localhost |
| `::1/128` | Localhost IPv6 |
| `102.219.208.0/22` | Authorised client range |
| `102.213.48.0/22` | Authorised client range |
| `102.209.56.0/22` | Authorised client range |

All other sources receive `REFUSED`.

---

## SSH

SSH is restricted to the physical interface only to prevent management access via the anycast address.

- **Listens on:** `102.219.210.213:22`
- **Config:** `/etc/ssh/sshd_config` (`ListenAddress 102.219.210.213`)
- **Socket override:** `/etc/systemd/system/ssh.socket.d/override.conf`

---

## Performance Test Results

**Tool:** `dnsperf 2.14.0`  
**Date:** 2026-07-16  
**Test:** 800 QPS target, 10 concurrent clients, 30 second duration, 30-domain query set (warmed cache)

| Metric | Result |
|--------|--------|
| Queries sent | 24,000 |
| Queries completed | 24,000 **(100%)** |
| Queries lost | 0 **(0.00%)** |
| Actual QPS | **800 QPS** |
| Average latency | **0.67 ms** |
| Minimum latency | 0.05 ms (cache hit) |
| Maximum latency | 485 ms (cold upstream) |
| Latency std dev | 10.7 ms |

> **Note on SERVFAILs at high QPS:** When testing above 1000 QPS from a single source IP,
> `ip-ratelimit` intentionally throttles excess queries. This is expected behaviour — in
> production, traffic arrives from many client IPs and each is subject to its own 1000 QPS
> limit, so the effective aggregate capacity far exceeds 1000 QPS.

---

## Service Status Verification

```bash
# Interface
networkctl status lo1

# BGP session and prefix advertisement
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show bgp ipv4 unicast"

# DNS resolution
dig @102.219.211.211 google.com +short

# Unbound stats
sudo unbound-control stats_noreset | grep -E "total\.|cache"

# All services enabled
systemctl is-enabled systemd-networkd unbound frr ssh.socket
```

---

## Boot Order

On reboot, services start in this order:

1. `systemd-networkd` — brings up `lo1` with `102.219.211.211/32`
2. `unbound` — binds to `102.219.211.211:53` and `127.0.0.1:53`
3. `frr` — establishes iBGP to `102.219.210.212`, advertises `102.219.211.211/32`
4. `ssh.socket` — accepts connections on `102.219.210.213:22`
