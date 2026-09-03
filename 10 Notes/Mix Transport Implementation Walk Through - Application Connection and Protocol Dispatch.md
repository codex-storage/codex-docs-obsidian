---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Stream Establishment Round Trip]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
---
This phase turns each established `TransportStream` into the libp2p connection object used by an application protocol. When the recipient accepts `OpenStream`, MixTransport now passes that same stream object to the handler of the mounted protocol selected by the frame's codec. The handler can therefore use the ordinary libp2p `Stream` and `Connection` interface rather than a Mix-specific callback API.

The connection is now active in both directions. `TransportStream.write` delegates to MixTransport's chunking and send path, while the stream's delivery task feeds ordered incoming Data into the inherited `BufferStream`. This note concentrates on the application-facing connection and protocol dispatch; [[Mix Transport Implementation Walk Through - Bounded Data Flow]] follows the byte and acknowledgement paths in detail.

## One Object Represents the Transport Stream and the libp2p Connection

`TransportStream` now inherits from libp2p's `BufferStream`, which inherits from `Connection`. This avoids creating a wrapper around the stream registry entry. Session lookup, transport state, application reads and the object passed to the protocol handler all refer to the same instance.

Construction initializes both sets of fields. The MixTransport fields retain `sessionId`, `streamId`, codec, transport direction and establishment state. The inherited libp2p fields expose `peerId`, `protocol`, `dir` and the buffered read implementation. `BufferStream.initStream()` initializes the read queue, close event, object identifier and libp2p stream bookkeeping.

The initiator-side connection uses the real destination peer ID because the initiator selected and knows that destination. The recipient-side connection uses the session pseudonym because the recipient does not know the initiator's authenticated libp2p identity:

```text
initiator TransportStream.peerId = destination PeerId
recipient TransportStream.peerId = sessionId
```

Every stream in one session exposes the same recipient-side pseudonym. The pseudonym changes only when that session is removed and a later connection creates another session. MixTransport does not register these virtual connections with the Switch connection manager, peer store, dialer or muxer, so the session ID does not enter libp2p's global connection state. The inherited stream initialization uses it only as connection metadata, while protocol admission control uses it as a per-peer counter key.

The protocol stream metrics are labelled by protocol and direction, not by peer ID, so a new time series is not created for every session. Libp2p's per-peer admission table removes the pseudonym when its last reserved stream is released. Identifier-space exhaustion is not a practical constraint; the separate recipient-session store still needs an explicit capacity and teardown policy so retained session state cannot grow without limit.

The connection's standard idle timeout is disabled with `ZeroDuration`. MixTransport owns flow control and will add retransmission and idle policy at its session and stream layers instead of allowing the base `Connection` timer to close a virtual stream independently of its remote endpoint.

## Protocol Selection and Admission

The recipient resolves the `OpenStream.codec` through `switch.ms.lookupProtocol`. This is the same mounted-protocol registry used by libp2p multistream and supports both exact codecs and registered matchers. If no protocol matches, the recipient sends `StreamReject` as described in the preceding phase.

Finding a protocol is not sufficient to accept the stream. MixTransport calls:

```nim
protocol.reserveIncoming(session.peerId)
```

This applies the `LPProtocol` limits for total incoming streams and incoming streams per peer. On the recipient, the per-peer key is the session pseudonym. If the protocol has reached either limit, the recipient does not create the stream and sends `StreamReject` with `requested protocol cannot accept another incoming stream`.

The normal libp2p multistream dispatcher performs the same reservation before invoking a handler. MixTransport selects the codec carried by the decoded `OpenStream` frame instead of negotiating it over the virtual byte stream, so it bypasses that dispatcher and must perform the reservation explicitly. When the handler ends or is cancelled, `protocol.releaseIncoming(session.peerId)` reverses the reservation and updates the corresponding libp2p counters and metrics.

## Starting the Application Handler

After registering the inbound stream, the recipient installs the stream's write callback, starts its Data-delivery and ACK tasks, and marks the stream established before submitting `StreamAck`. The first redundant acknowledgement may reach the initiator while the recipient is still sending the remaining copies. Preparing the bounded receive path first ensures that Data sent immediately by the initiator is retained rather than rejected because the recipient stream is still pending.

After at least one `StreamAck` copy has been submitted successfully, the recipient transfers stream and protocol-reservation cleanup to `runProtocolHandler`, starts that handler as an independent asynchronous task, and proactively requests a SURB refill when needed. The resulting handler future belongs to the `TransportStream` on which the handler operates. `handleOpenStream` stores that future in `stream.handlerTask` through `setHandlerTask` and does not wait for the application handler to finish. The stream also owns its ordered-delivery and ACK tasks and, when enabled by the transport configuration, its Data retransmission task.

The relevant sequence in `handleOpenStream` is:

```nim
self.configureStream(session, stream)
stream.establish()
if not await self.sendStreamResponse(session, stream.streamId, FrameKind.StreamAck):
  return

keepStream = true
keepReservation = true
let handlerTask = runProtocolHandler(session, stream, protocol)
if not handlerTask.finished:
  stream.setHandlerTask(handlerTask)
discard await self.requestRefill(session)
```

If every acknowledgement copy fails, the existing deferred cleanup removes and closes the stream and releases the incoming protocol reservation. Cancellation follows the same cleanup path because it explicitly terminates the local stream-opening operation, regardless of whether an earlier copy escaped. If submission completes with at least one successful copy, the stream remains established and `runProtocolHandler` owns its eventual cleanup.

The distinction is important because a protocol such as block exchange keeps its handler active while it reads messages from the connection. Waiting for that read loop inside `handleOpenStream` would prevent the `OpenStream` delivery from completing and could delay the later `Data` deliveries needed to satisfy the read. Instead, the execution is separated as follows:

```text
Mix delivery task                         protocol-handler task

receive OpenStream
register and establish stream
send StreamAck
start handler task --------------------> protocol.handler(stream, codec)
return                                    wait in stream.read...

receive later Data
deliver bytes to stream buffer --------> blocked read resumes
```

`runProtocolHandler` awaits the selected handler inside its own task. Awaiting there keeps the protocol reservation and stream alive for exactly as long as the application uses the connection. After the handler returns or is cancelled, deferred cleanup clears `handlerTask` before closing the stream so `closeImpl` does not request cancellation of the task that is currently performing cleanup. Closing the stream requests cancellation of its internal Data-delivery and ACK tasks. The handler then waits for those tasks, releases the protocol reservation and removes the stream from its session.

The handler is obtained through an explicitly typed `LPProtoHandler` binding before it is called. Libp2p's `handler` accessor is a template; binding its result preserves the handler type's `raises: [CancelledError]` contract, whereas calling the overloaded template directly loses that precise effect information.

## Transport Teardown

`MixTransport.stop` first unregisters the Mix delivery and raw-reply callbacks so no new work can enter. `takeSessions` then removes every session from both store indexes without yielding. Each detached session calls `takeStreams`, which removes all of its streams before asynchronous shutdown begins. Stream shutdown closes the buffered connection, requests cancellation of its handler and internal tasks, and waits for those tasks to finish. Cancellation therefore reaches handlers blocked in a connection read as well as ACK or Data-delivery tasks blocked inside a nested asynchronous operation.

Closing a pending session fires its `established` event. Closing a pending stream fires its `resolved` event. A concurrent `connect` or `dial` therefore resumes immediately and reports that the session or stream closed; the caller does not remain blocked until its establishment timeout expires.

The ownership hierarchy now follows the lifetime of the represented objects: `MixTransport` owns sessions, each `TransportSession` owns its registered streams, and each `TransportStream` owns its handler invocation and internal tasks. MixTransport no longer keeps flat transport-wide handler and stream task collections.

## Component Test

The five-node component test mounts a protocol whose handler reports the stream and selected codec through an `AsyncQueue`, reads a length-prefixed request from the virtual connection, writes a length-prefixed response, and then waits on an `AsyncEvent` so that it remains active while the test inspects the connection. Queues, connection reads and the event provide explicit synchronization rather than sleeps.

After `dial` receives `StreamAck`, the test verifies that the mounted handler was invoked with the exact recipient-side `TransportStream`, not a separate wrapper. It verifies the codec and peer-identity rules, then exchanges application bytes in both directions through `writeLp` and `readLp`. During teardown, cancelling the waiting handler exercises the tracked-task cleanup path.
