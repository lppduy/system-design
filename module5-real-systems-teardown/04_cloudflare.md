# Cloudflare Internals

## What It Is
Reverse proxy between users and origin server. Every request hits Cloudflare first.

**Reverse vs Forward proxy:**
- Forward: client → proxy → internet (proxy hides client, e.g. VPN)
- Reverse: client → proxy → origin (proxy hides server, e.g. Cloudflare)

## Anycast Routing

One IP announced from 330+ PoPs (Points of Presence) worldwide. BGP (Border Gateway Protocol) routes users to nearest PoP automatically.

```
User in Vietnam → BGP → Singapore PoP (closest)
User in France  → BGP → Paris PoP (closest)
Both connect to SAME IP address
```

**vs DNS-based routing (Route 53):** DNS returns different IPs per region. But attacker can discover origin IP and attack directly. Anycast hides origin IP — attacker only sees Cloudflare's IP.

## DDoS Mitigation

### Why Anycast helps
1 Tbps attack from global botnet → BGP splits across 330+ PoPs → ~3 Gbps per PoP → manageable. No single point absorbs full blast.

### Defense Layers
```
Layer 3/4 (Network): SYN floods, UDP floods
  → Dropped at network edge, never reaches app

Layer 7 (Application): HTTP floods, slowloris
  → WAF rules, rate limiting, bot detection, JS challenges

Bot detection:
  → Browser fingerprint, CAPTCHA, behavioral analysis
```

## Edge Caching (CDN)

```
User → PoP → Cache HIT? → return cached (fast, no origin hit)
           → Cache MISS? → fetch from origin → cache → return
```

**Cache-Control headers:**
- `public, max-age=3600` → cache 1 hour
- `no-store` → never cache
- `s-maxage=600` → CDN caches 10 min (separate from browser cache)

**Never cache user-specific API responses** (`/api/me/profile`). Risk: User A's data served to User B = security breach.

**What to cache:** static assets (JS, CSS, images), public API responses, HTML pages without personalization.

## Cloudflare Workers

Code running at the edge (330+ PoPs), not on origin:

```javascript
export default {
  async fetch(request) {
    // Runs at nearest PoP
    // Can modify request, return response, or proxy to origin
    return new Response("Hello from edge");
  }
}
```

Use cases: A/B testing, auth checks, URL rewriting, rate limiting, geo-routing — all at edge, zero origin load.

## SSL/TLS Termination

```
User ←→ Cloudflare: HTTPS (TLS terminated here)
Cloudflare ←→ Origin: HTTP or HTTPS (configurable)
```

Cloudflare handles certificate management. Modes:
- **Flexible:** User→CF is HTTPS, CF→Origin is HTTP (not recommended)
- **Full:** Both HTTPS, but origin cert not validated
- **Full (Strict):** Both HTTPS, origin cert validated (recommended)

## Practical: Dev Workflow with Cloudflare

### Setup
1. Buy domain → add to Cloudflare → update nameservers at registrar
2. Add DNS records: `A api 203.0.113.50 ☁️` (orange cloud = proxied)
3. SSL mode → Full (Strict) + install Cloudflare Origin Certificate on VPS

### Cache Headers in App Code (Spring Boot)
```java
// Public data — CF caches 5 min
@GetMapping("/api/products")
public ResponseEntity<List<Product>> list() {
    return ResponseEntity.ok()
        .header("Cache-Control", "public, s-maxage=300")
        .body(productService.findAll());
}

// Private data — NEVER cache
@GetMapping("/api/me")
public ResponseEntity<User> me(@AuthenticationPrincipal User user) {
    return ResponseEntity.ok()
        .header("Cache-Control", "private, no-store")
        .body(user);
}
```

### Deploy: Purge Cache
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache" \
  -H "Authorization: Bearer API_TOKEN" \
  -d '{"purge_everything":true}'
```

### Rate Limiting (Dashboard)
Rule: `/api/login` → 5 req/min per IP → block 10 min. Brute-force never reaches server.

### Impact
71% requests served from cache → $5 VPS handles traffic that would need $50 server.

## Interview Angles

- "How does CDN reduce latency?" → Content cached at PoP near user. No round-trip to origin.
- "Anycast vs DNS routing?" → Anycast: same IP everywhere, BGP routes to nearest, hides origin. DNS: different IPs, origin discoverable.
- "DDoS at scale?" → Anycast distributes attack across all PoPs. L3/4 dropped at edge. L7 filtered by WAF.
- "Edge computing?" → Workers run code at PoP. Auth, routing, transforms without origin hit.
