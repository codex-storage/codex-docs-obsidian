---
related:
  - "[[Libp2p Connection Lifecycle in Logos Storage]]"
  - "[[Sphinx SURBs implementation in the libp2p MIX protocol]]"
  - "[[New Logos Storage Discovery]]"
  - "[[Mix Transport - Pluggable Integration Model]]"
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
---

# Mix Transport Design Specification

## Purpose

`MixTransport` provides long-lived, multiplexed libp2p-style connections over the anonymous Mix packet service. Application protocols continue to work with normal `Connection` operations and mounted `LPProtocol` handlers, while the transport owns the session pseudonym, stream multiplexing, chunking, ordering, acknowledgements, return-path SURBs, backpressure and reliability policy.

The generic implementation lives in the `libp2p-mix-transport` repository. Logos Storage is its first consumer, but block exchange, DHT proxy behavior and Storage-specific peer management do not belong in the transport package.

The transport uses exit-equals-destination routing. The final Mix node is the application destination and dispatches the opaque service payload directly to the registered MixTransport handler. The final architecture does not require an exit proxy to dial a separate destination or interpret an application-specific read instruction.

## Layering and Ownership

`MixProtocol` owns the cryptographic packet service:

- construction and processing of Sphinx packets;
- anonymous routing through the selected hops;
- replay-tag checking at Mix packet level;
- creation and one-time use of public SURBs;
- recovery of a raw SURB reply when given its private `ReplyCredential`;
- dispatch of an opaque final-hop payload by service codec;
- offering an opaque raw SURB reply to the registered plug-in before the embedded legacy reply path.

`MixTransport` owns the connection protocol built from that service:

- long-lived pseudonymous sessions;
- virtual stream identifiers and stream lifecycle;
- the application-facing `Connection` objects;
- Data chunking, sequencing and ordered reconstruction;
- receive windows, acknowledgements and sender backpressure;
- private reply credential groups at the original sender;
- public received-SURB groups at the recipient;
- redundancy-group consumption, refill policy and return-send serialization;
- retransmission, timeouts, close and resource limits.

Mix remains usable without MixTransport. Another upper layer may use the stateless Mix service and SURB primitives to implement a different protocol. The embedded Mix connection behavior can coexist as a fallback while the plug-in architecture is introduced.

## Session Identity

The initiator calls `connect(destination)` with the real `PeerId` of the destination Mix node. For a new relationship it generates a random, valid `PeerId` value as `sessionId` and sends it in `Connect`.

The two endpoints intentionally expose different peer identifiers to their applications:

```text
initiator-side Connection.peerId = real destination PeerId
recipient-side Connection.peerId = anonymous sessionId
```

The recipient never learns the initiator's authenticated libp2p identity from this transport. Its `sessionId` is a local pseudonymous peer key used to associate streams and application state belonging to the same anonymous session. It is not inserted into the ordinary libp2p peer store and is not used with `Switch.connect` or `Switch.dial`.

`connect(destination)` reuses an established session for that destination. The session pseudonym remains stable while the consumer considers that peer connected. Opening or closing an individual stream does not create a new peer identity. Removing the complete session and later connecting again creates a new pseudonym, which the recipient observes as a new peer.

## Session Establishment

The initiator does not consider a session established merely because `MixProtocol.send` accepted `Connect`. It waits for a matching `ConnectAck` recovered through one of the supplied SURBs:

```text
initiator                                      recipient

create pending session S
create public SURB groups and private credentials
        |
        | Connect(S, public groups)
        v
                                      create pending recipient session S
                                      store public groups
                                      send ConnectAck through one group
                                      establish recipient session
        ^
        | ConnectAck(S) through SURBs
        |
recover with session-owned credential
establish initiator session S
```

The private credentials remain at the initiator. The recipient receives only public SURBs. Successful recovery consumes the complete redundancy group, and late copies from the same group are suppressed by retired identifiers.

## Virtual Streams and Protocol Dispatch

One session carries multiple virtual application streams. A stream is identified by `(sessionId, streamId)`. The endpoint that opens a stream chooses its ID, and the other endpoint uses the same ID.

To avoid simultaneous allocation collisions without another coordination exchange:

- the session initiator allocates odd stream IDs;
- the session recipient allocates even stream IDs.

The wire protocol represents a stream identifier with the `StreamId` alias, currently a 32-bit unsigned integer encoded as Protobuf `fixed32`. The session allocator owns the odd or even progression and records exhaustion explicitly instead of allowing the value to wrap. Keeping the primitive width behind `StreamId` confines a future width change to the transport's identifier domain, although such a wire-format change still requires a new protocol version.

`dial(destination, codec)` reuses or establishes the session, registers a pending outbound stream and sends `OpenStream`. The recipient resolves `codec` through the Switch multistream registry, applies `LPProtocol.reserveIncoming(session.peerId)`, and either sends `StreamReject` with a bounded diagnostic reason or registers the matching inbound stream, prepares its bounded receive path and sends `StreamAck`.

The initiator returns the stream only after recovering `StreamAck`. Before publishing that acknowledgement, the recipient configures and establishes its stream so Data sent immediately after the first redundant ACK copy is accepted. After at least one copy succeeds, the recipient starts the mounted protocol handler as a separate task so a long-running application read loop does not block later Mix deliveries. The same publication rule applies to `ConnectAck`: recipient session state becomes operational before the first positive acknowledgement copy can reach the initiator. Complete failure of every redundant copy permits rollback because no positive acknowledgement was published.

`TransportStream` inherits from libp2p's `BufferStream`. The stream stored in the session table, passed to the protocol handler and used for application reads and writes is one object rather than a wrapper around separate transport state.

## Wire Protocol

Transport frames are Protobuf messages carried as opaque payloads under `/libp2p/mix-transport/1.0.0`. Every frame contains a version, session pseudonym and kind. Optional fields are validated against that kind before any handler accesses them.

`StreamId` and `SequenceNumber` are transport-specific aliases for `uint32`. Stream identifiers, Data sequence numbers and ACK receive bases use fixed-width Protobuf encoding. The fixed representation gives every Data frame the same numeric-field overhead and keeps the payload bound independent of the current stream ID or sequence number. A future change to either alias remains localized in the implementation, but changing the encoded width is intentionally treated as a wire-protocol version change.

The binary session pseudonym is limited to 39 bytes, the representation produced by the current `PeerId.random` generator used by `connect`. Bounding the remaining variable-size Data identifier allows the complete maximum Data-frame overhead to be known before chunking.

The currently active frames are:

| Frame | Direction and purpose |
| --- | --- |
| `Connect` | Initiator to recipient; creates the session and supplies initial public SURB groups |
| `ConnectAck` | Recipient to initiator through SURBs; confirms the session round trip |
| `OpenStream` | Stream opener to remote endpoint; selects stream ID and application codec and may supply groups |
| `StreamAck` | Remote endpoint to opener; confirms registration and protocol admission |
| `StreamReject` | Remote endpoint to opener; rejects the stream with an optional bounded reason |
| `Data` | Either logical direction; carries one sequenced chunk |
| `Ack` | Either logical direction; reports an absolute receive-base and bitmap snapshot |
| `RefillRequest` | Recipient to initiator through a reserved SURB group |
| `Refill` | Initiator to recipient through the forward path; carries new public SURB groups |

`CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` have reserved wire values but are not implemented end to end yet.

Data frames never carry SURBs. `Connect`, `OpenStream` and `Refill` are the control boundaries through which public groups enter the recipient session.

## Data Chunking and Outbound Bounds

Application writes may exceed one Sphinx payload. MixTransport divides each write into consecutive `Data` frames using `MaxDataPayloadBytes`. The bound subtracts the complete maximum Data-frame overhead from the Sphinx payload space left after Mix service framing. Because the session identifier is bounded and the stream ID and sequence number are fixed-width fields, the transport uses one payload limit instead of repeatedly encoding candidate frames to determine the capacity of each chunk. Frame validation enforces the payload limit, and final encoding independently enforces the complete Mix frame limit.

Application write boundaries are not visible at the read side. Concurrent writes to one stream are serialized, and the remote endpoint reconstructs one ordered byte stream.

Every submitted chunk remains in the sender's `pendingOutbound` table until acknowledged. The current `MaxInflightChunks` is 64. When Data retransmission is enabled, each retained chunk receives a fixed retransmission deadline after its initial submission and after each retry. The default timeout is 30 seconds. Retries continue until an ACK removes the chunk or the stream closes; a future RTT-based policy may replace the fixed timeout.

Data retransmission is enabled by default. `newMixTransport` accepts `enableDataRetransmissions = false` for deployments that prefer delivery attempts without automatic Data retries. Disabling retransmission does not remove `pendingOutbound`: the retained chunks are still required for ACK processing, the in-flight bound and remote receive-window enforcement.

The sender also respects the remote receive limit derived from the latest acknowledged receive base. It cannot introduce a sequence outside the 256-position window advertised by the remote endpoint.

Data sequences use values from `1` through `MaxDataSequenceNumber`, where `MaxDataSequenceNumber` is `SequenceNumber.high - 1`. `SequenceNumber.high` is reserved for the terminal receive base. After the receiver delivers the final valid Data sequence, it advances `receiveBase` to that terminal value and can acknowledge complete delivery without integer wraparound. Reaching the limit exhausts the stream's sequence space; the transport does not reuse sequence numbers within that stream.

## Receive Window and ACK Semantics

Each stream has a 256-position receive window represented by:

```text
receiveBase + fixed 32-byte acknowledgement bitmap
```

Every sequence below `receiveBase` has entered the receiver's ordered `BufferStream`. Bitmap bit `i` states whether sequence `receiveBase + i` is currently retained. The receiver stores out-of-order payloads only inside this window, suppresses duplicates, and never delivers a later chunk across a missing earlier sequence.

An ACK is an absolute snapshot, not a delta. The sender removes every retained chunk below the reported base and every retained chunk selected by a set bitmap bit. An older base is ignored. Duplicate Data causes the receiver to send another snapshot because the sender may have retransmitted after losing an earlier ACK. Data above the receive window is invalid under the sender's flow-control rules, so the receiver discards such Data without sending an ACK. This prevents invalid frames from causing unnecessary Mix traffic: an ACK sent by the session recipient would consume a SURB group, while an ACK sent by the session initiator would consume a forward Mix delivery.

ACK generation is currently immediate. Delayed ACK policy may later reduce packet and SURB consumption without changing the wire representation.

## Application Backpressure

Ordered receive delivery awaits `BufferStream.pushData`, whose asynchronous queue has capacity one. If the application stops reading, that push blocks and `receiveBase` stops advancing. The sender eventually reaches the unchanged remote receive limit and cannot reserve more sequences.

This design bounds transport memory without modifying libp2p's read implementation. It grants credit when a chunk enters the bounded application-facing buffer, not when the application has consumed its final byte. More exact byte-level credit can be added only if measurements show that the existing fixed staging is insufficient.

## Return Delivery and SURB Supply

Forward frames from the session initiator use `MixProtocol.send` with the real destination. The session recipient cannot address the anonymous initiator through the forward Mix path. Every Data, ACK, or control frame sent by the recipient therefore consumes one session-owned SURB redundancy group and reaches the initiator through raw reply recovery.

SURB groups are shared by all streams in a session. A per-session send lock ensures that concurrent reverse Data, ACK, and control operations cannot remove the same group. The recipient keeps the group queue within a configured capacity, while the initiator keeps the corresponding private reply credentials until a group is used, expires, or the session closes.

The intended replenishment mechanism combines proactive initiator supply with recipient requests. The recipient advertises an absolute `surbSupplyLimit`, which authorizes the initiator to introduce only a bounded number of uniquely numbered groups. The initiator retains each numbered supply entry until the recipient acknowledges it and retransmits the same public groups after loss; retransmission never creates replacement credentials for an already numbered entry. The recipient uses a receive base and fixed bitmap to accept out-of-order supply, suppress duplicates, and report which public groups the initiator may stop retaining.

Reverse transport frames carry the recipient's latest supply acknowledgement and credit state when space permits. A recipient-side refill request remains available as an urgent signal when the local supply is low. If the recipient's session pool is empty and reverse status cannot be sent, the initiator can send a forward status probe with a dedicated reply group. The recipient uses the probe's group immediately to return its current supply state instead of storing the group in the session pool. This exchange restores synchronization without requiring the recipient to have a stored session group available beforehand.

The current implementation contains the recipient-initiated refill path and its protected control reserve. The proactive supplier, numbered supply acknowledgements, absolute credit, and terminal starvation probe remain to be implemented. [[Mix Transport SURB Replenishment Strategy]] defines the complete mechanism, its safety bounds, and the incremental transition from the current implementation.

## Task and Resource Lifetime

Task ownership follows the transport hierarchy. `MixTransport` owns its sessions. Each `TransportSession` owns its registered streams. Each `TransportStream` owns its ordered-delivery and ACK tasks and, for an accepted inbound application stream, the protocol-handler invocation and incoming protocol reservation.

Closing a `TransportStream` wakes Data, ACK, capacity and stream-opening waiters and explicitly requests cancellation of its handler and internal tasks. The explicit cancellation also reaches a task that is no longer waiting on a stream event because it is suspended inside Mix delivery or SURB replenishment. `closeImpl` does not wait for task completion because the protocol handler may be the task currently completing stream cleanup. External stream shutdown waits for every owned task; natural protocol-handler completion clears its handler-task reference, closes the stream, waits for the remaining internal tasks, releases its incoming reservation and removes the stream from its session. Closing a session similarly wakes a pending `connect`; the caller then observes the closed session instead of waiting until the connection timeout.

Complete shutdown proceeds through the same hierarchy. After unregistering Mix handlers, the transport synchronously detaches all sessions through `takeSessions`. Each session synchronously detaches its streams through `takeStreams`, starts their shutdown operations and waits for them. The transport clears reply credentials only after all detached sessions and streams have completed local teardown.

Reply credential capacity rejects a new group rather than evicting an unrelated in-flight credential. Successful recovery consumes its redundancy group. Cryptographic recovery failure preserves the remaining group opportunity, while successful cryptographic recovery followed by invalid transport decoding consumes the group because every redundant copy carries the same malformed plaintext.

Runtime close, reset and complete peer-drop behavior still require the corresponding remote wire exchanges and explicit session resource reclamation. Once a session or stream has been established, an endpoint that closes it normally should make a best-effort attempt to send `CloseStream` or `Disconnect` before removing its local state. Cancellation and failure should similarly use `ResetStream` or `ResetSession` when a return path is still available. These notifications allow the remote endpoint, including an initiator waiting for further traffic, to release its state immediately instead of discovering the closure only through a timeout. Sending a notification must not delay cancellation or make local cleanup depend on successful Mix delivery; if notification is no longer possible, teardown proceeds locally.

## Logos Storage Integration

Logos Storage will inject `MixTransport` into the network path used by block exchange. When Mix is enabled, peer establishment and stream dialing must go through the transport rather than calling `switch.connect` for the anonymous application peer.

The recipient-side block-exchange handler receives a normal `Connection` whose `peerId` is the session pseudonym. Transport session events, rather than raw Switch JOINED events from relay connections, must determine which anonymous application peers enter or leave the block-exchange peer set. A physical relay may also be a Storage node, but its direct Mix-overlay connection is not evidence that it opened an anonymous block-exchange session.

The existing exploratory `storage/mix/` code can inform initialization and dependency injection, but the generic package interface is the source of truth. Storage integration may replace that exploratory code where it does not match this design.

## Current Implementation Status

Implemented and covered by focused or live tests:

- Mix plug-in registration with embedded fallback when the plug-in does not handle a reply;
- stateless public SURB creation, SURB send, raw reply and recovery primitives in Mix;
- reply credential grouping, capacity, expiry and late-reply suppression;
- session creation, pseudonymous identity and destination-based reuse;
- odd/even stream allocation and `OpenStream` acknowledgement or rejection;
- mounted protocol lookup, incoming admission reservation and asynchronous handler dispatch;
- application-facing `BufferStream` connections;
- payload-aware chunking and bidirectional Data transfer;
- bounded sender state, fixed receive window, ordered delivery, absolute bitmap ACKs and optional Data retransmission enabled by default;
- application backpressure through `BufferStream`;
- recipient SURB control reserve, serialized return sends and one-packet refill;
- cancellation-safe local handler and flow-task shutdown;
- a five-node live request/response exchange through the standard connection API.

Not yet implemented:

- Data retransmission retry limits and RTT/RTO selection;
- ACK send retry and optional delayed-ACK batching;
- a persist probe when all receive-window updates are lost;
- proactive SURB supply, numbered supply acknowledgements and bounded supply credit;
- terminal SURB starvation recovery through an initiator-side status probe;
- `CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` processing;
- runtime session limits and complete peer-drop cleanup;
- authenticated Mix service discovery and destination record lifecycle;
- Logos Storage block-exchange integration and replacement peer events;
- removal of the embedded legacy path and final cleanup of forward mode.

## Migration Sequence

1. Complete the generic transport reliability and lifecycle mechanisms while retaining Mix's embedded fallback.
2. Integrate the package into Logos Storage and route block-exchange connect, dial and peer events through it.
3. Add authenticated Mix service discovery and destination record lifecycle.
4. Migrate remaining one-shot Mix users, including DHT proxy behavior, to exit-equals-destination service dispatch or a deliberately preserved compatibility facade.
5. Remove forward destination mode, destination read behaviors, external exit dialing and the obsolete exit-mode compile-time path after all consumers have migrated.

## Final Acceptance Criteria

- Two Mix-aware nodes establish a session only after a successful anonymous `Connect`/`ConnectAck` round trip.
- The recipient exposes only the session pseudonym as the incoming connection's peer identity.
- Repeated `connect` calls reuse the active session and its pseudonym.
- Multiple `dial` calls create independent streams whose incoming connections share the session identity.
- Closing a stream does not remove or re-identify the consumer peer; dropping the complete peer session does.
- Direct Switch connections between Mix relays do not create anonymous block-exchange peers.
- Arbitrarily sized writes are reconstructed as one ordered byte stream despite lost, reordered and duplicated packet delivery.
- Sender state, receive buffering, reply credentials, received SURBs and tracked tasks remain within configured bounds.
- Slow application reads stop the sender from introducing unbounded data.
- Lost ACKs cause duplicate Data to be acknowledged again rather than delivered twice.
- Retrying a return frame consumes a fresh SURB group and never reuses a sent SURB.
- Refill and persist mechanisms cannot permanently strand an otherwise live session after one lost control packet.
- Timeouts, close, reset, cancellation and capacity failures reclaim all session-owned state on both endpoints.
- Logos Storage block exchange reuses its normal frame reader and protocol handlers over the virtual connection.
- The final Mix routing model uses exit equals destination and does not require application-specific read behavior in Mix core.
