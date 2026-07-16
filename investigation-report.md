# Security Investigation & Firewall Report

**Date:** 2026-07-16  
**Server:** resolve01 (`102.219.211.211`)  
**Period analysed:** 16:41–17:01 UTC (20 minutes)

---

## Firewall Implementation

UFW enabled and persistent (survives reboots). Applied via `sudo ufw --force enable`.

### Rules Summary

| # | Rule | Action | Reason |
|---|------|--------|--------|
| 1 | `102.219.210.213:22/tcp on eno1` | ALLOW IN | SSH management access |
| 2–9 | Port 53 UDP+TCP from 3 authorised subnets | ALLOW IN | DNS client access |
| 10 | `23.92.30.7` → any | DENY IN | External probe IP |
| 11 | `10.250.70.253` → any | DENY IN | Unauthorised internal probe |
| 12–15 | Outbound port 853 TCP+UDP | DENY OUT | Block DNS-over-TLS from this server |

> **Note:** Blocking outbound DoT from end-clients requires rules on the upstream **gateway/router**, as resolve01 does not forward traffic (IP forwarding disabled in FRR config).

---

## Device Investigation: `102.219.209.234`

### Traffic Profile

| Metric | Value |
|--------|-------|
| Total queries (20 min) | 24,731 |
| Unique domains queried | 1,972 |
| Avg QPS | ~1,200/min |
| Peak QPS | 1,593/min (17:01) |
| Pattern | Sustained, multi-threaded — automated |

### Queries per Minute

| Time | QPS |
|------|-----|
| 16:42 | 734 |
| 16:43 | 1,146 |
| 16:44 | 1,378 |
| 16:45 | 1,301 |
| 16:47 | 1,449 |
| 16:50 | 1,371 |
| 16:55 | 1,516 |
| 17:01 | 1,593 ← peak |

---

### Confirmed Suspicious Behaviours

#### 1. Automated DoH Provider Resolution (DNS Bypass Attempts)

The device continuously attempts to resolve DoH provider hostnames to establish encrypted DNS sessions that would bypass this resolver entirely. All are blocked by `local-zone: always_nxdomain`.

| Domain | Queries | Result | Pattern |
|--------|---------|--------|---------|
| `dns.adguard.com` | 84 | NXDOMAIN (blocked) | Every ~20s, up to 16 concurrent queries at 17:01 |
| `cloudflare-dns.com` | 43 | NXDOMAIN (blocked) | Burst of 15 queries in <1 second at 17:00 |
| `dns.google` | Multiple | **NOERROR** (not yet blocked) | Resolves successfully — action required |

**Assessment:** This is **not human-initiated**. The retry frequency, burst patterns across 20 threads, and 20-minute persistence indicate an automated DoH client embedded in a mobile application (likely AdGuard for Android or a VPN app with built-in DoH fallback).

#### 2. Novel DoH Endpoint Discovery

The device also resolves Alibaba Cloud load balancer endpoints used for DoH:
- `nlb-uo231cms7bji7g2doh.ap-southeast-1.nlb.aliyuncsslbintl.com` — queried 4× and **resolving successfully (NOERROR)**. The domain contains `doh` in the NLB name — this is an Alibaba Cloud-hosted DoH endpoint being used by an app to bypass the resolver.

**Recommended block:**
```
local-zone: "aliyuncsslbintl.com." always_nxdomain
```

#### 3. DNS Amplification Probe

| Query | Response | Analysis |
|-------|----------|---------|
| `www.goooooooooooooooooooooooooooooooooooooooooooooooooooooooooogle.com.` | REFUSED | Classic amplification vector test — oversized domain |
| `216.58.202.4.in-addr.arpa.` | REFUSED/NXDOMAIN | Reverse DNS leak test (Google IP) |

These two queries were sent together at **17:00:36** and repeated at **17:01:30** — suggesting an automated test cycle.

#### 4. RFC-1918 Reverse DNS Discovery

Repeated PTR queries for private IP ranges:
- `192.168.1.3`, `192.168.100.21`, `192.168.100.51` — reverse lookups for local network hosts
- `lb._dns-sd._udp.0.100.168.192.in-addr.arpa` — DNS Service Discovery (DNS-SD) multicast probing via unicast DNS

This indicates the device is mapping the local network topology using DNS, consistent with a VPN or MDM application scanning the network.

#### 5. Long/Encoded Domain Names (Potential Data Exfiltration Watch)

| Domain | Queries | Pattern |
|--------|---------|---------|
| `71fbb0ea-2be6-4cd2-a1b0-6d24fad5118a-netseer-ipaddr-assoc.xy.fbcdn.net` | 36 | UUID in hostname — Facebook ad targeting |
| `2c-1138a2e087744ef69e662051fe044c44.ecs.us-west-1.on.aws` | 26 | Hash subdomain — AWS ECS service |
| `a771eb34b3544963a00ee48c2a69abff-9d23df9b5f7bf876.elb.eu-south-1.amazonaws.com` | 22 | AWS ELB hash — legitimate CDN pattern |

Assessment: These are long but **structurally legitimate** CDN/ad-network patterns — not DNS tunnelling indicators. No exfiltration suspected at this time.

---

### Risk Assessment

| Behaviour | Risk | Status |
|-----------|------|--------|
| AdGuard DoH bypass | High | Blocked (NXDOMAIN) |
| Cloudflare DoH bypass | High | Blocked (NXDOMAIN) |
| `dns.google` DoH bypass | High | **Still resolving — action needed** |
| Alibaba DoH endpoint | High | **Still resolving — action needed** |
| Amplification probes | Medium | Blocked (REFUSED by access-control) |
| Local network scanning (PTR/DNS-SD) | Medium | No impact on resolver — monitor |
| Long domain names | Low | Legitimate CDN — no action |

---

## Recommended Next Actions

### Immediate (DNS resolver level)

Add to `/etc/unbound/unbound.conf.d/fixes.conf`:

```yaml
local-zone: "dns.google." always_nxdomain
local-zone: "aliyuncsslbintl.com." always_nxdomain
```

### Network level (upstream gateway/router)

```
# Block outbound DoT from all clients
iptables -I FORWARD -p tcp --dport 853 -j REJECT
iptables -I FORWARD -p udp --dport 853 -j REJECT

# Block outbound DoH (HTTPS to known DoH IPs)
# Cloudflare DoH: 1.1.1.1, 1.0.0.1
iptables -I FORWARD -p tcp -d 1.1.1.1 --dport 443 -j REJECT
iptables -I FORWARD -p tcp -d 1.0.0.1 --dport 443 -j REJECT
# Google DoH: 8.8.8.8, 8.8.4.4
iptables -I FORWARD -p tcp -d 8.8.8.8 --dport 443 -j REJECT
iptables -I FORWARD -p tcp -d 8.8.4.4 --dport 443 -j REJECT
```

### Device level

Identify the device at `102.219.209.234` (MAC lookup on DHCP server), locate it physically, and:
1. Remove AdGuard/DNS-changing VPN apps
2. Disable private DNS settings (`Settings > Network > Private DNS > Off` on Android)
3. Apply MDM policy to prevent DNS override if enterprise-managed

---

## Probe IPs Blocked

| IP | Type | Queries | Block type |
|----|------|---------|-----------|
| `10.250.70.253` | Internal RFC-1918 | 60× `null TYPE0 CLASS0` every ~30s | UFW DENY IN |
| `23.92.30.7` | External public | 1× `null TYPE0 CLASS0` | UFW DENY IN |

The `null TYPE0 CLASS0` query is a standard open resolver scanner fingerprint. Both IPs were correctly REFUSED before firewall blocking; UFW adds an additional layer dropping packets before Unbound sees them.
