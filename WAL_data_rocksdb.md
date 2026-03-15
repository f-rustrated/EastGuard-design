# WAL → data.seg → RocksDB: Write & Recovery Pipeline

## Record Format (data.seg)

Variable-length records with a trailing length field for efficient backward scanning:

```
[crc: u32][type: u8][length: u32][payload: bytes][length: u32]
                                                  ↑ same value as leading length
```

**Record types:**

- `Data` — a single producer batch entry
- `Fence` — marks that all preceding records in this batch are durable; contains `wal_offset`

**Backward scan:** read trailing 4 bytes → jump back by `(length + header_size)` → repeat until a Fence record is found.

**O_DIRECT alignment:** the batch accumulator pads the entire batch (all records + fence) to the block boundary (
`align_up(batch_size, BLOCK_SIZE)`) before flushing. Alignment is per-flush, not per-record. Buffers must be allocated
with `posix_memalign`.

---

## Write Pipeline

```
1. Accumulate records into batch buffer
2. Write batch to WAL
3. fsync(WAL)                             ← WAL is now durable
4. Append Fence record to batch (wal_offset = current WAL end)
5. Pad batch to block boundary (align_up)
6. Write batch to data.seg (O_DIRECT)
7. fsync(data.seg)                        ← data + fence are now durable
8. WriteBatch to RocksDB (atomic):
     - index entries for each new record
     - last_indexed_position = fence position in data.seg
9. Delete WAL entries up to fence.wal_offset
10. ACK to producer
```

**Why WAL fsync before data.seg write (step 3 before step 6):**
WAL is the recovery safety net for incomplete data.seg writes. If WAL is not durable before the data.seg write begins, a
crash mid-write to data.seg leaves no recovery record.

**Why fence before data.seg fsync (step 4 before step 7):**
The fence is the last record written in the batch. Its presence on disk proves all preceding records in the batch are
durable. It must be included in the same fsync.

**Why RocksDB WriteBatch is atomic (step 8):**
If the index entries and `last_indexed_position` are written separately, a crash between them leaves RocksDB claiming it
has indexed up to position X while some entries for that range are missing. WriteBatch guarantees both are applied or
neither is.

---

## Recovery Procedure

```
1. Scan data.seg backward to find the last valid Fence record
2. Truncate data.seg to the fence position       ← discard partial writes after it
3. Read last_indexed_position from RocksDB
4. Scan data.seg forward from last_indexed_position to fence position
5. Rebuild missing index entries via RocksDB WriteBatch
     (includes updating last_indexed_position to fence position)
6. Re-apply WAL entries from fence.wal_offset onward
7. Append new Fence record + fsync(data.seg)
8. Delete WAL entries up to new fence.wal_offset
```

**Why truncate before re-applying WAL (step 2):**
A crash mid-fsync can leave partial records after the last fence. Re-applying WAL onto partial data produces corrupt or
duplicate records. Truncating first gives a clean base.

**Why scan only the gap in step 4 (not from start):**
RocksDB is persistent and survives crashes. `last_indexed_position` tells us exactly where RocksDB's state is
consistent. Scanning only from there to the fence avoids O(segment size) work on every recovery.

**Bootstrap condition for new Raft group (from B1):**
Before using a survivor's data.seg to seed a new group, the survivor must have been the Raft leader or a quorum of the
old group must be alive — guaranteeing the survivor holds all committed entries.

---

## MultiRaft: Shared Raft Log (H3)

A single Raft log is shared across all vnode groups on a node. Every entry is tagged with `group_id` to identify its Raft group.

**Why shared log:**
256 independent Raft logs = 256 fsyncs per commit round (~25ms overhead at 100μs/fsync on SSD). A shared log reduces this to 1 fsync via group commit, regardless of vnode count.

**Why group_id overhead is acceptable:**
`group_id: u32` adds 4 bytes per entry. A typical Raft entry carrying a producer batch is hundreds to thousands of bytes — well under 1% overhead.

---

## RocksDB Responsibilities

One shared RocksDB instance per node covers all vnode metadata. data.seg remains a separate append-only file; RocksDB holds only the index into it.

**What RocksDB stores:**

| Category | Fields |
|---|---|
| Raft hard state (per group) | `group_id`, `current_term`, `voted_for`, `commit_index`, `last_applied` |
| Segment metadata (per segment) | `segment_id`, `topic_id`, `range_id`, `state`, `replica_set`, `size_bytes`, `created_at`, `sealed_at` |
| Data index | `(topic_id, range_id, logical_offset)` → `(segment_id, physical_byte_offset)` |
| Housekeeping | `last_indexed_position`, `persisted_last_offset_raft_log` |
| Consumer group offsets | `(consumer_group_id, topic_id, range_id)` → `committed_offset` *(deferred — D3)* |

**Why `topic_id` instead of topic name:**
Variable-length string keys make composite key construction and prefix scans fragile. A fixed-size `u64` topic_id produces compact, predictable keys and enables efficient seek-then-scan in RocksDB. A separate `topic_name → topic_id` mapping is stored for lookup.
