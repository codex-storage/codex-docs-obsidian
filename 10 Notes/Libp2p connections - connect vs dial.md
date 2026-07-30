---
tags:
  - logos-storage/libp2p
  - logos-storage/connections
  - logos-storage/transports
  - libp2p
related:
  - "[[Closing libp2p connections and streams]]"
  - "[[Libp2p Connection Lifecycle in Logos Storage]]"
  - "[[New Logos Storage Block Exchange Protocol]]"
  - "[[Uploading and downloading content in new Logos Storage]]"
  - "[[New Logos Storage Download Flow]]"
  - "[[New Logos Storage Discovery]]"
  - "[[Block Exchange Peer Stores]]"
  - "[[Block Exchange Peer Performance and BDP]]"
---

# Libp2p connections: connect vs dial

> [!info] Source baseline
> This note describes `logos-storage-nim` `master` at commit `81f0d053`
> (2026-07-20), including its vendored nim-libp2p 2.1.4. It also refers to
> `/home/mc2/code/logos-storage/libp2p-mix-ping-example/multiple_transports_example.nim`
> when discussing address ordering across TCP and QUIC.

> [!note] Reading the excerpts
> Every Nim source excerpt includes the enclosing function, method, procedure,
> or type signature. Comments containing `...` indicate deliberately omitted
> code inside that named definition; they do not indicate an unnamed fragment.

The words **connection**, **connect**, **dial**, **stream**, and **muxer** are
easy to blur together because nim-libp2p deliberately presents several layers
through connection-like types. The practical distinction is:

```text
Switch.connect(peer, addresses)
  -> establish or reuse peer-level connectivity
  -> result stored by ConnManager as a Muxer
  -> no requested application protocol stream is returned

Switch.dial(peer, codec)
  -> open a new application stream on an existing Muxer
  -> negotiate the requested application codec
  -> return one endpoint of that stream

Switch.dial(peer, addresses, codec)
  -> establish or reuse peer-level connectivity
  -> open a new application stream on the selected Muxer
  -> negotiate the requested application codec
  -> return one endpoint of that stream
```

For current Logos Storage over TCP, the complete stack normally looks like:

```text
application protocol
  /storage/blockexc/1.0.0, manifest, ping, ...
                  │
                  │ one full-duplex application stream
                  ▼
             Yamux channel
                  │
                  │ one of many channels
                  ▼
             Yamux session                 <- Muxer
                  │
                  ▼
          Noise-secured stream
                  │
                  ▼
            TCP connection
```

Closing one application stream normally leaves the Yamux session and TCP
connection available for later streams. See
[[Closing libp2p connections and streams]] for all closing paths.

## The overloaded word `Connection`

The first source of confusion is that `Connection`, `Stream`, `RawConn`, and
`MuxedStream` are semantic aliases in nim-libp2p rather than four distinct Nim
types.

Source:
`vendor/nim-libp2p/libp2p/stream/connection.nim`

```nim
type
  Connection* = ref object of LPStream
    activity*: bool
    timeout*: Duration
    timerTaskFut: Future[void].Raising([])
    timeoutHandler*: TimeoutHandler
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

Consequently, a variable called `conn: Connection` may represent:

- a raw TCP-like transport stream before security and muxing;
- a secured transport stream wrapped by a muxer;
- one Yamux application channel returned by `Switch.dial`;
- one QUIC-native application stream.

The surrounding operation determines which layer is meant.

By contrast, `Muxer` is a separate abstraction. It owns or wraps the peer-level
session and can create streams:

Source:
`vendor/nim-libp2p/libp2p/muxers/muxer.nim`

```nim
method newStream*(
    m: Muxer, name: string = "", lazy: bool = false
): Future[MuxedStream] =
  raiseAssert("[Muxer.newStream] abstract method not implemented!")

method close*(m: Muxer) =
  if m.connection != nil:
    await m.connection.close()
```

The useful vocabulary in this note is therefore:

```text
transport connection
  TCP connection or QUIC connection

muxer
  ConnManager's uniform representation of one established peer connection

application stream
  one full-duplex logical stream opened through that muxer

stream endpoint object
  the local Connection object through which one side reads and writes
```

## Current Logos Storage transport stack

The Storage node currently builds a TCP-only switch:

Source:
`storage/storage.nim`

```nim
proc new*(
    T: type StorageServer,
    config: StorageConf,
    privateKey: StoragePrivateKey,
    logFile: Option[IoHandle] = IoHandle.none,
): StorageServer =
  ## create StorageServer including setting up datastore, repostore, etc
  let listenMultiAddr =
    getMultiAddrWithIpAndTcpPort(config.listenIp, config.listenPort)

  let switch = SwitchBuilder
    .new()
    .withPrivateKey(privateKey)
    .withAddresses(@[listenMultiAddr])
    .withWildcardResolver(true)
    .withIdentifyPusher(false)
    .withRng(random.Rng.instance().libp2pRng)
    .withNoise()
    .withYamux()
    .withMaxConnections(config.maxPeers)
    .withAgentVersion(config.agentString)
    .withSignedPeerRecord(true)
    .withTcpTransport({ServerFlags.ReuseAddr, ServerFlags.TcpNoDelay})
    .build()

  # ...construct the remaining StorageServer components...
```

This means the normal peer-level establishment flow is:

```text
TCP connect
  -> multistream-select chooses Noise
  -> Noise authenticates and secures the connection
  -> multistream-select chooses Yamux
  -> create Yamux session
  -> store Yamux as a Muxer
  -> run Identify
```

The phrase "`connect` does not negotiate a protocol" refers specifically to
the absence of a requested **application protocol**. Lower-level security and
muxer protocols are still negotiated.

## What `Switch.connect` does

The named call chain for a newly established TCP connection is:

```text
Switch.connect
  -> Dialer.connect
  -> Dialer.internalConnect
  -> Dialer.dialAndUpgrade(peerId, addresses)
  -> Dialer.dialAndUpgrade(peerId, hostname, address)
  -> TcpTransport.dial
  -> MuxedUpgrade.upgrade
     -> Upgrade.secure
     -> MuxedUpgrade.mux
  -> ConnManager.storeMuxer
  -> PeerStore.identify
  -> ConnManager.triggerPeerEvents(Identified)
```

The public switch method itself only delegates to its `Dialer`; the remaining
sections follow this chain one function at a time.

The public switch method delegates directly to its `Dialer`:

Source:
`vendor/nim-libp2p/libp2p/switch.nim`

```nim
method connect*(
    s: Switch,
    peerId: PeerId,
    addrs: seq[MultiAddress],
    forceDial = false,
    reuseConnection = true,
    dir = Direction.Out,
): Future[void] =
  ## Connects to a peer without opening a stream to it

  s.dialer.connect(peerId, addrs, forceDial, reuseConnection, dir)
```

The dialer first checks whether it already has at least one connection for the
peer:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
method connect*(
    self: Dialer,
    peerId: PeerId,
    addrs: seq[MultiAddress],
    forceDial = false,
    reuseConnection = true,
    dir = Direction.Out,
) =
  ## connect remote peer without negotiating
  ## a protocol

  if self.connManager.connCount(peerId) > 0 and reuseConnection:
    return

  discard
    await self.internalConnect(
      Opt.some(peerId), addrs, forceDial, reuseConnection, dir
    )
```

Thus a default `connect` means:

```text
If any muxer for Peer B already exists:
  do nothing successfully

Otherwise:
  establish, secure, identify, and store one connection to Peer B
```

It neither creates nor returns a block-exchange, manifest, or ping stream.

### Establishing a new transport connection

When reuse does not apply, `internalConnect` serializes simultaneous outgoing
attempts to the same peer and then calls `dialAndUpgrade`:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc internalConnect(
    self: Dialer,
    peerId: Opt[PeerId],
    addrs: seq[MultiAddress],
    forceDial: bool,
    reuseConnection = true,
    dir = Direction.Out,
): Future[Muxer] {.async: (raises: [DialFailedError, CancelledError]).} =
  let lock =
    self.dialLock.mgetOrPut(
      peerId.get(default(PeerId)),
      newAsyncLock()
    )
  await lock.acquire()
  defer:
    lock.release()

  if reuseConnection:
    peerId.withValue(peerId):
      self.tryReusingConnection(peerId).withValue(mux):
        return mux

  let slot = self.connManager.getOutgoingSlot(forceDial)
  let muxed = await self.dialAndUpgrade(peerId, addrs, dir)

  # ...track and store muxed, run Identify, then return muxed...
```

The per-peer lock prevents two concurrent local callers from independently
creating two ordinary outgoing connections just because both saw the peer as
disconnected at approximately the same time.

`forceDial` and `reuseConnection` have different meanings:

- `reuseConnection = true` permits returning an existing muxer;
- `reuseConnection = false` requires establishing another connection;
- `forceDial = true` bypasses ordinary outgoing capacity admission;
- `forceDial = true` does **not** by itself disable reuse.

### Choosing a transport from addresses

Address order matters only when a new connection must actually be established.
The dialer iterates address candidates in order:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc dialAndUpgrade*(
    self: Dialer,
    peerId: Opt[PeerId],
    addrs: seq[MultiAddress],
    dir = Direction.Out,
): Future[Muxer] {.
    async: (raises: [CancelledError, MaError, TransportAddressError, LPError])
.} =
  let dialAddrs = normalizedDialAddrs(peerId, addrs)

  for rawAddress in dialAddrs:
    let addresses = await self.expandDnsAddr(peerId, rawAddress)

    for (expandedAddress, addrPeerId) in addresses:
      let
        hostname = expandedAddress.getHostname()
        resolvedAddresses =
          if isNil(self.nameResolver):
            @[expandedAddress]
          else:
            await self.nameResolver.resolveMAddress(expandedAddress)

      for resolvedAddress in resolvedAddresses:
        let mux =
          await self.dialAndUpgrade(
            addrPeerId, hostname, resolvedAddress, dir
          )
        if not isNil(mux):
          return mux
```

For each resolved address, it finds a configured transport that handles that
multiaddress:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc dialAndUpgrade*(
    self: Dialer,
    peerId: Opt[PeerId],
    hostname: string,
    addrs: MultiAddress,
    dir = Direction.Out,
): Future[Muxer] {.async: (raises: [CancelledError]).} =
  for transport in self.transports:
    if transport.handles(addrs):
      let dialed =
        await transport.dial(hostname, addrs, peerId)

      let mux =
        await transport.upgrade(dialed, peerId)

      return mux

  return nil
```

A TCP multiaddress cannot be used by QUIC, and a QUIC multiaddress cannot be
used by TCP:

```text
/ip4/192.0.2.1/tcp/4001
  -> TcpTransport

/ip4/192.0.2.1/udp/4001/quic-v1
  -> QuicTransport
```

When there is no existing muxer, putting the TCP address first expresses
"try TCP first, then fall back to the remaining candidates."

It does not create one connection for every configured transport. The first
successful address/transport combination wins.

### Securing and multiplexing TCP

Ordinary stream transports use `MuxedUpgrade`. It first negotiates a secure
connection and then negotiates a muxer:

Source:
`vendor/nim-libp2p/libp2p/upgrademngrs/muxedupgrade.nim`

```nim
method upgrade*(
    self: MuxedUpgrade, conn: RawConn, peerId: Opt[PeerId]
): Future[Muxer] =
  let sconn = await self.secure(conn, peerId)
  if sconn == nil:
    raise (ref UpgradeFailedError)(
      msg: "unable to secure connection, stopping upgrade"
    )

  let muxer = (await self.mux(sconn)).valueOr:
    raise (ref UpgradeFailedError)(
      msg: "a muxer is required for outgoing connections"
    )

  muxer
```

The muxer negotiation chooses among the muxer codecs configured by the switch:

Source:
`vendor/nim-libp2p/libp2p/upgrademngrs/muxedupgrade.nim`

```nim
proc mux(
    self: MuxedUpgrade, secureConn: SecureConn
): Future[Opt[Muxer]] {.
    async: (raises: [CancelledError, LPStreamError, MultiStreamError])
.} =
  let
    muxerName =
      case secureConn.dir
      of Direction.Out:
        await self.ms.select(
          secureConn,
          self.muxers.mapIt(it.codec)
        )
      of Direction.In:
        await MultistreamSelect.handle(
          secureConn,
          self.muxers.mapIt(it.codec)
        )

    muxerProvider =
      self.getMuxerByCodec(muxerName).valueOr:
        return Opt.none(Muxer)

  let muxer = muxerProvider.newMuxer(secureConn)
  muxer.streamHandler = self.streamHandler
  muxer.handler = muxer.handle()
  Opt.some(muxer)
```

In current Logos Storage, the selected muxer is Yamux.

### QUIC still appears as a `Muxer`

QUIC already provides encryption and independent streams. It therefore does
not need Noise plus Yamux layered on top. Nevertheless, nim-libp2p wraps the
QUIC connection in a `QuicMuxer` so `ConnManager` can use the same abstraction.

Source:
`vendor/nim-libp2p/libp2p/transports/quictransport.nim`

```nim
method upgrade*(
    self: QuicTransport, conn: RawConn, peerId: Opt[PeerId]
): Future[Muxer] =
  let muxer = QuicMuxer.new(conn, peerId)

  muxer.streamHandler =
    proc(stream: MuxedStream) =
      try:
        await self.upgrader.ms.handle(stream)
      finally:
        await stream.closeWithEOF()

  muxer.handleFut = muxer.handle()
  return muxer
```

Therefore:

```text
TCP:
  Muxer = negotiated Yamux session over Noise over TCP

QUIC:
  Muxer = QuicMuxer adapter over QUIC's native session and streams
```

The useful general statement is:

> One successfully established underlying peer connection is represented in
> `ConnManager` by one `Muxer` abstraction.

### `connect` flow recap

After the detailed transport branches, the complete result can be summarized
as:

```text
Switch.connect(peerId, addrs)
  -> Dialer.connect
     -> return immediately if ConnManager already has peerId and reuse is true
     -> otherwise Dialer.internalConnect
        -> serialize local attempts with dialLock
        -> optionally reuse ConnManager.selectMuxer(peerId)
        -> Dialer.dialAndUpgrade over addresses in order
           -> matching Transport.dial
           -> Transport.upgrade
              -> TCP: secure with Noise, negotiate/create Yamux
              -> QUIC: create QuicMuxer over native QUIC session
        -> ConnManager.storeMuxer
        -> PeerStore.identify
        -> return without opening the caller's application protocol
```

## When the peer becomes “connected”

The upgraded muxer is stored in `ConnManager`:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc storeMuxer*(
    c: ConnManager, muxer: Muxer
) =
  let
    peerId = muxer.connection.peerId
    dir = muxer.connection.dir
    peerConnsCount = c.muxerStore.count(peerId)
    isNewPeer = peerConnsCount == 0

  if not c.muxerStore.add(muxer):
    raise newException(LPError, "muxer already stored")

  if isNewPeer and not c.peerStore.isNil:
    c.peerStore.markPeerConnected(peerId)

  c.onCloseFuts.trackFut(c.onClose(muxer))

  let connectedEvent = c.triggerConnEvent(
    peerId,
    ConnEvent(
      kind: ConnEventKind.Connected,
      incoming: dir == Direction.In
    )
  )

  c.notifyPeerReady(peerId)
  await connectedEvent

  if isNewPeer:
    joinedEvent = c.triggerPeerEvents(
      peerId,
      PeerEvent(
        kind: PeerEventKind.Joined,
        initiator: dir == Direction.Out
      )
    )
```

The precise local definition of connected is:

Source:
`vendor/nim-libp2p/libp2p/switch.nim`

```nim
proc isConnected*(s: Switch, peerId: PeerId): bool =
  ## returns true if the peer has one or more
  ## associated connections

  peerId in s.connManager
```

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc contains*(c: ConnManager, peerId: PeerId): bool =
  return c.muxerStore.contains(peerId)
```

It means:

> The local connection manager has at least one stored muxer associated with
> the peer, and it has not yet observed all such muxers terminate.

It does not continuously prove reachability. There is no periodic
application-level heartbeat implied by `isConnected`. A silent network failure
may leave this local state stale until traffic or a transport timeout exposes
the failure.

After outgoing `connect` continues past storage, it also performs Identify:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc internalConnect(
    self: Dialer,
    peerId: Opt[PeerId],
    addrs: seq[MultiAddress],
    forceDial: bool,
    reuseConnection = true,
    dir = Direction.Out,
): Future[Muxer] {.async: (raises: [DialFailedError, CancelledError]).} =
  # ...reuse or establish muxed...

  try:
    await self.connManager.storeMuxer(muxed)
    await self.peerStore.identify(muxed, dir)
    await self.connManager.triggerPeerEvents(
      muxed.connection.peerId,
      PeerEvent(
        kind: PeerEventKind.Identified,
        initiator: true
      ),
    )
    return muxed
  except CancelledError as exc:
    await muxed.close()
    raise exc
  except CatchableError as exc:
    await muxed.close()
    raise newException(DialFailedError, "Failed to finish outgoing upgrade", exc)
```

Thus a successful outgoing `Switch.connect` has established an authenticated,
secured, multiplexed peer connection and completed Identify. The `Joined`
event starts earlier, when the first muxer is stored; it is not an application
codec event.

## What `Switch.dial(peer, codec)` does

The peer-ID-only stream-opening chain is:

```text
Switch.dial(peerId, protocols)
  -> Dialer.dial(peerId, protocols)
  -> ConnManager.getStream(peerId)
     -> ConnManager.selectMuxer(peerId)
     -> ConnManager.getStream(muxer)
     -> Muxer.newStream
        -> Yamux.newStream in the current TCP Storage configuration
  -> Dialer.negotiateStream
     -> MultistreamSelect.select
  -> return the negotiated application Stream
```

The peer-ID-only overload requires an existing muxer:

Source:
`vendor/nim-libp2p/libp2p/switch.nim`

```nim
method dial*(
    s: Switch, peerId: PeerId, protos: seq[string]
): Future[Stream] =
  ## Open a stream to a connected peer with the specified `protos`

  s.dialer.dial(peerId, protos)
```

The dialer asks `ConnManager` for a new stream:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
method dial*(
    self: Dialer, peerId: PeerId, protos: seq[string]
): Future[Stream] =
  ## create a protocol stream over an
  ## existing connection

  let stream = await self.connManager.getStream(peerId)
  if stream.isNil:
    raise newException(
      DialFailedError,
      "Couldn't get muxed stream in dial"
    )

  return await self.negotiateStream(stream, protos)
```

`ConnManager.getStream` selects a muxer and calls `newStream`:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc getStream*(
    c: ConnManager, peerId: PeerId
): Future[MuxedStream] =
  return await c.getStream(c.selectMuxer(peerId))

proc getStream*(
    c: ConnManager, muxer: Muxer
): Future[MuxedStream] =
  if not muxer.isNil:
    return await muxer.newStream()
  return nil
```

For Yamux, every call creates a channel with a new channel ID:

Source:
`vendor/nim-libp2p/libp2p/muxers/yamux/yamux.nim`

```nim
method newStream*(
    m: Yamux, name: string = "", lazy: bool = false
): Future[MuxedStream] =
  let stream =
    m.createStream(
      m.currentId,
      true,
      m.windowSize,
      m.maxSendQueueSize
    )

  m.currentId += 2

  if not lazy:
    await stream.open()

  return stream
```

The requested application protocol is then negotiated on that new stream:

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc negotiateStream*(
    self: Dialer, stream: Stream, protos: seq[string]
): Future[Stream] =
  let selected =
    await MultistreamSelect.select(stream, protos)

  if not protos.contains(selected):
    await stream.closeWithEOF()
    raise newException(
      DialFailedError,
      "Unable to select sub-protocol"
    )

  return stream
```

The result is:

```text
existing peer muxer
  -> new Yamux channel
  -> multistream-select on that channel
  -> /storage/blockexc/1.0.0 selected
  -> return local endpoint object
```

So the long implementation thread reduces to:

```text
select existing muxer
  -> create a distinct muxed stream
  -> negotiate the requested application codec on that stream
  -> return the stream; leave the muxer stored
```

Calling `dial` twice for the same peer and same codec normally creates two
different application streams, not one reused stream:

```text
one Muxer to Peer B
  ├── stream A1/B1: /storage/blockexc/1.0.0
  └── stream A2/B2: /storage/blockexc/1.0.0
```

Application-level reuse, such as `NetworkPeer.sendConn`, must be implemented by
the application. The switch does not return an old protocol stream merely
because its codec matches.

## Full-duplex endpoints and incoming handlers

When A dials B, one logical full-duplex stream has two local endpoint objects:

```text
Node A                                       Node B

Switch.dial(B, Codec)
  -> endpoint A1   <===================>   endpoint B1
                                           protocol handler(B1)
```

A can write through A1 and B reads through B1. B can write its response through
B1 and A reads it through A1.

B writing to B1 does **not** invoke A's incoming protocol handler. A's handler
runs only when B independently calls `dial` and opens another stream:

```text
Node B calls Switch.dial(A, Codec)
  -> endpoint B2   <===================>   endpoint A2
                                           A's protocol handler(A2)
```

This is why the outgoing endpoint needs its own read loop if responses may
arrive on the same stream.

In current block exchange, a binary `WantBlocksResponse` is written onto the
same stream as its request:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
proc readLoop*(
    self: NetworkPeer, conn: Connection
) {.async: (raises: []).} =
  # ...read frame length and message type...

  case msgType
  of mtWantBlocksRequest:
    let reqResult =
      await readWantBlocksRequest(conn, dataLen)

    let
      req = reqResult.get
      blocks =
        await self.wantBlocksHandler(self.id, req)

    await writeWantBlocksResponse(
      conn,
      req.requestId,
      req.treeCid,
      blocks
    )

  # ...handle the other message types...
```

The requester's read loop on its outgoing endpoint receives that response.

## What `Switch.dial(peer, addresses, codec)` does

This overload prepends the `connect` chain to the stream-opening chain:

```text
Switch.dial(peerId, addresses, protocols)
  -> Dialer.dial(peerId, addresses, protocols)
  -> Dialer.internalConnect
     -> reuse an existing muxer, or
     -> Dialer.dialAndUpgrade and ConnManager.storeMuxer
  -> ConnManager.getStream(the returned muxer)
  -> Muxer.newStream
  -> Dialer.negotiateStream
  -> return the application Stream
```

The address-taking overload combines connectivity and stream creation:

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

  conn =
    await self.internalConnect(
      Opt.some(peerId),
      dialAddrs,
      forceDial
    )

  stream = await self.connManager.getStream(conn)
  return await self.negotiateStream(stream, protos)
```

`internalConnect` uses its default `reuseConnection = true`. This creates the
following exact semantics:

```text
1. Look for any existing reusable muxer for Peer B.
2. If one exists, return it and ignore the supplied address list.
3. Otherwise, try the supplied addresses in order.
4. Open a new stream on the returned or newly created muxer.
5. Negotiate the application codec.
```

The addresses are therefore **connection fallback addresses**, not a
requirement on the transport of the returned application stream.

### The TCP-first but existing-QUIC example

Assume A already has a QUIC muxer to B:

```text
ConnManager[A]
  └── Peer B
      └── existing QuicMuxer
```

Now A calls:

```nim
proc openPingWithTcpPreference(
    switch: Switch,
    peerB: PeerId,
    tcpAddress, quicAddress: MultiAddress,
): Future[Stream] {.async.} =
  return await switch.dial(
    peerB,
    @[tcpAddress, quicAddress],
    PingCodec
  )
```

The result is:

```text
internalConnect
  -> tryReusingConnection(peerB)
  -> returns existing QuicMuxer
  -> address list is never inspected
  -> QuicMuxer.newStream()
  -> negotiate Ping
```

Even the following normally reuses QUIC:

```nim
proc openPingWithOnlyTcpFallback(
    switch: Switch,
    peerB: PeerId,
    tcpAddress: MultiAddress,
): Future[Stream] {.async.} =
  return await switch.dial(
    peerB,
    @[tcpAddress],
    PingCodec
  )
```

The TCP address is not somehow run over QUIC. It is simply unused.

Source:
`vendor/nim-libp2p/libp2p/dialer.nim`

```nim
proc tryReusingConnection(
    self: Dialer, peerId: PeerId
): Opt[Muxer] =
  let muxer =
    self.connManager.selectMuxer(peerId)

  if muxer == nil:
    return Opt.none(Muxer)

  return Opt.some(muxer)
```

### What the multi-transport ping example demonstrates

The example orders addresses to prefer QUIC or TCP:

Source:
`/home/mc2/code/logos-storage/libp2p-mix-ping-example/multiple_transports_example.nim`

```nim
proc quicPreferred*(
    addrs: seq[MultiAddress]
): seq[MultiAddress] =
  addrs.filterIt("/quic-v1" in $it) &
    addrs.filterIt("/quic-v1" notin $it)

proc tcpPreferred*(
    addrs: seq[MultiAddress]
): seq[MultiAddress] =
  addrs.filterIt("/quic-v1" notin $it) &
    addrs.filterIt("/quic-v1" in $it)
```

It uses two independent client switches:

```nim
proc main() {.async: (raises: [CancelledError, LPError]).} =
  let rng = newRng()
  let pingProtocol = Ping.new(rng = rng)
  let
    quicClient = newTcpQuicSwitch(rng = rng)
    tcpClient = newTcpQuicSwitch(rng = rng)
    b = newTcpQuicSwitch(rng = rng)

  # ...start and eventually stop all three switches...

  let quicRtt =
    await quicClient.pingPreferringQuic(
      pingProtocol,
      b.peerInfo.peerId,
      b.peerInfo.addrs
    )

  let tcpRtt =
    await tcpClient.pingPreferringTcp(
      pingProtocol,
      b.peerInfo.peerId,
      b.peerInfo.addrs
    )
```

Neither client initially has a muxer to B:

```text
quicClient:
  no reusable muxer
  -> QUIC-first addresses choose QUIC

tcpClient:
  no reusable muxer
  -> TCP-first addresses choose TCP
```

No `reuseConnection = false` is needed because reuse cannot happen across two
different local switches.

If both pings were performed by the same client, the second call would normally
reuse the connection created by the first and ignore the second ordering.

## When can one peer have two muxers?

Two muxers mean two underlying peer connections:

```text
ConnManager[A]
  └── Peer B
      ├── Muxer 1 -> underlying connection 1
      └── Muxer 2 -> underlying connection 2
```

This can happen through:

1. **Simultaneous bidirectional dialing.** A dials B while B dials A. Each node
   may retain one outgoing and one incoming connection.
2. **Relay-to-direct upgrade.** DCUtR establishes a direct connection while a
   relayed connection still exists.
3. **Explicit non-reuse.** A caller invokes:

   ```nim
   await switch.connect(
     peerId,
     addresses,
     reuseConnection = false
   )
   ```

4. **A replacement/race.** A new connection is stored before the old connection
   has completed removal.

Ordinary repeated application `dial` calls do not create two muxers. They
create two application streams on one selected muxer.

The connection manager's default maximum in this version is two muxers per
peer, unless configured otherwise.

### Which muxer is selected?

Without a transport constraint, `ConnManager` prefers an outgoing muxer and
falls back to an incoming one:

Source:
`vendor/nim-libp2p/libp2p/connmanager.nim`

```nim
proc selectMuxer*(
    c: ConnManager, peerId: PeerId
): Muxer =
  var mux =
    c.selectMuxer(peerId, Direction.Out)

  if mux.isNil:
    mux =
      c.selectMuxer(peerId, Direction.In)

  return mux
```

The underlying muxer store returns the first muxer matching that direction:

Source:
`vendor/nim-libp2p/libp2p/muxer_store.nim`

```nim
proc selectMuxer*(
    s: MuxerStore,
    peerId: PeerId,
    dir: Direction
): Muxer =
  s.muxed.withValue(peerId, muxers):
    for _, m in muxers[]:
      if m.connection.dir == dir:
        return m
  return nil
```

There is no TCP-versus-QUIC preference in this selection path.

## Connection and peer events

The two event levels mirror the two levels of state:

```text
ConnEvent.Connected
  one particular muxer was stored

PeerEvent.Joined
  the first muxer for that PeerId was stored
```

For example:

```text
first muxer to B:
  Connected
  Joined

second muxer to B:
  Connected
  no second Joined
```

Likewise, as detailed in [[Closing libp2p connections and streams]]:

```text
one of two muxers closes:
  Disconnected
  no Left

last muxer closes:
  Disconnected
  Left
```

Closing an application stream produces neither a peer `Left` event nor a
muxer-level `Disconnected` event.

## How Logos Storage block exchange maps onto this

Discovery establishes peer-level connectivity:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc dialPeer*(
    self: BlockExcNetwork, peer: PeerRecord
) =
  if self.isSelf(peer.peerId):
    return

  if peer.peerId in self.peers:
    return

  await self.switch.connect(
    peer.peerId,
    peer.addresses.mapIt(it.address)
  )
```

This may create TCP + Noise + Yamux, causing `Joined`, but it does not yet open
`/storage/blockexc/1.0.0`.

The block-exchange stream is opened lazily:

Source:
`storage/blockexchange/network/network.nim`

```nim
proc getOrCreatePeer(
    self: BlockExcNetwork, peer: PeerId
): NetworkPeer =
  ## Creates or retrieves a BlockExcNetwork Peer

  if peer in self.peers:
    return self.peers.getOrDefault(peer, nil)

  var getConn: ConnProvider =
    proc(): Future[Connection] =
      try:
        return await self.switch.dial(peer, Codec)
      except CancelledError as error:
        raise error
      except CatchableError as exc:
        trace "Unable to connect to blockexc peer",
          exc = exc.msg

  # ...construct and store NetworkPeer with this provider...
```

The application then keeps that particular outgoing stream in
`NetworkPeer.sendConn`:

Source:
`storage/blockexchange/network/networkpeer.nim`

```nim
proc connect*(
    self: NetworkPeer
): Future[Connection] =
  if self.connected:
    return self.sendConn

  self.sendConn = await self.getConn()
  self.trackedFutures.track(
    self.readLoop(self.sendConn)
  )
  return self.sendConn
```

This `NetworkPeer.connect` is application-level naming. It does not call
`Switch.connect`; its `getConn` normally calls `Switch.dial` and obtains an
application stream.

That distinction gives us three separate operations all casually described as
"connect":

```text
BlockExcNetwork.dialPeer
  -> Switch.connect
  -> peer-level muxer

NetworkPeer.connect
  -> Switch.dial
  -> block-exchange application stream

TCP connect
  -> lowest-level transport establishment
```

### Logos Storage flow recap

The block-exchange-specific sequence is therefore:

```text
DiscoveryEngine result
  -> BlockExcNetwork.dialPeer
     -> Switch.connect
        -> establish/reuse peer-level muxer
        -> Joined may populate NetworkPeer and PeerContext

first block-exchange send
  -> BlockExcNetwork.send
  -> NetworkPeer.send
  -> NetworkPeer.connect
  -> ConnProvider created by BlockExcNetwork.getOrCreatePeer
  -> Switch.dial(peerId, BlockExcNetwork.Codec)
     -> select existing muxer
     -> open /storage/blockexc/1.0.0 stream
  -> cache stream as NetworkPeer.sendConn
  -> attach NetworkPeer.readLoop
```

The first chain creates peer connectivity. The second chain creates and caches
the application stream which carries the actual block-exchange frames.

## Practical comparison

| Operation | Needs addresses? | May establish transport? | Opens application stream? | Negotiates application codec? | Returns stream? |
|---|---:|---:|---:|---:|---:|
| `Switch.connect(peer, addrs)` | yes | yes, if no reuse | no | no | no |
| `Switch.dial(peer, codec)` | no | no | yes | yes | yes |
| `Switch.dial(peer, addrs, codec)` | yes as fallback | yes, if no reuse | yes | yes | yes |
| `NetworkPeer.connect()` | no directly | no | yes, through `Switch.dial` | block exchange | yes/cached |

## Worked examples

### Connect first, then open two protocols

```nim
proc openBlockExchangeAndPing(
    switch: Switch,
    peerB: PeerId,
    addressesB: seq[MultiAddress],
): Future[(Stream, Stream)] {.async.} =
  await switch.connect(peerB, addressesB)

  let blockExc =
    await switch.dial(peerB, BlockExcCodec)

  let ping =
    await switch.dial(peerB, PingCodec)

  return (blockExc, ping)
```

Result:

```text
Peer B
└── one TCP + Noise + Yamux muxer
    ├── one block-exchange stream
    └── one ping stream
```

### Dial with addresses and no existing connection

```nim
proc openManifest(
    switch: Switch,
    peerB: PeerId,
    addressesB: seq[MultiAddress],
): Future[Stream] {.async.} =
  return await switch.dial(
    peerB,
    addressesB,
    ManifestCodec
  )
```

Result:

```text
establish or reuse muxer
  -> new manifest stream
  -> return manifest endpoint
```

If B was not connected, the first muxer produces `Joined`. If B already had a
muxer, only a new stream is created and there is no new `Joined`.

### Dial twice with the same codec

```nim
proc openTwoBlockExchangeStreams(
    switch: Switch, peerB: PeerId
): Future[(Stream, Stream)] {.async.} =
  let first =
    await switch.dial(peerB, BlockExcCodec)

  let second =
    await switch.dial(peerB, BlockExcCodec)

  return (first, second)
```

Result:

```text
one muxer
  ├── block-exchange stream 1
  └── block-exchange stream 2
```

`first != second`. Each remote stream causes a separate invocation of the
remote application protocol handler.

### Open TCP after QUIC already exists

Address ordering alone cannot require TCP if a reusable QUIC muxer already
exists. A deliberate second underlying connection begins with:

```nim
proc addTcpConnection(
    switch: Switch,
    peerB: PeerId,
    tcpAddress: MultiAddress,
) {.async.} =
  await switch.connect(
    peerB,
    @[tcpAddress],
    reuseConnection = false
  )
```

This creates the possibility of two muxers, subject to connection limits. A
later unconstrained `dial(peerB, codec)` still uses the connection manager's
ordinary selection rules; it does not inherently mean "use the TCP muxer."

## Compact mental model

The most reliable model is:

```text
connect
  creates or reuses a road to the peer

dial
  opens a new lane for one application protocol on a road

application caching
  may keep using the same lane for many messages

close stream
  closes one lane

disconnect
  removes all roads to that peer
```

Or in libp2p terms:

```text
Switch.connect
  -> peer-level Muxer exists

Switch.dial
  -> new application Stream exists

application read/write
  -> full-duplex traffic on that Stream

Stream.close
  -> Muxer normally survives

Switch.disconnect
  -> all peer Muxers and all their Streams close
```
