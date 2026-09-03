---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport - Pluggable Integration Model]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
  - "[[Mix Transport Implementation Walk Through - SURB Replenishment]]"
---
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
  SurbSupply = 14
  SurbStatusProbe = 15
  SurbStatus = 16
```

The numeric values are explicit wire assignments. Existing values must not change if the enum is reordered, and a removed value must not be reused for another meaning.

The implementation currently handles:

| Frames | Current responsibility |
| --- | --- |
| `Connect`, `ConnectAck` | Establish and confirm a long-lived transport session |
| `OpenStream`, `StreamAck`, `StreamReject` | Open or reject one virtual application stream |
| `Data`, `Ack` | Carry ordered bytes and absolute receive-window state |
| `RefillRequest` | Report absolute SURB supply state and request urgent replenishment |
| `SurbSupply` | Carry consecutively numbered individual public SURBs |
| `SurbStatusProbe`, `SurbStatus` | Recover supply-state synchronization when the recipient's ordinary SURB queue can be empty |

`Refill` reserves the value used by the superseded request/response refill protocol. `CloseStream`, `ResetStream`, `Disconnect` and `ResetSession` also reserve wire values, but their end-to-end state transitions are not implemented yet.

## The Common Envelope

```nim
MixTransportFrame* {.proto2.} = object
  version* {.fieldNumber: 1, required, pint.}: uint32
  sessionId* {.fieldNumber: 2, required, ext.}: PeerId
  kind* {.fieldNumber: 3, required, ext.}: FrameKind
  streamId* {.fieldNumber: 4, fixed.}: Opt[StreamId]
  sequence* {.fieldNumber: 5, fixed.}: Opt[SequenceNumber]
  payload* {.fieldNumber: 6.}: Opt[seq[byte]]
  codec* {.fieldNumber: 7.}: Opt[string]
  receiveBase* {.fieldNumber: 8, fixed.}: Opt[SequenceNumber]
  acknowledgementBitmap* {.fieldNumber: 9.}: Opt[seq[byte]]
  firstSurbSequence* {.fieldNumber: 10, fixed.}: Opt[SurbSupplySequence]
  surbSupplyReceiveBase* {.fieldNumber: 11, fixed.}: Opt[SurbSupplySequence]
  surbSupplyAcknowledgementBitmap* {.fieldNumber: 12.}: Opt[seq[byte]]
  surbSupplyLimit* {.fieldNumber: 13, fixed.}: Opt[SurbSupplySequence]
  surbs* {.fieldNumber: 14.}: seq[seq[byte]]
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
| `firstSurbSequence` | `SurbSupply` |
| `surbSupplyReceiveBase`, `surbSupplyAcknowledgementBitmap`, `surbSupplyLimit` | complete snapshots on `ConnectAck`, `StreamAck`, `StreamReject`, reverse `Data`, reverse `Ack`, `RefillRequest` and `SurbStatus` |
| `rejectionReason` | optionally `StreamReject` |
| `surbs` | `Connect`, `OpenStream`, `SurbSupply` and `SurbStatusProbe` |

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

## Individual Public SURBs

The common frame carries every public SURB as a separate repeated byte field:

```nim
surbs* {.fieldNumber: 14.}: seq[seq[byte]]
```

The wire format does not record redundancy membership. The recipient stores every decoded SURB in one session queue and selects several SURBs only when it forms a temporary redundancy batch for one reverse frame. [[Mix Transport SURB Replenishment Strategy]] explains why redundancy is a send-time operation rather than persistent wire state.

The sender converts each public SURB with `serializeSurb`. The receiver applies `deserializeSurb` independently to every repeated value. An invalid serialized SURB can therefore be discarded without rejecting valid SURBs carried by the same transport frame.

MixTransport does not reconstruct private SURB fields manually. Local encoding checks that each value has `SurbSize`, and both encoding and decoding bound the total number of repeated values. Inbound structural validation deliberately leaves individual deserialization to the frame handler so one malformed SURB does not invalidate the remaining supply.

## Numbered SURB Supply and Absolute State

After session establishment, the initiator supplies public SURBs in `SurbSupply` frames. `firstSurbSequence` identifies the first repeated SURB, and each later item has the next consecutive sequence:

```nim
MixTransportFrame(
  version: MixTransportVersion,
  sessionId: session.sessionId,
  kind: FrameKind.SurbSupply,
  firstSurbSequence: Opt.some(firstSequence),
  surbs: encodedSurbs,
)
```

`MaxSurbSupplyPerFrame` is four. This is a packet-size bound rather than a persistent redundancy-group size: the recipient decodes and accepts every repeated SURB independently. The `SurbSupply` validator also checks that the complete consecutive range remains inside the valid supply sequence space.

The recipient reports supply receipt and replacement credit with one complete snapshot:

```nim
surbSupplyReceiveBase* {.fieldNumber: 11, fixed.}: Opt[SurbSupplySequence]
surbSupplyAcknowledgementBitmap* {.fieldNumber: 12.}: Opt[seq[byte]]
surbSupplyLimit* {.fieldNumber: 13, fixed.}: Opt[SurbSupplySequence]
```

`surbSupplyReceiveBase` is the first supply sequence whose receipt is not part of the contiguous acknowledged prefix. Bitmap bit `i` reports receipt of `surbSupplyReceiveBase + i`, allowing later sequences to be acknowledged across a gap. `surbSupplyLimit` is the exclusive absolute upper bound for new sequences the initiator may introduce.

All three snapshot fields must be present together. `ConnectAck`, `RefillRequest` and `SurbStatus` require a snapshot. Recipient-originated `StreamAck`, `StreamReject`, Data and ACK frames also carry a snapshot when the transport sends them through the reverse path. A duplicate `RefillRequest` repeats absolute state and only marks replenishment as urgent; the wire protocol therefore needs neither a request identifier nor a requested quantity.

The supply bitmap is tied to its receive window in the same way as the Data acknowledgement bitmap:

```nim
const
  SurbSupplyWindow* = 256
  SurbSupplyAckBitmapBytes* = SurbSupplyWindow div 8
  MaxSurbSupplyPerFrame* = 4
```

`SurbStatusProbe` carries fresh, unnumbered SURBs dedicated to one status response. `SurbStatus` returns the absolute snapshot through those SURBs. The dedicated response paths allow a recipient whose normal session queue is empty to report newly available credit.

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
require frame.firstSurbSequence.isSome == (frame.kind == FrameKind.SurbSupply),
  "firstSurbSequence does not match the frame kind"
require frame.surbSupplyReceiveBase.isSome == carriesSurbSupplyState and
  frame.surbSupplyAcknowledgementBitmap.isSome == carriesSurbSupplyState and
  frame.surbSupplyLimit.isSome == carriesSurbSupplyState,
  "SURB supply state is incomplete"
```

For example, `Data` without `sequence` is rejected, and `ConnectAck` carrying a Data sequence is also rejected. The same pattern covers payload, codec, Data ACK state, SURB supply state and rejection reason.

The bitmap has one exact legal size:

```nim
require frame.acknowledgementBitmap.isNone or
  frame.acknowledgementBitmap.get().len == AckBitmapBytes,
  "acknowledgement bitmap has the wrong size"
```

Kind-specific checks then reject empty Data, an empty Connect bootstrap supply, an empty OpenStream codec, an empty or oversized numbered supply, an overflowing supply sequence range and a status probe without response SURBs.

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

After `decode` returns `ok`, transport handlers may safely call required accessors such as `frame.streamId.get()` for a stream frame because validation already proved that the field is present. Inbound decoding validates the number of repeated SURBs but deliberately postpones validation of each serialized SURB value to the frame handler. That separation lets `handleSurbSupply` keep valid values when another value in the same frame is malformed.

## Fixed Data Payload Capacity

`StreamId`, Data `SequenceNumber`, the Data ACK receive base and `SurbSupplySequence` use fixed-width Protobuf encoding. The session identifier is bounded to the 39-byte representation generated by `PeerId.random`. The transport can therefore calculate one maximum Data-frame overhead without encoding candidate frames.

The overhead includes the complete fixed-size SURB supply snapshot because recipient-originated Data carries that snapshot. Initiator-originated Data does not need the snapshot, but using the same `MaxDataPayloadBytes` in both directions keeps chunking independent of the sender's session role.

`MaxDataPayloadBytes` subtracts this named overhead from `MaxTransportFrameBytes`. The final encoder still checks the complete serialized length, so the calculated bound and the actual Sphinx capacity are enforced independently. See [[Mix Transport Implementation Walk Through - Bounded Data Flow]] for the complete write path.

## Tests

`tests/test_wire.nim` verifies:

- Data preserves session, stream, sequence and payload across a Protobuf round trip.
- ACK preserves `receiveBase` and the fixed bitmap, while a 31-byte bitmap is rejected when 32 bytes are required.
- Numbered supply preserves its first sequence and individual public SURBs.
- A supply snapshot preserves its fixed receive base, 32-byte bitmap and absolute limit, while an incomplete or incorrectly sized snapshot is rejected.
- Individual SURBs agree with Mix's canonical `serializeSurb` and `deserializeSurb` boundary.
- Fields that contradict `kind` are rejected.
- Stream rejection preserves its diagnostic reason and also permits the defined no-reason fallback case.
- Unsupported versions, oversized frames, malformed Protobuf and incomplete Protobuf are rejected.

The live component test creates cryptographically valid SURBs and exercises proactive supply, recipient-requested supply, Data and ACK through a real five-node Mix path. The wire unit tests stay focused on the serialization and validation boundary.
