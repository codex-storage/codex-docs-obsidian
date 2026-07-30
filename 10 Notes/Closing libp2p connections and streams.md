---
tags:
  - logos-storage/libp2p
  - logos-storage/connections
  - logos-storage/streams
  - libp2p
related:
  - "[[Libp2p connections - connect vs dial]]"
  - "[[Libp2p Connection Lifecycle in Logos Storage]]"
  - "[[New Logos Storage Block Exchange Protocol]]"
  - "[[Uploading and downloading content in new Logos Storage]]"
  - "[[New Logos Storage Download Flow]]"
  - "[[New Logos Storage Discovery]]"
  - "[[Block Exchange Peer Stores]]"
  - "[[Block Exchange Peer Performance and BDP]]"
---

# Closing libp2p connections and streams

> [!info] Source baseline
> This note describes `logos-storage-nim` `master` at commit `81f0d053`
> (2026-07-20), including its vendored nim-libp2p 2.1.4.

> [!note] Reading the excerpts
> Every Nim source excerpt includes the enclosing function, method, procedure,
> or type signature. Comments containing `...` indicate deliberately omitted
> code inside that named definition; they do not indicate an unnamed fragment.

Closing a value called `Connection` does not always close the peer connection.
The effect depends on which layer that object represents.

The short answer is:

```text
close an application stream
  -> close one Yamux channel
  -> preserve Yamux muxer
  -> preserve Noise connection
  -> preserve TCP connection
  -> peer remains Joined/connected

reset an application stream
  -> abort one Yamux channel
  -> preserve muxer and underlying connection
  -> peer remains Joined/connected

close a muxer
  -> close all of its application streams
  -> close its secured transport connection
  -> close its TCP/QUIC connection
  -> one muxer-level Disconnected event
  -> PeerEvent.Left only if this was the peer's last muxer

Switch.disconnect(peer)
  -> remove and close every muxer for that peer
  -> close every stream carried by those muxers
  -> PeerEvent.Left

Switch.stop()
  -> close every peer connection and stop every transport
```

This note expands each path and relates it to the current Logos Storage
block-exchange read loops. The construction side is documented in
[[Libp2p connections - connect vs dial]].

## First identify the layer being closed

The stack for current TCP Logos Storage is:

```text
BlockExcNetwork / Manifest / Ping
                  │
                  │ application stream endpoint (`Connection`)
                  ▼
             Yamux channel
                  │
                  │ one channel among many
                  ▼
             Yamux session (`Muxer`)
                  │
                  ▼
          Noise-secured connection
                  │
                  ▼
             TCP socket
```

One muxer can carry many independent streams:

```text
Muxer to Peer B
├── block-exchange stream
├── manifest stream
├── ping stream
└── another block-exchange stream
```

The same Nim type name is used at several stages:

Source:
`vendor/nim-libp2p/libp2p/stream/connection.nim`

```nim
type
  Connection* = ref object of LPStream
    peerId*: PeerId
    observedAddr*: Opt[MultiAddress]
    localAddr*: Opt[MultiAddress]
    protocol*: string
    transportDir*: Direction

  ## Semantic aliases for stream endpoints at different stages of the stack.
  ## These are aliases, not distinct types.
  Stream* = Connection
  RawConn* = Stream
  MuxedStream* = Stream
```

Therefore, a bare call such as `await conn.close()` is not enough context by
itself. If `conn` is the application stream returned by `Switch.dial`, it
closes one stream. If it is the raw connection owned by a muxer, closing it
tears down the peer-level connection.

## The generic `LPStream.close`

All stream-like objects inherit the once-only close wrapper:

Source:
`vendor/nim-libp2p/libp2p/stream/lpstream.nim`

```nim
method close*(
    s: LPStream
): Future[void] =
  ## close the stream - this may block,
  ## but will not raise exceptions

  if s.isClosed:
    trace "Already closed", s
    return newFutureCompleted[void]()

  s.isClosed = true
  closeImpl(s)
```

The wrapper:

- makes repeated `close` calls idempotent at this level;
- marks the local object closed before invoking the transport/muxer-specific
  implementation;
- deliberately exposes a no-raise closing API;
- does not determine whether the close is graceful, half-close, reset, or
  whole-transport shutdown—the derived implementation does.

The base implementation performs local bookkeeping:

Source:
`vendor/nim-libp2p/libp2p/stream/lpstream.nim`

```nim
method closeImpl*(
    s: LPStream
): Future[void] =
  trace "Closing stream", s,
    objName = s.objName,
    dir = $s.dir

  libp2p_open_streams.dec(
    labelValues = [s.objName, $s.dir]
  )

  untrackCounter(s.objName)
  s.closeEvent.fire()

  trace "Closed stream", s,
    objName = s.objName,
    dir = $s.dir

  newFutureCompleted[void]()
```

The important behavior comes from `YamuxChannel.closeImpl`.

## Gracefully closing a Yamux application stream

The named call sequence for a normal outgoing application-stream close is:

```text
application caller
  -> LPStream.close
     -> dynamic YamuxChannel.closeImpl
        -> write Yamux FIN for this channel
        -> YamuxChannel.actuallyClose
           -> Connection.closeImpl when both halves are closed
              -> LPStream.closeImpl bookkeeping
```

An application stream returned by `Switch.dial` is a `YamuxChannel` in the
current Storage configuration. Its `closeImpl` sends a Yamux `FIN` when
possible:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
method closeImpl*(
    channel: YamuxChannel
) =
  if not channel.closedLocally:
    trace "Closing yamux channel locally",
      streamId = channel.id,
      conn = channel.conn

    channel.closedLocally = true

    if not channel.isReset and
        channel.sendQueue.len == 0:
      try:
        await channel.conn.write(
          YamuxHeader.data(
            channel.id,
            0,
            {Fin}
          )
        )
      except CancelledError, LPStreamError:
        discard

    await channel.actuallyClose()
```

For Yamux, `close` is a write-side half-close:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
method closeWrite*(
    channel: YamuxChannel
) =
  ## For yamux, closeWrite is the same as close -
  ## it implements half-close

  await channel.close()
```

It means:

```text
local side calls stream.close()
  -> local side promises to send no more bytes
  -> FIN is sent for this Yamux channel
  -> remote side may still finish sending its bytes
  -> channel is fully cleaned when both directions are closed
```

The channel-level cleanup waits for:

- the local side to be closed;
- the pending send queue to be empty;
- the remote side to close.

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
proc actuallyClose(
    channel: YamuxChannel
) =
  if channel.closedLocally and
      channel.sendQueue.len == 0 and
      channel.closedRemotely.isSet():
    await procCall Connection(channel).closeImpl()
```

This closes only the Yamux channel. It does not call `Yamux.close`, and it does
not close the Yamux session's underlying Noise/TCP connection.

The resulting state can be:

```text
before:
  Muxer B
  ├── stream 1
  └── stream 2

close stream 1:
  Muxer B
  └── stream 2

close stream 2:
  Muxer B
  └── zero application streams, still reusable
```

Even with zero application streams, the peer normally remains connected in
the local `ConnManager`.

## `closeWithEOF`: close and wait for the remote half

Protocol handlers frequently use `closeWithEOF` rather than plain `close`:

Source:
`vendor/nim-libp2p/libp2p/stream/lpstream.nim`

```nim
proc closeWithEOF*(
    s: LPStream
) =
  ## Close the stream and wait for EOF - use this
  ## with half-closed streams where an EOF is
  ## expected to arrive from the other end.

  if s.closedWithEOF:
    return

  s.closedWithEOF = true
  await s.close()

  if s.isResetLocally or s.atEof():
    return

  try:
    var buf: array[8, byte]
    if (await readOnce(
        s, addr buf[0], buf.len
    )) != 0:
      debug "Unexpected bytes while waiting for EOF", s
  except CancelledError:
    discard
  except LPStreamEOFError as e:
    trace "Expected EOF came",
      s, description = e.msg
  except LPStreamError as exc:
    debug "Unexpected error while waiting for EOF",
      s, description = exc.msg
```

This is appropriate only when the protocol has already reached a state in
which no more application data is expected. The code comment explicitly warns
against using it while another concurrent reader is active.

The semantic sequence is:

```text
send local FIN
  -> stop writing
  -> wait for remote FIN / EOF
  -> complete graceful stream teardown
```

It still closes only that application stream.

### Graceful stream-close recap

The longer code thread has two closely related entry points:

```text
stream.close()
  -> LPStream.close
  -> YamuxChannel.closeImpl
  -> send FIN for the local write half
  -> complete channel cleanup after remote FIN
  -> keep Yamux muxer and TCP connection

stream.closeWithEOF()
  -> LPStream.closeWithEOF
  -> stream.close()
  -> explicitly read until remote EOF
  -> keep Yamux muxer and TCP connection
```

## Incoming protocol streams are closed automatically

The incoming-stream lifetime is controlled by this call sequence:

```text
Yamux.handle receives a new channel
  -> Yamux.handleStream
  -> MuxedUpgrade.streamHandler
  -> MultistreamSelect.handle
  -> mounted LPProtocol handler
  -> handler returns
  -> MuxedUpgrade.streamHandler finally
  -> stream.closeWithEOF
```

For TCP/Yamux, the muxer's stream handler runs multistream-select and then
closes the incoming stream in a `finally` block:

Source:
`vendor/nim-libp2p/libp2p/upgrademngrs/muxedupgrade.nim`

```nim
proc new*(
    T: type MuxedUpgrade,
    muxers: seq[MuxerProvider],
    secureManagers: openArray[Secure] = [],
    ms: MultistreamSelect,
    connManager: Opt[ConnManager] = Opt.none(ConnManager),
): T =
  let upgrader =
    T(
      muxers: muxers,
      secureManagers: @secureManagers,
      ms: ms,
      connManager: connManager,
    )

  upgrader.streamHandler =
    proc(stream: MuxedStream) =
      try:
        upgrader.connManager.withValue(connManager):
          let ready =
            await connManager.waitForPeerReady(stream.peerId)
          if not ready:
            return

        await upgrader.ms.handle(stream)
      except CancelledError:
        return
      finally:
        await stream.closeWithEOF()

  return upgrader
```

`MultistreamSelect.handle` dispatches to the mounted application handler. Once
that handler returns, the wrapper closes the incoming stream gracefully.

This has an important application-design consequence:

> If an incoming protocol handler is meant to keep a long-lived stream open,
> it must remain active for the lifetime of that stream. Returning from the
> handler ends the incoming stream.

The QUIC transport installs an equivalent wrapper:

Source:
`vendor/nim-libp2p/libp2p/transports/quictransport.nim`

```nim
method upgrade*(
    self: QuicTransport, conn: RawConn, peerId: Opt[PeerId]
): Future[Muxer] {.async: (raises: [CancelledError, LPError]).} =
  let muxer = QuicMuxer.new(conn, peerId)

  muxer.streamHandler =
    proc(stream: MuxedStream) =
      try:
        await self.upgrader.ms.handle(stream)
      except CancelledError:
        return
      except CatchableError as exc:
        trace "exception in stream handler",
          stream, msg = exc.msg
      finally:
        await stream.closeWithEOF()

  muxer.handleFut = muxer.handle()
  return muxer
```

## Resetting one application stream

The abortive stream path is:

```text
application caller
  -> LPStream.reset template
  -> resetStream
  -> dynamic YamuxChannel.resetImpl
  -> YamuxChannel.reset(isLocal = true)
     -> fail and clear channel queues
     -> write Yamux RST when possible
     -> close/wake this channel
  -> leave sibling channels and the Yamux muxer intact
```

`reset` is the abortive alternative to graceful close:

Source:
`vendor/nim-libp2p/libp2p/stream/lpstream.nim`

```nim
proc resetStream(
    s: LPStream
): Future[void] =
  ## Abort the stream and best-effort notify
  ## the remote peer, if supported.

  if s.isClosed:
    return newFutureCompleted[void]()

  s.isClosed = true
  s.isResetLocally = true
  resetImpl(s)

template reset*[T: LPStream](s: T): untyped =
  resetStream(s)
```

For Yamux, reset:

- marks the channel reset and closed;
- fails queued writes;
- clears queued received data;
- sends a Yamux `RST` when possible;
- wakes pending readers;
- does not close sibling channels or the muxer.

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
proc reset(
    channel: YamuxChannel,
    isLocal: bool = false
) =
  if channel.isReset:
    return

  channel.isReset = true
  channel.remoteReset = not isLocal
  channel.isClosed = true
  channel.clearQueues(
    newLPStreamEOFError()
  )

  channel.sendWindow = 0

  if not channel.closedLocally:
    if isLocal and not channel.isSending:
      try:
        await channel.conn.write(
          YamuxHeader.data(
            channel.id,
            0,
            {Rst}
          )
        )
      except CancelledError, LPStreamError:
        discard

    await channel.closeImpl()

  if not channel.closedRemotely.isSet():
    await channel.remoteClosed()

  channel.receivedData.fire()
```

Use the distinction:

```text
close / closeWithEOF
  graceful protocol completion

reset
  abort this stream because completion is no longer possible or desired
```

Neither normally means “disconnect the peer.”

## Application-stream idle timeouts

An idle Yamux channel reaches the same reset path through:

```text
Connection.timeoutMonitor
  -> Connection.pollActivity sees no reads or writes during stream.timeout
  -> stream.timeoutHandler
  -> YamuxChannel.reset(isLocal = true)
  -> reset this application channel only
```

The configured Yamux timeouts apply to channels, not to the peer-level muxer:

Source:
`vendor/nim-libp2p/libp2p/builders.nim`

```nim
proc withYamux*(
    b: SwitchBuilder,
    maxChannCount: int = MaxChannelCount,
    windowSize: int = YamuxDefaultWindowSize,
    inTimeout: Duration = 5.minutes,
    outTimeout: Duration = 5.minutes,
): SwitchBuilder =
  proc newMuxer(conn: RawConn): Muxer =
    Yamux.new(
      conn,
      maxChannCount = maxChannCount,
      windowSize = windowSize,
      inTimeout = inTimeout,
      outTimeout = outTimeout,
    )
```

Every newly created channel receives the corresponding timeout:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
proc createStream(
    m: Yamux,
    id: uint32,
    isSrc: bool,
    recvWindow: int,
    maxSendQueueSize: int,
): YamuxChannel =
  var stream = YamuxChannel(
    id: id,
    conn: m.connection,
    # ...window and queue fields...
  )

  if isSrc:
    stream.dir = Direction.Out
    stream.timeout = m.outTimeout
  else:
    stream.dir = Direction.In
    stream.timeout = m.inTimeout

  stream.timeoutHandler =
    proc(): Future[void] =
      trace "Idle timeout expired, resetting YamuxChannel"
      stream.reset(isLocal = true)

  # ...initialize and register stream...
  return stream
```

An inactive application stream is therefore reset after its configured
timeout. This is not a peer heartbeat and does not close an idle muxer that has
zero channels.

Yamux understands and acknowledges Yamux ping frames:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
method handle*(m: Yamux) {.async: (raises: []).} =
  try:
    while not m.connection.atEof:
      let header = await m.connection.readHeader()

      case header.msgType
      of Ping:
        if MsgFlags.Syn in header.flags:
          await m.connection.write(
            YamuxHeader.ping(
              MsgFlags.Ack,
              header.length
            )
          )
      # ...handle GoAway, Data, and WindowUpdate...
  finally:
    await m.close()
```

But this implementation does not schedule a periodic Yamux ping as a
peer-level heartbeat.

## Closing a muxer

The peer-connection-level close sequence for TCP/Yamux is:

```text
caller
  -> Yamux.close
     -> mark every YamuxChannel reset/closed
     -> wake their pending readers and writers
     -> write Yamux GoAway
     -> close the secured m.connection
        -> secure wrapper close
        -> ChronosStream.closeImpl
        -> TCP client.closeWait
  -> ConnManager.onClose observes mux.connection.join
  -> remove this muxer from MuxerStore
  -> emit ConnEvent.Disconnected
  -> if it was the final muxer, onPeerDisconnected -> PeerEvent.Left
```

Closing the Yamux muxer is fundamentally different from closing one channel.
It marks every channel unusable, sends `GoAway` when possible, and closes the
underlying secured transport connection:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
method close*(m: Yamux) =
  if m.isClosed == true:
    return

  let channels =
    toSeq(m.channels.values())

  for channel in channels:
    channel.clearQueues(
      newLPStreamEOFError()
    )
    channel.recvWindow = 0
    channel.sendWindow = 0
    channel.closedLocally = true
    channel.isReset = true
    channel.opened = false
    channel.isClosed = true
    await channel.remoteClosed()
    channel.receivedData.fire()

  try:
    await m.connection.write(
      YamuxHeader.goAway(
        NormalTermination
      )
    )
  except CancelledError, LPStreamError:
    discard

  await m.connection.close()
  m.isClosed = true
```

For TCP, `m.connection` is the Noise-secured stream backed by a Chronos TCP
stream. Ultimately the transport stream closes its TCP client:

Source:
`vendor/nim-libp2p/libp2p/stream/chronosstream.nim`

```nim
method closeImpl*(
    s: ChronosStream
) =
  trace "Shutting down chronos stream",
    address = $s.client.remoteAddress(), s

  if not s.client.closed():
    await s.client.closeWait()

  trace "Shutdown chronos stream",
    address = $s.client.remoteAddress(), s

  await procCall Connection(s).closeImpl()
```

The consequence is:

```text
Yamux.close
  -> all Yamux channels fail/close
  -> Yamux GoAway
  -> Noise stream closes
  -> TCP socket closes
```

For QUIC, closing `QuicMuxer` closes the entire QUIC session and then drains its
stream handlers:

Source:
`vendor/nim-libp2p/libp2p/transports/quictransport.nim`

```nim
method close*(m: QuicMuxer) =
  if m.isNil:
    return

  if not m.session.isNil:
    await noCancel m.session.close()

  if not m.handleFut.isNil:
    await noCancel m.handleFut.cancelAndWait()

  let handleStreamFuts =
    move m.handleStreamFuts

  await noCancel
    handleStreamFuts.cancelAndWait()
```

The implementation differs, but the high-level meaning is the same:

> Closing one muxer closes one underlying peer connection and all streams
> carried by that connection.

In short:

```text
close stream
  -> one channel ends, muxer survives

close muxer
  -> all its channels end, underlying transport connection ends
```

## Natural muxer termination

Natural failure enters the cleanup chain from the opposite end:

```text
remote GoAway / TCP EOF / reset / transport error
  -> Yamux.handle exits
  -> Yamux.handle finally calls Yamux.close
  -> mux.connection finishes
  -> ConnManager.onClose
  -> remove muxer
  -> Disconnected
  -> Left only if no muxer remains
```

Every stored muxer has an `onClose` task:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc storeMuxer*(
    c: ConnManager, muxer: Muxer
) {.async: (raises: [CancelledError, LPError]).} =
  # ...validate and add muxer to c.muxerStore...

  c.onCloseFuts.trackFut(
    c.onClose(muxer)
  )

  # ...emit Connected and, for the first muxer, Joined...
```

The task waits for the muxer's underlying connection to finish:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc onClose(
    c: ConnManager, mux: Muxer
) =
  try:
    await mux.connection.join()
    trace "Connection closed, cleaning up", mux
  except CatchableError as exc:
    debug "Unexpected exception in connection manager's cleanup",
      description = exc.msg,
      mux
  finally:
    let peerId = mux.connection.peerId
    let removed = c.muxerStore.remove(mux)

    if removed and
        c.muxerStore.count(peerId) == 0:
      await c.onPeerDisconnected(peerId)

    await noCancel c.triggerConnEvent(
      peerId,
      ConnEvent(
        kind: ConnEventKind.Disconnected
      )
    )
```

Natural termination can be caused by:

- remote orderly shutdown;
- remote Yamux `GoAway`;
- TCP EOF or reset;
- QUIC session closure;
- security/muxer protocol error;
- local muxer closure;
- transport failure discovered during I/O.

Every muxer termination emits a connection-level `Disconnected` event after
cleanup. `PeerEvent.Left` has a different condition: the closed muxer must have
been the final stored muxer for that peer.

## One of two muxers closes

Assume:

```text
Peer B
├── Muxer 1: outgoing TCP
└── Muxer 2: incoming TCP
```

If Muxer 1 closes naturally:

```text
remove Muxer 1
  -> Muxer 2 still exists
  -> emit ConnEvent.Disconnected
  -> do not call onPeerDisconnected
  -> do not emit PeerEvent.Left
  -> switch.isConnected(B) remains true
```

This follows directly from:

```nim
proc onClose(
    c: ConnManager, mux: Muxer
) {.async: (raises: []).} =
  try:
    await mux.connection.join()
  finally:
    let peerId = mux.connection.peerId
    let removed = c.muxerStore.remove(mux)

    if removed and
        c.muxerStore.count(peerId) == 0:
      await c.onPeerDisconnected(peerId)

    await noCancel c.triggerConnEvent(
      peerId,
      ConnEvent(kind: ConnEventKind.Disconnected)
    )
```

Only when Muxer 2 also closes:

```text
remove Muxer 2
  -> zero muxers remain
  -> onPeerDisconnected(B)
  -> PeerEvent.Left
  -> switch.isConnected(B) becomes false
```

Logos Storage block exchange listens to the peer-level `Joined` and `Left`
events, not to every muxer-level `Disconnected` event.

## Explicitly disconnecting one peer

Explicit peer disconnection uses this named sequence:

```text
Switch.disconnect(peerId)
  -> ConnManager.dropPeer(peerId)
     -> MuxerStore.remove(peerId) returns every peer muxer
     -> schedule ConnManager.closeMuxer for each
        -> Muxer.close
        -> await muxer handler
     -> close every mux.connection
     -> ConnManager.onPeerDisconnected
        -> clear ready/connected/PeerStore state
        -> PeerEvent.Left
```

The public operation is:

Source:
`vendor/nim-libp2p/libp2p/switch.nim`

```nim
proc disconnect*(
    s: Switch, peerId: PeerId
) =
  await s.connManager.dropPeer(peerId)
```

`dropPeer` removes **all** muxers for that peer from the store and closes their
underlying connections:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc dropPeer*(
    c: ConnManager, peerId: PeerId
) =
  trace "Dropping peer", peerId

  let muxers =
    c.muxerStore.remove(peerId)

  if muxers.len > 0:
    try:
      for mux in muxers:
        c.onCloseFuts.trackFut(
          closeMuxer(mux)
        )

      await allFutures(
        muxers.mapIt(
          it.connection.close()
        )
      )
    finally:
      await noCancel
        c.onPeerDisconnected(peerId)

  trace "Peer dropped",
    peerId,
    connCount = muxers.len
```

The removal happens before draining the muxer handlers. This is deliberate:
`dropPeer` may be called from one of the peer's own stream handlers, and waiting
for that same handler to finish would deadlock.

The helper which fully drains a muxer is:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc closeMuxer(
    muxer: Muxer
) =
  await muxer.close()

  if not muxer.handler.isNil:
    try:
      await muxer.handler
    except CatchableError as exc:
      trace "Exception in close muxer handler",
        description = exc.msg
```

The explicit disconnect result is:

```text
Switch.disconnect(B)
  -> remove every B muxer from ConnManager
  -> close every B muxer
  -> close all streams on those muxers
  -> close all underlying TCP/QUIC connections to B
  -> clear peer-ready/peer-store connection state
  -> emit PeerEvent.Left
```

There is no retained TCP connection after `disconnect`.

## Address-taking `dial` failure can close the selected muxer

There is an exceptional cleanup path worth knowing because its scope is wider
than “close the stream which failed to negotiate.”

The address-taking overload
`Switch.dial(peerId, addresses, protocols)` first obtains a `Muxer`, then
creates and negotiates a stream. Its local cleanup procedure closes both the
incomplete stream and the selected muxer:

```text
Switch.dial(peerId, addresses, protocols)
  -> Dialer.dial(peerId, addresses, protocols)
  -> Dialer.internalConnect
     -> may return a pre-existing reused muxer
  -> ConnManager.getStream(muxer)
  -> Dialer.negotiateStream
  -> stream creation or negotiation fails
  -> Dialer.dial.cleanup
     -> stream.closeWithEOF, if a stream was created
     -> muxer.close, including when the muxer was reused
  -> ConnManager.onClose later removes that muxer
```

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
method dial*(
    self: Dialer,
    peerId: PeerId,
    addrs: seq[MultiAddress],
    protos: seq[string],
    forceDial = false,
): Future[Stream] =
  var
    conn: Muxer
    stream: Stream

  proc cleanup() =
    if not isNil(stream):
      await stream.closeWithEOF()

    if not isNil(conn):
      await conn.close()

  try:
    conn = await self.internalConnect(
      Opt.some(peerId),
      dialAddrs,
      forceDial
    )

    stream =
      await self.connManager.getStream(conn)

    if isNil(stream):
      raise newException(
        DialFailedError,
        "Couldn't get muxed stream"
      )

    return await self.negotiateStream(
      stream,
      protos
    )
  except CancelledError as exc:
    await cleanup()
    raise exc
  except CatchableError as exc:
    await cleanup()
    raise newException(
      DialFailedError,
      "failed new dial",
      exc
    )
```

`internalConnect` uses `reuseConnection = true` by default. Therefore `conn`
can be either:

- a muxer newly established for this call; or
- an already stored, reused muxer.

If opening or negotiating the application stream fails after that point,
`cleanup()` calls `conn.close()`. In the reused case this can close an
otherwise pre-existing peer connection and disrupt its sibling streams.

The intended conceptual model remains “protocol negotiation failure closes
the failed stream,” but the current concrete implementation has this broader
failure cleanup. This is particularly relevant when reasoning about error
paths in a future transport abstraction.

The peer is not necessarily immediately `Left`: normal `ConnManager.onClose`
processing removes this muxer, emits its `Disconnected` event, and emits
`Left` only if no other muxer for the peer remains.

## Connection-manager pruning also disconnects peers

`ConnManager` may be configured with high/low watermarks. When the number of
connected peers exceeds the high watermark, it selects unprotected candidates
and calls the same `dropPeer` path used by explicit disconnect:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc trimConnections(
    c: ConnManager
) {.async: (raises: []).} =
  let wm = c.watermark.get()

  # ...build and sort unprotected candidates...

  var dropFuts: seq[Future[void]]

  for (_, _, peerId) in candidates:
    if c.muxerStore.countPeers() <=
        wm.lowWater:
      break

    dropFuts.add(
      c.dropPeer(peerId)
    )

    libp2p_connmgr_pruned_peers_total.inc()

  await allFutures(dropFuts)
```

Pruning is consequently peer-wide:

```text
watermark exceeded
  -> choose unprotected peer
  -> dropPeer(peer)
  -> close every muxer for that peer
  -> close every application stream on them
  -> emit Left
```

It is not merely the eviction of one inactive application stream.

### Logos Storage's explicit drop path

Block exchange delegates its peer drop to the switch:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc dropPeer*(
    self: BlockExcNetwork, peer: PeerId
) =
  trace "Dropping peer", peer

  try:
    if not self.switch.isNil:
      await self.switch.disconnect(peer)
  except CatchableError as error:
    warn "Error attempting to disconnect from peer",
      peer = peer,
      error = error.msg
```

For example, validation failure can ban the peer in the current download and
then disconnect it:

Source:
`storage/blockexchange/engine/engine.nim`

```nim
proc banAndDropPeer(
    self: BlockExcEngine,
    download: ActiveDownload,
    peerId: PeerId
) =
  download.ctx.swarm.banPeer(peerId)
  download.handlePeerFailure(peerId)
  await self.network.dropPeer(peerId)
```

That is not merely closing a bad block-exchange stream. It drops all libp2p
connections and therefore all protocols currently using those connections.

## Peer-level cleanup and `Left`

When the final muxer disappears, `onPeerDisconnected` clears peer-level state:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc onPeerDisconnected(
    c: ConnManager, peerId: PeerId
) =
  if c.muxerStore.count(peerId) > 0:
    return

  c.clearPeerReadyState(peerId)
  c.connectedAt.del(peerId)

  if not c.peerStore.isNil:
    c.peerStore.markPeerDisconnected(peerId)
    c.peerStore.cleanup(peerId)

  await noCancel c.triggerPeerEvents(
    peerId,
    PeerEvent(
      kind: PeerEventKind.Left
    )
  )
```

The initial guard protects against a reconnection race: if another muxer has
already arrived, cleanup and `Left` are skipped.

The event meanings are:

```text
close one application stream:
  no ConnEvent.Disconnected
  no PeerEvent.Left

close one muxer:
  ConnEvent.Disconnected
  PeerEvent.Left only if zero muxers remain

Switch.disconnect:
  all peer muxers removed and closed
  PeerEvent.Left
```

## Closing the whole switch

Whole-node shutdown composes the earlier peer and muxer close paths:

```text
Switch.stop
  -> cancel accept loops
  -> cancel pending connection upgrades
  -> stop Switch services
  -> ConnManager.close
     -> clear MuxerStore
     -> ConnManager.closeMuxer for every muxer
     -> onPeerDisconnected for every connected peer
  -> Transport.stop for every configured transport
     -> TcpTransport.stop closes listeners and remaining clients
     -> QuicTransport.stop closes QUIC endpoints/sessions
  -> MultistreamSelect.stop
  -> PeerStore.close
```

`Switch.stop` closes all active peer connections and then stops transports:

Source:
`vendor/nim-libp2p/libp2p/switch.nim`

```nim
proc stop*(
    s: Switch
) =
  ## Stop listening on every transport, and
  ## close every active connections

  s.started = false

  await s.acceptFuts
    .cancelAndWait()
    .wait(1.seconds)

  await s.upgradeFuts.cancelAndWait()
  s.upgradeFuts = @[]

  for service in s.services:
    await service.stop(s)

  await s.connManager.close()

  for transp in s.transports:
    await transp.stop()

  await s.ms.stop()
  s.peerStore.close()
```

`ConnManager.close` clears the entire muxer store, closes every muxer, and
runs peer-level cleanup:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc close*(
    c: ConnManager
) =
  c.closed = true

  let muxed =
    c.muxerStore.getAll()

  c.muxerStore.clear()

  for peerId, muxers in muxed:
    if muxers.len > 0:
      await allFutures(
        muxers.mapIt(
          closeMuxer(it)
        )
      )
      await c.onPeerDisconnected(peerId)

  await c.drainOnCloseTasks()
```

The TCP transport stop closes listeners and every known incoming/outgoing TCP
client:

Source:
`vendor/nim-libp2p/libp2p/transports/tcptransport.nim`

```nim
method stop*(
    self: TcpTransport
) =
  self.stopping = true

  if self.running:
    await noCancel
      procCall Transport(self).stop()

    await noCancel allFutures(
      self.servers.mapIt(
        it.closeWait()
      ) &
      self.clients[Direction.In].mapIt(
        it.closeWait()
      ) &
      self.clients[Direction.Out].mapIt(
        it.closeWait()
      )
    )
```

Normally `ConnManager.close` has already closed managed peer connections; the
transport shutdown also ensures listeners and remaining transport resources
are gone.

The scope progression is therefore:

```text
stream.close
  -> one application stream

muxer.close
  -> one underlying connection to one peer

Switch.disconnect(peer)
  -> all underlying connections to one peer

Switch.stop
  -> all connections to all peers plus all listeners/transports
```

## How block exchange closes its application streams

Each `NetworkPeer` stores at most one preferred outgoing block-exchange stream
in `sendConn`:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
type NetworkPeer* =
  ref object of RootObj
    id*: PeerId
    handler*: RPCHandler
    wantBlocksHandler*: WantBlocksRequestHandler
    sendConn: Connection
    getConn: ConnProvider
    trackedFutures: TrackedFutures
    pendingWantBlocksRequests*:
      Table[
        uint64,
        WantBlocksResponseFuture
      ]
```

Its application-level definition of connected is narrower than
`Switch.isConnected`:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
proc connected*(
    self: NetworkPeer
): bool =
  not isNil(self.sendConn) and
  not (
    self.sendConn.closed or
    self.sendConn.atEof
  )
```

Compare:

```text
Switch.isConnected(peer)
  at least one peer-level muxer exists

NetworkPeer.connected
  this NetworkPeer has a usable cached outgoing
  block-exchange application stream
```

The first may be true while the second is false.

### Normal read-loop exit

The block-exchange-specific close sequence is:

```text
NetworkPeer.readLoop exits
  -> complete every pending WantBlocks future with ConnectionClosed
  -> clear pendingWantBlocksRequests
  -> clear NetworkPeer.sendConn if it is this stream
  -> Connection.close
     -> close only this block-exchange Yamux channel
  -> later NetworkPeer.connect
     -> ConnProvider
     -> Switch.dial(peer, Codec)
     -> open a replacement stream on a surviving muxer
```

The read loop owns the eventual application-stream close:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
proc readLoop*(
    self: NetworkPeer,
    conn: Connection
) =
  try:
    while not conn.atEof and
        not conn.closed:
      var lenBuf: array[4, byte]
      await conn.readExactly(
        addr lenBuf[0],
        4
      )

      # decode and dispatch one frame
      ...
  except CancelledError:
    trace "Read loop cancelled"
  except CatchableError as err:
    warn "Exception in blockexc read loop",
      msg = err.msg
  finally:
    for requestId, fut in
        self.pendingWantBlocksRequests:
      if not fut.finished:
        fut.complete(
          WantBlocksResult[
            WantBlocksResponse
          ].err(
            wantBlocksError(
              ConnectionClosed,
              "Read loop exited"
            )
          )
        )

    self.pendingWantBlocksRequests.clear()

    if self.sendConn == conn:
      self.sendConn = nil

    await conn.close()
```

When the read loop exits:

1. every pending `WantBlocks` response waiting on this peer is completed with
   `ConnectionClosed`;
2. `sendConn` is cleared if this was the cached outgoing stream;
3. this application stream is closed;
4. the underlying muxer normally remains connected.

The next send can lazily open another block-exchange stream on the existing
muxer:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
proc connect*(
    self: NetworkPeer
): Future[Connection] =
  if self.connected:
    return self.sendConn

  self.sendConn =
    await self.getConn()

  self.trackedFutures.track(
    self.readLoop(self.sendConn)
  )

  return self.sendConn
```

And the default provider is:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc getOrCreatePeer(
    self: BlockExcNetwork, peer: PeerId
): NetworkPeer =
  if peer in self.peers:
    return self.peers.getOrDefault(peer, nil)

  var getConn: ConnProvider =
    proc(): Future[Connection] =
      try:
        return await self.switch.dial(
          peer,
          Codec
        )
      except CancelledError as error:
        raise error
      except CatchableError as exc:
        trace "Unable to connect to blockexc peer",
          exc = exc.msg

  # ...construct and store NetworkPeer with this provider...
```

This is an important recovery distinction:

```text
only block-exchange stream failed:
  cached sendConn cleared
  -> Switch still has muxer
  -> next application send can dial a replacement stream

underlying muxer failed:
  all streams on it fail
  -> muxer removed
  -> if last muxer, Left removes BlockExc peer state
  -> discovery/connect may be required again
```

So after the detailed code:

```text
read-loop failure is normally stream-scoped
  -> replace sendConn lazily

muxer failure is connection-scoped
  -> every stream on that muxer fails

last-muxer failure is peer-scoped
  -> Left evicts node-wide block-exchange peer state
```

### Incoming block-exchange streams

Each remote `Switch.dial(..., Codec)` invokes the mounted handler with a new
incoming stream:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc handler(
    conn: Connection,
    proto: string
): Future[void] =
  let peerId = conn.peerId
  let blockexcPeer =
    self.getOrCreatePeer(peerId)

  await blockexcPeer.readLoop(conn)
```

The handler awaits the read loop for the lifetime of the stream. When that
loop ends, it closes the stream; after the handler returns, the generic
libp2p wrapper also calls `closeWithEOF`, which is safe because close is
idempotent.

An incoming stream is not automatically assigned to `NetworkPeer.sendConn`.
Consequently its closure does not necessarily clear the preferred outgoing
stream.

### Stopping block exchange versus stopping the switch

`BlockExcNetwork.stop` cancels its network-level tracked futures:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc stop*(
    self: BlockExcNetwork
) =
  await self.trackedFutures.cancelTracked()
```

`BlockExcEngine.stop` stops its engine tasks, network, discovery, and
advertiser:

Source:
`storage/blockexchange/engine/engine.nim`

```nim
proc stop*(
    self: BlockExcEngine
) =
  await self.trackedFutures.cancelTracked()
  await self.network.stop()
  await self.discovery.stop()
  await self.advertiser.stop()
  self.blockexcRunning = false
```

This is protocol-component shutdown. Whole-node shutdown subsequently stops
the switch and closes its peer-level connections. It is useful not to equate
"stop block exchange workers" with "`Switch.disconnect` every peer" unless the
owning node shutdown path actually performs both.

## Silent failures and stale “connected” state

For a clean remote shutdown, TCP FIN/RST, Yamux `GoAway`, or QUIC close is
normally detected by the muxer read loop. Cleanup then removes the muxer.

A silent failure is different:

```text
remote machine loses power
or route/NAT mapping disappears silently
  -> no immediate local EOF
  -> local muxer may remain stored
  -> Switch.isConnected(peer) can remain true temporarily
  -> later read/write/open/timeout discovers failure
  -> muxer closes and is removed
```

In other words, “connected” means:

> An authenticated muxed peer connection was established and the local node
> has not yet observed it terminate.

It does not mean:

- a heartbeat has just succeeded;
- the peer process is certainly responsive now;
- the route is currently usable;
- a new application stream is guaranteed to open;
- a specific application protocol is available.

The current Yamux implementation responds to ping frames but does not itself
schedule a periodic heartbeat. Application-stream idle timeouts reset
individual channels; they do not continuously validate an otherwise idle
peer-level muxer.

## Closing-operation matrix

| Operation | Affected application streams | Muxer survives? | Underlying TCP/QUIC survives? | `ConnEvent.Disconnected`? | `PeerEvent.Left`? |
|---|---|---:|---:|---:|---:|
| `stream.close()` | one, graceful half-close | yes | yes | no | no |
| `stream.closeWithEOF()` | one, graceful and waits for remote EOF | yes | yes | no | no |
| `stream.reset()` | one, abortive | yes | yes | no | no |
| application stream idle timeout | one, reset | yes | yes | no | no |
| `muxer.close()` | all streams on that muxer | no | no | yes | only if last muxer |
| natural underlying connection failure | all streams on that muxer | no | no | yes | only if last muxer |
| `Switch.disconnect(peer)` | all streams on every peer muxer | no peer muxer remains | no | per-muxer cleanup | yes |
| `ConnManager` pruning | same as peer disconnect | no peer muxer remains | no | per-muxer cleanup | yes |
| `Switch.stop()` | all streams for all peers | no | no | shutdown cleanup | each connected peer departs |

## Worked timelines

### Close one request stream, then dial another

```text
A and B have one Yamux muxer
  -> A dials Ping stream 1
  -> Ping completes
  -> A closes stream 1
  -> B closes its endpoint
  -> stream 1 is removed
  -> muxer remains
  -> A dials Ping stream 2
  -> stream 2 uses the same Yamux/TCP connection
```

No peer `Left` or new `Joined` occurs between the two pings.

### Remote closes one application stream

```text
B sends FIN on stream B1
  -> A observes EOF on A1
  -> A's read loop finishes
  -> A closes A1
  -> other muxer streams remain unaffected
  -> peer remains connected
```

### Remote closes the underlying TCP connection

```text
B closes TCP
  -> A's Yamux header read observes EOF
  -> Yamux handler enters finally
  -> Yamux closes every channel
  -> muxer connection finishes
  -> ConnManager.onClose removes muxer
  -> Disconnected
  -> Left if no other muxer exists
```

### Explicitly disconnect a peer with two muxers

```text
Peer B has Muxer 1 and Muxer 2
  -> Switch.disconnect(B)
  -> both muxers removed from store
  -> every stream on both muxers closes
  -> both underlying transport connections close
  -> peer-level cleanup
  -> one Left transition for B
```

### One of two muxers fails

```text
Peer B has Muxer 1 and Muxer 2
  -> Muxer 1 fails
  -> streams on Muxer 1 fail
  -> Muxer 1 removed
  -> Disconnected emitted
  -> Muxer 2 remains
  -> no Left
  -> Switch.isConnected(B) remains true
```

Application code which cached a stream on Muxer 1 must still recover that
stream, even though the peer remains connected through Muxer 2.

## Recommended wording

To avoid ambiguity in code reviews and design notes, prefer:

- “close the block-exchange stream” for one application stream;
- “reset the Yamux channel” for abortive stream termination;
- “close this muxer” for one underlying peer connection;
- “disconnect the peer” for all muxers to one peer;
- “stop the switch” for all peers, listeners, and transports.

Avoid saying only “close the connection” unless the layer is immediately
obvious.

The compact rule is:

```text
stream lifetime
  is application-protocol lifetime

muxer lifetime
  is one underlying peer-connection lifetime

peer connected lifetime
  spans from first stored muxer
  until last removed muxer

switch lifetime
  owns all muxers, transports, and listeners
```
