
This example assumes that the Mix overlay has already been constructed and started. In particular, each participating node already has a running libp2p `Switch`, a mounted and started `MixProtocol`, and a populated Mix node pool.

The example focuses on the application-facing `MixTransport` API.

## Recipient protocol

The recipient mounts an ordinary libp2p protocol on the same switch used by `MixProtocol`. The protocol handler receives a `TransportStream` through the normal libp2p `Stream` interface.

```nim
import chronos, results
import stew/byteutils
import libp2p/[peerid, protocols/protocol, stream/connection]
import libp2p_mix
import libp2p_mix_transport

const EchoCodec = "/example/mix-echo/1.0.0"

proc newEchoProtocol(): LPProtocol =
  let handler: LPProtoHandler = proc(
      stream: Stream,
      selectedCodec: string,
  ): Future[void] {.async: (raises: [CancelledError]).} =
    try:
      let request = await stream.readLp(1024)
      await stream.writeLp(request)
    except LPStreamError:
      discard

  LPProtocol.new(@[EchoCodec], handler)
```

The handler does not use a Mix-specific read or write API. `TransportStream` implements the ordinary libp2p connection operations, including `read`, `readExactly`, `readLp`, `write`, and `writeLp`.

## Starting the transports

Both endpoints construct `MixTransport` with their local `MixProtocol` instance. The recipient mounts the application protocol before accepting transport streams.

```nim
proc runExample(
    initiatorMix: MixProtocol,
    recipientMix: MixProtocol,
) {.async: (raises: [CancelledError, LPError]).} =
  recipientMix.switch.mount(newEchoProtocol())

  let
    initiatorTransport = newMixTransport(initiatorMix)
    recipientTransport = newMixTransport(recipientMix)

  (await initiatorTransport.start()).expect(
    "could not start initiator MixTransport"
  )
  (await recipientTransport.start()).expect(
    "could not start recipient MixTransport"
  )

  defer:
    await initiatorTransport.stop()
    await recipientTransport.stop()
```

`MixTransport.start` registers the transport delivery handler and the raw SURB reply handler with the injected `MixProtocol`. Starting `MixTransport` does not start the underlying switch or Mix overlay; those components must already be running.

## Opening and using a stream

The initiator identifies the recipient by the peer ID of the recipient's Mix node and requests a stream for `EchoCodec`.

```nim
  let destination = recipientMix.switch.peerInfo.peerId

  let stream = (await initiatorTransport.dial(destination, EchoCodec)).expect(
    "could not open MixTransport stream"
  )

  defer:
    await stream.close()

  await stream.writeLp("Hello through MixTransport")

  let response = await stream.readLp(1024)
  echo string.fromBytes(response)
```

`dial` establishes an anonymous transport session when no session for the destination exists. After establishing the session, `dial` sends an `OpenStream` frame and waits until the recipient accepts or rejects the requested codec.

The recipient's `MixTransport` looks up `EchoCodec` among the protocols mounted on the recipient's switch. When the protocol is available, the recipient creates the corresponding inbound `TransportStream`, returns `StreamAck`, and invokes the mounted protocol handler with that stream.

Application writes can exceed the payload capacity of one Sphinx packet. `MixTransport` divides the byte sequence into Data frames and handles acknowledgements, retransmission state, ordered delivery, backpressure, and SURB refills below the application-facing stream API.

## Complete application-facing flow

The essential initiator flow is:

```nim
let transport = newMixTransport(mixProtocol)
discard (await transport.start()).expect("start failed")

let stream = (await transport.dial(destinationPeerId, ApplicationCodec)).expect(
  "dial failed"
)

await stream.writeLp(message)
let response = await stream.readLp(MaxResponseBytes)
```

An explicit `connect` call is not required before `dial`. `dial` calls `connect` internally. The first call creates a transport session, while later calls for the same destination reuse the established session and open additional streams within that session.
