# Unbound Traffic Analysis Report

**Date:** 2026-07-16  
**Server:** resolve01 (`102.219.211.211`)  
**Analysis window:** ~24 minutes post-enable of query logging  
**Total queries:** 3,632

---

## Aggregate Statistics

| Metric | Value | Notes |
|--------|-------|-------|
| Total queries | 3,632 | |
| Cache hits | 1,917 (52.8%) | Rising as cache warms |
| Cache misses | 1,715 (47.2%) | Expected at early uptime |
| Prefetches | 179 | Proactive record refresh working |
| Stale cache served | 101 | `serve-expired` active |
| Timed out queries | 0 | Healthy |
| Request list exceeded | 131 | Burst saturation — see anomalies |
| Request list max depth | 92 | Peak concurrency |
| Avg recursion time | 436ms | Cold cache — expected to drop |
| Median recursion time | 299ms | Upstream latency to root/TLD |
| IP rate-limited queries | 0 | No abuse detected |

---

## Active Clients

| Source IP | Subnet | Profile |
|-----------|--------|---------|
| `102.209.56.4` | `102.219.208.0/22` | Light enterprise — Kenyan regional domains |
| `102.219.209.234` | `102.219.208.0/22` | Heavy mobile/Android — social, streaming, NTP |

---

## Traffic Breakdown by Category

| Category | Example Domains | Estimated Share |
|----------|----------------|-----------------|
| Google APIs & CDN | `googleapis.com`, `gstatic.com`, `googlevideo.com` | ~35% |
| TikTok | `tiktokv.com`, `tiktokcdn.com`, `pangle.io`, `capcutapi.com` | ~20% |
| Facebook / Instagram / WhatsApp | `graph.facebook.com`, `fbcdn.net`, `g.whatsapp.net` | ~15% |
| NTP | `pool.ntp.org`, `time.android.com`, `time.nist.gov` | ~10% |
| YouTube | `www.youtube.com`, `i.ytimg.com`, `youtubei.googleapis.com` | ~8% |
| Spotify | `gew1-spclient.spotify.com`, `gew1-dealer.g2.spotify.com` | ~3% |
| Microsoft | `login.microsoftonline.com`, `config.edge.skype.com` | ~2% |
| Ads / Analytics | `googleads.g.doubleclick.net`, `app-measurement.com`, `taboola.com` | ~2% |
| Other | NTP CDNs, Firebase, Samsung, Xiaomi, Konami, Zendesk | ~5% |

---

## Performance Anomalies

### 1. Request List Exceeded (131 queries dropped)
- **Severity:** Medium
- **Metric:** `total.requestlist.exceeded = 131`, `max = 92`
- **Cause:** Short burst from `102.219.209.234` exhausted the per-thread request queue during initial NTP + app startup storm
- **Recommendation:** Increase `num-queries-per-thread` from 4096 → 8192 in Unbound config

### 2. `i.ytimg.com` Persistent SERVFAIL
- **Severity:** Medium
- **Detail:** YouTube image CDN domain fails repeatedly across multiple threads and query cycles. Pattern indicates a broken delegation or DNSSEC issue in the `ytimg.com` zone at the authoritative level
- **Impact:** YouTube thumbnails/images failing to load for clients
- **Recommendation:** Monitor for 24h; if persistent, add `domain-insecure: ytimg.com` to bypass DNSSEC validation for this zone

### 3. `pool.ntp.org` SERVFAIL Burst
- **Severity:** Low
- **Cause:** Multiple devices initiated NTP sync simultaneously, triggering the `ratelimit: 1000` per-zone limit for `org.` zone
- **Impact:** Transient — self-resolved as infra cache populated
- **Recommendation:** Consider adding `ratelimit-below-domain: pool.ntp.org 0` to whitelist NTP traffic

### 4. High Recursion Latency (436ms avg)
- **Severity:** Low (transient)
- **Cause:** Cold cache post-restart; upstream round-trip to root and TLD servers
- **Expected:** Latency to drop to <100ms avg as cache fills over the next 1–2 hours

---

## Security Anomalies

### 1. `.bbrouter` DNS Search Domain (Client Misconfiguration)
- **Severity:** Informational
- **Source:** `102.219.209.234`
- **Detail:** Android device appending `.bbrouter` to every query (e.g. `login.microsoftonline.com.bbrouter`, `g.whatsapp.net.bbrouter`). This is a router-injected DNS search suffix being pushed to the client via DHCP
- **Impact:** Generates spurious NXDOMAIN queries; increases query volume ~10-15%
- **Recommendation:** Remove `.bbrouter` from the DHCP `search` domain option on the upstream router/DHCP server

### 2. Direct Root Server Query from Client
- **Severity:** Medium
- **Source:** `102.219.209.234`
- **Detail:** Client directly queried `a.root-servers.net` twice. Unusual for a standard device — indicates a DNS diagnostic tool, split-DNS VPN, or application performing its own resolver logic
- **Recommendation:** Monitor for recurrence; if persistent, investigate the device for rogue DNS client software

### 3. DoH/DoT Provider Resolution Attempts
- **Severity:** Informational
- **Source:** `102.219.209.234`
- **Detail:** Client resolved `dns.adguard.com`, `cloudflare-dns.com`, and `dns.google` — all well-known encrypted DNS providers. Device is likely attempting to establish DoH/DoT connections to bypass this resolver
- **Impact:** Not currently bypassing — resolver is being used to look up their IPs. If device succeeds in connecting via DoH/DoT, DNS queries will no longer flow through resolve01
- **Recommendation:** Consider blocking outbound TCP/UDP port 853 (DoT) and filtering DoH providers at the firewall level if DNS visibility is required

### 4. Adult Content Domain Queries
- **Severity:** Informational
- **Source:** `102.219.209.234`
- **Domains:** `bongamodels.com` (queried ~12 times with repeated retries)
- **Detail:** Adult content domain queried repeatedly with SERVFAIL responses. Eventually resolved to NOERROR. Repeated retry pattern may indicate an application rather than browser behaviour
- **Recommendation:** If policy requires content filtering, implement an RPZ (Response Policy Zone) or DNS blocklist

---

## Recommendations Summary

| Priority | Action |
|----------|--------|
| High | Increase `num-queries-per-thread: 8192` to prevent request queue exhaustion |
| High | Investigate device `102.219.209.234` — root server queries + DoH bypass attempts |
| Medium | Monitor `i.ytimg.com` — add `domain-insecure: ytimg.com` if SERVFAIL persists >24h |
| Medium | Fix DHCP search domain — remove `.bbrouter` suffix to reduce spurious queries |
| Low | Whitelist `pool.ntp.org` from zone ratelimit: `ratelimit-below-domain: pool.ntp.org 0` |
| Low | Consider DoT/DoH outbound blocking at firewall if DNS visibility is a policy requirement |
| Low | Implement RPZ blocklist for content filtering if required by policy |

---

## Raw Stats (unbound-control stats_noreset)

```
total.num.queries=3632
total.num.cachehits=1917
total.num.cachemiss=1715
total.num.prefetch=179
total.num.expired=101
total.num.queries_timed_out=0
total.requestlist.avg=2.23073
total.requestlist.max=92
total.requestlist.exceeded=131
total.recursion.time.avg=0.436398
total.recursion.time.median=0.299456
time.up=1468.844561
```
