# M2-03: Pastebin

## What is it
Paste text → get a short URL → share with others. Content can be code, logs, notes.
Supports expiry (TTL) and access control (public / unlisted / private).

## Access Pattern
**Read-heavy** (~10:1 read/write). 1 write → shared to many readers.

## Key Design Decisions

### Storage: Split metadata vs content
- **PostgreSQL** stores metadata only: `paste_id, short_key, owner_id, visibility, created_at, expires_at, s3_key`
- **S3** stores actual content (text blob)
- Reason: DB not designed for large blobs — slow index, bloat memory. S3 is cheap, scalable, immutable-friendly.

### Short Key Generation
- **Base62 random, 8 chars** = 62⁸ ≈ 218B keys
- Do NOT hash content — two pastes with same content are different objects (different owner, expiry)

### Read Scaling
- **Redis cache** — content is immutable after creation, very high cache hit rate
- **CDN** — public/unlisted pastes served from edge (static text)
- **Read replicas** — DB scale reads; write only to primary

### Expiry
- **Lazy deletion**: on read, check `expires_at` → if expired, return 404 + delete
- **Background cron job**: periodically `DELETE WHERE expires_at < NOW()` to free DB + S3
- Redis TTL handles cache expiry automatically

### Access Control
- 3 modes: `public`, `unlisted` (link-only), `private` (owner only)
- Enforced at **App Server** — only layer with user identity + paste metadata
- Private pastes **bypass CDN** entirely (CDN has no auth context)

## Architecture
```
Client
  ↓
Load Balancer
  ↓
App Server  ←→  Redis (cache)
  ↓
PostgreSQL (metadata) + Read Replicas
  ↓
S3 (content)  ←→  CDN (public/unlisted only)

Cleanup Job (cron) → delete expired from DB + S3
```

## Read Flow
```
GET /abc123
→ App Server: check cache (Redis)
  → miss: query DB for metadata (s3_key, visibility, expires_at)
    → access control check
    → fetch from S3, cache result
→ return content
```

## Key Insight
Private paste must go through App Server — CDN and S3 have no concept of user identity.
