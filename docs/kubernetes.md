

## Etcd 

On Talos, each control plane node runs one `etcd` member.

`etcd` uses the Raft protocol.

### Quorum

| Members | Quorum | Failures tolerated |
|---------|--------|--------------------|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |

Write needs a majority (quorum) of the members:

* Use an odd number of control plane nodes.
* A fourth member adds no tolerance, but one more node that can fail.

### Status

```bash
# asks each control plane node about its local etcd member
talosctl -n $nodes etcd status                 
```

| Column | Meaning |
|--------|---------|
| `NODE` | The node (endpoint) you queried, from `-n`. |
| `MEMBER` | The ID of the etcd member on that node. |
| `DB SIZE` | Total size of the etcd database on disk. |
| `IN USE` | Space actually used, with % of the total. High % = defrag candidate. |
| `LEADER` | The current leader ID. All rows must match. |
| `RAFT INDEX` | The current Raft log index. Same value = members in sync. |
| `RAFT TERM` | The current election term. Rises by 1 after each election. |
| `RAFT APPLIED INDEX` | `RAFT INDEX` when in sync. |
| `LEARNER` | `false` = voting member; `true` = catching up, not voting yet. |
| `PROTOCOL` | The etcd (server) version. |
| `STORAGE` | The data format (storage) version. |
| `ERRORS` | Any errors. Empty = healthy. |

### Snapshot

Write a consistent point-in-time backup of the entire `etcd` database

```bash
talosctl -n $node etcd snapshot $file.snapshot
```

**What it is** — The basis for disaster recovery

- A full, consistent copy of the `etcd` database: every key, i.e. every Kubernetes object.
- Captured at one `etcd` **revision** (a monotonically increasing commit number).

**Why the revision matters**

- Every successful write raises the revision.
- Recovering restores the state at the snapshot's revision.
- Data written after is lost.

### Defrag

Reclaims the unused space in one `etcd` member's database file.  
cf. [cluster-operations-and-maintenance/etcd-maintenance](https://docs.siderolabs.com/talos/v1.14/build-and-extend-talos/cluster-operations-and-maintenance/etcd-maintenance)

**Do not defrag all members at once!** (could lose quorum,take down API server)

```bash
talosctl -n $node etcd defrag
# verify state afterwards
talosctl -n $node etcd status
talosctl -n $node etcd alarm list
```
- Reclaims the free, fragmented space and shrinks the file on disk.
- After it, `DB SIZE` is smaller and `IN USE` is near 100 %.
- It applies to one member: the node you pass with `-n`.
- Compaction removes old revisions (logical); defrag reclaims the freed file space.

**Why the database grows?**

- `etcd` stores all cluster state in a single database file.
- Kubernetes updates keys constantly: leases, pods, endpoints, status.
- Updates and deletes free space but do not shrink the file (Fragmentation grows).
- `IN USE` is the share of the file holding live data...
- ...low value means much of the file is wasted. That is when defrag helps most.

**Why you care: the quota**

- `etcd` has a maximum database size (default 2 GB, `quota-backend-bytes`).
- If the file fills up, including unreclaimed fragmented space...
- ...etcd sets a `NOSPACE` alarm and goes read-only. The cluster stops accepting writes.
- Defrag reclaims space and helps avoid that...
- ...routine maintenance, alongside compaction and snapshots.

