────────────────────────────────────────────────────────
Attacker / Defender Review
────────────────────────────────────────────────────────

VERDICT: Plan is directionally correct but has 3 structural blockers
and ~10 high-severity gaps that must be resolved before implementation.

── STRUCTURAL BLOCKERS (must resolve before writing code) ──────────

[B1] SWIM and Raft are two separate membership systems — plan conflates them
SWIM says: "node N is dead"
Raft says: "node N is still a voter in group G"
These diverge during failures. Who initiates ConfChange to remove N from
Raft groups? If N held a Raft majority, the group stalls and cannot commit
the removal.
→ Need: explicit bridge protocol between SWIM NodeDead → Raft ConfChange,
with a defined procedure for single-node-loss quorum recovery.

[B2] WAL → segment transition has no atomic completion marker
"wal.log is deleted once fully reflected in data.seg" — but "fully reflected"
is never checkable without scanning the entire segment file.
A crash between segment fsync and WAL deletion leaves ambiguous state:
re-applying WAL duplicates records; skipping it loses records.
→ Need: a crash-safe "WAL fence" record appended to data.seg atomically
with the final flush, so startup can check the last byte of data.seg
rather than scanning it entirely.

[B3] Replication protocol is a placeholder ("fsync all replicas")
Who coordinates writes to followers? Fan-out or chain? What quorum
is required for ACK — all replicas, majority, one?
What happens if one replica fsyncs and two crash mid-batch?
Until this is specified, ACK durability guarantees are undefined.
→ Need: pick a model (chain replication or parallel fan-out + quorum ACK)
and define it explicitly before implementing produce sessions.

── HIGH-SEVERITY GAPS (resolve before the affected phase) ──────────

[H1] WAL itself is not fsynced — only data.seg is
The plan says "WAL write → segment file (O_DIRECT) → fsync"
The fsync covers the segment file. The WAL uses normal I/O.
On power failure, the WAL lives only in OS buffers → data loss.
→ WAL write must also fsync before data.seg write begins.

[H2] RocksDB index can diverge from data.seg after crash
Pipeline: WAL → data.seg → fsync → RocksDB index → ACK
Crash after fsync but before index write: records in data.seg,
index doesn't know. Without a WAL fence (B2), there's no recovery path.
→ Index write must be atomic with the WAL fence record.

[H3] 256 Raft groups per node — resource cost unanalyzed
256 Raft groups × (heartbeat timers + log FDs + snapshot FDs + memtables)
At 150ms heartbeat: ~1700 heartbeat msgs/sec per node at idle.
256 RocksDB instances for coordinator state: ~512 background threads
+ ~2GB block cache by default.
→ Either reduce vnodes_per_node significantly, or use a single
shared Raft log with logical partitioning (MultiRaft style).

[H4] O_DIRECT requires block-aligned writes — plan has no alignment strategy
Variable-length records produce non-aligned batches.
O_DIRECT returns EINVAL or silently corrupts on non-512-byte-aligned writes.
→ Batch accumulator must pad to block boundary before flush,
with posix_memalign buffers.

[H5] Stale ring views across brokers → duplicate CreateTopic
SWIM gossip propagates in O(log N) rounds. During that window, broker A
and broker B can both route hash(topic_name) to different vnodes and both
issue CreateTopic → duplicate topic records, each Raft-committed independently.
→ Coordinator CreateTopic must check for existence before committing,
and the client must be prepared to handle idempotent retries.

[H6] Mass re-replication after rack failure has no rate limit
If 10 nodes die simultaneously, every surviving Coordinator fires scans
simultaneously → O(256 × dead_nodes) replication sessions at once.
→ Need: admission control on replication sessions (max concurrent per node).

[H7] Concurrent Coordinators can start duplicate replication sessions
Two Coordinators owning different ranges but both with segments on the
dead node can both assign replication to the same destination.
→ Segment state Reassigning must be set atomically in Raft before any
replication session is opened; second Coordinator sees Reassigning and skips.

[H8] Client offset semantics undefined
Is logical_offset global (Kafka-style) or segment-local?
How does a consumer stitch offsets across a sealed segment boundary?
What do "earliest" / "latest" mean for a new consumer?
→ Must define offset model before implementing consume sessions.

[H9] Retention / segment deletion has no GC process
StoragePolicy.retention exists but there is no ticker that deletes
sealed segments past their retention. No coordination between Coordinator
(owns metadata) and Storage Engine (owns files). A consumer mid-read of
a file being deleted is a use-after-free.
→ Need: Coordinator-driven GC with a file reference count or
lease before unlink(2).

[H10] Raft snapshot mechanism undefined
"Full coordinator state at a log index" — for a busy vnode this could be
hundreds of MB. Must use RocksDB checkpoint API (not a naive serialize)
to avoid pausing writes during snapshot.
256 vnodes × snapshot_size triggered simultaneously = GBs of I/O.
→ Stagger snapshots across vnodes; use RocksDB::checkpoint().

── WHAT THE PLAN GETS RIGHT (defender findings) ────────────────────

[+] raft/ vs coordinator/ separation — reads never block on log replay (correct,
same pattern as TiKV and etcd)
[+] murmur3 with fixed seed — cross-process hash stability is correctly
required for routing correctness (DefaultHasher is unstable across runs)
[+] Physical-node deduplication in token_owners_for — already implemented and
tested; prevents all-replicas-on-one-machine failure
[+] SWIM Suspect state ignored by Topology — correct; acting on Suspect would
trigger unnecessary re-replication during transient network delays
[+] Gossip buffer with O(log N) decay and 900-byte UDP cap — prevents
fragmentation; mathematically correct dissemination bound
[+] Incarnation numbers for refutation — correctly prevents false-positive
resurrection suppression (from SWIM paper §4.2)
[+] Shared RocksDB for index with zero-padded key — lexicographic order
matches numeric order; enables correct seek-then-scan lookup
[+] Segment state Reassigning — prevents duplicate re-replication if set
atomically in Raft before opening a replication session (see H7)
[+] Co-located wal.log + data.seg — crash recovery is local, no cross-dir
coordination needed
[+] O_DIRECT for data.seg — correct for append-only immutable files;
frees page cache for index and Raft log

── APPROPRIATELY DEFERRED (out of scope for initial impl) ──────────

[D1] Rack/zone-aware placement — attributes field exists as a hook; constraint
logic deferred
[D2] Range split/merge filesystem ops — directory layout already accommodates it
[D3] Consumer groups and offset commits
[D4] Client-side metadata caching (broker-as-proxy is correct to start)
[D5] TLS / authentication
[D6] Follower reads

────────────────────────────────────────────────────────
Node Lifecycle: Join & Death
────────────────────────────────────────────────────────

── WHEN A NEW PHYSICAL NODE JOINS ──────────────────────────────────

Step 1 — SWIM Bootstrap
new_node sends Ping to seed nodes
seed nodes respond + gossip NodeAlive(new_node) to cluster
all brokers: Swim.members[new_node] → Alive

Step 2 — Topology Ring Update
SwimActor emits NodeAlive → topology.update(new_node, addr, Alive)
→ 256 new vnodes inserted into TokenRing
→ ring shifts: some key ranges now map to new vnodes on new_node

Step 3 — Raft Group Reconfiguration  ⚠ (unspecified in current plan)
For each vnode V now assigned to new_node:
current Raft leader of V proposes ConfChange(AddVoter, new_node)
new_node must receive a Raft snapshot of V's coordinator state
before it can vote — it cannot serve metadata reads until then
→ Need: "learner" mode where new_node receives data but doesn't vote yet,
so it doesn't reduce quorum availability during bootstrap

Step 4 — Metadata Bootstrapping
new_node receives Raft snapshot from leader (full RocksDB for vnode V)
once applied, new_node can serve metadata reads for V

Step 5 — Segment Data Transfer  ⚠ (not organic — needs explicit design)
new_node owns vnodes but has NO segment files on disk
for each segment assigned to a vnode on new_node:
Coordinator transitions segment state → Reassigning (atomic Raft write)
open replication session: existing replica → new_node
new_node writes to segments/{range_id}/{segment_id}/
once complete: new_node added to replica_set, state → Active/Sealed

── WHEN A PHYSICAL NODE DIES ────────────────────────────────────────

Step 1 — SWIM Detection  (up to ~6 seconds)
Ping → no Ack (300ms) → PingReq to 3 peers → no Ack (600ms)
→ Suspect; SuspectTimer fires (5s) → Dead
→ gossipped to all brokers via GossipBuffer

Step 2 — Topology Ring Update
SwimActor emits NodeDead → topology.update(dead_node, _, Dead)
→ 256 vnodes removed from TokenRing
→ ring shifts: those key ranges now map to surviving vnodes

Step 3 — Raft Group Impact  ⚠ (blocker B1)

    Case A: dead_node was a FOLLOWER
      Leader continues. Quorum met (e.g. 2/3 survive).
      Leader proposes ConfChange(RemoveVoter, dead_node) → commits normally.
      ✓ Safe.

    Case B: dead_node was the LEADER
      Followers detect heartbeat timeout → election → new leader elected.
      New leader proposes ConfChange(RemoveVoter, dead_node).
      ✓ Safe if quorum survives.

    Case C: dead_node held Raft majority  (e.g. 2 of 3 nodes die)
      Group is leaderless. Cannot commit ConfChange. Cannot proceed.
      ✗ STALL — requires operator intervention or pre-baked recovery.

Step 4 — Metadata Ownership
Ring shift changes which broker proxies client requests — NOT which
Raft group owns the metadata. Raft log is already replicated across
surviving nodes, so metadata is intact as long as quorum holds (Case A/B).
If Case C: metadata is frozen until quorum is restored.

Step 5 — Under-Replication Detection & Self-Healing
Surviving Coordinators scan segment metadata:
for each segment where dead_node ∈ replica_set:
transition state: Active/Sealed → Reassigning  (atomic Raft write)
pick new broker satisfying StoragePolicy constraints
open replication session: surviving replica → new broker
once complete: update replica_set → Active/Sealed

Step 6 — Producer/Consumer Impact During Recovery Window
Produce: Coordinator redirects to surviving replica in replica_set
if NO replica survives → produce blocked until Step 5 completes
Consume: same — redirect to surviving replica
if client was mid-read on dead node's file → error, must retry

── TWO SUBTLE PROBLEMS ──────────────────────────────────────────────

[P1] Ring shift ≠ metadata transfer
When the ring shifts after a node dies, a surviving vnode suddenly
"owns" a new key range in the routing table. But the Raft group for
that vnode was already replicated — the metadata is already on surviving
nodes. The ring shift only changes which broker acts as proxy.
This is safe as long as the Raft group has quorum (Cases A/B).

[P2] SWIM detection lag vs. client timeouts
SWIM takes up to 300ms + 600ms + 5000ms ≈ 6 seconds to declare Dead.
During that window, clients sending requests to the dead broker get TCP
timeouts. The plan has no defined client retry/redirect behavior during
SWIM's detection window.
→ Need: client-side timeout + retry-to-any-broker policy, independent
of SWIM convergence.