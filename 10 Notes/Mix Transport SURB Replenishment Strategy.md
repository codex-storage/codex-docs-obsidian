---
related:
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
  - "[[Mix Transport Implementation Walk Through - Session Registry]]"
---
This note defines how MixTransport supplies the session recipient with return paths to the anonymous session initiator. The current implementation uses recipient-initiated refill requests. The intended design adds proactive initiator supply while preserving refill requests as an urgent fallback. The resulting hybrid model keeps the recipient's memory bounded and can recover when the recipient no longer has a normal session SURB group with which to request more capacity.

## Communication Roles and Terminology

The session initiator knows the real Mix destination and can send forward packets to the session recipient at any time. The session recipient knows only the session pseudonym and cannot address the initiator through the forward Mix path. Every transport frame sent by the recipient therefore consumes a SURB redundancy group previously created by the initiator.

A SURB redundancy group contains several public SURBs that all carry the same logical reply. Sending a reverse frame through a group submits one copy through every SURB in that group; the first successfully recovered copy delivers the frame, and later copies are suppressed by the initiator's reply credential store. This note uses *group* as the unit of transport capacity because one reverse Data, ACK, or control frame consumes one complete redundancy group.

The initiator keeps the private reply credentials corresponding to every public SURB supplied to the recipient. The recipient stores only the public groups. Successful recovery consumes the complete credential group at the initiator, while session teardown retires all remaining credentials associated with that session.

## Why the Supply Belongs to the Session

SURB groups remain owned by `TransportSession`, not by individual `TransportStream` objects. A SURB identifies a return path to the session initiator; the `MixTransportFrame` sent through that path contains the `streamId` that selects a particular stream. The same group can therefore carry Data or an ACK for any stream in the session, as well as session-level control traffic.

Per-stream ownership would strand groups on idle streams while another stream was blocked, and closing a stream would discard return capacity that remained valid for the session. Session-level control frames would still require a separate pool, and a transport-wide memory bound would have to combine independently replenished stream pools. Keeping one session pool makes all stored groups fungible and gives session teardown one precise lifetime boundary.

A busy stream can still consume capacity needed by another stream. That is a scheduling concern rather than a reason to partition ownership. If contention requires stronger fairness, `TransportSession` can schedule pending reverse frames across streams or enforce per-stream demand quotas while continuing to draw every send from the common pool.

## Current Pull-Based Replenishment

The current implementation stores received groups in `TransportSession.receivedSurbGroups`. Recipient-side Data and ACK paths converge on the recipient branch of `sendStreamFrame`. A session-level `replySendLock` serializes the complete operation of checking supply, requesting a refill when necessary, removing one group, and submitting the reverse frame. Incoming refill processing does not acquire this lock because a task holding the lock may be waiting for the arriving groups.

Data and ACK traffic preserve `ReplyControlReserveGroups`, currently two groups. A group is not intrinsically marked as reserved; `takeUnreservedSurbGroup` permits removal only when the remaining queue will still contain the configured reserve:

```nim
proc takeUnreservedSurbGroup*(session: TransportSession): Result[seq[SURB], string] =
  if session.receivedSurbGroups.len <= ReplyControlReserveGroups:
    return err("session reply capacity is reserved for control traffic")
  ok(session.receivedSurbGroups.popFirst())
```

When only the reserve remains, `ensureUnreservedSurbGroup` calls `requestRefill`. Each request consumes one group from the reserve, asks for at most the number of groups that fit in one Sphinx packet, and receives one forward `Refill` frame from the initiator. A request and its response are correlated by `refillRequestId`. Several identifiers can remain outstanding so a valid response from an earlier attempt can still contribute its groups after a later retry was sent.

The recipient decodes every group in a refill independently and retains every valid group. A malformed group does not cause valid groups in the same response to be discarded. Accepting the response consumes its request identifier before adding groups, preventing a duplicate response from storing the same public SURBs twice.

`nextRefillRequestAt` limits request frequency. If supply remains low and no useful response arrives before the deadline, the next blocked reverse operation sends another request with a fresh identifier and a fresh SURB group. After any ordinary reverse send, `sendStreamFrame` also calls `requestRefill` so consumption that reaches the low-water mark can start replenishment before the next application operation blocks.

## The Pull-Only Failure

Every pull attempt spends part of the capacity it is trying to restore. If a request or its response is lost, the recipient must spend another group on the next attempt. Repeated loss can consume the complete control reserve. At that point the recipient cannot send another request, Data frame, ACK, or status message. The initiator still has an unrestricted forward path, but pull-only replenishment gives the initiator no reason to use that path. The reverse side of an otherwise live session is therefore permanently stranded and the blocked operation must fail.

Increasing the reserve only makes this failure less likely; no finite reserve removes it. A liveness mechanism must allow the initiator to act without first receiving a reverse request.

## Intended Hybrid Policy

Proactive initiator supply will become the normal mechanism, while the existing recipient request remains an urgent signal. The constructor should express this policy with:

```nim
enableProactiveSurbReplenishment = true
```

When the option is true, the initiator runs the proactive supply mechanism and the recipient may still send refill requests when it observes immediate pressure. When the option is false, the transport retains the current pull-only behavior. A separate push-only mode is not initially useful: the inexpensive request path gives the recipient a direct way to report urgency, and retaining it avoids an unnecessary compatibility matrix.

All endpoints should continue to understand both wire mechanisms regardless of the local emission policy. The option controls whether an initiator proactively sends supply; it should not make the recipient reject an unsolicited supply frame or make the initiator reject a valid refill request.

## Bounded Supply Credit

The initiator must not create and push new groups without limit. The recipient has a bounded group queue, while the initiator must retain reply credentials for every group that might be used. Unrestricted pushing would either overflow the recipient or fill the initiator's credential store with paths the recipient discarded.

The recipient therefore advertises an absolute, monotonically increasing `surbSupplyLimit`. The limit is the number of uniquely numbered groups that the initiator has permission to supply during the session. If the initial limit is 16, the initiator may send group sequences 0 through 15. Whenever the recipient consumes one stored group, the recipient increments the limit by one and thereby authorizes one replacement group.

This accounting maintains the central bound:

```text
authorized groups not yet consumed <= recipient SURB group capacity
```

The limit is absolute rather than a request for an additional quantity. Receiving the same limit twice has no cumulative effect, and receiving an older value cannot retract credit that the initiator has already used. The session pseudonym scopes the sequence space, so creating a new session resets the supply counters without ambiguity.

The recipient should advertise its capacity and initial supply limit in `ConnectAck`. A fixed protocol default may be sufficient for the first implementation, but carrying the accepted capacity in the handshake makes the bound explicit and permits endpoints configured with different limits to interoperate.

The groups attached to `Connect` provide the bootstrap path for `ConnectAck` and must count against the same recipient capacity. The initial supply limit must therefore authorize only the slots left after accounting for bootstrap groups that remain in the recipient queue. Once proactive session supply is operational, `OpenStream` should not routinely add another unnumbered set of groups. During an incremental transition, any groups still attached to `OpenStream` must pass through the same capacity check and reduce the available supply credit; otherwise opening streams could bypass the session bound.

## Numbered Supply and Safe Retransmission

Each public group supplied after session establishment receives a monotonically increasing `surbGroupSequence`. A `SurbSupply` frame can carry several consecutively numbered groups up to the Sphinx payload limit:

```text
SurbSupply
  firstGroupSequence
  groups[]
```

The recipient tracks accepted supply sequences using a receive base and a fixed bitmap. The structure serves the same purposes as the Data receive window: it suppresses duplicate groups, permits bounded out-of-order delivery, and lets the recipient report exactly which supplied groups arrived. Supply receipt and group consumption are separate state changes. A bitmap bit remains set after the corresponding group has been consumed, and the receive base advances when all preceding supply sequences have been received. The retained receipt state prevents retransmission from adding a consumed one-use group again.

The initiator must satisfy two limits before introducing a new supply sequence: the sequence must be authorized by `surbSupplyLimit`, and it must fit inside the supply receive window advertised by the recipient's receive base. A missing earlier supply can temporarily prevent the window from advancing even when later groups arrived and were consumed. The initiator continues retransmitting the missing retained supply; once that gap arrives, the recipient advances across the contiguous receipt bits and opens the window again. This behavior bounds duplicate-tracking state without treating a later group as a replacement for a missing sequence.

The initiator retains the encoded public groups and their supply sequences until the recipient acknowledges them. If a `SurbSupply` frame is lost, the initiator retransmits the same numbered groups rather than creating new ones. If the original and retransmitted frames both arrive, the recipient stores each group once. Retaining the serialized public representation is sufficient for retransmission; the initiator does not create or register another set of reply credentials.

Acknowledging receipt of a public group does not consume its private credential. The acknowledgement only allows the initiator to discard the retained `SurbSupply` payload. The private credential remains registered until the recipient sends through that group, the credential expires, or the session closes.

A supply frame can contain several groups for efficiency, but acknowledgement state is expressed per group rather than per frame. This distinction allows the recipient to retain and acknowledge valid groups independently if another encoded group in the same frame is malformed.

## Reporting Supply State

Reverse Data, ACK, and control frames should carry the recipient's latest supply state when wire space permits:

```text
surbSupplyReceiveBase
surbSupplyAcknowledgementBitmap
surbSupplyLimit
```

The receive base and bitmap tell the initiator which retained public groups no longer need retransmission. The supply limit tells the initiator which new group sequences it may send. These fields are absolute snapshots, so loss, duplication, and reordering do not require paired request/response state.

Piggybacking the snapshot on an existing reverse frame avoids consuming another group solely to acknowledge replenishment. The transport must nevertheless retain a standalone supply-status control frame because the recipient can need to report urgent capacity without having application Data or a Data ACK ready.

The current `RefillRequest` can evolve into that urgent notification. Under the credit model, the request does not need to demand an independently correlated batch. It tells the initiator to process the attached supply snapshot immediately. Existing `refillRequestId` correlation can remain during migration and be removed only after numbered supply acknowledgements completely replace its role.

## Normal Proactive Supply Flow

An initiator-side task belonging to `TransportSession` maintains the proactive supply. The task tracks the latest advertised limit, the next group sequence that has never been sent, retained unacknowledged supply entries, and their retransmission deadlines.

After `ConnectAck` establishes the session and advertises the initial limit, the task creates groups up to a configured target without exceeding the recipient's credit. It batches as many consecutive groups as fit in one `SurbSupply` frame, registers their reply credentials once, retains the encoded supply entry, and submits the frame through the forward Mix path.

The configured target can be lower than the recipient's hard capacity. The hard capacity defines the safety bound; the target controls how much return capacity is normally kept ready. This distinction avoids filling a large permitted queue when a session is mostly idle while still allowing temporary demand to raise the desired inventory.

When a reverse frame reports newly accepted supply sequences, the initiator removes the corresponding retained public encodings. When the same frame advances `surbSupplyLimit`, the task may create replacement groups. If no state changes before the earliest retransmission deadline, the task resubmits the retained numbered supply.

The initiator can observe successful reverse delivery before the application reads the recovered bytes. Successfully recovering a reverse frame proves that the recipient consumed a redundancy group. This transport-level observation is earlier and more directly related to SURB consumption than an application read.

## Complete Starvation Recovery

Supply credit and retransmission recover a lost initial `SurbSupply`, but a final feedback failure remains possible. The recipient can consume its remaining groups while every reverse frame carrying the increased limit is lost. The initiator then believes that all previously granted credit has been used to supply groups and has no permission to create another group. The recipient has no group with which to correct that belief.

The initiator resolves this state with a periodic forward `SurbStatusProbe` after the session has produced no reverse activity for the configured interval. The probe carries a small reply group dedicated to the status response. The recipient uses that attached group immediately rather than adding it to `receivedSurbGroups`, the session's general pool for Data, ACK, and control frames. The recipient can therefore return its current supply receive base, acknowledgement bitmap, and supply limit even when `receivedSurbGroups` is empty.

Each probe attempt must carry a fresh reply group because the initiator cannot know whether a preceding probe reached the recipient and consumed its group while the response was lost. Probe credentials remain subject to the normal credential capacity and expiry rules. The probe interval must be long enough to avoid accumulating excessive credentials during an extended partition.

The probe is a recovery path, not routine replenishment. Normal traffic communicates supply state through piggybacked snapshots, ordinary supply entries are retransmitted while unacknowledged, and a recipient refill request reports urgent pressure. The probe becomes active only when those paths have produced no usable reverse feedback.

## How Application Reads Relate to Supply

SURB provisioning should not be driven by `readLp` or `readExactly` calls made by the session initiator. `readLp` learns the message length only after the recipient has already spent return capacity to send the prefix. `readExactly(n)` describes the application's requested byte count, not the remote protocol's write schedule. Retransmitted Data and session control frames also consume groups without corresponding application reads.

MixTransport could override `TransportStream.readOnce` and observe the number of bytes actually removed from `BufferStream`. That information could support stricter byte-level Data backpressure by delaying receive-window advancement until the application consumes complete chunks. It would arrive too late to provision the SURBs that carried those chunks, however, and a slow application reader must not prevent the recipient from obtaining groups needed for ACK or control traffic. Application-read observation is therefore separate from SURB replenishment and should be added only if the current bounded `BufferStream` staging proves too permissive.

## How Recipient Writes Relate to Supply

MixTransport already observes every application write because `TransportStream.write` delegates to `MixTransport.writeStream`. On the recipient side, the transport can calculate the exact number of initial Data chunks produced from that byte sequence:

```text
initial Data groups = ceil(write length / MaxDataPayloadBytes)
```

For `writeLp`, libp2p constructs one length-prefixed byte sequence before calling the virtual `write` method, so MixTransport observes the complete encoded message. This calculation is still a lower bound on eventual consumption because Data retransmissions require fresh groups and other streams may concurrently generate Data, ACK, or control traffic.

Recipient write observation is useful as a demand hint. Beginning a large reverse write can mark replenishment as urgent and temporarily raise the desired proactive-supply target. The write should not wait for enough groups to cover its complete predicted size: one application write can require more groups than the recipient's bounded store. `writeStream` should continue chunk by chunk, consuming available unreserved groups, waiting when capacity is exhausted, and resuming as supply arrives.

The first proactive implementation does not need exact per-write demand accounting. Maintaining the ordinary session target and using write observation to trigger urgent replenishment is sufficient. If measurements later show contention between concurrent reverse writes, the session can maintain a bounded aggregate demand estimate and schedule frames fairly across streams.

## Task Ownership and Shutdown

The proactive supplier operates on session-wide credit, credentials, and public-group state, so its task belongs to `TransportSession`. A stream retransmission task may wait for a session group, but the stream does not own that group or the supplier that creates it. Closing one stream removes that stream's demand while preserving supply for the remaining streams.

Session shutdown cancels and waits for the supplier task before retiring the session's reply credentials and stored public groups. Transport shutdown continues to follow the existing hierarchy: `MixTransport` detaches sessions, each session detaches and shuts down its streams and session-level tasks, and credential storage is cleared only after those tasks have stopped.

## Implementation Sequence

1. Add a bounded recipient group capacity, supply sequence types, and the absolute `surbSupplyLimit` state.
2. Add `SurbSupply` encoding with multiple consecutively numbered groups and recipient duplicate suppression.
3. Add the initiator-side session supplier task, retained supply entries, and supply retransmission deadlines.
4. Carry absolute supply acknowledgement and credit snapshots in reverse frames and use them to retire retained public encodings and issue new groups.
5. Preserve `RefillRequest` as an urgent wake-up while adapting it to carry the same absolute supply state.
6. Add `enableProactiveSurbReplenishment = true`, with `false` preserving pull-only emission while both mechanisms remain accepted on the wire.
7. Use observed recipient writes as an urgency hint without coupling correctness to application write boundaries.
8. Add `SurbStatusProbe` to recover when the recipient's session pool is empty and the initiator's credit view is stale.
9. Exercise lost supply, duplicate supply, lost supply status, recipient exhaustion, concurrent streams, and session teardown under deterministic delivery control.

The incremental order keeps the current pull path operational while proactive supply is introduced. The completed hybrid design derives safety from bounded absolute credit and duplicate suppression, normal reliability from retained supply retransmission, and terminal starvation recovery from the initiator's forward status probe.
