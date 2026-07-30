---
related:
  - "[[Libp2p Connection Lifecycle in Logos Storage]]"
  - "[[New Logos Storage Discovery]]"
  - "[[New Logos Storage Block Exchange Protocol]]"
link: https://hackmd.io/i6vSY7gPTRqwn4XtVjIrmg
---
- Discovery: this gives us traditional `peerId`.
	- Normally, as part of the discovery process, `connect` on the switch is called, which populates the `peers` stores in both the network (`NetworkPeer`) and the engine (`PeerCtx`) in response to the `PeerEvent.Joined`:
		- we cannot do that here, because we have to use Mix to communicate with this peer
		- yet, we still populate the above mentioned `stores` with the discovered `peerId`
		- so far the behavior is identical except for calling `connect`
- Some time later, we will be trying to download some content.
- We will be selecting the candidate peers from the engine `peers` store.
	- peerId is added to the swarm
	- `self.network.request.sendWantList` - we descend to the network and will be asking for the ranges.
		- this will reach `BlockExcNetwork.send`
			- it gets the `NetworkPeer` corresponding to `peerId` from its `peers` store
			- normally, via `NetworkPeer.connect` if `self.sendConn` is not active, it will call `Switch.dial` (we should change this misleading naming to match the Switch operation)
				- we cannot do that - we are going via MIX
				- instead we have to use custom `self.sendConn = await self.getConn()`, which will get us a MIX connection from the Mix transport
				- we start reading loop on that transport MIX connection (as before)
			- we call `await conn.write(buf)` - effectively sending our `WantList` to the remote peer over MIX
- on the provider side, our MIX Transport will get `WantList` (and maybe already some SURBs)
	- obviously, there is no `peerId` at this end
	- instead we have some `sessionId`. MIX Transport assigns this `sessionId` to the underlying `Connection.peerId`
	- the provider gets the incoming connection form the MixTransport
	- normally it would already have the connection `peerId` added to its `peers` stores via "joined" Switch event (engine's peer context) and `getOrCreatePeer(peerId)` (`NetworkPeer`).
		- we will be probably creating a `NetworkPeer` but I am not yet sure if we will be adding it to the mentioned `peers` stores or to some other store.
		- read loop is started for the incoming stream
			- after sending what was to be sent, MIX Transport my decide to close that stream
			- but the corresponding `NetworkPeer` object stays.
		- we process the `WantHave` message. `BlockExcEngine.wantListHandler` needs the `peerId` to be present in its `peers` store.
			- if the provider has anything to offer, it will `sendPresence` via network (it will need the `peerId` to be in network's `peers` store).
			- this `peerId` will be MIX transport `sessionId`, which should let the MIX transport associate find the corresponding connection with the SURBs and if necessary request additional SURBs from the requester
				- this is a bit different than in the original where the outgoing messages will use a separate stream (with its corresponding `readLoop` if there is anything to read) - in the original protocol we basically do not know on which of the two stream the control messages will come - it is a bit messy...
			- in the end, the presence list should land on the requester's MIX transport side.
- back on the requester with the presence list
	- back in the incoming stream's `readLoop`
		- this is not stream really any more, it is the Mix Transport calling protocol's `handle` with its own connection object from while the `readLoop` will read as from a regular stream
		- the requester knows both Mix session id and the original provider's `peerId`.
	- as we know that some of the swarm peers have the data we want, we start requesting actual blocks
		- `self.sendWantBlocksRequest` reaches `NetworkPeer.sendWantBlocksRequest`
			- `NetworkPeer.sendConn` is (re)used - again it is a Mix Transport connection
- again at the provider via incoming Mix Transport connection
	- this time `readLoop` handles `mtWantBlocksRequest` and writes the answer directly to its `conn` (Mix Transport)
		- SURBs has to be requested as needed
- requester receives the block,
- etc...

We have a bunch of documents forming an overview of the block exchange protocol. To form some practical strategy for Mix Transport implementation here are the most importants excerpts from those document.


## What the libp2p connection manager stores

After an incoming or outgoing transport is connected, secured, and upgraded, libp2p stores its muxer in `ConnManager.muxerStore`.

The connection manager:

- indexes muxers by `PeerId`;
- tracks incoming and outgoing direction;
- prefers an outgoing muxer when selecting a connection, then falls back to an incoming muxer;
- opens new Yamux streams through the selected muxer;
- emits `ConnEvent.Connected` for every new muxed transport connection;
- emits `PeerEvent.Joined` only when the first connection for a peer is stored;
- watches the underlying connection for closure;
- emits `PeerEvent.Left` only when the last connection for that peer has disappeared.

Consequently, closing one application stream is not a peer departure. Even closing one of two physical connections is not a peer departure if another muxer for that peer remains.


## How outgoing peer connections are created

### Provider discovery for block exchange

The download worker asks `DiscoveryEngine` to find providers for the **manifest CID**. Direct provider lookup uses Discv5; it is not itself a libp2p stream.

For every returned signed peer record, `DiscoveryEngine.discoveryTaskLoop` calls:

```nim
BlockExcNetwork.dialPeer(peerRecord)
```

`dialPeer`:

1. skips the local peer ID;
2. skips a peer already present in `BlockExcNetwork.peers`;
3. calls `switch.connect(peerId, addresses)`.

`switch.connect` establishes or reuses a transport connection but deliberately does **not** negotiate an application protocol stream. With the default `reuseConnection = true`, the libp2p dialer returns immediately when a usable muxer for that peer already exists.

```nim
proc dialPeer*(self: BlockExcNetwork, peer: PeerRecord) {.async.} =
  if self.isSelf(peer.peerId):
    trace "Skipping dialing self", peer = peer.peerId
    return

  if peer.peerId in self.peers:
    trace "Already connected to peer", peer = peer.peerId
    return

  await self.switch.connect(peer.peerId, peer.addresses.mapIt(it.address))
```

Once a new muxer is stored, `PeerEvent.Joined` reaches `BlockExcNetwork.handlePeerJoined`. That creates the per-peer block-exchange objects before any block-exchange stream is necessarily open.

### Manifest fetching

Manifest fetching does not pre-connect through `BlockExcNetwork`. It directly calls:

```nim
switch.dial(peerId, addresses, "/storage/manifest/1.0.0")
```

This combined form:

1. reuses an existing muxer when possible;
2. otherwise establishes, secures, and stores a new peer connection;
3. opens a new Yamux stream;
4. negotiates the manifest codec on that stream.

### Opening a block-exchange stream

The switch-level connection may sit without a block-exchange stream until the first message needs to be sent.

`NetworkPeer.connect` checks `sendConn`:

```text
sendConn exists and is neither closed nor at EOF
  -> return the existing stream

otherwise
  -> switch.dial(peerId, "/storage/blockexc/1.0.0")
  -> store the returned stream in sendConn
  -> start a readLoop on it
```

This is a lazy, protocol-stream connection step. Because provider discovery has normally already called `switch.connect`, the codec-only `switch.dial` opens a stream over an existing muxer.

```nim
proc connect*(
    self: NetworkPeer
): Future[Connection] {.async: (raises: [CancelledError]).} =
  if self.connected:
    trace "Already connected", peer = self.id, connId = self.sendConn.oid
    return self.sendConn

  self.sendConn = await self.getConn()
  self.trackedFutures.track(self.readLoop(self.sendConn))
  return self.sendConn
```

## Incoming connection and stream creation

For an incoming transport connection, the switch:

1. accepts TCP;
2. performs the secure and multiplexing upgrade;
3. stores the muxer in the connection manager;
4. identifies the peer;
5. emits peer events;
6. accepts negotiated streams from that muxer.

When the remote peer opens `/storage/blockexc/1.0.0`, the protocol handler:

1. reads `conn.peerId`;
2. obtains or creates the shared `NetworkPeer`;
3. runs `NetworkPeer.readLoop(conn)` for the lifetime of that stream.

An incoming block-exchange stream is not assigned to `NetworkPeer.sendConn`. If the local node later needs an outgoing retained stream, it may open another block-exchange stream over the same muxer. The protocol is bidirectional at the framing level, but the implementation specifically retains the lazily opened outgoing stream as `sendConn`.

When the remote peer opens `/storage/manifest/1.0.0`, the manifest handler reads one CID, writes one `Found` or `NotFound` response, and returns. The libp2p multistream wrapper closes the stream after the protocol handler returns.

The block-exchange incoming-stream mapping is installed by
`BlockExcNetwork.init`:

```nim
proc handler(
    conn: Connection, proto: string
): Future[void] {.async: (raises: [CancelledError]).} =
  let peerId = conn.peerId
  let blockexcPeer = self.getOrCreatePeer(peerId)
  await blockexcPeer.readLoop(conn) # attach read loop

self.handler = handler
```

### Read Loop

The pending table is per `NetworkPeer`, not per stream. Therefore, if multiple read loops exist for the same peer, exit of any one of them currently fails all pending block requests for that peer. This is an implementation detail worth preserving or reconsidering when changing the transport abstraction.

### Send failure and reconnection

This whole section is relevant.

If a Protobuf write fails, `NetworkPeer.send` clears `sendConn` when it still refers to the failed stream and raises `LPStreamError`.

If a block request cannot be written or its stream ends, the pending request is removed or completed with an error. The download engine requeues work according to its failure and retry rules.

The next send calls `NetworkPeer.connect` again. If the peer's Yamux connection is still usable, this opens a replacement block-exchange stream over it. If the underlying peer connection has gone away, peer-departure cleanup removes the `NetworkPeer`, and later provider discovery may reconnect the peer from its signed addresses.

There is no explicit periodic reconnect task associated with `NetworkPeer`. 
