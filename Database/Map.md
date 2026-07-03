# Foundations 

## Dependency Map


| # | Topic | Requires | Core Idea (analogy) |
|---|-------|----------|----------------------|
| 1 | [Storage Fundamentals](./Storage%20Fundamentals.md) | — | RAM:disk is like a desk vs a warehouse across town — everything else exists because fetching from the warehouse is slow|
| 2 | [Database Types](./Types.md) | Storage Fundamentals | Schema enforcement = who's the bouncer, DB or app. OLTP = cashier (fast, one customer); OLAP = accountant (slow, whole ledger) |
| 3 | [Transactions](./Transactions.md) | Storage Fundamentals | ACID = a bank transfer's safety net — rollback log undoes half-done work, WAL is the receipt before the vault closes |
| 4 | [Indexes](./Indexes.md) | Storage Fundamentals | Like a book's index — doesn't move the pages, just tells you which page to flip to. Every new index = another list to keep updated on every edit |
| 5 | [Index Internals](./Index%20Internals.md) | Indexes | Five tools in a toolbox: B+Tree (adjustable wrench), Hash (exact-fit socket), LSM (write fast, tidy later), GIN (multi-tag lookup), Bitmap (light switch panel — only works with few switches) |
| 6 | [Distributed Consistency](./Distributed%20Consistency.md) | Transactions | ACID = one person's promise; BASE = a group chat eventually agreeing. CAP = forced choice during a network split; PACELC = that same choice exists even on a calm day |
| 7 | [Timeseries Databases](./Timeseries%20Databases%20%28TSDBS%29.md) | Index Internals, Database Types | Time never goes backward, so it's a free sorted index. Compression = vacuum-sealing a bag three times over. Failure mode = too many unique tags, not too much data |
| 8 | [Database Selection](./Database%20Selection.md) | All above | Same as choosing a vehicle — pattern, ratio, cardinality, consistency, scale decide car vs truck vs train, not brand loyalty |

