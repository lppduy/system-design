# M0-01: Latency Numbers Every Engineer Should Know

## Why it matters

Every estimation and every tradeoff depends on knowing how fast each layer is. Without this, your design decisions are guesswork.

---

## The Numbers

```
L1 cache reference              0.5 ns
L2 cache reference              7 ns
RAM reference                   100 ns
Send 1KB over 1Gbps network     10 µs      (10,000 ns)
SSD random read                 150 µs     (150,000 ns)
Round trip within same DC       500 µs
HDD seek                        10 ms      (10,000,000 ns)
Round trip CA → Netherlands     150 ms
```

**Key ratios:**
- RAM is ~200x faster than SSD
- SSD is ~1000x faster than HDD
- Same DC network call is ~5000x slower than RAM
- Cross-continent is ~300x slower than same DC

---

## Estimation Building Blocks

```
1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes

1K   = 10^3
1M   = 10^6
1B   = 10^9

Seconds in a day   = 86,400  ≈ 100K
Seconds in a month = 2.5M
Seconds in a year  = 31.5M   ≈ 30M
```

---

## DC (Data Center)

A physical building full of servers. When your app and DB are in the same DC, a network call takes ~500µs. Different countries → ~150ms. That's a 300x difference from physical distance alone.

---

## What This Means for Design

### One DB call = ~1ms minimum

Every `db.query(...)` has to:
1. Send query over network to DB server (~500µs)
2. DB reads from disk/RAM and processes
3. Send result back (~500µs)

5 serial DB calls per request = 5ms floor, just in waiting — before any compute.

### Serial vs Parallel calls

**Serial:**
```
fetch user       → wait 1ms
fetch orders     → wait 1ms  (starts only after user done)
fetch notifs     → wait 1ms  (starts only after orders done)
Total: 3ms
```

**Parallel:**
```
fetch user   ─┐
fetch orders  ├─→ all start together → 1ms total
fetch notifs ─┘
```

Same work, 3x faster.

---

## N+1 Problem

The most common serial call trap.

**Bad — 101 DB calls for 100 users:**
```java
List<User> users = db.query("SELECT * FROM users LIMIT 100"); // 1 query

for (User user : users) {
    List<Order> orders = db.query(
        "SELECT * FROM orders WHERE user_id = " + user.id   // 1 per user
    );
    user.setOrders(orders);
}
```

101 calls × 1ms = 101ms minimum. Grows linearly with data.

**Fix — 2 DB calls total:**
```java
List<User> users = db.query("SELECT * FROM users LIMIT 100");

List<Integer> ids = users.stream().map(u -> u.id).toList();
List<Order> orders = db.query(
    "SELECT * FROM orders WHERE user_id IN (" + ids + ")"
);

Map<Integer, List<Order>> ordersByUser = orders.stream()
    .collect(groupingBy(o -> o.userId));

for (User user : users) {
    user.setOrders(ordersByUser.get(user.id));
}
```

2 calls regardless of user count. Grouping happens in RAM — 200x faster than disk.

ORMs like Hibernate generate N+1 silently by default. Always check generated queries on large tables.

---

## Takeaway

- Cache = keep hot data in RAM, avoid hitting SSD/HDD
- Minimize serial network calls — parallelize where possible
- N+1 = silent killer, always batch with IN queries
- Same DC calls are cheap but not free — they add up at scale
