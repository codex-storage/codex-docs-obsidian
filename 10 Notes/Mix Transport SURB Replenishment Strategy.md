---
related:
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
  - "[[Mix Transport Implementation Walk Through - SURB Replenishment]]"
  - "[[Mix Transport Implementation Walk Through - Connect Handshake]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
  - "[[Mix Transport Implementation Walk Through - Session Registry]]"
  - "[[Mix Transport Implementation Walk Through - Stream Establishment Round Trip]]"
---

This note defines how MixTransport supplies the session recipient with return paths to the anonymous session initiator. SURBs are created, transmitted, acknowledged and stored individually. Several SURBs are grouped only while the recipient sends redundant copies of one reverse transport frame. A supplied SURB does not belong to a persistent wire-level group.

The implemented policy combines proactive supply from the session initiator with refill requests from the session recipient. Proactive supply is enabled by default. Disabling proactive supply leaves the request-driven path active, which is useful for comparison and controlled deployments but cannot recover after the recipient has exhausted every return path.

## Communication Roles

The session initiator knows the recipient's real Mix destination and can submit forward Mix packets at any time. The session recipient knows only the session pseudonym and cannot address the initiator through the forward path. Every frame sent from the recipient to the initiator must therefore use a SURB created by the initiator.

The initiator sends individual public SURBs to the recipient. The recipient stores those SURBs in one bounded queue owned by `TransportSession`. When the recipient needs to send reverse Data, an ACK or a control frame, the recipient removes `DefaultReplySurbRedundancy` SURBs from the queue and sends the same encoded frame through each selected SURB. The current redundancy value is two.

The initiator retains one private `ReplyCredential` for each public SURB. The recipient decides later which SURBs will carry the same logical frame, so the initiator cannot associate credentials with a persistent redundancy group. Each returned packet is recovered with its own credential. The transport frame's idempotent state transition or logical sequence identifier prevents redundant copies from applying the same effect twice.

## Session-Level Ownership

SURBs belong to a session rather than to one stream. A return packet contains a `streamId` when it addresses a stream, so any SURB in the session queue can carry traffic for any stream in that session. The same queue also supplies session-level control traffic.

Per-stream queues would leave valid SURBs unavailable when one stream is idle and another stream needs return capacity. Closing a stream would also discard return paths that remain valid for the session. Session ownership gives every reverse operation access to the same bounded supply and gives session teardown one precise cleanup boundary.

The queue is protected by a session-level reply-send lock. The lock serializes the sequence of checking available capacity, removing the SURBs for one temporary redundancy batch, encoding the latest supply snapshot and submitting the reverse frame. Incoming supply does not acquire this lock because the current sender may be waiting for those SURBs.

## Bootstrap Supply

`Connect` carries four unnumbered bootstrap SURBs. The recipient validates and stores them, removes two for the redundant `ConnectAck`, and retains two for later control traffic. The recipient then initializes numbered supply accounting and includes the initial supply snapshot in `ConnectAck`.

`OpenStream` carries two additional unnumbered SURBs. These SURBs are dedicated to the corresponding `StreamAck` or `StreamReject`; the recipient uses them immediately and does not add them to the session queue. This separation prevents stream-opening response paths from bypassing the numbered session-supply capacity.

After the session has been established, ordinary replenishment uses only numbered `SurbSupply` frames.

## Bounded Supply Credit

The recipient queue has a hard capacity named `recipientSurbCapacity`. The default is sixteen SURBs, and the minimum is the four-SURB control reserve plus one two-SURB ordinary reply batch.

The recipient advertises an absolute, monotonically increasing `surbSupplyLimit`. The limit is the exclusive upper bound on supply sequence numbers that the initiator may introduce. If the limit is fourteen, the initiator may create sequences zero through thirteen. Whenever the recipient removes a stored SURB for a reverse transmission, the recipient increments the limit by one and thereby authorizes one replacement.

The initial limit accounts for bootstrap SURBs that remain in the queue. With the default capacity, `Connect` leaves two SURBs after forming the `ConnectAck` batch, so `ConnectAck` advertises an initial limit of fourteen. Supplying sequences zero through thirteen fills the queue to sixteen without exceeding its configured capacity.

The limit is absolute rather than an instruction to add a quantity. Repeating the same limit grants no additional capacity, and an older snapshot cannot retract capacity already observed by the initiator.

## Numbered Supply

Every public SURB supplied after establishment receives a monotonically increasing `SurbSupplySequence`. One `SurbSupply` frame carries `firstSurbSequence` followed by as many as four serialized SURBs. Sequence numbers for later values are consecutive.

The recipient tracks accepted supply sequences with `surbSupplyReceiveBase` and a fixed 256-bit acknowledgement bitmap. A bitmap bit at offset `i` records receipt of sequence `surbSupplyReceiveBase + i`. The recipient accepts valid supply out of order, stores each SURB once and advances the receive base across a contiguous prefix of set bits.

Receipt and consumption are different events. A bitmap bit remains recorded after the corresponding SURB has been consumed from the queue. This receipt history is necessary because retransmission of a consumed one-use SURB must still be recognized as a duplicate rather than inserted into the queue again.

The initiator may introduce a sequence only when the sequence is below `surbSupplyLimit` and inside the 256-position receive window beginning at the reported receive base. These two checks bound the recipient queue and the initiator's retained retransmission state.

Each serialized SURB is decoded independently. If one item in a supply frame is malformed, the recipient retains the other valid items and leaves a sequence gap for the malformed item. Retransmission of the original valid serialization can later fill that gap.

## Absolute Supply Snapshots

A supply snapshot contains three fields:

```text
surbSupplyReceiveBase
surbSupplyAcknowledgementBitmap
surbSupplyLimit
```

The receive base and bitmap tell the initiator which public SURB serializations the recipient has accepted. The initiator removes acknowledged serializations from its retransmission table. The supply limit tells the initiator which new sequences the recipient has authorized.

`ConnectAck` carries the initial snapshot. Later recipient-originated Data, ACK and stream-response frames carry the latest snapshot. `RefillRequest` and `SurbStatus` also carry a complete snapshot. The three fields are present together or absent together, so the initiator never applies a partial supply update.

Snapshots are absolute and idempotent. Duplicate or reordered snapshots can acknowledge additional known sequences, but they cannot allocate the same sequence twice or decrease a limit already observed by the initiator.

## Proactive Supply

After `ConnectAck` establishes an initiator-side session, MixTransport starts one SURB supplier task owned by that `TransportSession`. With `enableProactiveSurbReplenishment = true`, the supplier creates numbered SURBs until it has used the credit advertised by the recipient.

The supplier creates no more than four new SURBs per `SurbSupply` frame. For every new SURB, the initiator registers the private credential once and retains the public serialization under its supply sequence. The forward frame is then submitted to the recipient through the ordinary Mix path.

When a reverse snapshot increases `surbSupplyLimit`, the session signals the supplier task. The supplier uses the newly authorized sequence positions to replace SURBs that the recipient consumed. When a snapshot acknowledges supplied sequences, the supplier discards only the retained public serializations; their private reply credentials remain registered until those SURBs carry replies, expire or the session closes.

The constructor option controls proactive emission only. Both endpoints continue to understand numbered supply and refill requests regardless of the option. With proactive emission disabled, an initiator waits until a `RefillRequest` marks the session as urgently needing supply.

## Recipient-Initiated Refill

The recipient protects four stored SURBs for control traffic. An ordinary reverse Data or ACK transmission is permitted only when removing its two-SURB redundancy batch leaves that reserve intact.

When the recipient has fewer than six SURBs, `ensureUnreservedSurbs` calls `requestRefill`. A due refill request consumes two SURBs, attaches the recipient's current absolute supply snapshot and travels to the initiator through both return paths. The request does not contain a requested quantity or a request identifier.

The initiator applies the snapshot and marks the session's supplier as urgent. The same supplier task used for proactive replenishment then creates as many new numbered SURBs as the current credit permits. Repeating a refill request does not create a second logical operation: the repeated absolute snapshot is safe to apply, and the urgency flag remains a boolean condition.

`nextRefillRequestAt` limits request frequency. If useful supply has not arrived after thirty seconds, a blocked reverse sender can submit another refill request while it still has a redundancy batch available. Receiving enough SURBs clears the pending retry deadline.

With proactive replenishment enabled, exhausting the queue does not immediately fail the waiting reverse operation because the status-probe path can restore synchronization. In pull-only mode, the operation returns an error when the recipient no longer has two SURBs with which to send another request.

## Retransmitting Supply

New supply remains in the initiator's `pendingSurbSupply` table until a recipient snapshot acknowledges it. Each entry retains the supplied SURB's public serialization and the identifier of its private reply credential. After a supply submission completes, the initiator assigns each still-pending entry a retransmission deadline. The default delay is thirty seconds.

When the earliest deadline expires, the supplier first purges expired credentials and expired retired-identifier records from the reply credential store. The supplier then verifies that the private credential identified by the pending entry remains active. If the credential is absent, the supplier removes the pending public serialization instead of retransmitting a return path that the initiator can no longer recover. Otherwise, the supplier retransmits the same serialized public SURB with the same sequence number. The retransmission does not create a new SURB, register another private credential or extend the existing credential's TTL. If the original and retransmitted packets both arrive, the recipient's receive base and bitmap cause one copy to be stored and the other to be ignored.

Taking an entry for retransmission temporarily clears its deadline while the send is in progress. After the attempt completes, the sender schedules another deadline only if the entry remains pending. A snapshot can acknowledge and remove the entry while the asynchronous send is running; the completion path never recreates an acknowledged entry.

## Recovering from Complete Recipient Starvation

Supply retransmission alone cannot repair every feedback loss. The recipient can consume its last stored SURBs while all reverse frames carrying the increased limit are lost. The initiator then sees no new credit, while the recipient has no return path through which to report its actual state.

With proactive replenishment enabled, the initiator sends a `SurbStatusProbe` after the session has produced no reverse supply snapshot for the configured interval. The default interval is two minutes. A probe carries two freshly created SURBs dedicated to one redundant `SurbStatus` response.

The recipient uses the probe SURBs immediately and does not insert them into the session queue. The returned `SurbStatus` contains the recipient's current receive base, acknowledgement bitmap and supply limit. Applying that snapshot lets the initiator resume ordinary numbered supply even when the recipient queue was empty.

Every probe uses fresh SURBs because the initiator cannot know whether a previous probe reached the recipient and its response was lost. Probe credentials remain governed by the reply credential store's capacity and expiry policy.

## Interaction with Application Traffic

SURB replenishment is driven by transport-level consumption rather than by calls to `readLp` or `readExactly`. The recipient increments supply credit when MixTransport removes SURBs for a reverse frame. This event occurs before the initiator can recover the reply and before an application can read its payload.

MixTransport observes application writes through `TransportStream.write`, but the current supplier does not derive a speculative SURB target from the write length. The recipient's hard queue capacity and absolute credit remain the authoritative limits. A future scheduler can use known reverse demand to prioritize replenishment without changing the wire protocol.

Application read observation is a separate flow-control concern. The current receive window limits chunks accepted by the transport, while libp2p's `BufferStream` stores bytes that have been delivered in order but not yet consumed by the application. Overriding the read path could provide byte-level consumption feedback later, but that feedback is not required to account for SURB consumption.

## Session Shutdown

The supplier task is owned by `TransportSession`. Session shutdown signals the supplier's state event, cancels the task and waits for its completion before shutting down the session's streams. Reply credential cleanup then removes credentials associated with the session, including unused supply and status-probe credentials.

The recipient's queue, supply bitmap and credit counters share the session lifetime. A new anonymous session starts a new supply sequence space and cannot reuse acknowledgements or credit from an earlier session.

## Implemented Safety Properties

- The recipient queue cannot exceed `recipientSurbCapacity`.
- The initiator cannot create new numbered supply beyond the recipient's absolute credit or the fixed receive window.
- A retransmitted SURB is stored at most once, including after the first copy has already been consumed.
- Every valid SURB in a partly malformed supply frame is accepted independently.
- Public SURB retransmission does not duplicate private credentials.
- Refill requests are idempotent urgency signals rather than correlated refill transactions.
- Proactive status probes can restore the recipient's ability to report credit after its ordinary SURB queue reaches zero.
- Session shutdown terminates the supplier and bounds the lifetime of session-owned supply state.
