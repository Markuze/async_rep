Reverse Ganesha: Feasibility of Live CephFS Async Replication
=============================================================

Abstract
--------

This proposal argues that live asynchronous CephFS replication can be built by
observing committed CephFS client traffic, translating it into logical
filesystem operations, and replaying those operations on a passive CephFS
replica.

The design is the inverse of NFS-Ganesha. Ganesha receives file-service
protocol operations and issues CephFS client operations. Reverse Ganesha
observes CephFS client operations and reconstructs a filesystem operation
stream: ``create``, ``write``, ``rename``, ``unlink``, ``setattr``, and related
mutations.

The core claim is that CephFS already exposes the information needed for this
translation. MDS traffic carries namespace and metadata semantics. OSD traffic
carries file data mutations. Source-assigned versions and operation identifiers
carry the ordering needed for convergence. A gateway that observes these facts
can produce an ordered, idempotent, path-based replay stream. A remote libceph
client can then apply that stream to a passive replica.

This is not synchronous replication, not snapshot replication, and not
active-active service. It is single-source asynchronous replication with
bounded lag and eventual convergence to the source's POSIX-visible state.

Model
-----

The initial design assumes:

* one source CephFS filesystem;
* one passive replica CephFS filesystem;
* unmodified source clients;
* parseable committed MDS and OSD client traffic;
* one active MDS rank for the proof of concept;
* replay only after durable source success is observed;
* no writes to the replica except through the replay backend.

These assumptions define the proof boundary. They are not the final product
boundary. The important question is whether the semantic translation is
possible. If it is, stronger deployment models are extensions of the same
abstraction.

Architecture
------------

::

   CephFS client traffic
        |
        v
   transparent capture layer
        |
        v
   translation gateway
        |
        v
   ordered path-operation stream
        |
        v
   remote libceph client
        |
        v
   passive CephFS replica

The capture layer forwards source traffic unchanged and copies relevant MDS
and OSD messages to the gateway. It does not decide filesystem order.

The gateway is the semantic boundary. It correlates requests with durable
replies, maintains source-side identity state, maps object writes back to file
offsets, and emits logical records:

::

   { op=CREATE, path=/d/f, args=... }
   { op=WRITE,  path=/d/f, offset=x, bytes=..., tokens=... }
   { op=RENAME, old_path=/d/f, new_path=/e/g, tokens=... }
   { op=SETATTR, path=/e/g, args=..., tokens=... }

The remote backend applies these records to the replica. Source inode numbers
and source object names are not target identities. They are translation and
deduplication evidence. The replica allocates its own inodes and may use its
own layout.

Feasibility Argument
--------------------

The design is feasible if four claims hold.

First, MDS traffic yields metadata operations. CephFS namespace mutations pass
through the MDS. A successful MDS request/reply pair identifies the operation,
the target namespace object, the result, and any allocated source inode or
metadata needed for later translation. Failed operations are ignored. Durable
successful operations become path-based replay records.

Second, OSD traffic yields data operations. CephFS file data is stored in RADOS
objects derived from source file identity and layout. A committed OSD write
identifies the source file object, object offset, payload, and committed object
version. Using the file layout known from CephFS metadata, the gateway maps the
object extent to a logical file extent and emits a ``WRITE`` at a file offset.
The replica need not share the source layout because the replay stream is
logical, not object-level.

Third, MDS and OSD traffic can be joined. MDS traffic establishes the mapping
between source inodes, paths, dentries, hardlinks, and layouts. OSD traffic
names objects derived from source inodes. The gateway uses source inode
identity only inside the translation boundary, then emits path operations. This
bridges CephFS's split metadata/data architecture without requiring source
inode reuse on the replica.

Fourth, correct order is recoverable from the source. Packet arrival order is
not sufficient: clients talk to MDSs and OSDs over many connections, and one
file may be striped across many objects. The gateway instead uses
source-assigned order evidence:

* RADOS object versions order committed writes to the same object;
* truncate sequencing orders size changes against writes;
* client session operation identifiers order one client's operations;
* MDS authority and metadata versions order namespace and metadata changes.

These tokens are meaningful because CephFS already serializes conflicting
behavior. Capabilities prevent incompatible client-side mutations from
proceeding concurrently without coordination. Revocation requires dirty state
to be flushed before conflicting authority moves. The gateway therefore
recovers the source's order; it does not invent one.

Only non-commuting operations require a common order. Independent writes,
independent metadata updates, and operations on disjoint namespace regions may
replay in parallel. Conflicting writes, truncate/write races, same-inode
metadata changes, and path-changing namespace operations must share an ordering
domain. This observation is both the correctness rule and the scaling rule.

Bootstrap
---------

Live capture handles mutations after capture begins. Bootstrap handles state
that already existed. The correctness invariant is simple:

No committed mutation may be lost between the start of capture and the moment
an entry becomes replicated.

There are two high-level ways to satisfy the invariant.

A quarantined copy temporarily blocks writes to a subtree, copies that subtree
to the replica, seeds the gateway's source identity state, and then releases
the subtree into live replay. Because mutations are excluded during the copy,
the copied state is a consistent base.

A buffered copy allows mutations during the copy but records live changes from
the moment copying begins. The copied data and buffered mutations are reconciled
using the same source versions used by live replay. If the copy already
includes a mutation, the buffered record is a duplicate. If it does not, the
record is applied.

Both strategies reduce bootstrap to the same principle: establish an initial
equivalent state, then apply every later committed mutation exactly once in the
orders that matter.

Why This Converges
------------------

The replica converges when these conditions hold:

* every committed source mutation in scope is observed or included in
  bootstrap;
* no failed or uncommitted mutation is replayed;
* each mutation is translated to its logical filesystem effect;
* non-commuting mutations are replayed in source order;
* duplicate records are rejected by source tokens.

Under those conditions, the replica applies the same sequence of logical
effects as the source, modulo operations whose relative order cannot affect the
final filesystem state. That is sufficient for convergence to the source's
POSIX-visible state.

The proof does not rely on a global clock, a single packet observation point,
matching source and target layouts, or target reuse of source inode numbers. It
relies on complete committed traffic, source-assigned ordering evidence, and
faithful semantic translation.

Extensions
----------

If the proof of concept establishes this translation, several extensions follow
naturally.

Multiple active MDS ranks extend the ordering model from one MDS authority to
many. The abstraction remains the same: namespace authority movement and
cross-rank operations become additional source ordering evidence.

Encrypted transport changes where plaintext is observed, not what is replayed.
A trusted source-side observation point or event export path can provide the
same committed operation stream.

Gateway sharding follows ordering domains. Data can shard by source file
identity; namespace work can shard by subtree; operations that may affect the
same file or path decision are routed together.

WAN optimization can compress, batch, coalesce, or deduplicate the stream as
long as the receiver reconstructs the same ordered logical records.

Snapshots, quotas, and failover extend the replicated state from ordinary
filesystem contents to disaster-recovery semantics.

Active-active replication is a different model. Reverse Ganesha can move
changes between clusters, but accepting writes on both sides introduces
conflicts that require policy, not only replay.

Conclusion
----------

Reverse Ganesha is feasible because CephFS already contains the required
semantic evidence. MDS traffic describes namespace and metadata mutations. OSD
traffic describes data mutations. Layout metadata maps object writes back to
file offsets. Capabilities and source-assigned tokens define the order of
conflicting effects.

The proposed system makes that evidence explicit. It turns committed CephFS
client traffic into an ordered stream of logical filesystem operations and
replays that stream on a passive CephFS replica. The hard problem is not writing
to the remote cluster; it is proving that the source semantics can be
recovered. This architecture shows that they can.
