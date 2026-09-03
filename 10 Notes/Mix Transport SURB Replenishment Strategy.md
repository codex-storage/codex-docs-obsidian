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
This note explains why MixTransport needs a SURB replenishment mechanism and how the push and pull parts of that mechanism work together. The note develops the communication model first and introduces the protocol mechanisms only after explaining the problems they solve. [[Mix Transport Implementation Walk Through - SURB Replenishment]] maps this design to concrete types and procedures.

The implemented strategy stores and supplies SURBs individually. Several SURBs are brought together only when the recipient sends redundant copies of one reverse frame. No persistent redundancy group is represented in the wire format.

## Communication Directions and Session Roles

A MixTransport session has an initiator and a recipient. The initiator creates the session by sending `Connect` to the recipient's real Mix destination. The recipient learns the session pseudonym carried by `Connect`, but the recipient does not learn an address through which it could send a normal Mix packet back to the anonymous initiator.

This document calls a packet from the initiator to the recipient a forward packet. The initiator can create a forward packet whenever it knows the recipient's Mix destination. A packet from the recipient back to the initiator is a reverse packet. The recipient can send a reverse packet only through a single-use reply block, or SURB, previously created by the initiator.

Creating a SURB produces two related values. The public SURB gives the recipient a one-use mechanism for sending one encrypted reply. The corresponding private reply credential remains with the initiator and is required to recover the payload when the returned packet arrives. The initiator must therefore create return paths before the recipient can send anything in the reverse direction.

## Storing Return Paths for a Session

The initiator sends public SURBs to the recipient. The recipient keeps available SURBs in one queue owned by the recipient-side transport session. This queue is called the shared session queue in the remainder of this note.

The queue belongs to the session rather than to an individual stream. A reverse stream frame identifies its stream, so any SURB in the shared queue can carry traffic for any stream in that session. The same queue can also carry session-level control frames.

One shared queue prevents SURBs from becoming stranded in an idle stream while another stream needs a return path. Closing one stream also does not discard SURBs that remain useful to other streams in the same session.

When the recipient sends one reverse frame, it removes several individual SURBs from the shared queue and sends the same encoded frame through each selected SURB. This temporary collection is a redundancy batch. The current policy selects two SURBs, giving the logical frame two independent opportunities to reach the initiator.

The redundancy batch exists only for that send operation. The recipient decides which SURBs to select and how many copies to send. Because persistent groups are not imposed on the queue or wire format, a later policy can choose a different redundancy count for a particular session, stream or frame kind.

The initiator retains the private credential for every supplied SURB independently. Each returned packet is recovered through the credential corresponding to the SURB that carried it. Transport operations that may arrive redundantly are idempotent or carry a logical sequence identifier, so recovering a later copy does not apply the same logical operation twice.

## Establishing the Initial Supply

The replenishment protocol cannot start until both endpoints have created the session. The initiator must nevertheless give the recipient enough return paths to acknowledge `Connect`. For that reason, `Connect` carries four bootstrap SURBs.

After accepting `Connect`, the recipient stores the four bootstrap SURBs in the shared session queue. The recipient removes two SURBs to send two copies of `ConnectAck`. The other two remain available for one subsequent reverse control operation.

The initiator also supplies dedicated response paths when opening a stream. `OpenStream` carries two SURBs that the recipient uses immediately for `StreamAck` or `StreamReject`. These two SURBs exist only for that stream-opening exchange and are not inserted into the shared session queue.

Once `ConnectAck` confirms that the session exists at both endpoints, the initiator can begin maintaining the shared session queue through the post-establishment replenishment mechanism described below.

## Why Replenishment Must Be Bounded

The initiator can create and send SURBs faster than the recipient can consume them. Allowing every arriving SURB to remain in memory would let the recipient's queue grow without limit. MixTransport therefore configures a maximum number of SURBs that the recipient may retain in the shared session queue. The current default is sixteen.

A simple request for a fixed number of new SURBs is not sufficient. A request or its response may be duplicated, delayed or delivered out of order. Interpreting each copy as permission to add another quantity would allow duplicate messages to create duplicate capacity.

Instead, the recipient grants absolute supply credit. Every SURB sent after session establishment receives a sequence number. The credit is an exclusive upper limit: a credit value of fourteen permits the initiator to send sequence numbers zero through thirteen. Repeating the same credit value grants no additional permission.

The initial credit reflects the queue positions still free after `ConnectAck` is sent. With the default capacity of sixteen, two bootstrap SURBs remain in the queue and fourteen positions are free. The first `ConnectAck` therefore authorizes fourteen numbered SURBs.

Whenever the recipient removes SURBs from the shared queue for a reverse transmission, the same number of queue positions become free. The recipient increases the absolute credit by that number, allowing the initiator to replace the consumed SURBs without exceeding the configured queue capacity.

The two SURBs attached to `OpenStream` remain outside the shared queue and outside this credit calculation. If they were inserted into the queue, they would occupy positions without consuming numbered supply credit. The initiator would then believe more queue positions were available than the recipient could actually accept.

## Identifying Supplied SURBs

Loss and reordering require the recipient to distinguish a new SURB from a retransmission of one it has already stored or consumed. The initiator therefore assigns a monotonically increasing sequence number to every SURB supplied after establishment.

The initiator sends numbered SURBs in `SurbSupply` frames. One frame states the sequence number of its first SURB and carries as many as four serialized SURBs. Later entries in the same frame use consecutive sequence numbers.

The recipient records the first sequence number not covered by a contiguous prefix of received supply. This value is the supply receive base. The recipient also keeps a fixed bitmap describing which later sequences have already arrived beyond a gap.

For example, suppose the receive base is ten. If sequences ten and twelve arrive while sequence eleven is missing, the bitmap records both arrivals. The recipient can advance the receive base to eleven because sequence ten completed the next contiguous position, while the shifted bitmap continues to record sequence twelve.

The bitmap covers a fixed window of 256 sequence positions beginning at the receive base. The recipient accepts a supplied SURB only when its sequence is inside this window, is permitted by the current absolute credit, has not been recorded before, and the shared queue still has capacity. These conditions bound both the queue and the state required to remember out-of-order arrivals.

Receiving a SURB and consuming that SURB are different events. When the recipient later removes a SURB from the shared queue, the receive base and bitmap continue recording that its sequence arrived. A retransmitted copy can therefore be rejected as a duplicate even after the first copy has already been consumed.

Each serialized SURB in a supply frame is decoded independently. If one entry is malformed, the recipient keeps the other valid entries and leaves a gap at the malformed entry's sequence. Retransmission can later fill that gap.

## Reporting Receipt and Available Capacity

The initiator must learn two facts maintained by the recipient. First, the initiator needs to know which numbered SURBs arrived so that their public serializations no longer need retransmission. Second, the initiator needs to know how much absolute credit is available for sending new SURBs.

The recipient reports both facts in one supply snapshot. The snapshot contains the receive base, the fixed acknowledgement bitmap and the absolute supply limit. The implementation represents these values as `surbSupplyReceiveBase`, `surbSupplyAcknowledgementBitmap` and `surbSupplyLimit`.

`ConnectAck` carries the initial snapshot. Later reverse Data, ACK, `StreamAck`, `StreamReject`, `RefillRequest` and `SurbStatus` frames carry the recipient's latest snapshot. Carrying all three values together prevents the initiator from applying receipt information without the corresponding view of available capacity.

Snapshots describe absolute state rather than changes relative to the previous message. Receiving the same snapshot twice does not acknowledge another SURB or grant additional credit. A delayed snapshot also cannot move the initiator's stored receive base or supply limit backwards.

When a snapshot acknowledges a numbered SURB, the initiator removes that SURB's public serialization from its retransmission state. The initiator keeps the corresponding private reply credential because the recipient may still hold the public SURB and use it for a future reverse frame.

## Proactive Replenishment

After recovering `ConnectAck`, the initiator starts one supplier task for the established session. The supplier compares the next unused sequence number with the most recent credit reported by the recipient. When credit remains available, the supplier creates new SURBs and sends them to the recipient without waiting for an explicit request.

Before sending a new public SURB, the initiator registers its private credential and retains the public serialization under the assigned sequence number. Recording both values before the asynchronous send ensures that a fast returned packet cannot refer to state that the initiator has not yet installed.

When a later reverse snapshot increases the absolute credit, the supplier wakes and replaces the SURBs that the recipient consumed. When the snapshot acknowledges receipt of supplied sequences, the initiator stops retaining their public serializations for retransmission.

Proactive replenishment is enabled by default. The `enableProactiveSurbReplenishment` constructor option can disable creation triggered solely by unused credit. Disabling proactive creation does not disable the supply protocol; the same supplier task waits for a refill request from the recipient instead.

## Recipient-Initiated Replenishment

The pull path lets the recipient explicitly wake the supplier when the shared queue is becoming too small for reverse application traffic. The recipient protects four SURBs for control traffic. Sending reverse Data or ACK requires two additional SURBs, so an application-related reverse send waits until at least six SURBs are available.

While the queue remains below that level, the recipient attempts to send `RefillRequest`. A refill request removes two SURBs from the protected control supply and sends the same request through both return paths. Removing those SURBs frees two queue positions, so the request includes the resulting higher absolute credit together with the latest receipt state.

The request contains neither a requested quantity nor a transaction identifier. After receiving `RefillRequest`, the initiator applies its absolute snapshot and marks the existing supplier task as urgent. The supplier then uses all credit currently available. Receiving the same request more than once repeats the same state and urgency condition rather than creating multiple refill transactions.

The recipient rate-limits refill requests while waiting for new supply. If sufficient supply has not arrived after thirty seconds, the recipient may use another control redundancy batch to repeat the request. Once enough SURBs are available for an application-related reverse send, the recipient clears the scheduled retry.

Pull-only operation still depends on the recipient retaining enough SURBs to send another request. If fewer than two remain, the recipient cannot reach the anonymous initiator through the pull path. The proactive recovery mechanism described below addresses this condition when proactive replenishment is enabled.

## Retransmitting Supply

A forward `SurbSupply` frame may be lost. Until the recipient acknowledges a supplied sequence, the initiator retains the serialized public SURB and assigns it a retransmission deadline. The default deadline is thirty seconds after the previous send attempt.

The retained entry also records the identifier of the corresponding private reply credential. Before retransmission, the initiator purges expired credentials and expired retired-identifier records from the reply credential store. The initiator then verifies that the selected credential remains active.

If the credential is absent, the initiator removes the retained public serialization instead of retransmitting it. A reply sent through that SURB could no longer be recovered. If the credential remains active, the initiator retransmits the same public serialization with the same supply sequence. Retransmission creates neither a new SURB nor a new credential and does not extend the credential's original lifetime.

If the original and retransmitted packets both reach the recipient, the sequence tracking described above causes one copy to be stored and the other to be ignored. If a supply acknowledgement arrives while retransmission is being submitted, the acknowledgement removes the retained entry and completion of the send does not recreate it.

## Recovering When the Recipient Has No Return Path

Proactive supply and supply retransmission still depend on the initiator learning when the recipient has freed queue positions. The recipient can consume its final SURBs and increase its absolute credit, but every reverse frame carrying the updated snapshot can be lost. The initiator then sees no new credit, while the recipient has no SURB left with which to report its current state.

When proactive replenishment is enabled, the initiator treats a long absence of reverse snapshots as a reason to check the recipient's state. After the configured interval, currently two minutes, the initiator sends a forward `SurbStatusProbe`.

The probe carries two fresh SURBs dedicated to its response. The recipient uses these SURBs immediately to send two copies of `SurbStatus`; the recipient does not insert them into the shared queue. `SurbStatus` carries the current receive base, acknowledgement bitmap and absolute credit.

Applying the returned snapshot lets the initiator resume numbered supply even when the recipient's shared queue was empty. Every probe uses fresh SURBs because the initiator cannot determine whether an earlier probe was lost or whether its response was lost.

## Relationship to Application Data Flow

Replenishment accounts for SURBs when MixTransport removes them from the shared queue for a reverse frame. The recipient grants replacement credit at that moment. The mechanism does not depend on when the initiator's application reads the recovered frame payload.

MixTransport observes application writes through the virtual stream, but the supplier does not predict future SURB demand from the size of a write. The recipient's bounded queue and absolute credit remain the authoritative limits on supply.

Application byte-flow control solves a separate problem. The Data receive window bounds chunks accepted out of order, and libp2p's `BufferStream` bounds bytes delivered in order but not yet consumed by the application. These byte limits do not replace the SURB queue limit because one mechanism bounds application data while the other bounds one-use anonymous return paths.

## Session Lifetime

The supplier task, shared SURB queue, supply sequence state and registered streams share the lifetime of one transport session. Session shutdown cancels and awaits the supplier task, closes the session's streams and removes reply credentials associated with the session.

A new anonymous session begins with an empty shared queue and a new supply sequence space. Receipt acknowledgements and credit from an earlier session cannot be applied to the new session.

## Resulting Safety Properties

- The recipient never retains more SURBs than the configured shared-queue capacity.
- The initiator cannot introduce new numbered SURBs beyond the recipient's absolute credit or the fixed receive window.
- Retransmitting a supplied SURB does not create another private credential.
- The recipient stores at most one copy of a numbered SURB, including when the original has already been consumed.
- A malformed SURB does not prevent other valid SURBs from the same supply frame from being accepted.
- Refill requests repeat absolute state and cannot grant duplicate capacity.
- Status probes allow the recipient to report available capacity even after its shared queue becomes empty.
- Session shutdown bounds the lifetime of session-owned supply state and reply credentials.
