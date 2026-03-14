# Elasticsearch Internals

## What It Is
Distributed search engine built on Apache Lucene. Answers "given a word, which documents contain it?" across billions of docs in milliseconds.

**Lucene** = Java library providing inverted index, text analysis, scoring. Not a server.
**Elasticsearch** = Lucene + distribution + REST API + cluster management. Lucene is the engine, ES is the car.

## Inverted Index

Flips the relationship: instead of "doc → words", stores "word → docs".

```
Forward:  doc1 → "kafka uses sequential writes"
Inverted: "kafka" → [doc1, doc3],  "sequential" → [doc1]
```

Search `kafka AND sequential`: intersect posting lists → {doc1}. O(1) per term lookup.

## Text Analysis Pipeline

Normalizes text so "K8s Containers" matches search "kubernetes container":

```
Input: "The K8s Containers were Running!!!"
  1. Character filter → strip special chars
  2. Tokenizer → ["The", "K8s", "Containers", "were", "Running"]
  3. Token filters (in order):
     - Lowercase:  → ["the", "k8s", "containers", "were", "running"]
     - Synonyms:   → ["the", "kubernetes", "containers", "were", "running"]
     - Stop words:  → ["kubernetes", "containers", "running"]  (remove "the","were")
     - Stemming:   → ["kubernetes", "contain", "run"]
```

Same pipeline runs on search query. Both sides normalized → they match.

**Token filters solve:** case mismatch, singular/plural, synonyms, noise words.

## Architecture: Index → Shards → Segments

- **Index:** logical collection (like a DB table)
- **Shard:** a Lucene index, actual data unit. Index split across shards for parallelism.
- **Replica:** copy of shard on different node (fault tolerance + read scaling)
- **Segment:** immutable file within a shard. New docs → new segment on refresh.

Same concept as Kafka partitions — split for parallelism, replicate for durability.

## Distributed Search (Scatter-Gather)

Query hits ALL shards (any might have matches):
1. Scatter: send query to all shards in parallel
2. Each shard searches locally, returns top N doc IDs + scores
3. Gather: coordinating node merges, sorts by score, picks global top N
4. Fetch full documents for final results

## Near Real-Time Search (~1s delay)

```
Document indexed:
  1. Written to in-memory buffer + translog (WAL for crash recovery)
  2. ~1s later: REFRESH → buffer flushed to new segment → now searchable
  3. Minutes later: FLUSH → segments persisted to disk, translog cleared
```

**Segments are immutable** — no locking needed, cache-friendly, similar to Kafka's log segments.
**Deletes** = mark as deleted (bitmap). Merge process removes deleted docs + combines small segments.

## Relevance Scoring (BM25)

Not just "matches or not" but "how well":
- **TF (Term Frequency):** word appears 5× in doc → higher score
- **IDF (Inverse Doc Frequency):** rare word across all docs → higher score
- **Field length:** match in short title > match in long body

## ES vs PostgreSQL Full-Text Search

| | PostgreSQL (GIN + tsvector) | Elasticsearch |
|---|---|---|
| Distribution | Single node | Built-in sharding + replication |
| Scoring | Basic | BM25, customizable |
| Analyzers | Limited | Rich, language-specific |
| Best for | Search as secondary feature | Search IS the product |

## Operational Concepts

- **Cluster health:** Green (all OK), Yellow (replicas missing), Red (primaries missing)
- **Shard sizing:** 10-50GB per shard
- **Time-series:** index-per-day/week, roll over old indices, delete entire old indices

## Interview Angle
"Elasticsearch is Lucene (inverted index + text analysis + scoring) distributed across a cluster. Shards split data for parallelism, replicas for fault tolerance. Search uses scatter-gather across all shards. Near real-time via immutable segments refreshed every ~1s. The analyzer pipeline (tokenize → lowercase → stem → filter) is what makes fuzzy human searches match stored documents."
