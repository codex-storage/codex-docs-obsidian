---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport - Pluggable Integration Model]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
---

# Mix Transport Implementation Walk Through - Wire Format Foundation

`libp2p_mix_transport/wire.nim` defines the Protobuf envelope exchanged by two MixTransport endpoints. The file is not merely a collection of data types: it is the boundary that rejects malformed combinations of fields before transport state is changed, converts public SURBs through Mix's canonical serialization API and calculates how much application data fits in one Sphinx packet.

The tests are in `tests/test_wire.nim`. Package users obtain these public types through the root `libp2p_mix_transport.nim` facade.

## Service Identity and Sphinx Capacity

```nim
const
  MixTransportCodec* = "/libp2p/mix-transport/1.0.0"
  MixTransportVersion* = 1'u32
```

`MixTransportCodec` is the service name registered with `MixProtocol`. A final-hop Mix delivery carrying this codec is passed to MixTransport rather than interpreted by Mix as a connection protocol. `MixTransportVersion` is inside the opaque service payload and lets the transport reject an incompatible envelope.

```nim
let MaxTransportFrameBytes* = getMaxMessageSizeForCodec(MixTransportCodec, 0).expect(
    "MixTransportCodec framing leaves no room for a transport frame"
  )
```

This calculates the Sphinx payload space left after the Mix service codec. The second argument is zero because MixTransport does not use the embedded legacy SURB envelope. When MixTransport needs public SURBs, it serializes them in its own control frames.

## Frame Kinds and Their Current Status

```nim
FrameKind* {.pure.} = enum
  Connect = 1
  ConnectAck = 2
  OpenStream = 3
  StreamAck = 4
  Data = 5
  Ack = 6
  RefillRequest = 7
  Refill = 8
  CloseStream = 9
  ResetStream = 10
  Disconnect = 11
  ResetSession = 12
  StreamReject = 13
```

The numeric values are explicit wire assignments. Existing values must not change if the enum is reordered, and a removed value must not be reused for another meaning.

The implementation currently handles:

| Frames | Current responsibility |
| --- | --- |
| `Connect`, `ConnectAck` | Establish and confirm a long-lived transport session |
| `OpenStream`, `StreamAck`, `StreamReject` | Open or reject one virtual application stream |
| `Data`, `Ack` | Carry ordered bytes and absolute receive-window state |
| `RefillRequest`, `Refill` | Replenish recipient-owned SURB groups |

`CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` reserve the intended wire values, but their end-to-end state transitions are not implemented yet.

## The Common Envelope

```nim
MixTransportFrame* {.proto2.} = object
  version* {.fieldNumber: 1, required, pint.}: uint32
  sessionId* {.fieldNumber: 2, required, ext.}: PeerId
  kind* {.fieldNumber: 3, required, ext.}: FrameKind
  streamId* {.fieldNumber: 4, pint.}: Opt[uint64]
  sequence* {.fieldNumber: 5, pint.}: Opt[uint64]
  payload* {.fieldNumber: 6.}: Opt[seq[byte]]
  codec* {.fieldNumber: 7.}: Opt[string]
  receiveBase* {.fieldNumber: 8, pint.}: Opt[uint64]
  acknowledgementBitmap* {.fieldNumber: 9.}: Opt[seq[byte]]
  batchId* {.fieldNumber: 10, pint.}: Opt[uint64]
  requestedGroups* {.fieldNumber: 11, pint.}: Opt[uint32]
  partIndex* {.fieldNumber: 12, pint.}: Opt[uint32]
  partCount* {.fieldNumber: 13, pint.}: Opt[uint32]
  surbGroups* {.fieldNumber: 14.}: seq[SurbGroup]
  rejectionReason* {.fieldNumber: 15.}: Opt[string]
```

Every frame carries `version`, `sessionId` and `kind`. `sessionId` is the initiator-generated session pseudonym, not the initiator's authenticated network identity. On the recipient it becomes the `peerId` metadata of incoming virtual connections, but it is never passed to `Switch.connect`, `Switch.dial` or the ordinary peer store.

`Opt` distinguishes an absent field from a field carrying its numeric default. The semantic validator then relates presence to `kind`.

| Field | Frame kind |
| --- | --- |
| `streamId` | `OpenStream`, `StreamAck`, `StreamReject`, `Data`, `Ack`, `CloseStream`, `ResetStream` |
| `sequence`, `payload` | `Data` |
| `codec` | `OpenStream` |
| `receiveBase`, `acknowledgementBitmap` | `Ack` |
| `batchId` | `RefillRequest`, `Refill` |
| `requestedGroups` | `RefillRequest` |
| `partIndex`, `partCount`, `surbGroups` | `Refill` |
| `rejectionReason` | optionally `StreamReject` |
| `surbGroups` | also `Connect` and optionally `OpenStream` |

## Data and Absolute Acknowledgement State

A `Data` frame identifies one transport chunk:

```nim
MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.Data,
  streamId: Opt.some(stream.streamId),
  sequence: Opt.some(reservedSequence),
  payload: Opt.some(move(chunk)),
)
```

An `Ack` frame contains an absolute snapshot of the receiver:

```nim
MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.Ack,
  streamId: Opt.some(stream.streamId),
  receiveBase: Opt.some(snapshot.receiveBase),
  acknowledgementBitmap: Opt.some(snapshot.acknowledgementBitmap),
)
```

The constants tie the bitmap length directly to the receive window:

```nim
const
  ReceiveWindowChunks* = 256
  AckBitmapBytes* = ReceiveWindowChunks div 8
  MaxInflightChunks* = 64

static:
  doAssert ReceiveWindowChunks mod 8 == 0
  doAssert MaxInflightChunks <= ReceiveWindowChunks
```

Every sequence below `receiveBase` has entered the receiver's ordered `BufferStream`. Bitmap bit `i` says whether sequence `receiveBase + i` is retained in the current receive window. The fixed 32-byte representation has the same size for a contiguous prefix, one isolated gap or alternating received and missing chunks.

The sender-side 64-chunk in-flight limit is separate from the 256-position receive window. It bounds retained retransmission state without changing what one ACK can describe.

## Public SURB Groups

```nim
SurbGroup* {.proto2.} = object
  surbs* {.fieldNumber: 1.}: seq[seq[byte]]
```

One `SurbGroup` represents redundant delivery attempts for one logical reply. The recipient sends the same payload through every SURB in the group; the initiator consumes the corresponding credential group after the first successful recovery.

The wire layer treats each SURB as opaque bytes and uses Mix's canonical conversion functions:

```nim
proc init*(T: type SurbGroup, surbs: openArray[SURB]): Result[T, string] =
  require surbs.len > 0, "SURB groups must not be empty"

  var encoded = newSeqOfCap[seq[byte]](surbs.len)
  for surb in surbs:
    encoded.add(surb.serializeSurb())
  ok(T(surbs: encoded))
```

```nim
proc decodeSurbs*(group: SurbGroup): Result[seq[SURB], string] =
  require group.surbs.len > 0, "SURB groups must not be empty"

  var decoded = newSeqOfCap[SURB](group.surbs.len)
  for encoded in group.surbs:
    let surb = encoded.deserializeSurb().valueOr:
      return err("could not deserialize SURB: " & error)
    decoded.add(surb)
  ok(decoded)
```

MixTransport does not reconstruct private SURB fields. Validation checks that each serialized value has `SurbSize`, groups are non-empty and the aggregate count cannot exceed the transport-frame capacity.

## The Current One-Packet Refill Contract

`RefillRequest` carries `batchId` and `requestedGroups`. Validation limits the request:

```nim
of FrameKind.RefillRequest:
  require frame.requestedGroups.get() > 0, "refill must request at least one group"
  require frame.requestedGroups.get() <= MaxRefillGroupsPerFrame,
    "refill requests too many groups"
```

`MaxRefillGroupsPerFrame` is currently two. The recipient never requests more groups than can fit in one Sphinx packet. If it needs more, it sends another request after the current batch completes.

The returned frame still carries `partIndex` and `partCount`, but the current sender always emits one part:

```nim
MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.Refill,
  batchId: frame.batchId,
  partIndex: Opt.some(0'u32),
  partCount: Opt.some(1'u32),
  surbGroups: prepared.encoded,
)
```

The recipient does not accumulate multipart refills. It accepts a `Refill` only when its `batchId` matches the one pending request, then adds all groups from that frame. `partIndex` and `partCount` preserve a possible extension point, but they do not describe a current multipart implementation.

## Stream Rejection Is Diagnostic, Not Enumerated

`StreamReject` carries the rejected `streamId` and may carry a bounded string:

```nim
rejectionReason* {.fieldNumber: 15.}: Opt[string]
```

The recipient knows why protocol lookup or admission failed and truncates its reason to `MaxStreamRejectionReasonBytes`, currently 255. The decoder rejects an overlong value. An absent or empty reason remains a valid rejection; the initiator substitutes `remote rejected the stream for an unknown reason`.

## Semantic Validation

Protobuf checks field encoding, but it cannot express that one field is mandatory for one frame kind and forbidden for every other kind. The local `require` template returns a `Result` error:

```nim
template require(condition: bool, message: string): untyped =
  if not condition:
    return err(message)
```

The compact equality checks both directions:

```nim
require frame.sequence.isSome == (frame.kind == FrameKind.Data),
  "sequence does not match the frame kind"
require frame.receiveBase.isSome == (frame.kind == FrameKind.Ack),
  "receiveBase does not match the frame kind"
require frame.batchId.isSome ==
  (frame.kind in {FrameKind.RefillRequest, FrameKind.Refill}),
  "batchId does not match the frame kind"
```

For example, `Data` without `sequence` is rejected, and `ConnectAck` carrying `sequence` is also rejected. The same pattern covers payload, codec, ACK state, refill metadata and rejection reason.

The bitmap has one exact legal size:

```nim
require frame.acknowledgementBitmap.isNone or
  frame.acknowledgementBitmap.get().len == AckBitmapBytes,
  "acknowledgement bitmap has the wrong size"
```

Kind-specific checks then reject empty Data, an empty Connect SURB supply, an empty OpenStream codec and invalid refill indices.

## Encoding and Decoding

Encoding validates before Protobuf serialization and checks the final size afterward:

```nim
proc encode*(frame: MixTransportFrame): Result[seq[byte], string] =
  frame.validate().isOkOr:
    return err(error)

  let encoded = Protobuf.encode(frame)
  require encoded.len <= MaxTransportFrameBytes, "transport frame is too large"
  ok(encoded)
```

Decoding rejects oversized input before allocating protocol state, catches `SerializationError`, and runs the same semantic validator:

```nim
let frame =
  try:
    decodeFrame(data)
  except SerializationError as exc:
    return err("could not decode transport frame: " & exc.msg)
frame.validate().isOkOr:
  return err(error)
ok(frame)
```

After `decode` returns `ok`, transport handlers may safely call required accessors such as `frame.streamId.get()` for a stream frame because validation already proved that the field is present.

## Payload-Aware Data Chunking

`dataPayloadCapacity(sessionId, streamId, sequence)` uses the actual identifiers to find the largest encodable Data payload by binary search. This matters because the varint lengths of `streamId` and `sequence` and the binary size of `PeerId` contribute to the Protobuf envelope.

The chunker therefore asks the wire layer for each sequence instead of subtracting a fixed guessed header length. See [[Mix Transport Implementation Walk Through - Bounded Data Flow]] for the full write path.

## Tests

`tests/test_wire.nim` verifies:

- Data preserves session, stream, sequence and payload across a Protobuf round trip.
- ACK preserves `receiveBase` and the fixed bitmap, while a 31-byte bitmap is rejected when 32 bytes are required.
- Refill preserves its batch metadata and grouped public SURBs.
- `SurbGroup.init` and `decodeSurbs` agree with Mix's canonical serialization.
- Fields that contradict `kind` are rejected.
- Stream rejection preserves its diagnostic reason and also permits the defined no-reason fallback case.
- Unsupported versions, oversized frames, malformed Protobuf and incomplete Protobuf are rejected.

The live component test creates cryptographically valid SURBs and exercises Data, ACK and refill through the real five-node Mix path. The wire unit tests stay focused on the serialization and validation boundary.
