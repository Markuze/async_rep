Live CephFS Async Replication POC — Reverse Ganesha
===================================================

Goal
----

Demonstrate that asynchronous cross-cluster CephFS replication can be built
by observing client wire traffic and replaying it as filesystem operations
on a passive replica.

We propose a transparent proxy in the client-to-cluster network path
captures MDS and OSD traffic in both directions. A translation gateway
parses captured packets, maintains source-side inode-to-path and layout
state, and emits an ordered stream of path-based filesystem operations.
A remote libceph client backend applies that stream to the replica,
which converges to the source's POSIX-visible state.

The replication is live and async with bounded lag — not snapshot-based,
not synchronous, not active-active. The mechanism scales to a full filesystem.

There are two feasibility claims that need proving. First, that the
gateway can reconstruct VFS-level filesystem operations from Ceph wire
packets — recovering the operation, target path, arguments, and payload
from MDS request/reply pairs and OSD object writes, for the hard cases
as well as the easy ones (truncate/write races, unlink, rename,
hardlinks, bootstrap of pre-existing data). Second, that those
reconstructed operations can be replayed in an order that converges the
replica to the source's state. The ordering claim rests on CephFS
capabilities: caps serialize concurrent access at the source, so
wire-arrival order is a valid replay order — with one named exception,
truncate, which uses an OSD-level sequencing mechanism the gateway
cannot observe and handles instead through per-inode serialization in
the replay path.

The replay side is straightforward libceph once derivation and ordering
are correct.

Status
------

Proposal for discussion and proof-of-concept development.

Purpose
-------

This document proposes a POC to test one specific feasibility claim:

    A packet-sniffing proxy can observe enough CephFS client traffic to
    derive an ordered stream of filesystem operations, and a remote
    libceph client backend can replay those ``path + op`` records into a
    passive replica CephFS cluster.

The POC is not a production DR design. It is a proof that the semantic
translation is possible:

::

   Ceph client wire traffic  ->  ordered filesystem operation stream
                              ->  remote libceph client backend
                              ->  replica CephFS state

This is the "reverse Ganesha" idea. NFS-Ganesha accepts NFS/SMB protocol
operations and translates them into CephFS client operations. This POC
observes Ceph client protocol operations and translates them back into a
filesystem operation stream that can be replayed on another CephFS cluster.

The important proof point is the first half of the pipeline: can the proxy
and gateway derive the exact logical operation, target path, file offset,
metadata arguments, and ordering constraints from network traffic? Once the
system has derived a correct ordered ``path + op`` stream, the remote
libceph client backend is a straightforward replay sink. The exact backend
implementation details are intentionally not the subject of this POC.

Scope
-----

The POC demonstrates that the system can:

* observe MDS request/reply traffic for namespace and metadata mutations;
* observe OSD write request/reply traffic for file data mutations;
* build a source-local inode to path mapping;
* map OSD object writes back to logical file writes;
* produce an ordered stream of filesystem operations;
* feed those operations to a remote libceph client backend;
* bootstrap a pre-existing subtree into the replicated state, including
  the MDS-side quarantine mechanism, table seeding, and the
  bootstrap-to-live-capture seam;
* converge the replica to the source's POSIX-visible state;
* verify convergence with a source-vs-replica comparison.

POC Assumptions
---------------

The POC assumes:

* one source CephFS filesystem;
* one passive replica CephFS filesystem;
* unmodified source clients;
* a logical capture point that sees all relevant client-to-MDS,
  MDS-to-client, client-to-OSD, and OSD-to-client traffic;
* inspectable traffic, for example msgr2 crc/plaintext-inspectable mode;
* strict capture: successful source replies are observed before events are
  considered replayable;
* source-rejected operations are not replayed;
* the replica is written only by the remote libceph client backend.

Core Claim
----------

The POC proves feasibility if these statements are true in practice.

F1. MDS traffic can produce namespace operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MDS mutation requests carry operation intent: create, mkdir, unlink,
rename, link, symlink, setattr, setxattr, removexattr, and similar
filesystem operations.

MDS replies carry the source cluster's result: success/failure, allocated
source-local inode number, type, attributes, and layout where relevant.

Together, request and reply are enough to derive a logical metadata
operation such as:

::

   { seq, op=CREATE, path=/dir/file, type=file, mode=0644 }
   { seq, op=RENAME, old_path=/dir/a, new_path=/dir/b }
   { seq, op=SETXATTR, path=/dir/file, name=user.k, value=v }

F2. Source-local inode to path translation can bridge MDS and OSD traffic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MDS traffic and OSD traffic identify files differently:

* MDS traffic is namespace-oriented: path, parent inode, dentry name.
* OSD traffic is object-oriented: source-local inode, object number,
  object offset, length, payload.

The gateway must therefore maintain a local inode to path translation
model. The source-local inode is the join key.

::

   MDS create reply
        teaches: source-local inode I belongs to /dir/file

   Later OSD write
        names:   source-local inode I, object N, offset X

   Gateway joins them
        I -> /dir/file
        object extent -> file offset
        emits: { op=WRITE, path=/dir/file, offset=Y, bytes=B }

Source-local inode numbers are source-side identities only. The gateway
must not emit source-local inode numbers into the replay stream as
identifiers — only paths and arguments cross the gateway boundary. The
replay record carries ``source_inode`` as bookkeeping for the gateway
itself (e.g. for the per-inode ordering queue), but the remote backend
operates entirely on paths. The replica cluster allocates its own target
inodes naturally when the remote libceph backend replays the derived
path operations.

F3. OSD traffic can produce data operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CephFS file data is written to RADOS objects. The object identity and write
extent can be decoded into:

::

   source-local inode
   object number
   object offset
   length
   payload

Using the source file layout, the gateway maps that object extent back to
one or more logical file extents:

::

   (source_inode, object_number, object_offset, length)
      -> [(file_offset, length_i, payload_i), ...]

The emitted replay operation is then simply:

::

   { seq, op=WRITE, path=/dir/file, offset=file_offset, bytes=payload_i }

The replica does not need to use the same CephFS layout as the source. The
translation is to logical file offsets, not to target RADOS objects.

F4. Correct ordering can be derived before replay
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The POC must prove that the gateway can merge MDS-derived metadata events
and OSD-derived data events into a consistent order.

For ordinary metadata and writes, the gateway preserves source-observed
order within the relevant inode or namespace domain.

For size-affecting operations, especially truncate/write races, the gateway
uses a per-inode size-affecting queue. This is required because source
CephFS uses internal truncate sequencing — a per-inode ``truncate_seq``
counter stamped on every OSD write, and MDS-driven ``CEPH_OSD_OP_TRIMTRUNC``
ops issued directly from MDS to OSDs on the cluster network — that the
gateway neither sees nor can replay through libcephfs. The per-inode queue
imposes source-equivalent order on the replay stream instead. See the
Size-affecting operations section under Ordering and Consistency.

F5. The remote side can remain simple
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The remote side is a remote libceph client backend. It receives already
normalized and ordered replay records:

::

   { seq, op, path, args }

and applies them to the replica CephFS filesystem.

This document does not need to describe exactly how each operation is
implemented inside the remote backend. Once the gateway has derived the
correct operation, target path, arguments, and order, replay through a
CephFS-capable client backend is the easy side of the design. The POC is
about proving the derivation.

F6. Cap-based source serialization is the foundation that makes wire order a valid replay order
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The gateway emits operations in source wire-arrival order and expects
this to be a source-equivalent replay order. The reason this works at
all is CephFS capabilities. The proof obligation is to demonstrate that
caps are a sufficient ordering foundation for the replay stream, and to
name the case where they are not.

**What caps give us.** CephFS issues per-inode capabilities (Fr, Fw, Fb,
Fc, Fx, Ax, Lx, Xx, etc.) that gate what clients may do without
coordinating with the MDS. A client cannot modify state it does not hold
the appropriate cap for; the MDS revokes conflicting caps from other
clients before granting them. So:

* Two clients cannot simultaneously hold conflicting exclusive caps on
  the same inode. Cross-client conflicts are serialized by the MDS
  through cap revoke/grant cycles.
* A client modifying state under exclusive caps does so without any
  concurrent modification from another client. The order in which that
  client issues operations on the wire is the order they were committed.
* When the MDS revokes a cap, the client must flush the dirty state it
  held under that cap before releasing. The flushed state crosses the
  proxy in observable wire order, after which the next holder can
  proceed.

The combined effect is that per-inode commit order on the source equals
per-source-TCP wire order observed by the proxy, modulo the cap handoff
sequence. The gateway's only job is to preserve that order across the
MDS and OSD streams it has merged.

**Multi-writer behavior.** When multiple clients write the same file,
the MDS transitions the filelock to LOCK_MIX and revokes Fb (buffered
write) from all holders, forcing all writers to issue synchronous OSD
writes. Synchronous writes commit at the OSD in wire-arrival order; no
client-side reordering can occur. The proxy sees the same total order
the OSDs do.

**The exception: size-affecting operations.** Caps do not serialize
truncate against in-flight writes. Truncate uses a separate mechanism —
the per-inode ``truncate_seq`` counter stamped on OSD writes, with
MDS-driven ``CEPH_OSD_OP_TRIMTRUNC`` ops issued directly from MDS to
OSDs on the cluster network — which the gateway cannot observe and
cannot replay through libcephfs. For size-affecting operations the
gateway substitutes its own per-inode serialization (see Ordering and
Consistency). This is the named exception to the cap-foundation claim;
everywhere else, caps are sufficient.

**The non-exception that looks like one: cross-object-boundary writes
from simultaneous writers.** CephFS already deviates from strict POSIX
atomicity for this case (the "aa|bb" tearing example in the Ceph
documentation). Whatever interleaving the source produces is what the
replica receives, so convergence holds — but the convergence target is
the source's behavior, not strict POSIX. This is a property of CephFS,
not a flaw in the replication design.

**What the POC must demonstrate.**

#. Two clients writing the same file under cap revocation produce a
   wire stream that, when replayed in order, yields the same final file
   contents on the replica.
#. A reader on one client observing size/mtime under cap-mediated stat
   sees a consistent view that the replica reproduces.
#. The truncate exception is handled by the per-inode size-affecting
   queue, not by caps — verified by the truncate/write race test in
   the test matrix.
#. No cap-mediated handoff (Fb revoke and flush, LOCK_MIX transition,
   stat-driven size collection) leaves a write unaccounted for on the
   replica.

Together these prove that caps are a sufficient ordering foundation
for everything except the explicitly-handled truncate case, and that
the gateway's wire-order replay is therefore source-equivalent.

Architecture
------------

High-level design
~~~~~~~~~~~~~~~~~

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
   | unmodified  |<->|   Proxy                      |
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
                                    | { seq, op, path, args }
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
                                    ^
                                    |
                              +-----+-----+
                              | Verifier  |
                              +-----------+

Reverse-Ganesha view
~~~~~~~~~~~~~~~~~~~~

::

   NFS-Ganesha:

       NFS/SMB protocol  ->  Ganesha translation  ->  CephFS client ops

   This POC:

       Ceph client wire traffic  ->  gateway translation  ->  path+op stream
                                                          ->  remote libceph backend
                                                          ->  replica CephFS

Component responsibilities
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Component
     - Responsibility
   * - Bidirectional Capture Proxy
     - Observes and copies relevant MDS and OSD traffic while forwarding
       source traffic unchanged.
   * - Translation Gateway
     - Decodes packets, correlates requests and replies, maintains
       source-local inode/path/layout state, derives ordered filesystem
       operations.
   * - Remote libceph Client Backend
     - Receives the ordered ``path + op`` stream and applies it to the
       replica CephFS filesystem.
   * - Verifier
     - Compares source and replica state after replay drains.

Traffic to Operation Derivation
-------------------------------

The core of the POC is the derivation path from packets to filesystem
operations.

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
   { seq, op, path, args }

Replay record shape
~~~~~~~~~~~~~~~~~~~

The internal representation does not need to be final for the POC, but the
conceptual replay record should contain:

::

   seq                 # ordering key
   op                  # CREATE, MKDIR, WRITE, RENAME, SETXATTR, TRUNCATE, ...
   path                # replica-relative logical path
   old_path/new_path   # for rename-like operations
   source_inode        # source-local inode, for gateway bookkeeping
   args                # mode, uid/gid, xattr, offset, length, payload, size
   ack_state           # observed, confirmed, dropped

The remote backend consumes only the operation stream it needs: operation,
path, arguments, and order. Source-local inode and layout details are
gateway-side derivation state.

Local Inode to Path Translation
-------------------------------

Why this is required
~~~~~~~~~~~~~~~~~~~~

OSD writes do not say "write to /dir/file". They say, in effect, "write to
object N for source-local inode I". The remote backend cannot replay that
without a path. The gateway therefore creates and maintains a live mapping
from source-local inode to source path.

Core tables
~~~~~~~~~~~

::

   inode_table:
       source_inode -> {
           paths[],
           type,
           nlink,
           state,
           layout_ref
       }

   path_table:
       source_path -> source_inode

   dentry_table:
       (parent_source_inode, name) -> child_source_inode

   layout_table:
       source_inode -> {
           stripe_unit,
           stripe_count,
           object_size,
           pool
       }

The mapping is updated only from confirmed MDS operations. A failed create,
rename, unlink, or setattr does not change the table and does not produce a
replayable operation.

Create
~~~~~~

Captured source traffic:

::

   client -> MDS: CREATE(parent_inode=P, name="file")
   MDS -> client: success, new_inode=I, layout=L

Gateway derivation:

::

   parent_path = inode_table[P].path
   path        = parent_path + "/file"

   inode_table[I].paths += path
   path_table[path] = I
   dentry_table[(P, "file")] = I
   layout_table[I] = L

   emit { seq, op=CREATE, path=path, args={mode,type} }

The emitted operation is path-based. The target inode number will be chosen
by the replica cluster, not copied from the source.

Write
~~~~~

Captured source traffic:

::

   client -> OSD: WRITE(object="I.objno", object_offset=X, length=N, bytes=B)
   OSD -> client: success

Gateway derivation:

::

   I           = parse_source_inode(object)
   path        = choose_current_path(inode_table[I].paths)
   layout      = layout_table[I]
   file_extent = map_object_extent_to_file_extent(layout, objno, X, N)

   emit { seq, op=WRITE, path=path, args={offset=file_offset, bytes=B} }

The gateway has converted object identity into filesystem identity.

Rename
~~~~~~

Captured source traffic:

::

   client -> MDS: RENAME(old_parent=P1, old_name="a",
                        new_parent=P2, new_name="b")
   MDS -> client: success

Gateway derivation:

::

   old_path = path(P1) + "/a"
   new_path = path(P2) + "/b"
   I        = path_table[old_path]

   update inode_table[I].paths: old_path -> new_path
   update path_table and dentry_table

   emit { seq, op=RENAME, old_path=old_path, new_path=new_path }

Future writes to source-local inode ``I`` resolve to the new path.

Link
~~~~

Captured source traffic:

::

   client -> MDS: LINK(existing_inode=I, new_parent=P, new_name="b")
   MDS -> client: success

Gateway derivation:

::

   new_path = path(P) + "/b"
   inode_table[I].paths += new_path
   path_table[new_path] = I

   emit { seq, op=LINK, existing_path=some_path(I), new_path=new_path }

A hardlinked inode may have multiple paths. For data writes, any current
path for the inode is sufficient because all paths name the same file.

Unlink
~~~~~~

Captured source traffic:

::

   client -> MDS: UNLINK(parent_inode=P, name="file")
   MDS -> client: success

Gateway derivation:

::

   path = path(P) + "/file"
   I    = path_table[path]

   remove path from inode_table[I].paths
   remove path_table[path]

   emit { seq, op=UNLINK, path=path }

If an inode has no remaining path, later data writes to it are treated as
unlink-while-open behavior. For the POC, transient writes to pathless
inodes are out of scope; final-state convergence is still correct because
the file is deleted.

Setattr and xattr
~~~~~~~~~~~~~~~~~

Captured source traffic:

::

   client -> MDS: SETATTR(target, fields)
   MDS -> client: success

Gateway derivation:

::

   path = resolve_target_to_path(target)

   emit { seq, op=SETATTR, path=path, args=fields }

If the setattr includes file size, it is classified as size-affecting and
enters the per-inode ordering queue with writes.

Data Extent Translation
-----------------------

The gateway needs source layout only to convert source object coordinates
to logical file offsets. It does not replicate source layout to the target.

Object extent input
~~~~~~~~~~~~~~~~~~~

From OSD traffic:

::

   source_inode
   object_number
   object_offset
   length
   payload

From the source layout table:

::

   stripe_unit
   stripe_count
   object_size

Logical output:

::

   file_offset
   length
   payload

Stripe math
~~~~~~~~~~~

Given the source inode layout:

::

   stripes_per_object = object_size / stripe_unit
   objsetno           = object_number / stripe_count
   stripepos          = object_number % stripe_count
   objsetpos          = object_offset / stripe_unit
   blockoff           = object_offset % stripe_unit
   stripeno           = objsetno * stripes_per_object + objsetpos
   blockno            = stripeno * stripe_count + stripepos
   file_offset        = blockno * stripe_unit + blockoff

If a write crosses a stripe boundary, the gateway splits it into multiple
logical file extents and emits multiple ordered WRITE records for the same
source event.

Data derivation drawing
~~~~~~~~~~~~~~~~~~~~~~~

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
~~~~~~~~~~~~~~~~~~~~~~~~~~

The POC uses strict capture. Events are emitted to the replay stream only
when the source operation is known to have succeeded.

* Successful MDS mutation reply -> emit metadata operation.
* Failed MDS mutation reply -> drop event.
* Successful OSD write reply -> emit data operation.
* Failed OSD write reply -> drop event.

This is what prevents the replica from containing writes or metadata that
the source rejected.

Ordering domains
~~~~~~~~~~~~~~~~

The gateway preserves order in the domains where order affects the final
filesystem state:

* per-file data and size operations;
* same-inode metadata operations;
* namespace operations involving the same path or parent directory;
* rename/link/unlink operations that change the inode-to-path table.

Rename against in-flight writes is handled by the order of table updates,
not by coupling at replay time: a rename emits its path-table update in
``wire_seq`` order, and any write event resolves the inode's current path
at the moment it is emitted. Because the source committed the rename
before any subsequent write, the gateway sees the rename first on the
wire, updates the table first, and the later write resolves to the new
path. No cross-stream coupling between rename and writes is required.

The exact concurrency strategy is outside the POC. The required property is
simple: the emitted replay stream must be ordered so that the remote backend
applies dependent operations in source-equivalent order.

Size-affecting operations
~~~~~~~~~~~~~~~~~~~~~~~~~

Truncate/write ordering needs special handling. Source CephFS uses internal
truncate sequencing and MDS-to-OSD trim/truncate behavior that is not
visible as ordinary path operations to the remote backend. The gateway
therefore serializes size-affecting operations per source-local inode.

Size-affecting operations include:

* ``OSD_OP_WRITE``;
* ``OSD_OP_WRITEFULL``;
* ``OSD_OP_TRUNCATE`` (client-issued, distinct from MDS-issued
  ``CEPH_OSD_OP_TRIMTRUNC`` which travels MDS-to-OSD on the cluster
  network and is not visible to the proxy);
* ``OSD_OP_ZERO`` (punch hole);
* MDS ``setattr`` with ``size`` set (truncate).

Each entry is tracked as:

::

   Observed   -> parsed and ordered, waiting for source reply
   Confirmed  -> source accepted it, eligible for replay
   Dropped    -> source rejected it, skip it

Per-inode queue:

::

   queue[source_inode] = [event1, event2, event3, ...]

   replay only from the head:

       wait until head is Confirmed or Dropped
       if Dropped: remove it
       if Confirmed: emit its path+op record, then remove it

The head-of-queue wait is essential. A later-confirmed write must not jump
ahead of an earlier unconfirmed truncate for the same file.

Stamp-and-gate drawing
~~~~~~~~~~~~~~~~~~~~~~

::

   MDS setattr(size) ----+
                         |
   OSD write ------------+--> single logical ordering point
                         |         |
   OSD zero -------------+         v
                            +-------------+
                            | wire_seq    |
                            +------+------+ 
                                   |
                                   v
                            +-------------+
                            | queue[I]    |
                            |  10 Obs     |
                            |  11 Conf    |
                            |  12 Drop    |
                            +------+------+ 
                                   |
                                   v
                            emit ordered path+op records

Bootstrap
---------

Bootstrap is the phase that brings the pre-existing source filesystem
into the replicated state. In any deployment the source filesystem
exists before replication starts, and bootstrap is the only mechanism
that closes the gap between "filesystem exists" and "replica converges
from live capture alone." Bootstrap correctness sits on the same level
as live-capture correctness.

Bootstrap correctness claim
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The claim that makes bootstrap tractable: once capture is running, every
new mutation is observable. The bootstrap mechanism only needs to bring
the *pre-existing* state across; the live stream takes care of everything
from that point onward. So the bootstrap problem reduces to: copy a
consistent snapshot of an existing subtree to the replica, seed the
gateway's translation tables for the copied files, and hand off to live
replay without missing any mutation in the seam.

Every file and directory is in one of three replication states:

* **Replicated** — present on the replica and consistent with the source.
  Live capture mutates it normally. Steady state.
* **Priming** — being copied to the replica. Live capture against this
  entry is held until copy completes.
* **Non-replicated** — pre-existing, not yet copied. Live capture for
  this entry is held until the entry transitions to Priming.

State transitions: ``Non-replicated -> Priming -> Replicated``. Live
mutations on Replicated entries flow through the normal capture path.

Strategy A: Quarantine-then-copy (primary mechanism)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The primary bootstrap strategy uses an MDS-level quarantine on the
subtree being copied. Quarantine blocks all client mutations within the
subtree — namespace operations (create, unlink, mkdir, rmdir, rename,
link, symlink), attribute changes (setattr, setxattr), and data writes.
Reads are permitted. No entry can be created, removed, renamed into or
out of, or written to in a quarantined subtree.

This quarantine guarantee is what makes copy-then-transition safe: no
mutations can occur during the copy, so the copied data is a consistent
point-in-time snapshot. There is no race between copy and live capture
for the quarantined subtree, because there is no live activity to race
against.

**MDS support required.** Quarantine in the form needed for bootstrap
does not exist in upstream CephFS today. It requires minor MDS changes:
a per-subvolume or per-subtree flag that, when set, causes the MDS to
reject (or block) all mutating client requests within the subtree until
the flag is cleared. Read-side caps are unaffected. The flag must be
durable across MDS restart for the duration of bootstrap.

This is a small, well-scoped MDS feature — far smaller than the rest of
the replication design. The POC can prototype it as a patch against the
MDS; production will need it upstream or carried as a vendor patch.

Procedure (Strategy A)
~~~~~~~~~~~~~~~~~~~~~~

#. Start capture. The proxy begins forwarding MDS and OSD packets to the
   gateway. All pre-existing entries are Non-replicated. All entries
   created after this point are tracked live and transition directly to
   Replicated.
#. Walk the source tree. Identify the next subtree to bootstrap. Subtree
   granularity is a tuning parameter, discussed below.
#. Quarantine the subtree via the MDS quarantine flag.
#. Copy the subtree contents to the replica. The copy reads from the
   source side and writes to the replica side directly — it does *not*
   go through the live capture/replay pipeline, because there is nothing
   to capture (quarantine blocks mutations) and live replay only handles
   deltas.
#. Seed the gateway's translation tables for the copied subtree:

   * ``inode_table`` entries (source-local inode, paths, type, nlink);
   * ``layout_table`` entries (read source layout via ``ceph_get_layout``
     on each file — this is how layout for pre-existing files enters the
     table at all, since they have no MDS create response to observe);
   * ``path_table`` and ``dentry_table`` entries for the copied paths.

#. Transition all entries in the subtree from Non-replicated to
   Replicated. Safe because quarantine guarantees no mutations occurred
   during the copy.
#. Release quarantine. Writes resume and are captured/replayed through
   the normal pipeline.
#. Repeat for the next subtree until the entire scope is bootstrapped.

Subtree granularity
~~~~~~~~~~~~~~~~~~~

Quarantine windows are client-visible write pauses on the quarantined
subtree. The duration is bounded by:

::

   quarantine_window ≈ subtree_size / min(source_read_bw,
                                          link_bw,
                                          replica_write_bw)

Smaller subtrees mean shorter individual pauses at the cost of more
quarantine cycles. The granularity choice is a deployment-time tuning
question, not a design-time one — but the POC must measure the
relationship and report it. A subtree-sizing policy that bounds maximum
pause duration (e.g., "no single quarantine window exceeds 30 seconds")
is more operationally useful than a fixed subtree size.

Boundary rules
~~~~~~~~~~~~~~

**Rename across quarantine boundary.** Quarantine blocks rename into or
out of the quarantined subtree, by definition. Renames between two
non-quarantined subtrees are captured live and replayed normally. No
entry can move into a subtree that is currently being copied, which is
what makes the copy snapshot consistent.

**Hardlinks across subtree boundaries.** A file with links in multiple
subtrees is copied during bootstrap of the first containing subtree.
When a later subtree is bootstrapped and the gateway encounters a link
to an already-replicated inode, the replay backend creates a hardlink on
the replica rather than re-copying the data. The gateway identifies
this case via ``inode_table[I].paths``, which tracks all paths per
source-local inode.

**Failures mid-copy.** If the copy fails partway through a subtree, the
quarantine is held, partial replica state is rolled back (or the
subtree on the replica is dropped and recopied), and the bootstrap step
retries. The subtree stays Non-replicated until the copy completes and
transitions succeed atomically. There is no partial-replicated state.

Strategy B: Buffer-and-replay (experimental)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Strategy A's quarantine windows are client-visible stalls. For workloads
that cannot tolerate any pause, an alternative is to allow mutations
during copy and reconcile afterward.

Outline:

#. Mark the subtree as Priming. Live capture for entries in this state
   is *buffered* rather than replayed — events are tagged with their
   ``wire_seq`` and held in a per-inode buffer.
#. Copy the subtree contents to the replica concurrently with live
   client activity on the source. The copy reads whatever the source
   shows at the moment each file is read.
#. After copy completes, drain the per-inode buffers in ``wire_seq``
   order against the copied state. Each buffered event is replayed
   against the replica.
#. Transition the subtree to Replicated once buffers are drained.

The complication is that the copy and the buffered mutations race
against each other. A mutation captured at ``wire_seq = K`` may have
been committed on the source *before* the copy read its file (so the
copy already includes it, and replaying the buffered event
double-applies), or *after* (so the copy missed it, and replaying is
necessary). Per-file timestamping of when the copy read each file, plus
careful idempotency in the replay (writes at a specific offset are
idempotent; create/unlink/rename are not), is needed to reconcile this.

For the POC, Strategy B is **experimental** and worth implementing only
after Strategy A is working. The interesting POC question for B is
whether the reconciliation logic can be made simple enough to trust in
production. If it can, Strategy B becomes the production default and A
becomes the fallback for workloads that don't justify B's complexity.

Strategy B is not free from the MDS-support discussion: it does not
require quarantine, but it does require capture to be live across the
entire to-be-bootstrapped subtree from the moment Priming begins, which
puts the same robustness demands on the capture path as steady-state
replication.

Seam between bootstrap and live capture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The handoff from bootstrap to live capture is where bugs hide. The
correctness requirement is: no mutation committed on the source between
"capture started" and "subtree is Replicated" is lost.

Strategy A satisfies this because quarantine excludes mutations during
the copy window; any mutation that happened before the quarantine is
either in the live stream (if captured) or in the copy (if it predated
capture); any mutation that happens after release is in the live stream
by construction.

Strategy B satisfies this only if the buffer captures every mutation
from the moment Priming begins, and the drain reconciles correctly
against the copy.

The POC must explicitly test this seam: start capture, make mutations,
bootstrap, make more mutations, verify the replica has every committed
mutation and none of the rejected ones.

Bootstrap drawing (Strategy A)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
                                                                 |
                                                                 v
                                                            verify

Potential Extensions
--------------------

The following are intentionally outside the core POC but are useful
follow-up topics once semantic feasibility is established.

Deployment variants, DNAT, and placement
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The logical capture point could be implemented in several ways:

* external transparent proxy;
* DNAT/nftables redirection;
* tunnel-based redirection;
* client-local proxy or host agent;
* source-side sidecar;
* colocated proxy and gateway;
* remote gateway with source-side capture;
* colocated gateway and remote libceph client backend;
* fully separated proxy, gateway, transport, and remote backend.

Example DNAT-style deployment:

::

   clients
      |
      | redirected by DNAT/routing/tunnel
      v
   +----------+         forwarded Ceph traffic        +---------------+
   | proxy    |-------------------------------------->| source Ceph   |
   +----+-----+                                       +---------------+
        |
        | copied MDS/OSD event stream
        v
   +----------+       ordered path+op stream          +---------------+
   | gateway  |-------------------------------------->| remote libceph|
   +----------+                                       | backend       |
                                                      +-------+-------+
                                                              |
                                                              v
                                                        replica CephFS

These are topology choices. They don't affect whether the semantic
translation works.

WANOPT: compression and TRE
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A production design can optimize traffic between capture/gateway and the
remote replay side using WANOPT techniques such as:

* compression of metadata events and write payloads;
* batching of small operations;
* write coalescing;
* traffic redundancy elimination (TRE);
* content-defined chunking;
* remote cache dictionaries for repeated byte ranges;
* backpressure based on remote lag and link utilization.

WANOPT must preserve the derived operation stream. It may transform the
transport representation, but the remote side must recover the same
ordered ``{seq, op, path, args}`` records.

Sharding and load balancing
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A production design can scale by using multiple proxies or gateway shards,
with each shard handling a subset of files.

Possible strategies:

* subtree-based sharding, where proxy/gateway shard A handles ``/a``, shard
  B handles ``/b``, and so on;
* source-local-inode hash sharding for data-heavy workloads;
* client-session sharding at the proxy, followed by gateway-side routing by
  source-local inode;
* hybrid sharding, where namespace events are routed by subtree and OSD
  writes are routed by inode.

The ordering rule is non-negotiable: all events that can affect the same
file must reach the same ordering domain, or a coordinator must preserve
that file's order.

Sharding drawing:

::

                         source clients
                              |
        +---------------------+---------------------+
        |                     |                     |
        v                     v                     v
   +---------+           +---------+           +---------+
   | proxy A |           | proxy B |           | proxy C |
   | files A |           | files B |           | files C |
   +----+----+           +----+----+           +----+----+
        |                     |                     |
        v                     v                     v
   +---------+           +---------+           +---------+
   | shard A |           | shard B |           | shard C |
   +----+----+           +----+----+           +----+----+
        |                     |                     |
        +---------------------+---------------------+
                              |
                              v
                    ordered streams to remote backend

Secure msgr2 mode
~~~~~~~~~~~~~~~~~

Encrypted Ceph traffic cannot be passively parsed. Future designs could
use a trusted terminating proxy, an integration point where plaintext is
available, or a source-side event export path.

High availability and recovery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Production recovery needs explicit design for proxy failure, gateway
failure, replay-log overflow, and missed traffic. Possible mechanisms
include fail-closed capture, durable proxy-side logs, bounded dirty sets,
selective recopy, and verifier-driven drift detection.

Fd cache and unlink-while-open
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A future fd cache could preserve transient writes to files that have been
unlinked but remain open on the source. The POC does not require this
because final visible state convergence for deleted files does not depend
on replaying transient pathless writes.

Snapshots, quotas, and failover
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Snapshots, quota enforcement, replica promotion, client remount behavior,
fencing, and post-failover resync are production DR topics outside the POC.

Implementation framework and language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The packet-processing framework, language choice, and runtime are
engineering decisions independent of the semantic claim this document
makes. Plausible options range from kernel-bypass packet frameworks
(DPDK, AF_XDP, eBPF/XDP) for high-throughput captures, to userspace
packet capture (libpcap, netfilter queues) for simpler deployments, to
fully managed implementations in Rust, Go, C++, or extensions of the
existing Ceph codebase. The choice affects performance and operational
characteristics but not whether the translation works.

Remote backend internals
~~~~~~~~~~~~~~~~~~~~~~~~

The POC treats the remote libceph client backend as a replay sink that
consumes the ordered ``{seq, op, path, args}`` stream. Production
implementations have meaningful internal design space:

* parallelism strategy (per-inode workers, op-class lanes, pipelined
  batches);
* fd caching for hot files;
* write coalescing and small-op batching at the replay boundary;
* error handling and per-op retry policy against the replica MDS/OSDs;
* lag-driven backpressure to the gateway;
* integration with the replica cluster's own admin and monitoring
  surfaces.

These are real engineering decisions but they sit downstream of the
feasibility claim. Once the gateway emits a correct ordered stream, the
backend's job is well-defined.

Active-active replication
~~~~~~~~~~~~~~~~~~~~~~~~~

The POC assumes a single source and a passive replica. Active-active
replication — where both clusters accept client writes and changes
propagate in both directions — requires conflict detection, conflict
resolution policy, and a stronger consistency model than eventual
convergence. It is a substantially different design and is not a
straightforward extension of this one.
