---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Connect Handshake]]"
  - "[[Mix Transport Implementation Walk Through - Virtual Stream Registry]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
  - "[[Mix Transport Implementation Walk Through - SURB Replenishment]]"
---
This phase implements the first complete virtual-stream handshake over a live Mix network. The initiator calls `MixTransport.dial(destination, codec)`. The call returns an established `TransportStream` only after the destination has registered a matching inbound stream and returned `StreamAck` anonymously through a SURB.

The established `TransportStream` is also the libp2p `Connection` passed to the mounted application handler. After this handshake succeeds, the configured stream can transfer Data and ACK frames as described in [[Mix Transport Implementation Walk Through - Bounded Data Flow]]. This note concentrates on the control round trip that decides whether that connection may exist.

## Initiator: Preparing `OpenStream`

`dial` first calls `connect(destination)`. If an established session already exists for the destination, `connect` returns it without starting another session handshake. Otherwise it completes the `Connect` and `ConnectAck` exchange before stream creation continues.

The initiator then calls `session.addOutboundStream(codec)`. This allocates the next locally owned stream identifier and stores a pending outbound stream. For a session created by this endpoint, the first identifier is `1`.

The beginning of `dial` therefore separates session establishment from stream establishment:

```nim
let session = (await self.connect(destination)).valueOr:
  return err(error)
let stream = session.addOutboundStream(codec).valueOr:
  return err(error)
var keepStream = false
defer:
  if not keepStream:
    discard session.removeStream(stream.streamId)
```

The `defer` gives the pending stream transactional lifetime. Every error path removes it unless the complete round trip reaches the final `keepStream = true`.

The initiator creates two independent reply SURBs. It stores the corresponding private reply credentials locally under the session ID and serializes only the public SURBs into the `OpenStream` frame:

```text
OpenStream
  sessionId = existing session pseudonym
  streamId  = newly allocated stream ID
  codec     = requested application protocol
  surbs      = dedicated StreamAck or StreamReject return paths
```

`OpenStream` is a control frame, so attaching SURBs does not change Data chunk sizing. These two SURBs are dedicated to the response for this opening attempt. The recipient decodes them into a temporary redundancy batch and sends `StreamAck` or `StreamReject` through that batch without inserting the SURBs into the session queue. Numbered `SurbSupply` remains the only post-establishment mechanism that increases ordinary session return capacity.

The public SURBs and their private credentials are produced together. Only the encoded public SURBs cross the Mix network:

```nim
let prepared = self.createReplySurbs(
  destination, session.sessionId, DefaultOpenStreamReplySurbs
).valueOr:
  return err("could not prepare OpenStream reply SURBs: " & error)

let frame = MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.OpenStream,
  streamId: Opt.some(stream.streamId),
  codec: Opt.some(codec),
  surbs: prepared.encoded,
)
```

`prepared.credentials` remain in the initiator's `ReplyCredentialStore`. They let `handleRawSurbReply` recover the `StreamAck` or `StreamReject` sent through these dedicated SURBs.

After sending `OpenStream` through `MixProtocol.send`, `dial` waits on the outbound stream's resolution event. `StreamAck` establishes the stream, while `StreamReject` rejects it and lets `dial` return an error without waiting for `streamOpenTimeout`. The timeout remains necessary when no response arrives.

`waitUntilResolved` exposes that event as a directly delegated future. The raw async annotation records that cancellation can propagate from the caller without creating an additional wrapper future:

```nim
proc waitUntilResolved*(
    stream: TransportStream
): Future[void] {.async: (raw: true, raises: [CancelledError]).} =
  stream.resolved.wait()
```

```nim
(await self.mix.send(
  MixDestination.exitNode(destination), MixTransportCodec, payload
)).isOkOr:
  return err("could not send OpenStream frame: " & error)

if not await stream.waitUntilResolved().withTimeout(self.streamOpenTimeout):
  return err("MixTransport stream opening timed out")
if stream.state == StreamState.Rejected:
  return err(stream.rejectionReason)

self.configureStream(session, stream)
keepStream = true
ok(stream)
```

`configureStream` is intentionally after remote acceptance on the opener. It installs the Data write callback and starts the Data-delivery and ACK tasks only for a stream that `dial` is about to return to its caller.

If the operation fails before `OpenStream` is submitted to Mix, `dial` removes the pending stream and retires the credentials it just prepared. After successful submission, the recipient may already have consumed the dedicated public SURBs even if its response is lost. The corresponding credentials therefore remain registered until each reply arrives, the credentials expire, or the complete session is removed. The rejected or timed-out stream itself is removed, while the established session remains available for another operation.

## Recipient: Registering the Inbound Stream

The destination receives `OpenStream` through the Mix delivery handler registered for `MixTransportCodec`. The handler first verifies that the frame refers to an established recipient-side session. The handler then attempts to deserialize every attached SURB. If at least two values are valid, the handler moves the first two valid SURBs into a local `replyBatch`; it does not add any of the attached SURBs to `TransportSession.receivedSurbs`.

```nim
let session = self.sessions.get(frame.sessionId).valueOr:
  return
if session.role != SessionRole.Recipient or
    session.state != SessionState.Established:
  return

var decodedSurbs = newSeqOfCap[SURB](frame.surbs.len)
for encodedSurb in frame.surbs:
  let surb = encodedSurb.deserializeSurb().valueOr:
    continue
  decodedSurbs.add(surb)
if decodedSurbs.len < DefaultReplySurbRedundancy:
  return

var replyBatch = newSeqOfCap[SURB](DefaultReplySurbRedundancy)
for index in 0 ..< DefaultReplySurbRedundancy:
  replyBatch.add(move(decodedSurbs[index]))
```

The recipient prepares the response paths before inspecting the requested codec because every rejection also needs an anonymous return path. If fewer than two attached values decode successfully, the recipient cannot acknowledge or reject the opening attempt and drops the frame without creating a stream.

The recipient uses the Switch's multistream registry to find a mounted protocol matching the requested codec. If no protocol matches, it does not register an inbound stream. Instead, it sends `StreamReject` through the `replyBatch` supplied by `OpenStream`. The frame includes the recipient's diagnostic reason, `requested protocol is not supported`, allowing the initiator to return that specific error as soon as the first valid redundant rejection arrives.

```nim
let protocol = self.mix.switch.ms.lookupProtocol(frame.codec.get()).valueOr:
  discard await self.sendStreamResponse(
    session,
    move(replyBatch),
    frame.streamId.get(),
    FrameKind.StreamReject,
    "requested protocol is not supported",
  )
  return

if not protocol.reserveIncoming(session.peerId):
  discard await self.sendStreamResponse(
    session,
    move(replyBatch),
    frame.streamId.get(),
    FrameKind.StreamReject,
    "requested protocol cannot accept another incoming stream",
  )
  return
```

`lookupProtocol` uses the same multistream registry as an ordinary libp2p dial. `reserveIncoming` applies the mounted protocol's incoming-stream admission limit. The `PeerId` passed to that bookkeeping is the recipient-side session pseudonym; it is not used to reveal or dial the anonymous initiator.

The rejection reason is a bounded string rather than a shared enum. The recipient has the most accurate information about why it refused the stream, and the initiator does not need to reproduce the recipient's decision logic. Before encoding the frame, the recipient truncates a reason longer than 255 bytes. The decoder independently rejects an overlong wire value. A received `StreamReject` remains valid when the optional reason is absent or contains an empty string; in that case the initiator reports `remote rejected the stream for an unknown reason`.

This lookup supports exact codecs and matchers registered with libp2p multistream. `StreamAck` must confirm that the destination can service the codec, not merely store its string. After lookup, the current handler also calls `protocol.reserveIncoming(session.peerId)` before creating the stream. Admission failure returns `StreamReject` with `requested protocol cannot accept another incoming stream`.

The recipient calls `addInboundStream` with the exact `streamId` and codec from the frame. The session registry verifies that the ID belongs to the remote endpoint's allocation space and is not already present. The resulting stream is pending, inbound, and uses the same `(sessionId, streamId)` pair as the initiator's outbound stream.

The recipient sends the same encoded `StreamAck` through both one-shot SURBs in the local `replyBatch`. `sendStreamResponse` attaches the recipient's latest numbered supply snapshot before encoding the response. If neither reply can be submitted, the recipient removes the newly registered stream. If at least one submission succeeds, the recipient keeps the inbound stream established.

Before submitting `StreamAck`, the recipient configures the stream's write callback, starts its Data-delivery, ACK and optional retransmission tasks, and marks the stream established. This ordering matters because the first redundant `StreamAck` can reach the initiator while the recipient is still submitting another copy. Once the initiator receives that first acknowledgement, it may immediately send Data. Establishing the recipient-side stream before the acknowledgement leaves ensures that such Data enters the configured receive path instead of being discarded as traffic for a pending stream.

After at least one `StreamAck` copy has been submitted successfully, the recipient retains the stream and its libp2p incoming-stream reservation. The recipient then starts the mounted protocol handler as a task owned by the stream and requests a refill if the session queue is low. The handler receives the same `TransportStream` and may use normal connection reads and writes without blocking `handleOpenStream`.

```nim
self.configureStream(session, stream)
stream.establish()
if not await self.sendStreamResponse(
  session, move(replyBatch), stream.streamId, FrameKind.StreamAck
):
  return

keepStream = true
keepReservation = true
let handlerTask = runProtocolHandler(session, stream, protocol)
if not handlerTask.finished:
  stream.setHandlerTask(handlerTask)
discard await self.requestRefill(session)
```

The protocol handler starts only after `sendStreamResponse` reports that at least one acknowledgement copy was submitted. A long-running handler read loop runs as the stream's `handlerTask`; it does not block the Mix delivery handler from processing later frames. Keeping the task on `TransportStream` also gives stream shutdown a direct task to cancel and await. A handler is recorded only while its future remains unfinished because an asynchronous Nim procedure can complete synchronously when none of its awaited futures suspend. In that case `runProtocolHandler` has already performed its deferred cleanup and no task remains for the stream to own.

`runProtocolHandler` supplies the virtual connection to the mounted protocol and balances the admission reservation when the handler ends:

```nim
proc runProtocolHandler(
    session: TransportSession, stream: TransportStream, protocol: LPProtocol
) {.async: (raises: [CancelledError]).} =
  defer:
    stream.clearHandlerTask()
    await noCancel stream.close()
    await noCancel stream.cancelAndWaitForStreamTasks()
    protocol.releaseIncoming(stream.peerId)
    discard session.removeStream(stream.streamId)

  # Binding the template accessor preserves LPProtoHandler's raises list.
  let handler: LPProtoHandler = protocol.handler
  await handler(stream, stream.codec)
```

## Initiator: Processing `StreamAck` or `StreamReject`

Each raw SURB reply first enters the shared reply credential store. The SURB identifier selects one private reply credential and the session that owns it. Successful recovery consumes only that credential. A second reply carrying the same logical `StreamAck` uses its own identifier and credential, so it can also be recovered. The pending-state check described below prevents the second copy from resolving the stream again.

After decoding the recovered frame, the transport verifies that the frame's `sessionId` matches the session recorded with the credentials. It then looks up `streamId` inside that established session. Only a pending outbound stream can be resolved. `StreamAck` calls `stream.establish()`, while `StreamReject` records the transmitted reason and calls `stream.reject()`. Both operations fire the stream's resolution event and wake `dial`. `dial` returns the established stream after an acknowledgement, or removes the rejected stream and returns the recorded reason as its error after a rejection.

The state transition is small because all routing and cryptographic validation has already happened before `handleReplyFrame` receives the frame:

```nim
of FrameKind.StreamAck, FrameKind.StreamReject:
  if session.state != SessionState.Established:
    return
  let stream = session.getStream(frame.streamId.get()).valueOr:
    return
  if stream.direction != StreamDirection.Outbound or
      stream.state != StreamState.Pending:
    return
  if frame.kind == FrameKind.StreamAck:
    stream.establish()
  else:
    let rejectionReason = frame.rejectionReason.get("")
    stream.reject(
      if rejectionReason.len == 0:
        UnknownStreamRejectionReason
      else:
        rejectionReason
    )
```

Every redundant reply is recovered independently, so this transition can receive the same logical response more than once. The pending-state check makes the response idempotent: only the first valid `StreamAck` or `StreamReject` can resolve the pending stream, and later copies cannot change an already resolved stream.

The two endpoints now retain matching state:

```text
initiator                                      recipient

session S, outbound stream 1, pending
        |
        | OpenStream(S, 1, codec, individual SURBs)
        v
                                      session S, inbound stream 1, pending
        ^
        | StreamAck(S, 1) through a temporary SURB redundancy batch
        |
session S, outbound stream 1, established      inbound stream 1, established
```

For an unsupported codec, no inbound stream is created:

```text
initiator                                      recipient

session S, outbound stream 3, pending
        |
        | OpenStream(S, 3, unsupported codec, individual SURBs)
        v
                                      no matching mounted protocol
        ^
        | StreamReject(S, 3) through a temporary SURB redundancy batch
        |
dial returns an error; stream 3 is removed     no stream 3 was registered
```

The initiator knows the destination's real libp2p peer ID because it selected that destination. The recipient identifies the remote transport peer only by session pseudonym `S`. Opening the stream does not alter either identity rule.

## Component Test

`tests/test_connect.nim` starts five real Mix nodes with deterministic zero relay delay. The first and last nodes run `MixTransport`; the three middle nodes provide the Mix paths. The test performs `Connect`, reuses the established session, and then calls `dial` for a test codec.

The destination Switch mounts the test codec before `dial` is called. After the first `dial` returns, the test verifies that the initiator and recipient hold the same session ID, stream ID and codec, and that both streams are established with the expected local directions. The same test now proceeds through a request and response using the ordinary connection API.

The test then calls `dial` with a codec that is not mounted at the destination. It verifies that the call returns `requested protocol is not supported`, the reason provided by the destination, and that neither endpoint retains the rejected stream. The wire test separately verifies that a rejection without a reason still encodes and decodes, allowing the receiving transport to apply its unknown-reason fallback. Finally, the component test waits for both redundant `ConnectAck` replies, both redundant `StreamAck` replies, and both redundant `StreamReject` replies. This explicit synchronization ensures that teardown begins only after every expected return packet has passed through the raw reply handler.
