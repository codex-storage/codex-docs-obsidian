---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Application Connection and Protocol Dispatch]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
---
This walkthrough follows application bytes through an established `TransportStream`. The forward path begins when the session initiator writes to the stream, divides the byte sequence into Sphinx-sized transport frames and sends those frames to the session recipient. The recipient restores the ordered byte stream, exposes the bytes to the mounted libp2p protocol and acknowledges the received sequences. The walkthrough then follows the reverse path, where the recipient sends Data and ACK frames through SURB groups supplied by the initiator. Because every reverse frame consumes one group, the recipient must replenish that supply before it loses its final return path.

The implementation is divided across four modules:

- `wire.nim` defines the `Data`, `Ack`, `RefillRequest` and `Refill` frames. The module also defines the fixed-width stream and sequence types, derives one maximum Data payload size and validates the fixed-size acknowledgement bitmap.
- `streams.nim` owns sequence numbers, retained outbound chunks, the receive window, its bitmap and the events used by the flow tasks.
- `sessions.nim` owns the recipient's SURB groups, refill state and the lock that serializes return sends within one session.
- `transport.nim` connects the standard libp2p `Connection` methods to those state machines. The module selects forward Mix delivery for frames sent by the session initiator and SURB delivery for frames sent by the session recipient.

The current implementation bounds memory, propagates application backpressure, retransmits unacknowledged Data and retries a SURB refill while the recipient still has a group with which to send another request. Data retransmission is enabled by default and can be disabled when constructing `MixTransport`. The transport does not yet probe a stalled receive window when every ACK carrying the advanced window has been lost.

## The State Behind One Virtual Connection

`TransportStream` inherits from libp2p's `BufferStream`. The same object therefore contains the transport state and the connection read buffer passed to the application protocol.

The receiver accepts Data sequence numbers only within a fixed-size receive window. The receive base is the lowest sequence number in that window and, equivalently, the next sequence that has not yet been passed in order into `BufferStream`. Sequence numbers below the receive base have entered `BufferStream`, although the application may not have read all of those bytes yet. The local field `receiveBase` stores this value and starts at `1`. `SequenceNumber.high` is reserved as the terminal receive-base value, so the last valid Data sequence is `MaxDataSequenceNumber = SequenceNumber.high - 1`. After passing that final Data sequence to `BufferStream`, the receiver can still represent the next expected position without wrapping to zero.

Each ACK reports the receiver's current receive base to the sender. The sender stores the most recently reported value in `remoteReceiveBase`. The local `receiveBase` therefore describes this endpoint's incoming Data, while `remoteReceiveBase` records how far the other endpoint has progressed when passing Data sent by this endpoint into its `BufferStream`. The sender combines `remoteReceiveBase` with the receive-window size to determine the highest sequence number that the receiver currently permits.

The outbound part of a stream records `nextOutboundSequence`, `remoteReceiveBase`, and the chunks that have been submitted but not acknowledged. Each retained `OutboundChunk` contains the payload and an optional retransmission deadline. The deadline is absent while the initial transmission or a retransmission is being submitted. After submission completes, the transport sets the next deadline unless an ACK has already removed the chunk.

The outbound part owns two events. An application writer waits on `sendStateChanged` when the stream cannot allocate another sequence. Applying a received ACK signals `sendStateChanged` after removing acknowledged chunks or advancing `remoteReceiveBase`. Removing a chunk that could not be submitted and closing the stream also signal `sendStateChanged`. The retransmission task waits on `retransmissionStateChanged` when no deadline exists or when the next deadline is still in the future. Scheduling a deadline, removing an acknowledged chunk and closing the stream signal `retransmissionStateChanged`, causing the task to inspect the current table state again.

The inbound part records `receiveBase`, a bitmap describing received positions at and above `receiveBase`, and the out-of-order payloads retained inside the receive window. Retaining a payload means inserting the payload into `pendingInbound` under its sequence number and setting the corresponding bit in the acknowledgement bitmap. The payload remains in `pendingInbound` until all preceding sequences have been passed to `BufferStream` and the payload becomes the next one eligible for ordered delivery.

The inbound part also owns two events. `dataAvailable` tells the ordered-delivery task to check whether the payload for `receiveBase` is available. The transport signals `dataAvailable` after `receiveData` accepts and retains any in-window payload, including a payload that arrived out of order. After advancing `receiveBase`, the transport signals `dataAvailable` again when `pendingInbound` already contains the payload for the new base. Closing the stream also signals `dataAvailable` so that the delivery task can terminate. A `dataAvailable` signal therefore does not assert that the required payload is present - meaning it does not assert that payload at the `receiveBase` has been received - it signals that some payload within the receive window has been received.

`shouldSendAck` tells the ACK task to capture and send the current `receiveBase` and acknowledgement bitmap. The transport signals `shouldSendAck` after accepting an in-window payload, after receiving duplicate Data, and after advancing `receiveBase`. Closing the stream also signals `shouldSendAck`; the awakened ACK task checks `stream.closed` and terminates without sending another ACK.

The fields are declared together on `TransportStream`:

```nim
TransportStream* = ref object of BufferStream
  sessionId: PeerId
  streamId: StreamId
  codec: string
  direction: StreamDirection
  state: StreamState
  writeHandler: StreamWriteHandler
  writeLock: AsyncLock

  nextOutboundSequence: SequenceNumber
  remoteReceiveBase: SequenceNumber
  pendingOutbound: Table[SequenceNumber, OutboundChunk]
  sendStateChanged: AsyncEvent
  retransmissionStateChanged: AsyncEvent

  receiveBase: SequenceNumber
  acknowledgementBitmap: seq[byte]
  pendingInbound: Table[SequenceNumber, seq[byte]]
  dataAvailable: AsyncEvent
  shouldSendAck: AsyncEvent
```

The outbound and inbound state describe different directions of the same bidirectional stream. `pendingOutbound` contains chunks sent by this endpoint and awaiting acknowledgement. `pendingInbound` contains chunks received from the remote endpoint but not yet passed in order into this endpoint's `BufferStream`.

The four events carry notifications rather than the associated state. After waking, a task checks the corresponding counters, tables or bitmap again. A notification means "check again," not "capacity definitely exists" or "a particular chunk is definitely available." The waiting loops therefore re-evaluate their conditions after every wake-up.

Construction starts both sequence spaces at `1` and allocates the fixed bitmap:

```nim
nextOutboundSequence: 1,
remoteReceiveBase: 1,
pendingOutbound: initTable[SequenceNumber, OutboundChunk](),
sendStateChanged: newAsyncEvent(),
retransmissionStateChanged: newAsyncEvent(),
receiveBase: 1,
acknowledgementBitmap: newSeq[byte](AckBitmapBytes),
pendingInbound: initTable[SequenceNumber, seq[byte]](),
dataAvailable: newAsyncEvent(),
shouldSendAck: newAsyncEvent(),
```

`configureStream` installs the callback used by `TransportStream.write` and starts one ordered-delivery task and one acknowledgement task. When `enableDataRetransmissions` is true, `configureStream` also starts the stream's retransmission task:

```nim
proc configureStream(
    self: MixTransport, session: TransportSession, stream: TransportStream
) =
  let writeHandler: StreamWriteHandler = proc(
      data: sink seq[byte]
  ): Future[void] {.async: (raw: true, raises: [CancelledError, LPStreamError]).} =
    self.writeStream(session, stream, move(data))
  stream.setWriteHandler(writeHandler)
  stream.trackStreamTask(runInboundDelivery(stream))
  stream.trackStreamTask(runAcknowledgements(self, session, stream))
  if self.dataRetransmissionsEnabled:
    stream.trackStreamTask(runRetransmissions(self, session, stream))
```

An outbound stream is configured after its `StreamAck` arrives. At the recipient, an inbound stream is configured immediately before it is established and passed to the mounted protocol. A rejected stream never starts these tasks. The tasks are stored on the stream whose state and events they use; they are not stored in a transport-wide task collection.

## 1. An Application Write Enters MixTransport

The application uses a normal libp2p connection. In this example it calls the inherited `LPStream.writeLp` method:

```nim
await initiatorStream.writeLp(TestRequest)
```

`writeLp` constructs one byte sequence containing the varint length prefix followed by `TestRequest`, then passes that sequence to the virtual `write` method:

```nim
method writeLp*(
    s: LPStream, msg: openArray[byte]
): Future[void] {.base, async: (raises: [CancelledError, LPStreamError], raw: true).} =
  let vbytes = PB.toBytes(msg.len().uint64)
  var buf = newSeqUninit[byte](msg.len() + vbytes.len)
  buf[0 ..< vbytes.len] = vbytes.toOpenArray()
  buf[vbytes.len ..< buf.len] = msg
  s.write(move(buf))
```

Because `s` is a `TransportStream`, dynamic dispatch selects `TransportStream.write` for the final call. The application therefore invokes `writeLp`, while the transport implements the lower-level `write` operation that receives the complete length-prefixed byte sequence.

Before the stream is exposed to the application, `configureStream` assigns `stream.writeHandler` a closure that captures the `MixTransport`, `TransportSession` and `TransportStream` objects associated with this virtual connection. Calling that closure invokes `self.writeStream(session, stream, ...)`. The write handler is therefore the adapter between the application-facing `TransportStream.write` method and the transport's frame-producing `MixTransport.writeStream` procedure.

`TransportStream.write` checks that `configureStream` installed this handler, serializes complete application writes with `writeLock`, and invokes the handler with the application bytes:

```nim
method write*(
    stream: TransportStream, msg: sink seq[byte]
): Future[void] {.async: (raises: [CancelledError, LPStreamError]).} =
  if stream.writeHandler.isNil:
    raise newException(LPStreamError, "MixTransport stream is not writable")

  await stream.writeLock.acquire()
  defer:
    stream.writeLock.release()
  await stream.writeHandler(move(msg))
```

The lock covers one complete application write. Two application tasks can call `write` concurrently on the same stream; for example, a protocol can produce a response while another task writes a control message. Without the lock, `writeStream` could alternate between those calls while assigning sequence numbers, so chunks from the two byte sequences could be interleaved on the receiving side. Applying received ACKs and delivering inbound Data do not use `writeLock`; both operations continue while one writer waits for capacity or Mix delivery.

Application write boundaries are not encoded on the wire. The receiver reconstructs one ordered byte stream, which is what `readOnce`, `readExactly` and `readLp` expect.

## 2. The Writer Waits for Capacity

Before taking bytes from the application write, `writeStream` waits until the stream may allocate another outbound sequence:

```nim
proc writeStream(
    self: MixTransport,
    session: TransportSession,
    stream: TransportStream,
    data: sink seq[byte],
): Future[void] {.async: (raises: [CancelledError, LPStreamError]).} =
  if stream.state != StreamState.Established:
    raise newException(LPStreamError, "MixTransport stream is not established")

  var offset = 0
  while offset < data.len:
    await stream.waitForOutboundCapacity()
    let chunkLength = min(MaxDataPayloadBytes, data.len - offset)
    # Chunk reservation and frame submission continue below.
```

To avoid polling while capacity is unavailable, `waitForOutboundCapacity` suspends the application writer on `sendStateChanged`. The `waitForOutboundCapacity` procedure is defined in `streams.nim` and repeatedly evaluates `canReserveOutbound`, the stream's predicate for whether another chunk may be allocated:

```nim
proc waitForOutboundCapacity*(
    stream: TransportStream
): Future[void] {.async: (raises: [CancelledError, LPStreamError]).} =
  while true:
    if stream.closed:
      raise newLPStreamClosedError()
    if stream.nextOutboundSequence > MaxDataSequenceNumber:
      raise newException(LPStreamError, "stream sequence space is exhausted")
    if stream.canReserveOutbound:
      return
    stream.sendStateChanged.clear()
    await stream.sendStateChanged.wait()
```

Chronos schedules these tasks cooperatively. Clearing `sendStateChanged` and registering the wait happen without an intervening `await`, so another task cannot change capacity between those operations. After `sendStateChanged` wakes the writer, the loop checks closure, sequence exhaustion and capacity again. Checking `stream.closed` before capacity prevents a closed stream from returning successfully merely because its outbound table still has room. Reporting sequence exhaustion before waiting prevents a writer from sleeping forever after the final sequence has been allocated.

The predicate used by that loop combines two limits:

```nim
func canReserveOutbound*(stream: TransportStream): bool =
  stream.pendingOutbound.len < MaxInflightChunks and
    stream.nextOutboundSequence <= MaxDataSequenceNumber and
    stream.nextOutboundSequence < stream.remoteReceiveLimit
```

`MaxInflightChunks` is currently `64`. It bounds the sent payloads retained while awaiting acknowledgement. The remote receive limit is derived from the latest base reported by the other endpoint:

```nim
func remoteReceiveLimit*(stream: TransportStream): SequenceNumber =
  let window = SequenceNumber(ReceiveWindowChunks)
  if stream.remoteReceiveBase > SequenceNumber.high - window:
    SequenceNumber.high
  else:
    stream.remoteReceiveBase + window
```

If the remote base is `20`, its 256-position window admits sequences `20` through `275`; `276` is the exclusive limit. The sender can stop because it already has 64 unacknowledged chunks or because its next sequence falls outside that remote window.

## 3. Every Data Frame Uses One Payload Bound

`wire.nim` names the two numeric domains instead of spreading primitive integer types through the transport:

```nim
type
  StreamId* = uint32
  SequenceNumber* = uint32

const MaxDataSequenceNumber* = SequenceNumber.high - 1
```

The frame encodes stream IDs, Data sequence numbers and ACK receive bases as Protobuf `fixed32` fields:

```nim
streamId* {.fieldNumber: 4, fixed.}: Opt[StreamId]
sequence* {.fieldNumber: 5, fixed.}: Opt[SequenceNumber]
receiveBase* {.fieldNumber: 8, fixed.}: Opt[SequenceNumber]
```

Each fixed numeric field occupies four value bytes regardless of the current identifier or sequence. Consequently, a stream that has just opened and a stream approaching sequence exhaustion use the same Data payload bound. The transport can calculate one payload bound for every Data frame; it does not need to encode candidate frames repeatedly to determine how many application bytes fit at a particular stream or sequence number.

The remaining variable-size Data field is `sessionId`. `connect` generates session pseudonyms with `PeerId.random`, which currently produces a 39-byte binary Peer ID. `MaxSessionIdBytes` records that wire constraint, and both session registration and frame validation reject a longer identifier.

`MaxDataFrameOverheadBytes` accounts for the Protobuf tags and encoded values of `version`, the maximum-size `sessionId`, `kind`, `streamId`, `sequence` and the payload length prefix. `MaxDataPayloadBytes` subtracts that overhead from the Sphinx payload space available after Mix service framing:

```nim
const
  MaxSessionIdBytes* = 39

  # Every Data-frame field number fits in a one-byte Protobuf field tag.
  ProtobufFieldTagBytes = 1
  SingleByteVarintBytes = 1
  MaxDataPayloadLengthPrefixBytes = 2

  VersionFieldBytes = ProtobufFieldTagBytes + SingleByteVarintBytes
  SessionIdFieldBytes =
    ProtobufFieldTagBytes + SingleByteVarintBytes + MaxSessionIdBytes
  FrameKindFieldBytes = ProtobufFieldTagBytes + SingleByteVarintBytes
  StreamIdFieldBytes = ProtobufFieldTagBytes + sizeof(StreamId)
  SequenceFieldBytes = ProtobufFieldTagBytes + sizeof(SequenceNumber)
  DataPayloadHeaderBytes =
    ProtobufFieldTagBytes + MaxDataPayloadLengthPrefixBytes

  MaxDataFrameOverheadBytes =
    VersionFieldBytes + SessionIdFieldBytes + FrameKindFieldBytes +
    StreamIdFieldBytes + SequenceFieldBytes + DataPayloadHeaderBytes

let MaxDataPayloadBytes* = MaxTransportFrameBytes - MaxDataFrameOverheadBytes
```

The expression uses `sizeof(StreamId)` and `sizeof(SequenceNumber)` rather than hard-coded numeric widths. Changing either alias and retaining fixed-width Protobuf encoding therefore updates the calculated payload bound automatically. The frame validator independently rejects a larger payload, and `encode` retains the final complete-frame size check. The wire-format test encodes `MaxDataPayloadBytes` with both the smallest identifiers and the largest valid identifiers, verifies that both frames exactly reach `MaxTransportFrameBytes`, and verifies that one additional payload byte is rejected. This test makes a future frame-layout change fail visibly if the remaining overhead expression is not updated.

## 4. A Sequence Is Assigned and the Chunk Is Retained

Before sending a chunk, `writeStream` must assign the chunk a sequence number and retain its payload until an ACK confirms receipt. The implementation calls this combined operation a reservation. A successful reservation increments `nextOutboundSequence` and adds the payload to `pendingOutbound` under the assigned sequence number.

`writeStream` copies the next slice, calls `reserveOutbound` to perform that operation, and builds the wire frame:

```nim
proc writeStream(
    self: MixTransport,
    session: TransportSession,
    stream: TransportStream,
    data: sink seq[byte],
): Future[void] {.async: (raises: [CancelledError, LPStreamError]).} =
  # Establishment check and capacity wait precede this excerpt.
  let chunkLength = min(MaxDataPayloadBytes, data.len - offset)
  var chunk = data[offset ..< offset + chunkLength]
  let reservedSequence = stream.reserveOutbound(chunk).valueOr:
    raise newException(LPStreamError, error)

  let frame = MixTransportFrame(
    version: MixTransportVersion,
    sessionId: session.sessionId,
    kind: FrameKind.Data,
    streamId: Opt.some(stream.streamId),
    sequence: Opt.some(reservedSequence),
    payload: Opt.some(move(chunk)),
  )
```

The `reserveOutbound` implementation first repeats the capacity check, then increments the sequence and retains the payload. The new chunk has no retransmission deadline because its initial submission has not completed:

```nim
proc reserveOutbound*(
    stream: TransportStream, payload: seq[byte]
): Result[SequenceNumber, string] =
  if stream.nextOutboundSequence > MaxDataSequenceNumber:
    return err("stream sequence space is exhausted")
  if not stream.canReserveOutbound:
    return err("stream has no outbound capacity")
  let sequence = stream.nextOutboundSequence
  inc stream.nextOutboundSequence
  stream.pendingOutbound[sequence] =
    OutboundChunk(payload: payload, nextRetransmissionAt: Opt.none(Moment))
  ok(sequence)
```

The frame receives `move(chunk)`, while `pendingOutbound` retains its own payload for possible retransmission. An ACK removes the retained `OutboundChunk`; immediate submission failure calls `cancelOutbound`.

The wire validator requires exactly the fields appropriate for `Data`:

```nim
proc validateFrame(
    frame: MixTransportFrame, requireValidSurbEncoding: bool
): Result[void, string] =
  # Validation shared by all frame kinds precedes these Data checks.
  require frame.sequence.isSome == (frame.kind == FrameKind.Data),
    "sequence does not match the frame kind"
  require frame.payload.isSome == (frame.kind == FrameKind.Data),
    "payload does not match the frame kind"
  # Validation for the remaining frame-specific fields follows.
```

After `sendStreamFrame` accepts the frame for Mix submission, `writeStream` schedules the first retransmission deadline when Data retransmission is enabled:

```nim
(await self.sendStreamFrame(session, frame)).isOkOr:
  stream.cancelOutbound(reservedSequence)
  raise newException(LPStreamError, error)
if self.dataRetransmissionsEnabled:
  stream.scheduleOutboundRetransmission(
    reservedSequence, self.dataRetransmissionTimeout
  )
```

Submission does not mean that the remote endpoint received the Data. The chunk remains in `pendingOutbound` until an ACK removes it. If an ACK arrives before `sendStreamFrame` returns, the ACK removes the chunk first and `scheduleOutboundRetransmission` finds no matching entry, so the completed initial submission does not recreate acknowledged state.

## 5. Session Role Selects the Delivery Path

`sendStreamFrame` is the common submission procedure for an already constructed `MixTransportFrame`. The procedure is not specific to Data frames. `writeStream` passes the `Data` frame constructed in Section 4 to `sendStreamFrame`. The `runAcknowledgements` task described in Section 10 uses the same procedure for each `Ack` frame. Centralizing this decision ensures that Data and ACK traffic follow the same delivery rule for a given session role.

`sendStreamFrame` first encodes the complete transport frame. The local variable `payload` therefore contains an encoded `MixTransportFrame`; `payload` does not necessarily contain application Data. The session role then selects how that encoded frame reaches the other endpoint:

```nim
proc sendStreamFrame(
    self: MixTransport, session: TransportSession, frame: MixTransportFrame
): Future[Result[void, string]] {.async: (raises: [CancelledError]).} =
  let payload = frame.encode().valueOr:
    return err("could not encode " & $frame.kind & " frame: " & error)

  case session.role
  of SessionRole.Initiator:
    let destination = session.destination.valueOr:
      return err("initiator session has no destination")
    (
      await self.mix.send(
        MixDestination.exitNode(destination), MixTransportCodec, payload
      )
    ).isOkOr:
      return err("could not send " & $frame.kind & " frame: " & error)
  of SessionRole.Recipient:
    await session.acquireReplySend()
    defer:
      session.releaseReplySend()
    (await self.ensureUnreservedSurbGroup(session)).isOkOr:
      return err(error)
    var replyGroup = session.takeUnreservedSurbGroup().valueOr:
      return err(error)
    (await self.sendWithSurbGroup(replyGroup, payload)).isOkOr:
      return err("could not send " & $frame.kind & " frame: " & error)
    (await self.requestRefill(session)).isOkOr:
      return err(error)
  ok()
```

When the local endpoint is the session initiator, the initiator knows the destination and submits the encoded frame through the forward Mix path. This path carries both Data written by the initiator and ACKs produced after the initiator receives reverse Data from the recipient.

When the local endpoint is the session recipient, the recipient does not have a forward destination for the anonymous initiator. The recipient submits the encoded frame through a SURB group supplied by the initiator. This path carries both response Data written by the recipient's application handler and ACKs produced after the recipient receives forward Data from the initiator.

`acquireReplySend` serializes all return sends belonging to the same session. The session has one shared supply of received SURB groups, so an application response and an ACK task must not concurrently inspect and consume that supply. The recipient preserves a control reserve, waits for an unreserved group before sending ordinary traffic, and checks whether another refill should be requested after consuming the selected group.

The definitions of an unreserved group, the existing pull-based refill sequence, and the intended hybrid replenishment mechanism are documented in [[Mix Transport SURB Replenishment Strategy]]. This walkthrough relies on their resulting contract: one reverse transport frame consumes one session-owned redundancy group, and Data or ACK transmission waits rather than consuming capacity protected for replenishment control.

After the session recipient submits the same encoded frame through every SURB in the selected group, the explanation moves to the session initiator that receives those redundant replies. The initiator's reply credential store contains the private credentials corresponding to that SURB group. When the initiator successfully recovers the first reply, the store consumes the complete credential group because every SURB in the group carries a redundant copy of the same transport frame. If another copy arrives later, its retired SURB identifier tells the initiator that the group has already produced a valid reply, so the later copy does not enter the transport state machine again.

## 6. Both Receive Paths Converge on `handleData`

Forward frames arrive through the registered Mix service handler and `handleDelivery`. SURB replies are recovered by `handleRawSurbReply`, decoded, checked against the credential group's session ID and passed to `handleReplyFrame`.

Both paths converge on `handleData`:

```nim
proc handleData(self: MixTransport, frame: MixTransportFrame) {.gcsafe, raises: [].} =
  let session = self.sessions.get(frame.sessionId).valueOr:
    return
  if session.state != SessionState.Established:
    return
  let stream = session.getStream(frame.streamId.get()).valueOr:
    return
  if stream.state != StreamState.Established:
    return
  discard stream.receiveData(frame.sequence.get(), frame.payload.get())
```

The `(sessionId, streamId)` pair selects the connection. Data for an unknown or non-established session or stream is ignored and cannot enter an unrelated connection.

## 7. `receiveData` Updates the Fixed Window

The receive window contains `256` positions, so the bitmap is exactly `32` bytes:

```nim
const
  ReceiveWindowChunks* = 256
  AckBitmapBytes* = ReceiveWindowChunks div 8
  MaxInflightChunks* = 64
```

Bit `i` represents `receiveBase + i`. With `receiveBase = 10`, bit zero represents sequence 10 and bit 255 represents sequence 265.

`handleData` passes the sequence number and payload to `receiveData`. The complete `receiveData` procedure rejects sequences outside the window, recognizes duplicates and retains every newly accepted payload:

```nim
proc receiveData*(
    stream: TransportStream, sequence: SequenceNumber, payload: sink seq[byte]
): InboundDataDisposition =
  if sequence > MaxDataSequenceNumber:
    return InboundDataDisposition.OutsideWindow
  if sequence < stream.receiveBase:
    stream.fireShouldSendAckEvent()
    return InboundDataDisposition.Duplicate

  let offset = sequence - stream.receiveBase
  if offset >= SequenceNumber(ReceiveWindowChunks):
    return InboundDataDisposition.OutsideWindow
  if stream.acknowledgementBitmap.bitmapContains(offset):
    stream.fireShouldSendAckEvent()
    return InboundDataDisposition.Duplicate

  stream.pendingInbound[sequence] = move(payload)
  stream.acknowledgementBitmap.setBitmapBit(offset)
  stream.dataAvailable.fire()
  stream.fireShouldSendAckEvent()
  InboundDataDisposition.Accepted
```

A sequence below the base was already delivered in order. A set bit identifies a duplicate still inside the window. In both duplicate cases, the sender may be retrying because the previous ACK was lost, so the receiver signals `shouldSendAck` and sends its current snapshot again.

An above-window sequence cannot be produced by a compliant sender because the sender limits new sequences using the latest `remoteReceiveBase` reported by the receiver. The receiver classifies such a sequence as `OutsideWindow`, discards its payload and does not signal `shouldSendAck`. This prevents an invalid frame from causing unnecessary ACK traffic. If the receiver is the session recipient, sending that ACK would consume a SURB group. If the receiver is the session initiator, sending that ACK would consume a forward Mix delivery.

An accepted payload is inserted into `pendingInbound` and its acknowledgement-bitmap bit is set. If the chunk at `receiveBase` has not arrived, the receiver may still retain chunks with higher sequence numbers that arrived first. Those out-of-order chunks remain in `pendingInbound` until the missing earlier chunk arrives and ordered delivery can continue. The receiver accepts an out-of-order chunk only when its sequence number is lower than `receiveBase + ReceiveWindowChunks`, so every retained chunk remains inside the fixed receive window.

## 8. Ordered Delivery Feeds `BufferStream`

The preceding section explained how `receiveData` accepts an inbound Data frame, stores its payload in `pendingInbound` and fires the stream's `dataAvailable` event. Storing the payload does not immediately expose it to the application because an earlier sequence may still be missing.

The `configureStream` procedure shown before Section 1 starts one `runInboundDelivery` task for every established stream. This task moves contiguous payloads from `pendingInbound` into the inherited `BufferStream` in sequence order. The same `configureStream` call also starts the ACK task described in Section 10.

`runInboundDelivery` starts as a background task and repeatedly calls `takeNextInbound`. The inner loop stops when `takeNextInbound` reports that the sequence currently required for ordered delivery is unavailable. The task then waits on `dataAvailable`. Accepting another inbound payload, or advancing `receiveBase` to a sequence that is already present, fires `dataAvailable` and wakes the task for another delivery attempt:

```nim
proc runInboundDelivery(stream: TransportStream) {.async: (raises: [CancelledError]).} =
  while not stream.closed:
    stream.clearInboundDataAvailable()
    while true:
      var inbound = stream.takeNextInbound().valueOr:
        break
      try:
        await stream.pushData(move(inbound.payload))
      except LPStreamError:
        return
      stream.advanceReceiveWindow(inbound.sequence)
    await stream.waitForInboundData()
```

`waitForInboundData` is a raw async wrapper around `AsyncEvent.wait`. The wrapper returns the event's future directly instead of creating another async future and continuation:

```nim
proc waitForInboundData*(
    stream: TransportStream
): Future[void] {.async: (raw: true, raises: [CancelledError]).} =
  stream.dataAvailable.wait()
```

Returning the original future also preserves the intended cancellation path. Cancelling `runInboundDelivery` cancels the active `AsyncEvent.wait`, whose cancellation callback removes that future from `dataAvailable`'s waiter list. Using `wait().join()` here would create an observer future whose cancellation deliberately does not cancel the underlying event wait.

`takeNextInbound` uses `receiveBase` as the sequence that must be delivered next. The procedure looks up that exact sequence in `pendingInbound`. If no payload is stored under `receiveBase`, a higher sequence may still be present in the table, but the procedure returns `none` because delivering that higher sequence would create a gap in the application byte stream:

```nim
proc takeNextInbound*(
    stream: TransportStream
): Opt[tuple[sequence: SequenceNumber, payload: seq[byte]]] =
  stream.pendingInbound.withValue(stream.receiveBase, payload):
    let value = (sequence: stream.receiveBase, payload: move(payload[]))
    stream.pendingInbound.del(stream.receiveBase)
    return Opt.some(value)
  Opt.none(tuple[sequence: SequenceNumber, payload: seq[byte]])
```

When the table contains `receiveBase`, `takeNextInbound` removes that entry and returns both the sequence and the payload to `runInboundDelivery`. Ownership of the payload therefore moves from `pendingInbound` to the local `inbound` value held by the delivery task. `runInboundDelivery` passes that payload to `BufferStream.pushData` and calls `advanceReceiveWindow` only after `pushData` succeeds.

The acknowledgement bitmap is not consulted when selecting the next payload, but its bit for `receiveBase` remains set while `pushData` is suspended. During that interval, `pendingInbound` no longer contains the payload for `receiveBase`; the delivery task owns the payload instead. If another Data frame carrying the same sequence arrives, `receiveData` sees the set bit and classifies the frame as a duplicate rather than inserting another payload into `pendingInbound`. After `pushData` succeeds, `advanceReceiveWindow` shifts the bitmap and advances `receiveBase`.

The precise invariant is therefore not that every set bit has a matching `pendingInbound` entry. A set bit means that the corresponding sequence has been accepted and has not yet advanced below `receiveBase`. The accepted payload is normally stored in `pendingInbound`; for the one sequence currently being passed to `BufferStream`, the delivery task temporarily owns the payload. This bitmap state removes the need for a separate field that identifies the sequence currently being delivered.

If sequence `2` arrives before `1`, `pendingInbound` contains sequence `2` but does not contain the current `receiveBase`, sequence `1`. `takeNextInbound` therefore returns `none`. When sequence `1` arrives, the task pushes sequence `1`, advances `receiveBase` and can then push sequence `2`.

The window advances only after `pushData` succeeds:

```nim
proc advanceReceiveWindow*(stream: TransportStream, sequence: SequenceNumber) =
  doAssert sequence == stream.receiveBase
  doAssert stream.acknowledgementBitmap.bitmapContains(0)
  stream.shiftBitmap()
  inc stream.receiveBase
  stream.fireShouldSendAckEvent()
  if stream.acknowledgementBitmap.bitmapContains(0):
    stream.dataAvailable.fire()
```

Shifting makes the former bit one the new bit zero. Incrementing `receiveBase` extends the far edge of the receive window by one sequence.

## 9. Application Backpressure Reaches the Sender

libp2p's `BufferStream.pushData` uses a capacity-one asynchronous queue and waits when that queue is occupied. If the mounted protocol stops reading, `runInboundDelivery` eventually blocks in `pushData` and does not advance `receiveBase`.

```text
application stops reading
        ↓
BufferStream queue fills
        ↓
runInboundDelivery waits in pushData
        ↓
receiveBase stops advancing
        ↓
ACKs stop extending the remote receive limit
        ↓
sender reaches the limit and waitForOutboundCapacity blocks
```

Successful `pushData` means that the payload entered `BufferStream`, where it is available for the application to read. The application may not have read those bytes yet. The receiver can hold payloads in three places: out-of-order chunks in `pendingInbound`, the one chunk currently being passed from the delivery task to `BufferStream`, and the capacity-one queue inside `BufferStream`. The fixed receive window limits the number of chunks retained in `pendingInbound`, while the delivery task and `BufferStream` each add only a fixed amount of additional buffering. A slow application therefore cannot cause the receiver to retain an unlimited number of chunks.

After `pushData` succeeds, the receiver advances `receiveBase` and reports the new base in an ACK. Advancing the base permits the sender to allocate another sequence number, even though the application may not yet have read the payload from `BufferStream`. If the transport instead permitted another chunk only after the application completed a read, the transport would have to observe or wrap the `BufferStream` read operations. The current implementation does not provide that stricter form of flow control.

## 10. The Receiver Produces an Absolute ACK

The receiver must inform the sender when the receive state changes. Accepting a new Data chunk changes the acknowledgement bitmap. Advancing `receiveBase` reports additional space in the receive window. Receiving duplicate Data also requires another ACK because the duplicate may mean that an earlier ACK was lost.

These state changes do not send an ACK directly. They call `fireShouldSendAckEvent`, which signals the stream's `shouldSendAck` event:

```nim
proc fireShouldSendAckEvent(stream: TransportStream) =
  stream.shouldSendAck.fire()
```

`configureStream` starts one `runAcknowledgements` task for each established stream. `shouldSendAck` is a manual-reset Chronos `AsyncEvent`. `runAcknowledgements` waits for the event, verifies that the stream is still open, clears the event and calls `acknowledgementSnapshot`. The task then creates an `Ack` frame from that snapshot and submits the frame through `sendStreamFrame`:

```nim
proc runAcknowledgements(
    self: MixTransport, session: TransportSession, stream: TransportStream
) {.async: (raises: [CancelledError]).} =
  while not stream.closed:
    await stream.waitForShouldSendAck()
    if stream.closed:
      return
    stream.clearShouldSendAck()
    let snapshot = stream.acknowledgementSnapshot()

    let frame = MixTransportFrame(
      version: MixTransportVersion,
      sessionId: session.sessionId,
      kind: FrameKind.Ack,
      streamId: Opt.some(stream.streamId),
      receiveBase: Opt.some(snapshot.receiveBase),
      acknowledgementBitmap: Opt.some(snapshot.acknowledgementBitmap),
    )
    if (await self.sendStreamFrame(session, frame)).isErr:
      return
```

`waitForShouldSendAck` follows the same raw-wrapper pattern as `waitForInboundData`:

```nim
proc waitForShouldSendAck*(
    stream: TransportStream
): Future[void] {.async: (raw: true, raises: [CancelledError]).} =
  stream.shouldSendAck.wait()
```

`acknowledgementSnapshot` is the operation called immediately before the frame is constructed. The operation copies the current bitmap and pairs the copy with the current `receiveBase`:

```nim
proc acknowledgementSnapshot*(stream: TransportStream): AckSnapshot =
  var bitmap = newSeq[byte](AckBitmapBytes)
  for index, value in stream.acknowledgementBitmap:
    bitmap[index] = value
  AckSnapshot(receiveBase: stream.receiveBase, acknowledgementBitmap: move(bitmap))
```

The copy gives the asynchronous send operation a stable representation of the receiver's state at that moment. If another Data frame changes `stream.acknowledgementBitmap` while `sendStreamFrame` is awaiting the Mix send operation, that change does not modify the bitmap already stored in the outgoing `Ack` frame.

Several calls to `fireShouldSendAckEvent` that occur before the ACK task runs are coalesced into the event's single signalled state. The snapshot still includes every Data chunk accepted into the receive window and every `receiveBase` advancement completed before `acknowledgementSnapshot` is called. If `receiveData` or `advanceReceiveWindow` signals `shouldSendAck` while `sendStreamFrame` is awaiting completion, the event remains signalled and causes the next loop iteration to send a newer snapshot.

The ACK is absolute. `receiveBase` declares that every lower sequence entered the ordered `BufferStream`. Set bitmap bits identify additional chunks that the receiver has accepted but cannot yet deliver because an earlier sequence is missing. One ACK can therefore report the acceptance of several chunks together with every advancement already reflected in the current `receiveBase`.

ACKs are currently sent immediately. There is no delayed-ACK timer, and a send error ends the ACK task rather than scheduling a retry.

## 11. The Sender Applies the ACK

This section moves from the endpoint that generated the ACK to the endpoint that previously sent the acknowledged Data. That endpoint is called the _sender_ in this section. When the session initiator sends forward Data, the recipient's ACK returns through a SURB. When the session recipient sends reverse Data through a SURB, the initiator's ACK arrives through the forward Mix path. Both arrival paths decode the same `Ack` frame and dispatch it to `handleAcknowledgement`.

`handleAcknowledgement` uses the frame's `sessionId` and `streamId` to find the outbound state to which the ACK applies. An ACK for an unknown session, a session that has not been established, an unknown stream or a stream that has not been established is ignored. For an established session and stream, the procedure passes the advertised `receiveBase` and bitmap to `applyAcknowledgement`:

```nim
proc handleAcknowledgement(
    self: MixTransport, frame: MixTransportFrame
) {.gcsafe, raises: [].} =
  let session = self.sessions.get(frame.sessionId).valueOr:
    return
  if session.state != SessionState.Established:
    return
  let stream = session.getStream(frame.streamId.get()).valueOr:
    return
  if stream.state != StreamState.Established:
    return
  discard stream.applyAcknowledgement(
    frame.receiveBase.get(), frame.acknowledgementBitmap.get()
  )
```

The sender retains every unacknowledged Data chunk in `stream.pendingOutbound`, indexed by its sequence number. The current implementation uses the number of retained entries to enforce `MaxInflightChunks`. Each entry retains both the payload needed for retransmission and its optional next retransmission deadline.

`applyAcknowledgement` first validates the received snapshot. The bitmap must have exactly `AckBitmapBytes` bytes. `receiveBase` must not be lower than `stream.remoteReceiveBase`, because accepting an older base would move the sender's view of the receive window backwards. `receiveBase` must not be higher than `stream.nextOutboundSequence`, because the receiver cannot have advanced beyond every sequence the sender has allocated:

```nim
proc applyAcknowledgement*(
    stream: TransportStream, receiveBase: SequenceNumber, bitmap: openArray[byte]
): bool =
  if bitmap.len != AckBitmapBytes or receiveBase < stream.remoteReceiveBase or
      receiveBase > stream.nextOutboundSequence:
    return false

  var acknowledged: seq[SequenceNumber]
  for sequence in stream.pendingOutbound.keys:
    if sequence < receiveBase:
      acknowledged.add(sequence)
    else:
      let offset = sequence - receiveBase
      if offset < SequenceNumber(ReceiveWindowChunks) and bitmap.bitmapContains(offset):
        acknowledged.add(sequence)

  for sequence in acknowledged:
    stream.pendingOutbound.del(sequence)

  let changed = acknowledged.len > 0 or stream.remoteReceiveBase != receiveBase
  stream.remoteReceiveBase = receiveBase
  if changed:
    stream.sendStateChanged.fire()
    stream.retransmissionStateChanged.fire()
  changed
```

Every sequence below `receiveBase` is cumulatively acknowledged, so its retained payload can be removed. For a retained sequence at or above `receiveBase`, the procedure calculates the sequence's offset inside the advertised receive window. A set bit at that offset confirms that the receiver has accepted that specific sequence even if an earlier sequence is still missing.

After removing the acknowledged chunks, `applyAcknowledgement` records the new `remoteReceiveBase`. The procedure fires `sendStateChanged` when at least one chunk was removed or the base advanced. Removing a chunk can bring `pendingOutbound.len` below `MaxInflightChunks`. Advancing `remoteReceiveBase` moves the upper boundary of the remote receive window. Either change can make `canReserveOutbound` true and wake a writer waiting in `waitForOutboundCapacity`. The procedure also fires `retransmissionStateChanged` so that the retransmission task recalculates its earliest deadline after acknowledged entries have disappeared.

`applyAcknowledgement` returns `false` for an invalid snapshot and for a valid snapshot that does not change the sender's state. `handleAcknowledgement` currently ignores that return value because both cases require no further action.

## 12. Unacknowledged Data Is Retransmitted

`newMixTransport` enables Data retransmission by default and uses a fixed 30-second timeout. A caller can disable this behavior explicitly without changing ACK processing or flow control:

```nim
proc newMixTransport*(
    mix: MixProtocol,
    connectTimeout = DefaultConnectTimeout,
    streamOpenTimeout = DefaultStreamOpenTimeout,
    refillRequestTimeout = DefaultRefillRequestTimeout,
    dataRetransmissionTimeout = DefaultDataRetransmissionTimeout,
    enableDataRetransmissions = true,
    refillResponseLifetime = DefaultRefillResponseLifetime,
    maxOutstandingRefillRequests = DefaultMaxOutstandingRefillRequests,
): MixTransport
```

When `enableDataRetransmissions` is false, `configureStream` does not start `runRetransmissions`, and `writeStream` does not assign retransmission deadlines. The sender still retains each outbound chunk until it is acknowledged because `pendingOutbound` also enforces the in-flight bound and prevents the sender from outrunning the receiver's window.

When retransmission is enabled, `runRetransmissions` belongs to the same `TransportStream` as the ACK and ordered-delivery tasks. The task first clears its notification event and asks `takeDueOutboundRetransmission` for the due chunk with the earliest deadline:

```nim
proc runRetransmissions(
    self: MixTransport, session: TransportSession, stream: TransportStream
) {.async: (raises: [CancelledError]).} =
  while not stream.closed:
    stream.clearRetransmissionStateChanged()

    var retransmission = stream.takeDueOutboundRetransmission().valueOr:
      let deadline = stream.earliestRetransmissionDeadline().valueOr:
        await stream.waitForRetransmissionStateChange()
        continue
      let waitTime = deadline - Moment.now()
      if waitTime > ZeroDuration:
        discard await stream.waitForRetransmissionStateChange().withTimeout(waitTime)
      continue
```

If no retained chunk has a deadline, the task waits until initial Data submission schedules one, an ACK changes the table, or stream closure requests cancellation. If the earliest deadline is in the future, the task waits until either that deadline expires or `retransmissionStateChanged` reports an earlier state change. In both cases the loop inspects `pendingOutbound` again instead of assuming that the state which caused the wake-up is still current.

`takeDueOutboundRetransmission` copies the selected payload for the asynchronous send and clears that chunk's deadline. Clearing the deadline marks that a retransmission is currently being submitted and prevents the loop from selecting the same chunk again concurrently. The task constructs another Data frame with the original stream ID, sequence number and payload, then submits the frame through the same session-role-dependent path used by the initial send:

```nim
let frame = MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.Data,
  streamId: Opt.some(stream.streamId),
  sequence: Opt.some(retransmission.sequence),
  payload: Opt.some(move(retransmission.payload)),
)
let sent = await self.sendStreamFrame(session, frame)
stream.scheduleOutboundRetransmission(
  retransmission.sequence, self.dataRetransmissionTimeout
)
```

The next fixed deadline is scheduled after the submission finishes, whether that submission succeeded or returned an error. A transient local send failure therefore does not silently abandon an unacknowledged chunk. There is currently no retry-count limit: retransmission continues until an ACK removes the chunk or the stream closes.

An ACK can arrive while `sendStreamFrame` is suspended. In that case `applyAcknowledgement` removes the chunk from `pendingOutbound`. The later call to `scheduleOutboundRetransmission` changes an entry only when that entry still exists, so completion of the overlapping retransmission cannot restore an acknowledged chunk. The receiver may observe the already-submitted duplicate; `receiveData` suppresses the duplicate and requests another absolute ACK snapshot.

## 13. Stream Shutdown Cancels and Waits for Its Tasks

An established stream owns an ordered Data-delivery task and an ACK task. An accepted inbound application stream also owns the protocol-handler invocation operating on that stream. `runInboundDelivery` may wait on `dataAvailable`, `runAcknowledgements` may wait on `shouldSendAck`, and an application writer may wait on `sendStateChanged` because the in-flight or remote-window limit has been reached.

Firing the events is sufficient when a task is idle at one of those waits, but it does not stop a task that has moved into another asynchronous operation. For example, `runAcknowledgements` may be inside `sendStreamFrame`, waiting for a session SURB refill. The ACK task is no longer waiting on `shouldSendAck`, so firing that event cannot terminate it. `TransportStream` therefore owns explicit references to its internal `streamTasks` and optional `handlerTask`.

`TransportStream.closeImpl` fires the ordinary state events, requests cancellation of every owned task and then delegates to `BufferStream.closeImpl`:

```nim
method closeImpl*(
    stream: TransportStream
): Future[void] {.async: (raises: [], raw: true).} =
  stream.dataAvailable.fire()
  stream.shouldSendAck.fire()
  stream.sendStateChanged.fire()
  stream.resolved.fire()
  stream.streamTasks.cancelSoon()
  if not stream.handlerTask.isNil:
    stream.handlerTask.cancelSoon()
  procCall BufferStream(stream).closeImpl()
```

The inherited `LPStream.close` operation marks the stream closed before dispatching to `closeImpl`. The fired events allow ordinary waiters to observe that state, while explicit cancellation reaches a task suspended deeper inside Mix or SURB processing. `closeImpl` requests cancellation but does not wait: the protocol handler may be the task currently completing stream cleanup, and a task must not wait for itself.

External teardown uses `TransportStream.shutdown`. The procedure retains a local reference to `handlerTask`, closes the stream and then waits for the internal tasks and handler to finish. The local reference remains valid if handler cleanup clears the field while cancellation is being processed:

```nim
proc shutdown*(stream: TransportStream): Future[void] {.async: (raises: []).} =
  let handlerTask = stream.handlerTask
  await stream.close()
  await stream.cancelAndWaitForStreamTasks()
  if not handlerTask.isNil:
    await noCancel handlerTask.cancelAndWait()
  stream.handlerTask = nil
```

When `runProtocolHandler` finishes naturally, it clears `handlerTask` before closing the stream and waits only for the internal `streamTasks`. The handler therefore never waits for its own future.

Stopping the complete transport follows the ownership hierarchy. After unregistering both Mix handlers, `takeSessions` synchronously detaches every session from the store. Each session synchronously detaches its streams through `takeStreams`; session shutdown then starts every stream shutdown and waits for all of them. No table iterator survives across an `await`, and a handler completing during cancellation cannot mutate the container currently being iterated. After all detached sessions finish, the transport clears reply credentials:

```nim
proc stop*(self: MixTransport): Future[void] {.async: (raises: [CancelledError]).} =
  if not self.started:
    return

  self.mix.unregisterRawSurbReplyHandler()
  self.mix.unregisterMixDeliveryHandler(MixTransportCodec)
  let sessions = self.sessions.takeSessions()
  var shutdownTasks = newSeqOfCap[Future[void].Raising([])](sessions.len)
  for session in sessions:
    shutdownTasks.add(session.shutdown())
  await noCancel shutdownTasks.allFutures()
  self.replyCredentials.clear()
  self.started = false
```

`CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` already have wire-format identifiers, but neither endpoint currently sends or handles those frames through the complete transport path. The implemented shutdown therefore cleans up local tasks and state; it does not yet notify the remote endpoint that a stream or session has ended.

A future lifecycle increment should send a best-effort notification when an established stream or session is closed and a delivery path remains available. A normal close should use `CloseStream` or `Disconnect`; cancellation or failure should use `ResetStream` or `ResetSession`. The remote endpoint can then remove its corresponding state immediately rather than waiting for a traffic or liveness timeout. This notification is an optimization of remote cleanup, not a prerequisite for local teardown: cancellation must continue immediately, and local cleanup must complete even when the notification cannot be sent or is lost.

## The Tests as an Executable Walkthrough

The tests cover the flow at three different boundaries. `tests/test_streams.nim` tests receive-window state without a live Mix network. `tests/test_wire.nim` tests the Protobuf representation accepted by both endpoints. `tests/test_connect.nim` runs the application-visible exchange through five live Mix nodes.

The test named `the acknowledgement bitmap retains and orders received chunks` creates one established inbound stream and calls `receiveData` directly. The test inserts sequence `2` before sequence `1`. Because `pendingInbound` does not yet contain the current `receiveBase`, `takeNextInbound` must return `none`. After sequence `1` arrives, the test verifies that a repeated sequence `2` is classified as a duplicate and that both original payloads are delivered in order:

```nim
test "the acknowledgement bitmap retains and orders received chunks":
  # Session and stream construction omitted from this excerpt.
  check:
    stream.receiveData(2, @[2'u8]) == InboundDataDisposition.Accepted
    stream.takeNextInbound().isNone
    stream.receiveData(1, @[1'u8]) == InboundDataDisposition.Accepted
    stream.receiveData(2, @[2'u8]) == InboundDataDisposition.Duplicate

  var first = stream.takeNextInbound().expect("sequence 1 was not ready")
  stream.advanceReceiveWindow(first.sequence)
  var second = stream.takeNextInbound().expect("sequence 2 was not ready")
  stream.advanceReceiveWindow(second.sequence)

  check:
    stream.receiveBase == 3
    stream.pendingInboundCount == 0
```

The test named `acknowledgement bitmap has the fixed receive-window size` constructs an `Ack` frame, encodes and decodes it, and compares the decoded base and bitmap with the original values. It then replaces the bitmap with a value one byte shorter than `AckBitmapBytes` and verifies that frame validation rejects it:

```nim
test "acknowledgement bitmap has the fixed receive-window size":
  # Valid frame construction and round trip omitted from this excerpt.
  var wrongSize = frame
  wrongSize.acknowledgementBitmap = Opt.some(newSeq[byte](AckBitmapBytes - 1))
  check wrongSize.encode().isErr
```

The component test in `tests/test_connect.nim` starts an initiator-side `MixTransport`, a recipient-side `MixTransport` and the five-node live Mix overlay between them. After the `Connect` and `OpenStream` round trips complete, the initiator writes a length-prefixed request through the standard libp2p connection API:

```nim
await initiatorStream.writeLp(TestRequest)
```

On the recipient, the mounted protocol handler receives the established inbound `TransportStream`. The handler reads the request from that stream and writes a response through the same connection object:

```nim
let request = await stream.readLp(1024)
await requests.put(request)
await stream.writeLp(TestResponse)
```

The response is divided into transport Data frames and sent from the recipient through SURB groups. The initiator reorders those frames, feeds the reconstructed bytes into its `BufferStream`, and completes the pending `readLp`:

```nim
let receivedResponseFuture = initiatorStream.readLp(1024)
if not await receivedResponseFuture.withTimeout(TestOperationTimeout):
  raise newException(LPError, "initiator did not receive stream response")
let receivedResponse = await receivedResponseFuture
```

This exchange exercises both directions explicitly. Initiator Data travels through forward Mix delivery, and the recipient returns its ACK through a SURB group. Recipient Data travels through SURB groups, and the initiator returns its ACK through forward Mix delivery. The same exchange exercises ordered delivery at both endpoints, the recipient's control reserve and a one-frame refill exchange. `TestOperationTimeout` is a test failure guard for operations that never complete; successful synchronization comes from transport handshakes, queue notifications and stream reads rather than fixed sleeps.

## Remaining Reliability Work

The implemented flow bounds memory and carries application bytes in both directions, but it does not yet guarantee recovery from every packet-loss pattern. The following mechanisms remain to be added:

- Data retransmission currently uses one fixed timeout and retries without a retry-count limit. The transport does not estimate RTT, apply exponential backoff or close a stream after a configured number of unsuccessful retries.

- Receiving duplicate Data causes the receiver to send its latest absolute ACK again, which recovers when the Data arrived but the preceding ACK was lost. If submitting an ACK itself returns an error, `runAcknowledgements` currently exits, so later changes to the receive window no longer produce ACKs on that stream.

- The implemented pull-only SURB refill can exhaust the recipient's final return paths after repeated loss. [[Mix Transport SURB Replenishment Strategy]] defines the intended hybrid proactive-supply, bounded-credit, supply-retransmission and starvation-recovery design.

- A sender stops allocating new sequences when `nextOutboundSequence` reaches the remote receive-window limit. If the receiver advanced its window but every ACK carrying the new `receiveBase` was lost, the sender has no persist probe: it does not periodically send a small control frame that prompts the receiver to repeat its current window information.

- Every accepted chunk, duplicate and `receiveBase` advancement currently wakes the ACK task immediately. A delayed-ACK policy could combine state changes that occur within a short interval and reduce Mix-packet and SURB consumption, but no delay timer or threshold policy is implemented.

- `CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` are represented in the wire enum but are not sent and handled by both endpoints. Consequently, local cancellation removes local state, while the remote endpoint learns about closure only through later failures or its own cleanup. A future lifecycle increment should send the applicable notification on a best-effort basis when a delivery path remains, without delaying cancellation or making local teardown depend on its delivery.

These additions affect transport scheduling and lifecycle management. They do not require changing the application-facing `Connection` API, and the retransmission and delayed-ACK work can continue to use the existing sequence numbers, `receiveBase` and fixed acknowledgement bitmap.
