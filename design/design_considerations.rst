Design Considerations for Reverse Ganesha
=========================================

Purpose
-------

This note preserves implementation considerations that appeared in
``live_ceph_async_replication_v2.rst`` but were intentionally compressed or
removed from ``live_ceph_async_replication_v3.rst``. The main proposal should
remain a feasibility argument. This document is for engineering details that
are useful after the feasibility claim is accepted.

The detailed proposal is ``../live_ceph_async_replication_v3.rst``. The older
``../live_ceph_async_replication_v2.rst`` remains the source for the notes
captured here.

Replay Record State
-------------------

The main replay record in ``v3`` carries the semantic fields needed by the
backend:

::

   op
   path
   old_path/new_path
   source_inode
   object
   user_version
   truncate_seq
   op_id
   args

``v2`` also carried an operational acknowledgement state:

::

   ack_state           # observed, confirmed, dropped

This state is not part of the logical replay stream. It is gateway-side
control state used to prevent speculative replay.

The states are:

* ``Observed`` — the request has been parsed and placed in the relevant order,
  but the durable source reply has not yet been matched.
* ``Confirmed`` — the source durably accepted the operation; the record is
  eligible for replay once all earlier dependent records are resolved.
* ``Dropped`` — the source rejected the operation; the gateway removes it from
  the ordering structure without replaying it.

The backend should not need ``ack_state`` as an API field. It receives only
confirmed logical records, plus source tokens for dedupe and idempotence.

Strict Confirmation
-------------------

Replay must be driven by durable source success, never by request observation
or an early acknowledgement.

Rules:

* MDS mutation: emit only after a durable success reply; drop on failure.
* OSD write: emit only after the commit/ondisk reply; drop on failure.
* Early ack is not sufficient, because replaying it could create replica state
  for a write the source later loses during OSD failover or peering.
* The OSD ``user_version`` used for ordering arrives with the commit reply, so
  confirmation and the data-path ordering token become available together.

Request/reply correlation is therefore a correctness boundary. A parsed request
without a matching durable reply is not replayable.

Capture Continuity and Fail-Closed Behavior
-------------------------------------------

``v3`` states the high-level assumption: capture is inline with the source TCP
stream, so a collector observes every byte its daemon observes.

The implementation consequence from ``v2`` is fail-closed checking:

* per-collector TCP stream contiguity should be checked;
* per-object ``user_version`` continuity should be checked where applicable;
* any discontinuity is an alarm, not a tolerated condition;
* the gateway should not guess through missing traffic.

Inline capture rules out ordinary packet loss as a semantic condition, but it
does not rule out parser bugs, collector faults, process crashes, or bad
request/reply correlation. Those failures must stop or isolate the affected
ordering domain. They must not silently produce replay records.

Size-Affecting Ordering Queue
-----------------------------

``v3`` keeps the core rule: size-affecting operations share a single per-inode
ordering point ordered by ``truncate_seq`` and then ``user_version``. ``v2``
also spelled out the queue discipline.

The size-affecting class is:

* ``OSD_OP_WRITE``;
* ``OSD_OP_WRITEFULL``;
* ``OSD_OP_TRUNCATE``;
* ``OSD_OP_ZERO``;
* MDS ``setattr`` with ``size`` set.

For each source inode:

::

   queue[source_inode] = [event1, event2, event3, ...]
       # ordered by truncate_seq, then user_version where applicable

   replay only from the head:
       wait until head is Confirmed or Dropped
       if Dropped:
           remove it
       if Confirmed:
           emit its path+op record
           remove it

The head-of-queue rule is essential. A later-confirmed write must not pass an
earlier unconfirmed truncate for the same file. The later write may already
have its OSD commit reply, but replaying it before resolving the earlier size
operation can produce a state the source never exposed.

Stalled Queue Heads
-------------------

A strict queue can stall if the head entry remains ``Observed`` forever. In
this design that is not normal backpressure; it indicates a parse,
correlation, or collector failure.

The gateway should treat a stalled head as a fault in that ordering domain:

* alarm with the source inode, token values, and unmatched request identity;
* stop replay for the affected inode or subtree;
* avoid letting later confirmed events bypass the stalled head;
* recover by repairing correlation state, replaying from durable logs, or
  recopying the affected region.

The important invariant is simpler than the recovery policy: an unresolved
earlier size-affecting operation blocks later dependent replay.

Non-Goals
---------

This document does not duplicate details already retained in ``v3``:

* inode/path/layout tables;
* per-operation metadata derivation;
* object-to-file stripe math;
* bootstrap strategy and subtree quarantine;
* future architecture extensions.

Those belong in the main proposal until they are moved to a broader design
appendix.
