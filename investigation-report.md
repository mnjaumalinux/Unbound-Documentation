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

## Investigation: `102.219.209.234` — MikroTik NAT Gateway

> **Correction from initial assessment:** `102.219.209.234` is a **MikroTik router performing NAT** for a downstream client network. All queries attributed to this IP are from multiple end-user devices behind the NAT. It is an authorised gateway — not a rogue device. Investigation findings below have been updated accordingly.

### Traffic Profile

| Metric | Value |
|--------|-------|
| Source | MikroTik NAT gateway (multiple clients masked behind single IP) |
| Total queries (20 min) | 24,731 |
| Unique domains queried | 1,972 (across all NAT'd clients) |
| Avg QPS | ~1,200/min |
| Peak QPS | 1,593/min (17:01) |
| Pattern | Normal aggregate traffic for a multi-user NAT network |

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

#### 1. DoH Provider Resolution (DNS Bypass Attempts from NAT clients)

One or more devices **behind the MikroTik NAT** are running applications with embedded DoH clients attempting to bypass this resolver. All provider hostnames are blocked by `local-zone: always_nxdomain` on resolve01.

| Domain | Queries | Result | Pattern |
|--------|---------|--------|---------|
| `dns.adguard.com` | 84 | NXDOMAIN (blocked) | Every ~20s — automated app retry (AdGuard for Android / VPN app) |
| `cloudflare-dns.com` | 43 | NXDOMAIN (blocked) | Burst of 15 queries in <1s — multiple apps retrying simultaneously |
| `dns.google` | Multiple | NOERROR (not blocked — per operator decision) | Google DNS left resolvable |

**Assessment:** Originating from one or more end-user devices behind the MikroTik. The high retry rate and multi-thread bursts indicate **automated DoH fallback logic** in a mobile app (likely AdGuard for Android, a VPN client, or browser with DoH enabled). The MikroTik is the correct enforcement point for blocking this traffic.

#### 2. Novel DoH Endpoint Discovery

A client behind the MikroTik resolves Alibaba Cloud load balancer endpoints used for DoH:
- `nlb-uo231cms7bji7g2doh.ap-southeast-1.nlb.aliyuncsslbintl.com` — queried 4× (NOERROR). The `doh` in the NLB name confirms this is an Alibaba Cloud-hosted DoH endpoint used by an app to bypass the resolver.

**Recommended block:**
```
local-zone: "aliyuncsslbintl.com." always_nxdomain
```

#### 3. DNS Amplification Probe

| Query | Response | Analysis |
|-------|----------|---------|
| `www.goooooooooooooooooooooooooooooooooooooooooooooooooooooooooogle.com.` | REFUSED | Classic amplification vector test — oversized domain |
| `216.58.202.4.in-addr.arpa.` | REFUSED/NXDOMAIN | Reverse DNS leak test (Google IP) |

These queries were sent at **17:00:36** and repeated at **17:01:30** — an automated test cycle from a client behind the NAT running a network diagnostic/security tool. Resolver correctly returned REFUSED.

#### 4. RFC-1918 Reverse DNS Discovery

Repeated PTR queries for private IP ranges from NAT clients:
- `192.168.1.3`, `192.168.100.21`, `192.168.100.51` — reverse lookups for local network hosts
- `lb._dns-sd._udp.0.100.168.192.in-addr.arpa` — DNS Service Discovery (DNS-SD) probing via unicast DNS

This is **normal behaviour** for devices on the MikroTik's LAN (mDNS/DNS-SD service discovery is common on Android/iOS/macOS). No action required.

#### 5. Long/Encoded Domain Names (Potential Data Exfiltration Watch)

| Domain | Queries | Pattern |
|--------|---------|---------|
| `71fbb0ea-2be6-4cd2-a1b0-6d24fad5118a-netseer-ipaddr-assoc.xy.fbcdn.net` | 36 | UUID in hostname — Facebook ad targeting |
| `2c-1138a2e087744ef69e662051fe044c44.ecs.us-west-1.on.aws` | 26 | Hash subdomain — AWS ECS service |
| `a771eb34b3544963a00ee48c2a69abff-9d23df9b5f7bf876.elb.eu-south-1.amazonaws.com` | 22 | AWS ELB hash — legitimate CDN pattern |

Assessment: These are long but **structurally legitimate** CDN/ad-network patterns — not DNS tunnelling indicators. No exfiltration suspected at this time.

---

### Risk Assessment

| Behaviour | Source | Risk | Status |
|-----------|--------|------|--------|
| AdGuard DoH bypass | Client(s) behind MikroTik | High | Blocked at resolver (NXDOMAIN) |
| Cloudflare DoH bypass | Client(s) behind MikroTik | High | Blocked at resolver (NXDOMAIN) |
| `dns.google` resolution | Client(s) behind MikroTik | Low | Allowed per operator decision |
| Alibaba Cloud DoH endpoint | Client(s) behind MikroTik | High | Resolving — block at MikroTik or resolver |
| Amplification probes | Client behind MikroTik | Medium | Blocked (REFUSED by access-control) |
| PTR/DNS-SD queries | Normal LAN device behaviour | Low | No action needed |
| Long domain names | Normal CDN patterns | Low | No action needed |

---

## Recommended Next Actions

### 1. MikroTik router — enforce DNS and block DoT (highest priority)

Apply these rules on the MikroTik to intercept all client DNS and block DoT:

```routeros
# Force all client DNS queries to resolve01
/ip firewall nat
add chain=dstnat protocol=udp dst-port=53 src-address=!102.219.211.211 \
    action=dst-nat to-addresses=102.219.211.211 comment="Force DNS to resolve01"
add chain=dstnat protocol=tcp dst-port=53 src-address=!102.219.211.211 \
    action=dst-nat to-addresses=102.219.211.211 comment="Force DNS to resolve01 TCP"

# Block outbound DNS-over-TLS (port 853)
/ip firewall filter
add chain=forward protocol=tcp dst-port=853 action=drop comment="Block DoT TCP"
add chain=forward protocol=udp dst-port=853 action=drop comment="Block DoT UDP"

# Block outbound DoH to known provider IPs
add chain=forward protocol=tcp dst-address=1.1.1.1 dst-port=443 action=drop comment="Block Cloudflare DoH"
add chain=forward protocol=tcp dst-address=1.0.0.1 dst-port=443 action=drop comment="Block Cloudflare DoH"
add chain=forward protocol=tcp dst-address=8.8.8.8 dst-port=443 action=drop comment="Block Google DoH"
add chain=forward protocol=tcp dst-address=8.8.4.4 dst-port=443 action=drop comment="Block Google DoH"
add chain=forward protocol=tcp dst-address=9.9.9.9 dst-port=443 action=drop comment="Block Quad9 DoH"
```

### 2. Resolver level (optional, additional hardening)

Add to `/etc/unbound/unbound.conf.d/fixes.conf` if Alibaba DoH blocking is desired:

```
local-zone: "aliyuncsslbintl.com." always_nxdomain
```

### 3. Client identification

To identify which client behind the MikroTik is generating DoH bypass traffic:
```routeros
# Enable connection tracking logging on MikroTik for port 853
/ip firewall filter
add chain=forward protocol=tcp dst-port=853 action=log log-prefix="DoT-attempt" \
    comment="Log DoT attempts for client identification"
```
Then check `/log` on the MikroTik to see the source IP of the DoH-bypassing client.

---

## Probe IPs Blocked

| IP | Type | Queries | Block type |
|----|------|---------|-----------|
| `10.250.70.253` | Internal RFC-1918 | 60× `null TYPE0 CLASS0` every ~30s | UFW DENY IN |
| `23.92.30.7` | External public | 1× `null TYPE0 CLASS0` | UFW DENY IN |

The `null TYPE0 CLASS0` query is a standard open resolver scanner fingerprint. Both IPs were correctly REFUSED before firewall blocking; UFW adds an additional layer dropping packets before Unbound sees them.
