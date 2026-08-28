---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Session Registry]]"
  - "[[Mix Transport Implementation Walk Through - Connect Handshake]]"
---

# Mix Transport Implementation Walk Through - Virtual Stream Registry

This phase adds the state needed to represent multiple virtual application streams inside one established MixTransport session. It defines `TransportStream` in `libp2p_mix_transport/streams.nim` and makes each `TransportSession` responsible for registering and removing its streams.

`TransportStream` now inherits from libp2p's `BufferStream`, and therefore from `Connection`. The session registry, frame router, application protocol handler and buffered read/write path all refer to the same stream object; there is no separate application wrapper.

## Stream Identity Is Scoped by the Session

A `streamId` is unique within one transport session. The complete identity of a stream is therefore the pair:

```text
(sessionId, streamId)
```

Two different sessions may both contain stream `1` without ambiguity because their `sessionId` values differ.

The endpoint opening a stream selects its `streamId`. It will send that ID in an `OpenStream` frame, and the remote endpoint will register an inbound stream with the same ID. One logical stream never receives separate local and remote identifiers:

```text
session initiator                         session recipient

outbound stream 1
        |
        | OpenStream(streamId = 1)
        v
                                          inbound stream 1
```

## Preventing Concurrent Allocation Collisions

Both endpoints may open streams within the same established session. If both started independent counters at `1`, simultaneous opens could assign `1` to two different streams before either `OpenStream` frame arrived.

The registry avoids coordination by partitioning the positive `uint64` identifier space:

- The endpoint that initiated the transport session allocates odd identifiers: `1`, `3`, `5`, and so on.
- The endpoint that received the transport session allocates even identifiers: `2`, `4`, `6`, and so on.

This partition determines only which endpoint may originate an identifier. Both endpoints still use the same ID for the resulting stream. For example, stream `2` is outbound at the session recipient and inbound at the session initiator.

The IDs are sequential because they need to be unique only within one session. Small sequential values also have a compact Protobuf varint representation. The available space is large enough that exhaustion is not a practical concern during one session's lifetime.

## Outbound Stream Registration

`addOutboundStream(codec)` may be called only on an established session and requires a non-empty application codec. It takes the session's next locally assigned identifier, creates a pending outbound `TransportStream`, stores it by `streamId`, and advances the local counter by two.

```nim
let stream = newTransportStream(
  session.sessionId, session.peerId, session.nextOutboundStreamId, codec,
  StreamDirection.Outbound,
)
session.streams[stream.streamId] = stream
session.nextOutboundStreamId += 2
```

The stream remains pending because local registration alone does not show that the remote endpoint accepted the application codec. `MixTransport.dial` sends `OpenStream` and waits for this stream to become established after `StreamAck` returns.

## Inbound Stream Registration

`addInboundStream(streamId, codec)` also requires an established session and a non-empty codec. Before changing the registry, it verifies two independent conditions:

1. The identifier belongs to the remote endpoint's odd or even allocation space. Identifier zero is never valid.
2. No stream with that exact identifier is already registered in the session.

The first check rejects a remote frame that attempts to use an identifier reserved for locally opened streams. The second check detects duplicate `OpenStream` frames and prevents them from replacing the existing stream.

The role-based validation is deliberately local: it answers whether an identifier received from the other endpoint belongs to that endpoint's allocation space.

```nim
func isValidInboundStreamId(session: TransportSession, streamId: uint64): bool =
  if streamId == 0:
    return false

  case session.role
  of SessionRole.Initiator:
    streamId mod 2 == 0
  of SessionRole.Recipient:
    streamId mod 2 == 1
```

An initiator allocates odd outbound IDs itself, so it accepts even IDs in incoming `OpenStream` frames. A recipient allocates even outbound IDs itself, so it accepts odd incoming IDs. The modulo test does not claim that every identifier with the expected parity already exists; `addInboundStream` creates the stream only after also checking the session state, codec and duplicate table entry.

After validation, the session creates a pending inbound stream with the exact `streamId` supplied by the opener. The recipient's `OpenStream` handler establishes it only after registering the stream and successfully submitting `StreamAck` through a SURB reply group.

```nim
if not session.isValidInboundStreamId(streamId):
  return err("stream identifier was not allocated by the remote endpoint")
if session.streams.hasKey(streamId):
  return err("stream identifier is already registered")

let stream = newTransportStream(
  session.sessionId, session.peerId, streamId, codec, StreamDirection.Inbound
)
session.streams[streamId] = stream
ok(stream)
```

At this point the stream is registered but still `Pending`. Registration lets subsequent control code refer to the same object, while the pending state prevents application use before the remote opener has been sent an acknowledgement.

## Direction and State

`StreamDirection` is local to an endpoint:

```text
opener:          Outbound
remote endpoint: Inbound
```

It is independent of `SessionRole`. A session initiator can receive an inbound stream opened later by the session recipient, and a session recipient can hold an outbound stream that it opened itself.

Every stream starts in `Pending`. `establish` changes the state to `Established`, while `reject` changes it to `Rejected`. Both operations fire the stream's resolution `AsyncEvent`. On the recipient, the `OpenStream` handler establishes its inbound stream after successfully submitting `StreamAck`. On the opener, the reply handler establishes the pending outbound stream after `StreamAck`, or rejects it after `StreamReject`. `dial` waits on the resolution event instead of polling and then returns either the established stream or an error.

## Construction Boundary

`newTransportStream` is exported from the internal `streams.nim` module because `sessions.nim` is a separate Nim module and must be able to call it. The package facade deliberately excludes that constructor from its exports. Normal consumers create registered streams through `addOutboundStream` and `addInboundStream`, which enforce the session and identifier rules described above.

## Stream and Session Lifetimes

The session owns its stream table. Removing one stream deletes only that stream's entry and leaves the established transport session registered. This preserves the session pseudonym and allows later streams to reuse the same anonymous peer relationship.

Removing the complete session releases its stream table together with the other session-owned state. Local stream closure already wakes Data, ACK and send-capacity waiters, and transport shutdown cancels and waits for the tracked flow tasks. The wire-level close, reset and disconnect exchanges are still pending; those operations must coordinate remote state before normal runtime session removal is complete.

## Tests

`tests/test_streams.nim` verifies that:

- the session initiator allocates odd IDs while the recipient allocates even IDs;
- multiple locally opened streams receive distinct sequential identifiers;
- an inbound stream keeps the exact identifier chosen by its remote opener;
- an endpoint rejects inbound IDs from its own allocation space, identifier zero, and duplicate IDs;
- streams cannot be created before their transport session is established;
- removing a stream leaves its established session available.
