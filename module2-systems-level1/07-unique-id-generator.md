# M2-07: Unique ID Generator

## Estimation
- ~10K IDs/sec normal, 100K peak
- ~1B IDs/day
- 64-bit (8 bytes) per ID
- ~3 TB/year (IDs only)

## Requirements
**Functional:** Globally unique, numeric, roughly time-ordered.
**Non-functional:** High availability, <1ms latency, distributed, no SPOF.

## Approaches

### 1. UUID (128-bit random)
- No coordination needed, each node generates independently.
- **Problems:** 128 bits (2x waste), not time-sorted, random B-tree insert → index fragmentation → slow writes.
- **Use when:** Small systems, no time-sort needed (session IDs, correlation IDs).

### 2. DB Auto-increment (Multi-master)
- Multiple DBs with different start + step: Server1=1,4,7... Server2=2,5,8...
- **Problems:** Adding/removing servers requires changing step for all. Not time-sorted across servers. Network round-trip per ID.
- **Use when:** Small-medium systems, stable server count.

### 3. Ticket Server (Flickr)
- Dedicated DB just for ID generation. Flickr uses 2 (odd/even) for redundancy.
- **Problems:** SPOF risk. Network hop every ID. Bottleneck at high scale.
- **Use when:** Medium scale, need sequential IDs.

### 4. Snowflake (Twitter) ★
- Encode info into 64 bits:
```
[0][41-bit timestamp ms][10-bit machine ID][12-bit sequence]
 1        69 years           1024 machines     4096/ms
```
- **Why it wins:** No network call (local generate), time-sorted (timestamp in high bits), 64-bit, no SPOF, ~4M IDs/sec/machine.
- **Weakness:** Clock-dependent, needs machine ID assignment (ZooKeeper).
- **Use when:** High scale. Standard interview answer.

### 5. Variants
- **Sonyflake (63-bit):** `[39-bit ts 10ms][8-bit seq][16-bit machine]` — trades throughput (256/10ms) for more machines (65K) and longer lifespan (174 years).
- **ULID (128-bit):** `[48-bit ts ms][80-bit random]` — no machine ID needed (random replaces it), but 128-bit = more space.
- **Key diff:** All same idea (timestamp in high bits). Different bit allocation for different tradeoffs.

## Comparison

| | UUID | DB Auto | Ticket | Snowflake |
|---|---|---|---|---|
| Bits | 128 | 64 | 64 | 64 |
| Time-sorted | ✗ | ✗ | ✓ | ✓ |
| Need network | ✗ | ✓ | ✓ | ✗ |
| SPOF | ✗ | ✓ | ✓ | ✗ |
| Scale | ∞ | Low | Medium | High |

## Deep Dive: Failure Cases

### Clock Skew
- NTP sync can push clock backwards → same timestamp reused → duplicate IDs.
- **Fix:** Refuse to generate if clock goes backwards. Wait until clock catches up. Twitter does this.

### Sequence Overflow
- 4096 IDs in same ms on same machine. If exceeded, sequence bits overflow.
- **Fix:** Wait until next millisecond, reset sequence to 0. Rare in practice.

### Machine ID Assignment
- Two machines with same ID → duplicate IDs.
- Options: ZooKeeper (Twitter), config file, IP hash, DB registration.

### Epoch Exhaustion
- 41 bits = 69 years. Using Unix epoch (1970) → expires ~2039.
- **Fix:** Custom epoch (e.g., 2024-01-01) → expires ~2093.

## Key Insight
> Bit position determines sort order. Whatever you want to sort by first → put in most significant bits (leftmost).

## Interview Flow (45 min)
1. Clarify requirements (unique? sorted? 64-bit? distributed?)
2. Estimation (QPS, storage, bit budget)
3. Mention 4 approaches, tradeoffs, pick Snowflake
4. Draw bit layout, explain each field
5. Deep dive: clock skew, sequence overflow, machine ID
6. Tradeoffs: Snowflake vs ULID, clock dependency, 69-year limit
