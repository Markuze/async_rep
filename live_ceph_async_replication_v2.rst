Live CephFS Async Replication POC — Reverse Ganesha
===================================================

Abstract
--------

We propose a proof of concept for asynchronous, cross-cluster CephFS
replication built by observing CephFS client wire traffic and replaying it
as filesystem operations on a passive replica. A transparent capture layer
in the client-to-cluster path copies MDS and OSD traffic in both directions.
A translation gateway decodes that traffic, maintains source-side
inode-to-path and layout state, and emits an ordered stream of path-based
filesystem operations. A remote libceph client backend applies that stream
to the replica, which converges to the source's committed state.

This is the inverse of NFS-Ganesha. Ganesha accepts NFS/SMB protocol
operations and translates them into CephFS client operations; this design
observes CephFS client operations and translates them back into a filesystem
operation stream replayable on a second CephFS cluster — "reverse Ganesha."

Replication is live and asynchronous with bounded lag: not snapshot-based,
not synchronous, not active-active. Correct replay order is recovered from
source-assigned sequence tokens carried in the wire messages — per-object
``user_version``, ``truncate_seq``, and per-session op ids — so ordering does
not depend on the order in which packets are observed. The POC proves the
hard half of the pipeline: that the gateway can derive a correct, ordered
``path + op`` stream from wire traffic. Replay through a libceph backend is
the straightforward half.

The live capture-and-replay mechanism needs no changes to Ceph clients,
OSDs, or MDS; the only optional cluster change is a small MDS quarantine
feature used by one bootstrap strategy (Strategy A), which the alternative
(Strategy B) avoids entirely.

Introduction
------------

In any deployment the source filesystem exists and serves clients before
replication begins, and clients do not funnel through a single chokepoint:
they compute placement via CRUSH and talk directly to the primary OSD of
each placement group and to the active MDS, each over its own connection. A
replication design must therefore reconstruct a single coherent stream of
filesystem operations from traffic that is inherently distributed across
many connections, and must do so without modifying clients.

The pipeline:

::

   Ceph client wire traffic  ->  ordered filesystem operation stream
                             ->  remote libceph client backend
                             ->  replica CephFS state

The contribution of this POC is semantic translation, not transport. The
question it answers is whether the capture layer and gateway can recover the
exact logical operation, target path, file offset, metadata arguments, and
ordering constraints from network traffic. Once a correct ordered
``path + op`` stream exists, the remote libceph backend is a replay sink. The
document presents the architecture at the level needed to show the approach
is feasible and how it works; backend internals and transport choices are
out of scope.

Scope
-----

The POC demonstrates that the system can:

* observe MDS request/reply traffic for namespace and metadata mutations;
* observe OSD write request/reply traffic for file data mutations;
* build a source-local inode to path mapping;
* map OSD object writes back to logical file writes;
* derive a correct replay order from source-assigned tokens;
* feed the ordered operation stream to a remote libceph client backend;
* bootstrap a pre-existing subtree into the replicated state, including
  table seeding and the bootstrap-to-live-capture seam;
* converge the replica to the source's state (defined next).

*Convergence* is the proof target. After the replay stream drains on a
quiesced source, the replica's namespace, file content, and attributes —
mode, owner, xattrs, size, nlink, and symlink target — equal the source's
actual committed state. Timestamps (mtime, atime, ctime) are not preserved,
as expected for asynchronous disaster-recovery replication. Stating the
target as the source's *actual* committed state also settles the cross-object
case: where simultaneous writers tear a write across an object boundary,
CephFS itself is not atomic and the replica reproduces whatever the source
committed.

Assumptions
-----------

* one source CephFS filesystem and one passive replica CephFS filesystem;
* a single active MDS rank (``max_mds=1``);
* unmodified source clients;
* a capture layer that sees all relevant client-to-MDS, MDS-to-client,
  client-to-OSD, and OSD-to-client traffic;
* inspectable traffic (msgr2 crc / plaintext-inspectable mode);
* strict capture: an operation is replayable only after its commit/ondisk
  reply is observed, never on an early ack;
* source-rejected operations are not replayed;
* the replica is written only by the remote libceph client backend.

Core Feasibility Claims
-----------------------

The POC proves feasibility if the following six statements hold in practice.

F1. MDS traffic yields namespace operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MDS mutation requests carry operation intent: create, mkdir, unlink, rename,
link, symlink, setattr, setxattr, removexattr, and similar. MDS replies
carry the result: success or failure, allocated source-local inode number,
type, attributes, and layout where relevant. Request and reply together are
enough to derive a logical metadata operation:

::

   { op=CREATE,   path=/dir/file, type=file, mode=0644 }
   { op=RENAME,   old_path=/dir/a, new_path=/dir/b }
   { op=SETXATTR, path=/dir/file, name=user.k, value=v }

F2. Inode-to-path translation bridges MDS and OSD traffic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MDS traffic is namespace-oriented (path, parent inode, dentry name); OSD
traffic is object-oriented. A CephFS data object is named
``<inode-hex>.<objno-hex>`` (e.g. ``10000000abc.00000004``), so the
source-local inode and object number are read directly off the object name —
the join key is on the wire, not inferred. The source-local inode is that
join key, so the gateway maintains an inode-to-path model:

::

   MDS create reply   teaches:  source-local inode I belongs to /dir/file
   Later OSD write    names:    inode I, object N, offset X
   Gateway joins:     I -> /dir/file, object extent -> file offset
                      emits { op=WRITE, path=/dir/file, offset=Y, bytes=B }

Source-local inode numbers are source-side identities. Only paths and
arguments cross the gateway boundary; ``source_inode`` is gateway
bookkeeping (e.g. for the per-inode ordering queue). The replica allocates
its own inodes as the backend replays the path operations.

F3. OSD traffic yields data operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CephFS file data lives in RADOS objects. An OSD write decodes to
``(source_inode, object_number, object_offset, length, payload)``. Using the
source file layout, the gateway maps that object extent to one or more
logical file extents and emits:

::

   { op=WRITE, path=/dir/file, offset=file_offset, bytes=payload }

The replica need not share the source's layout: the translation targets
logical file offsets, not target RADOS objects.

The gateway learns a file's layout from the MDS create reply and treats it as
immutable once the file holds data; a layout set via the ``ceph.*.layout``
xattr on an empty file before its first write updates ``layout_table`` too, so
the stripe math never runs on a stale layout.

F4. Correct order is derived from source tokens before replay
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The order recovered here is well-defined only because the source has already
serialized conflicting operations — the cap foundation established in F6
below. Given that, the gateway merges MDS-derived metadata events and
OSD-derived data events into one order. That order is reconstructed from
source-assigned sequence
tokens carried in the wire messages, not from observation order. Because a
file is striped across many objects on many OSDs over many connections, even
one client writing one file produces parallel streams with no inherent
ordering between them. The source's tokens are identical regardless of which
connection or collector observes a message, which is what makes a coherent
order recoverable without a shared clock.

The ordering tokens:

* **per-object** ``user_version`` — RADOS stamps each object with a
  monotonic version, returned in the OSD op reply (``MOSDOpReply``). Writes
  that conflict land on the same object and are totally ordered by
  ``user_version``; writes to different objects do not conflict, so their
  relative order does not affect final state. This is the canonical
  data-path key.
* ``truncate_seq`` — carried on every OSD op request; orders truncate
  against surrounding writes.
* **per-session op id** — OSD ``reqid`` and MDS request ``tid``, monotonic
  per client session; orders one client's operations across connections.
* **MDS per-inode metadata version / journal sequence** — orders metadata
  operations on an inode.

The gateway joins a request's ``truncate_seq`` to a reply's ``user_version``
on the op ``reqid``/``tid`` — the same request/reply correlation that strict
capture already performs.

Replaying an object's writes in ``user_version`` order therefore reproduces
that object's exact final bytes — ``user_version`` *is* the source's commit
order for that object — and per-object equality, with the object-to-file
offset mapping of F3, gives whole-file equality. Because the gateway keys on
committed OSD writes, logical writes a client buffered and coalesced away
never entered the source's durable state and so need not be replayed.

F5. The remote side stays simple
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The remote side receives normalized, ordered, token-carrying records
``{ op, path, args, tokens }`` and applies them to the replica. Because the
tokens travel with each record, the backend can dedupe idempotently per
object — a data record whose ``user_version`` is not newer than what the
replica already holds for that object is a duplicate and is dropped — and so
tolerates slight reordering and parallelizes per inode. Once derivation and
order are correct, replay is the easy side of the design.

F6. Caps make the source order well-defined; tokens carry it to the gateway
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The tokens of F4 are only useful if the source has actually serialized
conflicting operations into a well-defined order. CephFS capabilities
provide that serialization.

CephFS issues per-inode capabilities (Fr, Fw, Fb, Fc, Fx, Ax, Lx, Xx) that
gate what a client may do without coordinating with the MDS. A client cannot
modify state it does not hold the cap for, and the MDS revokes conflicting
caps before granting them. Consequently two clients never hold conflicting
exclusive caps on an inode at once; a client modifying under exclusive caps
does so with no concurrent modification; and on revocation a client flushes
its dirty state before releasing, with the next holder granted only after
that flush is acknowledged. The source therefore produces a well-defined
per-inode commit order, and the gateway recovers it from the tokens stamped
on the committed operations.

Cap handoff — Fb revoke, flush, grant, where two clients touch one inode in
sequence — is the case most likely to reorder. It needs no special handling:
because the successor is granted its cap only after the predecessor's flush
is acknowledged, the successor's writes carry strictly higher
``user_version`` values on the shared objects, so ``user_version`` ordering
reproduces the handoff order directly. The same holds when multiple writers
force the filelock to LOCK_MIX and Fb is revoked from all of them: writes
become synchronous and the OSD assigns ``user_version`` in commit order,
which the gateway recovers from the replies.

Architecture
------------

High-level design
~~~~~~~~~~~~~~~~~~

::

                         Source CephFS Cluster
                  +--------------------------------+
                  |              MDS               |
                  |              OSDs              |
                  +--------^-----------------^-----+
                           |                 |
                           | replies         | replies
                           |                 |
   CephFS clients    +-----+-----------------+-----+
   +-------------+   |   Bidirectional Capture      |
   | unmodified  |<->|   Layer                      |
   | clients     |   |   forwards traffic unchanged |
   +-------------+   +-----+------------------+-----+
                           |                  |
                           | MDS req/reply    | OSD write req/reply
                           v                  v
                    +-------------------------------+
                    |      Translation Gateway      |
                    |  decode -> correlate -> order |
                    |  inode -> path -> path+op     |
                    +---------------+---------------+
                                    |
                                    | ordered replay stream
                                    | { op, path, args, tokens }
                                    v
                    +-------------------------------+
                    | Remote libceph Client Backend |
                    |  applies path+op records      |
                    +---------------+---------------+
                                    |
                                    v
                         Replica CephFS Cluster
                    +-------------------------------+
                    |          MDS + OSDs           |
                    |    passive filesystem copy    |
                    +-------------------------------+

Reverse-Ganesha view
~~~~~~~~~~~~~~~~~~~~~~

::

   NFS-Ganesha:
       NFS/SMB protocol  ->  Ganesha translation  ->  CephFS client ops

   This POC:
       Ceph client wire traffic  ->  gateway translation  ->  path+op stream
                                                          ->  remote libceph backend
                                                          ->  replica CephFS

Component responsibilities
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Component
     - Responsibility
   * - Capture layer
     - Stands inline between the source daemons and the world, forwarding
       source traffic unchanged while copying relevant MDS and OSD traffic.
       Holds minimal state: enough parsing to decide whether a message is
       relevant and to extract its tokens, then forwards. No path or
       namespace state lives in the capture layer.
   * - Translation gateway
     - Decodes events, correlates requests and replies, maintains
       source-local inode/path/layout state, and orders by source token.
       Ordering state is sharded by source inode so every event for a file
       lands in one ordering domain.
   * - Remote libceph backend
     - Applies the ordered ``path + op`` records to the replica, deduping
       idempotently per object/inode using the carried tokens.

Distributed capture
~~~~~~~~~~~~~~~~~~~~~

The capture layer is drawn above as a single inline proxy, and the POC may
implement it that way. The design does not depend on a single observation
point: it is equally a set of daemon-adjacent collectors, one inline in
front of each OSD and the MDS, each forwarding its daemon's traffic
unchanged while copying relevant events to the gateway.

This is sound because ordering is recovered from source-assigned tokens that
are identical regardless of which collector observes a message. Collectors
need not agree on a global order, share a clock, or coordinate. Each parses
minimally, stamps the source tokens, and forwards; the gateway reassembles
by source inode and orders by token. Routing by source inode is therefore a
correctness requirement of the distributed model — every event for a file
must reach one ordering domain — not merely a scaling choice.

Traffic to Operation Derivation
-------------------------------

::

   captured packet
      |
      v
   decoded Ceph message
      |
      v
   normalized source event
      |
      v
   identity enrichment
      |       MDS parent inode/name -> path
      |       MDS reply inode       -> source-local inode
      |       OSD object name       -> source-local inode/object
      |       layout table          -> file offset
      v
   ordered replay record
      |
      v
   { op, path, args, tokens }

Replay record shape
~~~~~~~~~~~~~~~~~~~~~

::

   op                  # CREATE, MKDIR, WRITE, RENAME, SETXATTR, TRUNCATE, ...
   path                # replica-relative logical path
   old_path/new_path   # for rename-like operations
   source_inode        # source-local inode, gateway bookkeeping
   object              # source object id, for data ops
   user_version        # per-object RADOS version (data-path order key)
   truncate_seq        # truncate-vs-write order key
   op_id               # source-session op id (OSD reqid / MDS tid)
   args                # mode, uid/gid, xattr, offset, length, payload, size

The backend applies ``op``, ``path``, and ``args``, and uses the tokens to
order and dedupe per object/inode. Source-local inode, object, and layout
details are gateway-side derivation state and never identify anything on the
replica.

Local Inode to Path Translation
-------------------------------

OSD writes do not name a path; they name an object for a source-local inode.
The gateway therefore maintains a live inode-to-path mapping, updated only
from confirmed MDS operations — a failed create, rename, unlink, or setattr
changes nothing and emits nothing.

Core tables
~~~~~~~~~~~~

::

   inode_table:   source_inode -> { paths[], type, nlink, state, layout_ref }
   path_table:    source_path  -> source_inode
   dentry_table:  (parent_source_inode, name) -> child_source_inode
   layout_table:  source_inode -> { stripe_unit, stripe_count,
                                    object_size, pool }

Per-operation derivation
~~~~~~~~~~~~~~~~~~~~~~~~~~

Each confirmed operation updates the tables and emits a path-based record.
The emitted record is always path-based; the replica chooses its own inode
numbers.

.. list-table::
   :header-rows: 1
   :widths: 14 40 46

   * - Op
     - Captured source traffic
     - Gateway derivation and table updates
   * - CREATE
     - ``CREATE(parent=P, name="file")`` -> ``ok, inode=I, layout=L``
     - ``path = path(P)+"/file"``; add ``I->path`` to ``inode_table``,
       ``path_table``, ``dentry_table``; ``layout_table[I]=L``. Emit
       ``{CREATE, path, mode, type}``.
   * - WRITE
     - ``WRITE(object="I.objno", object_offset=X, length=N, bytes=B)`` ->
       ``ok, user_version=V``
     - Resolve ``I -> path``; map object extent to ``file_offset`` via
       ``layout_table[I]`` (see Data Extent Translation). Emit
       ``{WRITE, path, offset=file_offset, bytes=B}`` with ``V`` and
       ``truncate_seq``.
   * - RENAME
     - ``RENAME(old_parent=P1, old_name="a", new_parent=P2, new_name="b")``
       -> ``ok``
     - ``I = path_table[path(P1)+"/a"]``; move ``I``'s path to
       ``path(P2)+"/b"`` in ``inode_table``/``path_table``/``dentry_table``.
       Emit ``{RENAME, old_path, new_path}``. Subsequent writes to ``I``
       resolve to the new path.
   * - LINK
     - ``LINK(existing_inode=I, new_parent=P, new_name="b")`` -> ``ok``
     - Add ``path(P)+"/b"`` to ``inode_table[I].paths`` and ``path_table``.
       Emit ``{LINK, existing_path=some_path(I), new_path}``. An inode may
       hold several paths; any current path names the same file for data
       writes.
   * - UNLINK
     - ``UNLINK(parent=P, name="file")`` -> ``ok``
     - ``path = path(P)+"/file"``; ``I = path_table[path]``; remove that
       path from ``inode_table[I].paths`` and ``path_table``. Emit
       ``{UNLINK, path}``. If ``I`` has no remaining path, later writes to
       it are unlink-while-open; final-state convergence holds because the
       file is deleted.
   * - SETATTR / xattr
     - ``SETATTR(target, fields)`` or ``SETXATTR(...)`` -> ``ok``
     - Resolve ``target -> path``. Emit ``{SETATTR, path, fields}``. A
       setattr that sets ``size`` is size-affecting and enters the per-inode
       ordering queue with writes (see Size-affecting operations).

Data Extent Translation
-----------------------

The gateway uses source layout only to convert source object coordinates to
logical file offsets; it does not replicate source layout to the target.

Inputs: from OSD traffic, ``(source_inode, object_number, object_offset,
length, payload)``; from the layout table, ``(stripe_unit, stripe_count,
object_size)``. Output: ``(file_offset, length, payload)``.

Stripe math
~~~~~~~~~~~

::

   stripes_per_object = object_size / stripe_unit
   objsetno           = object_number / stripe_count
   stripepos          = object_number % stripe_count
   objsetpos          = object_offset / stripe_unit
   blockoff           = object_offset % stripe_unit
   stripeno           = objsetno * stripes_per_object + objsetpos
   blockno            = stripeno * stripe_count + stripepos
   file_offset        = blockno * stripe_unit + blockoff

A write crossing a stripe boundary is split into multiple logical file
extents, each emitted as an ordered WRITE record for the same source event.

::

   OSD write packet
   +------------------------------------------------------+
   | object = <source_inode>.<object_number>              |
   | object_offset, length, payload                       |
   +---------------------------+--------------------------+
                               |
                               v
                  +--------------------------+
                  | Source Layout Table      |
                  | stripe math              |
                  +-------------+------------+
                                |
                                v
                  +--------------------------+
                  | file_offset + payload    |
                  +-------------+------------+
                                |
                                v
                  +--------------------------+
                  | Source inode -> path     |
                  +-------------+------------+
                                |
                                v
                  +--------------------------+
                  | { op=WRITE, path,        |
                  |   offset, bytes }        |
                  +--------------------------+

Ordering and Consistency
------------------------

Strict source confirmation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Events are emitted only when the source operation has durably committed:

* MDS mutation: durable success reply emits; failure reply drops.
* OSD write: commit/ondisk reply emits; failure reply drops.

"Confirmed" means the commit/ondisk reply, never the early ack. Ceph can
send an early ack before a write is durable; replaying on the ack would let
the replica hold a write the source later loses on OSD failover or peering.
The ``user_version`` the gateway orders on arrives in that same commit reply,
so confirmation and ordering token arrive together.

Capture is inline with the source's TCP stream, so a collector observes every
byte its daemon does: capture is lossless by construction, and the design
carries no capture-loss-recovery machinery.

Ordering domains
~~~~~~~~~~~~~~~~~

The gateway preserves order in the domains where order affects final state:
per-file data and size operations; same-inode metadata operations; namespace
operations on the same path or parent directory; and rename/link/unlink
operations that change the inode-to-path table.

Rename against in-flight writes needs no replay-time coupling, because a
rename and a write to the same inode commute for convergence: the write
targets the inode's objects regardless of the inode's current name, and the
rename relocates the inode together with its data, so the replica converges
whichever the gateway emits first — including across clients. The mechanism
that realizes this is path resolution at emit time: a write names only an
inode, so the gateway resolves it to the inode's current path against the
table as it emits the record, while a rename updates that table. Competing
namespace operations on the same name are ordered by the MDS journal
sequence.

Size-affecting operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Caps do not serialize truncate against in-flight writes; ``truncate_seq``
does. The client stamps the post-truncate ``truncate_seq`` on every
subsequent write, and the OSD discards writes carrying a stale value, so the
truncate's effect is encoded in the ``truncate_seq`` of the writes the
gateway already captures, together with the MDS ``setattr(size)`` reply. No
visibility into the MDS-issued ``CEPH_OSD_OP_TRIMTRUNC`` (which travels
MDS-to-OSD on the cluster network) is required.

The size-affecting operations — ``OSD_OP_WRITE``, ``OSD_OP_WRITEFULL``,
``OSD_OP_TRUNCATE``, ``OSD_OP_ZERO``, and MDS ``setattr`` with ``size`` set —
share a single per-inode ordering point, ordered by ``truncate_seq`` and by
``user_version`` within a ``truncate_seq`` epoch.

::

   MDS setattr(size) --+
   OSD_OP_WRITE -------+
   OSD_OP_WRITEFULL ---+--> single per-inode ordering point,
   OSD_OP_TRUNCATE ----+    ordered by truncate_seq, then user_version
   OSD_OP_ZERO --------+
                       |
                       v
              emit ordered path+op records

Bootstrap
---------

Bootstrap brings the pre-existing source filesystem into the replicated
state. The source filesystem exists before replication starts, so bootstrap
is what closes the gap between "filesystem exists" and "replica converges
from live capture alone." Its correctness sits on the same level as
live-capture correctness.

Once capture is running, every new mutation is observable; bootstrap only
needs to bring the *pre-existing* state across, and the live stream handles
everything after. So bootstrap reduces to: copy a consistent snapshot of an
existing subtree to the replica, seed the gateway's translation tables for
the copied files, and hand off to live replay without missing a mutation in
the seam.

Every entry is in one of three states:

* **Replicated** — present on the replica and consistent with the source;
  live capture mutates it normally. Steady state.
* **Priming** — being copied; live capture against it is held until copy
  completes.
* **Non-replicated** — pre-existing, not yet copied; live capture is held
  until the entry transitions to Priming.

Transitions run ``Non-replicated -> Priming -> Replicated``.

Strategy A: Quarantine-then-copy (primary)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The primary strategy uses an MDS-level quarantine on the subtree being
copied. Quarantine blocks all client mutations within the subtree —
namespace operations, attribute changes, and data writes — while permitting
reads. Because no mutation can occur during the copy, the copied data is a
consistent point-in-time snapshot and there is no race between copy and live
capture.

Quarantine in this form is a small, well-scoped MDS feature: a per-subtree
flag that causes the MDS to reject or block mutating client requests within
the subtree until cleared, durable across MDS restart for the duration of
bootstrap. Read-side caps are unaffected.

Procedure:

#. Start capture. The capture layer forwards MDS and OSD events to the
   gateway. All pre-existing entries are Non-replicated; entries created
   after this point are tracked live and go directly to Replicated.
#. Walk the source tree and select the next subtree to bootstrap.
#. Quarantine the subtree.
#. Copy the subtree contents to the replica directly (not through the
   live capture/replay pipeline, which carries only deltas).
#. Seed the gateway's tables for the copied subtree: ``inode_table``
   (inode, paths, type, nlink); ``layout_table`` read via
   ``ceph_get_layout`` per file, since pre-existing files have no MDS create
   reply to observe; and ``path_table`` / ``dentry_table`` for the paths.
#. Transition the subtree from Non-replicated to Replicated.
#. Release quarantine; writes resume and flow through the live pipeline.
#. Repeat until the whole scope is bootstrapped.

Quarantine windows are client-visible write pauses, bounded by:

::

   quarantine_window ~= subtree_size / min(source_read_bw,
                                           link_bw,
                                           replica_write_bw)

Smaller subtrees give shorter pauses at the cost of more cycles. The POC
measures this relationship; a policy that bounds maximum pause duration is
more useful operationally than a fixed subtree size.

Boundary rules:

* **Rename across the quarantine boundary** is blocked by definition;
  renames between two non-quarantined subtrees are captured and replayed
  live. Nothing can move into a subtree being copied, which keeps the copy
  snapshot consistent.
* **Hardlinks across subtree boundaries**: a multiply-linked file is copied
  during bootstrap of the first containing subtree; when a later subtree is
  bootstrapped and the gateway encounters a link to an already-replicated
  inode (identified via ``inode_table[I].paths``), the backend creates a
  hardlink rather than re-copying data.
* **Failure mid-copy**: quarantine is held, partial replica state is rolled
  back or dropped and recopied, and the step retries. The subtree stays
  Non-replicated until copy and transition succeed atomically; there is no
  partial-replicated state.

Strategy B: Buffer-and-replay
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For workloads that cannot tolerate quarantine pauses, mutations are allowed
during copy and reconciled afterward. The source-token ordering of live
capture is what makes the reconciliation tractable.

Per file:

#. Mark the subtree Priming. Live capture for it is buffered rather than
   replayed — events are tagged with their tokens and held per inode.
#. Copy each file's contents to the replica concurrently with live client
   activity, recording per object the ``user_version`` the copy observed.
#. Drain the per-inode buffers against the copied state, ordered by token.
   A buffered data event whose ``user_version`` is not newer than the copied
   version for that object is a duplicate and is dropped; anything newer is
   applied. Writes at a given offset are idempotent, so a misjudged
   duplicate replayed anyway is harmless on the data path. Metadata
   operations are reconciled by MDS metadata version / op id.
#. Transition the subtree to Replicated once buffers are drained.

Version comparison resolves the copy-vs-mutation race deterministically:
whether the copy already included a mutation is answered by comparing
source-assigned object versions, not by reconstructing wall-clock timing. B
requires no MDS support, but it does require capture to be live across the
whole subtree from the moment Priming begins.

Seam between bootstrap and live capture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The correctness requirement at handoff is that no mutation committed between
"capture started" and "subtree Replicated" is lost.

Strategy A satisfies this because quarantine excludes mutations during the
copy window: a mutation before quarantine is either in the live stream (if
captured) or in the copy (if it predated capture); a mutation after release
is in the live stream by construction.

Strategy B satisfies this when the buffer captures every mutation from the
moment Priming begins and the drain reconciles by object version — applying
buffered events newer than the copied version and dropping those already
included.

::

   start capture (entire scope, all files Non-replicated)
        |
        v
   for each subtree in scan order:
        |
        v
   +-----------------+      +------------------+      +-------------------+
   | MDS quarantine  |----->| copy subtree to  |----->| seed inode/path   |
   | subtree         |      | replica          |      | layout tables     |
   +-----------------+      +------------------+      +-------------------+
        |                                                       |
        v                                                       v
   client writes blocked                          transition to Replicated
   within subtree                                                |
                                                                 v
                                                       release quarantine
                                                                 |
                                                                 v
                                                         live replay

Future Work
-----------

The following extend the approach once feasibility is established. Each
assumes the POC's core mechanism and adds capability above it.

Multiple active MDS ranks
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Supporting ``max_mds > 1`` adds capacity by partitioning the namespace
across ranks. Cross-rank rename and subtree export/import coordinate over
the inter-MDS network, so multi-rank support orders these from inter-MDS
traffic or from the authoritative client-facing completion replies.

Active-active replication
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Both clusters accept client writes, with changes propagating in both
directions. This requires conflict detection, a conflict-resolution policy,
and a stronger consistency model than eventual convergence — a substantially
different design from the single-source, passive-replica model here.

Snapshots, quotas, and failover
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Replicating CephFS snapshots and quota state, and supporting replica
promotion with client failover, extend the model from data/metadata
convergence toward full disaster-recovery semantics.

Secure msgr2
~~~~~~~~~~~~~

Encrypted traffic cannot be parsed passively. A future design obtains
plaintext at a trusted integration point or via a source-side event export
path, preserving the same token-based derivation.

WAN optimization
~~~~~~~~~~~~~~~~~

Traffic between the gateway and the remote backend can be compressed,
batched, write-coalesced, and deduplicated (traffic redundancy elimination,
content-defined chunking), with backpressure from remote lag. Optimization
may transform the transport representation as long as the remote recovers
the same ordered token-carrying records.

Sharding and load balancing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Multiple gateway shards scale throughput, each owning a subset of files —
by subtree for namespace events, by source-local inode for data. The
ordering rule is fixed: all events that can affect one file reach the same
ordering domain.

High availability and recovery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Production recovery adds explicit handling for capture-layer and gateway
failure, replay-log overflow, and missed traffic — fail-closed capture,
durable logs, bounded dirty sets, selective recopy, and drift detection.

Live verifier on a consistent cut
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The POC verifies against a quiesced source. A live verifier compares a
mutating source against a lagging replica using a consistent cut — CephFS
snapshots or a version-stamped frontier from the same ordering tokens — with
drift triggering selective recopy.
