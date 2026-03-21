# M2-04 Web Crawler

## Mode: Mentor

## Estimation
- 1B pages, crawl monthly, avg 500KB/page (HTML + metadata)
- Pages/sec: 1B / (30 × 86400) ≈ 400 pages/sec
- Bandwidth: 400 × 500KB = 200 MB/s = 1.6 Gbps
- Storage: 1B × 500KB = 500 TB/month
- Conclusion: distributed system required, object storage (S3) for HTML

## Requirements
**Functional:** Given seed URLs, crawl reachable pages, extract links, store content.
**Non-functional:** Politeness (respect robots.txt), distributed, URL dedup, fault-tolerant, priority-based crawling.

## High-Level Design
```
Seed URLs → URL Queue (Kafka) → Workers (Fetcher) → HTML Parser → URL Filter → back to Queue
                                      ↓
                                Storage (S3)
```
1. **URL Queue** — holds URLs to crawl, starts with seeds
2. **Worker/Fetcher** — pulls URL, downloads HTML (async I/O)
3. **Storage (S3)** — saves raw HTML, cheap at 500TB scale
4. **HTML Parser** — extracts all links from page
5. **URL Filter** — checks: already crawled? robots.txt allows? → Bloom Filter
6. New valid URLs → push back to queue → loop

## Deep Dive

### URL Dedup — Bloom Filter
- HashSet for 1B URLs = 100GB RAM → too much
- Bloom Filter: bit array + hash functions, 1B URLs ≈ 1.2GB RAM
- Trade-off: 1-2% false positive (crawl a few dupes) OK, zero false negatives
- Centralized on Redis, shared across workers

### Politeness
- Respect robots.txt (crawl-delay, disallow paths)
- Rate limit per domain (max 1 req/sec per domain)
- Partition by domain: each worker owns a group of domains → no concurrent hits

### Priority Queue
- Homepage > deep pages
- Frequently changing pages (news) > static pages
- Use priority queue instead of FIFO

### Fault Tolerance
- Queue ACK: unACKed URLs auto-retry on different worker
- Periodic checkpointing to DB
- Workers are stateless → restart = resume

### Content Dedup
- Different URLs, same content (e.g., ?ref=123 params)
- SHA-256 hash of page content → second Bloom Filter

### Spider Trap Prevention
- Max depth limit (e.g., 15 levels)
- Max URLs per domain cap
- Detect repeating URL patterns

## Why Reasoning
- **BFS over DFS**: DFS gets stuck on one site; BFS ensures breadth across domains
- **Kafka over in-memory queue**: persistence, replay, consumer groups for load balancing
- **S3 over HDFS**: managed, unlimited scale, cheaper ops (most companies moved away from self-hosted HDFS)
- **Partition by domain over random**: controls politeness, caches robots.txt, reuses TCP connections

## Key Concepts
| Concept | Application |
|---------|------------|
| BFS | Traverse web breadth-first, avoid getting stuck |
| Bloom Filter | O(1) membership check, 1.2GB vs 100GB for 1B URLs |
| Consistent Hashing | Partition domains across workers, easy add/remove |
| Async I/O | Handle network-bound fetching efficiently |
| Idempotency | Re-crawling same URL produces same result |
| robots.txt | Standard protocol for crawler politeness |
| Content Hashing | Detect duplicate pages with different URLs |

## Interview Angle (45 min)
```
0-5:   Clarify scope (web scale? incremental? content type?)
5-10:  Estimation (pages/s, bandwidth, storage)
10-20: High-level (queue → workers → parser → filter → loop)
20-35: Deep dive (pick 2-3: URL dedup, politeness, fault tolerance)
35-40: Tradeoffs (BFS vs DFS, S3 vs HDFS, Bloom Filter vs HashSet)
40-45: Extensions (SPA rendering, recrawl strategy, redirects)
```

## Extensions
- **SPA crawling**: Headless browser (Puppeteer) for JS-rendered content
- **Recrawl strategy**: Track change frequency per page, prioritize dynamic pages
- **Redirect handling**: Follow 301/302, normalize URLs, update mapping
- **HDFS**: Hadoop Distributed File System — splits files into 128MB blocks, replicates 3x across machines. Self-hosted alternative to S3, good for batch processing but high ops cost.
