# Storage Fundamentals

---

## Why this doc exists

This doesn't directly make you better at databases. Understanding comes from using these concepts to explain something, not from reading about them.

What this actually does: gives you a **cost model and shared vocabulary** that makes every other DB concept click faster instead of being memorised in isolation.

- Reading about LSM trees choosing sequential writes — you already know *why* sequential wins. You don't take it on faith.
- Index explanations mention "heap fetch" — you already have the picture.
- "Buffer pool hit rate dropped" — you know what that means in latency terms, not just abstractly.

---

## 1. Mental Model

> A database never reads a single row. It reads the **page** the row lives on — always.

Ask for one row, get an 8KB block. Whether you need 1 row or 80, the disk cost is identical. Every DB design decision — indexes, buffer pools, sequential I/O — is either reducing how many pages get read, or making sure the right pages are already in RAM.

---

## 2. The Speed Hierarchy

| Storage | Latency | Feel |
|---|---|---|
| RAM | ~100 ns | Instant |
| SSD (NVMe) | ~100 µs | 1,000× slower than RAM |
| HDD (spinning) | ~10 ms | 100,000× slower than RAM |

One HDD random read = ~10ms. In that same window, your CPU executes ~10 million instructions. The disk is the bottleneck, not your code. This is why everything in DB engineering is about minimising disk trips.

---

## 3. Disk vs In-Memory

| | Disk-Based | In-Memory |
|---|---|---|
| Data lives in | Disk (RAM is cache) | RAM |
| Latency | I/O-bound | RAM speed (~100 ns) |
| Dataset limit | Disk size | RAM size |
| Survives crash | Yes | Needs WAL/snapshot |
| Examples | PostgreSQL, MySQL, MongoDB | Redis, VoltDB |

**The nuance:** a disk-based DB with a hot dataset that fits in its buffer pool *behaves* like in-memory at read time. The difference surfaces on cold reads and crash recovery.

---

## 4. Pages — The Atomic Unit

Fixed-size block (PostgreSQL: 8KB, MySQL: 16KB). The smallest thing a DB reads from or writes to disk.

```
┌──────────────────────────────────────┐
│  Page Header (ID, free space ptr)    │
├──────────────────────────────────────┤
│  Slot Array  [→r1] [→r2] [→r3]      │  → grows forward
├──────────────────────────────────────┤
│          Free Space                  │
├──────────────────────────────────────┤
│    ...row3 | row2 | row1             │  ← grows backward
└──────────────────────────────────────┘
```

- **Slot array** — fixed-size pointers to row offsets. Indexes point here, not at the raw byte offset. If a row moves within the page, only the slot updates — external pointers stay valid.
- **Write amplification** — update 10 bytes, the DB still writes 8KB back to disk. The page is the atomic write unit too.

---

## 5. Heap Files — How Tables Live on Disk

A table = a collection of pages in **no particular order**. New rows go wherever there's free space.

```
Table: orders

Page 1  │ id=9823 │ id=102  │ id=44001 │
Page 2  │ id=5    │ id=76543│ id=219   │
Page 3  │ id=88   │ id=30001│ ...      │
```

No ordering. `id=5` lives between `id=76543` and `id=219`. Without an index, finding it means reading every page — a full scan. That's what indexes solve.

**Dead rows** — deletes don't immediately compact a page, they mark slots as dead. Space reclaimed by background processes (Postgres: VACUUM, InnoDB: purge thread).

---

## 6. TIDs — Row Addressing

Every row has a physical address:

```
TID = (page_number, slot_number)
      e.g. (47, 3) → Page 47, Slot 3
```

This is what an index stores, not the actual row data. Index lookup flow:

```
Index traversal → TID (47, 3) → load Page 47 → go to Slot 3 → row
```

Two steps. Always. This is why **covering indexes** win — if the index already has every column the query needs, step 2 (heap fetch) is skipped entirely.

---

## 7. Buffer Pool

A chunk of RAM the DB manages itself. Keeps recently-used pages in memory to avoid disk reads.

```
           ┌─────────────────┐
  Query ──▶│   Buffer Pool   │──HIT──▶ return row (no disk)
           │   (RAM cache)   │
           └────────┬────────┘
                    │ MISS
                    ▼
               Disk (SSD/HDD)
               load page → store in buffer pool → return row
```

**Why the DB manages its own cache instead of trusting the OS:** the DB knows which pages are hot, which are part of a sequential scan (don't bother caching), and controls when dirty pages flush to disk for durability. The OS doesn't know any of this.

**Buffer pool size is the single highest-leverage config knob for read performance.** Most production setups: 60–80% of available RAM.

---

## 8. Sequential vs Random I/O

The biggest practical consequence of how disks work.

| Pattern | I/O type | Why |
|---|---|---|
| Full table scan | Sequential | Reads pages in order, disk arm doesn't move (HDD) |
| Index point lookup | Random | 3–4 tree levels + 1 heap page, scattered locations |
| B+Tree range scan | Sequential | Leaf pages are linked — walk the chain |
| UUID primary key inserts | Random writes | Each insert lands on a random page |
| Auto-increment PK inserts | Sequential writes | Always appends to the last page |

```
Sequential read:  [Page1][Page2][Page3][Page4]...  ← one sweep
Random read:      [Page47]...[Page3]...[Page891]...[Page12]  ← jumping around
```

**HDD:** sequential is ~100× faster than random (no arm movement).  
**SSD:** gap is smaller but still real — ~10–20× at high operation counts.

**Practical implication:** if a query returns 30–40% of a table, the planner may choose a full sequential scan over thousands of random index-to-heap fetches. More pages, less total time.

---

## 9. One-Line Principle

> Everything in storage engineering is a negotiation between two facts: disk is 100,000× slower than RAM, and you can only read/write in page-sized chunks — never a single row.
