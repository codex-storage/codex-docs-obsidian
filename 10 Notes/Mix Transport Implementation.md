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

### Initializing Mix Protocol

`StorageServer.start` already provides conditional Mix Protocol initialization for the purpose of DHT queries over Mix. More specifically, `config.mixEnabled`, `config.mixPoolJson`, and `config.mixPool` can be reused also for blocks exchange protocol.

At the end of the `if s.config.mixEnabled:` code path, `mixProto` holds fully initialized instance of the `MixProtocol`. This is the right moment create an instance of the `MixTransport`, passing `mixProto` as a constructor argument. We set the instance of the `MixTransport` as optional on the `BlockExcNetwork.mixTransport`. If it is set, it means that Mix is enabled and should be used to carry the block exchange and manifest protocols.

Initially I thought that `MixTransport` will not be another libp2p protocol on its own, but because of how current Mix implementation works this may be the easiest if not the only option. Mix implementation always needs to send the data to a concrete protocol: if one is not mounted on the `Switch`, we will get an error:

**libp2p_mix/exit_layer.nim:**

```nim
var hasHandler: bool = false
for index, handler in enumerate(self.switch.ms.handlers):
  if codec in handler.protos:
	try:
	  hasHandler = true
	  await handler.protocol.handler(exitConn, codec)
	except CatchableError as e:
	  error "Error during execution of MixProtocol handler: ", err = e.msg

if not hasHandler:
  error "Handler doesn't exist", codec = codec
  return
```

Thus, `MixTransport` will be having its own protocol - `MixTransportProtocol`. 

We add the MixTransport initialization code to `storage/storage.nim`, in `StorageServer.start` under the `s.config.mixEnabled` code path where we initialize Mix Transport passing the already initialized instance of `MixProtocol` to it:

```nim
proc startMixTransport*(s: StorageServer, mixProto: MixProtocol) {.async: (raises: [CancelledError, StorageError]).} =
  if not s.config.mixEnabled or mixProto.isNil:
    return

  let switch = s.storageNode.switch

  let mixTransport = newMixTransport(switch, mixProto)
  (await mixTransport.start()).isOkOr:
    raise newException(StorageError, "Failed to start Mix transport: " & error.msg)
  s.storageNode.engine.network.mixTransport = some(mixTransport)
```

As we see, it also sets the  `mixTransport` optional on the instance of `BlockExcNetwork`, which is defined as:

```nim
mixTransport*: Option[MixTransport] = none(MixTransport)
```

With this, the `BlockExcNetwork` can easily detect if Mix is enabled.

### Initiating outgoing connections

The party that wants to connect to other peers anonymously, will naturally avoid using the libp2p-level `Switch.connect` or `Switch.dial` operation: these are irrelevant in the Mix environment as connecting to a remote peer using a direct libp2p connection would de-anonymize the original sender. Yet, before adding the discovered provider to list of peer, it might be desired to established an anonymous connection with the discovered provider in order to confirm that this peer is reachable before attempting regular block exchange traffic with that peer.

To facilitate this, `MixTransport` provides its own `connect` operation. `connect` will attempt an anonymous round-trip exchange with the remote peer, sending it a number of SURBs that fit within a single Sphynx block. The remote provider will use some of these SURBs to acknowledge its reachability.

> To remote provider will use a number of SURBs to establish some redundancy in order to 
> increase the chance that at least one packet arrives at the original sender.

This initial exchange will be of a simple request-response type, using the default Mix protocol implementation.

After successful exchange, the originating `peerId` can be added to the list of network peers in the same way as it is done for the regular block exchange protocol.

Recall there are two local stores keeping track of the `peerId`: one on the network level and one on the engine level. Both are normally populated in response to the `PeerEventKind.Joined` event handlers. Correspondingly, a peer is normally removed from the stores in response to the `PeerEventKind.Left` event. Explicit removal via `BlockExcNetwork.dropPeer` delegates to the switch, which in turn which cause the `PeerEventKind.Left` event to be triggered. 

We mention it here as it will be important to properly manege the `peerId` lifecycle also in the Mix Transport as the original  `PeerEventKind.Joined` and `PeerEventKind.Left` will not be triggered in this case (see also `excludedPeers: HashSet[PeerId]` property on `BlockExcNetwork`).

For the peer initiating the connection (the original sender in the Mix terms), the very first connection to the remote provider will be caused by the `discoveryTaskLoop`, where for each discovered `peerId` that advertises the given CID

```nim
  proc discoveryTaskLoop(b: DiscoveryEngine) {.async: (raises: []).} =
    ## Run discovery tasks
    ## Peer availability is tracked per-download in DownloadContext.swarm.
    ## This loop just runs discovery for CIDs that are queued.

    try:
      while b.discEngineRunning:
        let cid = await b.discoveryQueue.get()

        if cid in b.inFlightDiscReqs:
          trace "Discovery request already in progress", cid
          continue

        trace "Running discovery task for cid", cid

        let request = b.discovery.find(cid)
        b.inFlightDiscReqs[cid] = request
        storage_inflight_discovery.set(b.inFlightDiscReqs.len.int64)

        defer:
          b.inFlightDiscReqs.del(cid)
          storage_inflight_discovery.set(b.inFlightDiscReqs.len.int64)

        if (await request.withTimeout(DefaultDiscoveryTimeout)) and
            peers =? (await request).catch:
          let dialed = await allFinished(peers.mapIt(b.network.dialPeer(it.data)))

          for i, f in dialed:
            if f.failed:
              await b.discovery.removeProvider(peers[i].data.peerId)
    except CancelledError:
      trace "Discovery task cancelled"
      return

    info "Exiting discovery task runner"

```

### Processing Incoming Frames

The Mix Transport protocol helper will use the provided libp2p `Connection` object to read the incoming frames, process them (asynchronously) write back to it using the `sessionId` that we will use as the Mix Transport equivelnt of the regular connection's `peerId`.
