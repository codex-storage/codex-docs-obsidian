---
related:
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Implementation Walk Through - Connect Handshake]]"
  - "[[Mix Transport Implementation Walk Through - Stream Establishment Round Trip]]"
  - "[[Mix Transport Implementation Walk Through - Bounded Data Flow]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
---

This walk-through follows the implemented hybrid SURB replenishment path from the initial `ConnectAck` credit advertisement through proactive supply, recipient-initiated refill, supply retransmission and complete-starvation recovery. The transport sends and stores individual SURBs. A redundancy batch exists only while the recipient sends several copies of one reverse frame.

The implementation is divided between three files:

- `libp2p_mix_transport/wire.nim` defines numbered supply frames and the absolute supply snapshot.
- `libp2p_mix_transport/sessions.nim` owns recipient queue capacity, received-sequence state, initiator retransmission state and the supplier task.
- `libp2p_mix_transport/transport.nim` creates SURBs, sends supply and probes, handles inbound supply, and connects the refill path to ordinary reverse traffic.

## 1. The Connect Handshake Establishes Initial Credit

The session initiator attaches four unnumbered bootstrap SURBs to `Connect`. The session recipient stores all valid bootstrap SURBs, removes two for the `ConnectAck` redundancy batch, and retains two in the session queue.

After removing the `ConnectAck` batch, `handleConnect` initializes numbered supply accounting:

```nim
proc handleConnect(
    self: MixTransport, frame: MixTransportFrame
): Future[void] {.async: (raises: [CancelledError]).} =
  # Validation and session creation precede this excerpt.
  session.addReceivedSurbs(decodedSurbs).isOkOr:
    return

  var replyBatch = session.takeReceivedSurbs(DefaultReplySurbRedundancy).valueOr:
    return
  session.initializeSurbSupply().isOkOr:
    return
  var acknowledgement = MixTransportFrame(
    version: MixTransportVersion,
    sessionId: frame.sessionId,
    kind: FrameKind.ConnectAck,
  )
  session.attachSurbSupplySnapshot(acknowledgement)
```

`initializeSurbSupply` calculates the number of currently empty queue positions. The resulting value becomes the first exclusive supply limit:

```nim
proc initializeSurbSupply*(session: TransportSession): Result[void, string] =
  if session.role != SessionRole.Recipient:
    return err("only recipient sessions advertise SURB supply state")
  if session.surbSupplyInitialized:
    return err("SURB supply state is already initialized")
  if session.receivedSurbs.len > session.recipientSurbCapacity:
    return err("recipient SURB capacity is already exceeded")

  session.surbSupplyLimit =
    SurbSupplySequence(session.recipientSurbCapacity - session.receivedSurbs.len)
  session.surbSupplyInitialized = true
  ok()
```

With the default capacity of sixteen, two bootstrap SURBs remain after the `ConnectAck` batch is removed. The initial limit is therefore fourteen, authorizing supply sequences zero through thirteen. Accepting those fourteen SURBs fills the queue to sixteen.

The recipient attaches three fields as one absolute snapshot:

```nim
proc attachSurbSupplySnapshot(
    session: TransportSession, frame: var MixTransportFrame
) =
  let snapshot = session.surbSupplySnapshot()
  frame.surbSupplyReceiveBase = Opt.some(snapshot.receiveBase)
  frame.surbSupplyAcknowledgementBitmap =
    Opt.some(snapshot.acknowledgementBitmap)
  frame.surbSupplyLimit = Opt.some(snapshot.supplyLimit)
```

The receive base and bitmap initially report that no numbered SURB has arrived. The limit grants the first numbered supply credit.

## 2. ConnectAck Starts the Initiator's Supplier Task

Every recovered reverse frame first applies a supply snapshot when the frame carries one. `ConnectAck` then establishes the initiator-side session and starts its supplier:

```nim
proc handleReplyFrame(
    self: MixTransport, frame: MixTransportFrame
): Future[void] {.async: (raises: [CancelledError]).} =
  let session = self.sessions.get(frame.sessionId).valueOr:
    return

  if not session.applySurbSupplySnapshot(frame):
    return
  if frame.surbSupplyReceiveBase.isSome:
    session.noteReverseActivity(self.surbStatusProbeInterval)

  case frame.kind
  of FrameKind.ConnectAck:
    if session.role != SessionRole.Initiator or
        session.state != SessionState.Pending:
      return
    session.establish()
    self.startSurbSupplier(session)
  # Other reply kinds follow.
```

`applySurbSupplySnapshot` validates the snapshot, removes public serializations acknowledged by the receive base or bitmap, and advances the initiator's view of the recipient's receive base and supply limit. The procedure signals `surbSupplyStateChanged` when any of those state changes can affect the supplier.

`startSurbSupplier` creates one long-lived task and records that task on the session:

```nim
proc startSurbSupplier(
    self: MixTransport, session: TransportSession
) {.gcsafe, raises: [].} =
  let task = self.runSurbSupplier(session)
  if not task.finished:
    session.setSurbSupplierTask(task)
```

The task belongs to the session because supply sequences, reply credentials and the recipient queue all share the session lifetime.

## 3. Absolute Credit Bounds New Supply

The initiator stores the next unused supply sequence, the latest absolute limit reported by the recipient and the latest supply receive base. `availableSurbSupplySlots` combines the advertised credit with the fixed 256-position receive window:

```nim
func availableSurbSupplySlots*(session: TransportSession): int =
  if session.role != SessionRole.Initiator:
    return 0
  let nextSequence = session.nextSurbSupplySequence.valueOr:
    return 0
  let upperBound = min(
    uint64(session.remoteSurbSupplyLimit),
    uint64(session.remoteSurbSupplyReceiveBase) + uint64(SurbSupplyWindow),
  )
  if uint64(nextSequence) >= upperBound:
    return 0
  int(upperBound - uint64(nextSequence))
```

The absolute limit prevents the initiator from overflowing the recipient's queue. The receive-window limit prevents an old missing sequence from causing unbounded duplicate-tracking and retransmission state.

The default constructor enables proactive supply:

```nim
proc newMixTransport*(
    mix: MixProtocol,
    # Other options omitted.
    enableProactiveSurbReplenishment = true,
    recipientSurbCapacity = DefaultRecipientSurbCapacity,
): MixTransport
```

When proactive supply is disabled, the supplier task remains present but does not create new SURBs until the recipient sends a refill request. Keeping one task for both policies avoids separate push and pull state machines.

## 4. The Supplier Creates Numbered SURBs

`runSurbSupplier` checks whether proactive policy or a recipient request currently calls for new supply. If credit is available, the task creates at most four SURBs for the next frame:

```nim
proc runSurbSupplier(
    self: MixTransport, session: TransportSession
) {.async: (raises: [CancelledError]), gcsafe.} =
  defer:
    session.clearSurbSupplierTask()

  session.noteReverseActivity(self.surbStatusProbeInterval)
  while session.state == SessionState.Established:
    session.clearSurbSupplyStateChanged()

    let shouldCreateSupply =
      self.proactiveSurbReplenishmentEnabled or session.isSurbSupplyRequested
    if shouldCreateSupply and session.availableSurbSupplySlots > 0:
      let count = min(MaxSurbSupplyPerFrame, session.availableSurbSupplySlots)
      if await self.createAndSendSurbSupply(session, count):
        continue

    # Retransmission, status probing and event waiting follow.
```

`createAndSendSurbSupply` creates each public SURB together with its private credential. `registerSurbSupply` assigns consecutive sequence numbers and retains every public serialization before the frame is submitted:

```nim
proc createAndSendSurbSupply(
    self: MixTransport, session: TransportSession, count: int
): Future[bool] {.async: (raises: [CancelledError]).} =
  let destination = session.destination.valueOr:
    return false
  let prepared = self.createReplySurbs(
    destination, session.sessionId, count
  ).valueOr:
    debug "Could not create proactive SURBs", sessionId = session.sessionId, error
    return false
  let firstSequence = session.registerSurbSupply(prepared.encoded).valueOr:
    self.retireReplyCredentials(prepared.credentials)
    debug "Could not register proactive SURBs", sessionId = session.sessionId, error
    return false
  await self.sendSurbSupply(session, firstSequence, prepared.encoded)
  true
```

Registration happens before asynchronous submission so that a fast reverse snapshot cannot acknowledge a sequence that the initiator has not recorded. A registration failure retires the newly created credentials because the recipient cannot possess those unsent SURBs.

`sendSurbSupply` places the first sequence and serialized SURBs in one forward frame. After the send attempt finishes, the procedure schedules a retransmission deadline for every entry that is still pending:

```nim
proc sendSurbSupply(
    self: MixTransport,
    session: TransportSession,
    firstSequence: SurbSupplySequence,
    encodedSurbs: seq[seq[byte]],
): Future[void] {.async: (raises: [CancelledError]).} =
  let frame = MixTransportFrame(
    version: MixTransportVersion,
    sessionId: session.sessionId,
    kind: FrameKind.SurbSupply,
    firstSurbSequence: Opt.some(firstSequence),
    surbs: encodedSurbs,
  )
  let sent = await self.sendStreamFrame(session, frame)
  if sent.isErr:
    debug "Could not send SURB supply",
      sessionId = session.sessionId, firstSequence, error = sent.error
  session.scheduleSurbSupplyRetransmission(
    firstSequence, encodedSurbs.len, self.surbSupplyRetransmissionTimeout
  )
```

The private credentials stay in `ReplyCredentialStore`. The public serializations stay in `pendingSurbSupply` only until the recipient acknowledges their supply sequences.

## 5. The Recipient Accepts Every Valid SURB Independently

An ordinary Mix delivery containing `SurbSupply` reaches `handleSurbSupply`. The handler derives the sequence number of each attached SURB from `firstSurbSequence` and the SURB's position in the frame. The handler then decodes each serialized SURB independently and passes every valid value to the session:

```nim
proc handleSurbSupply(
    self: MixTransport, frame: MixTransportFrame
) {.gcsafe, raises: [].} =
  # Session and role checks precede this excerpt.
  let firstSequence = frame.firstSurbSequence.get()
  for index, encodedSurb in frame.surbs:
    let sequence = firstSequence + SurbSupplySequence(index)
    let surb = encodedSurb.deserializeSurb().valueOr:
      continue
    discard session.acceptSurbSupply(sequence, surb)
```

`acceptSurbSupply` rejects a sequence below the receive base as a duplicate, rejects a sequence outside the advertised credit or receive window, and rejects new supply when the queue is already at its hard capacity. A newly accepted SURB is inserted into the queue and represented in the acknowledgement bitmap:

```nim
proc acceptSurbSupply*(
    session: TransportSession,
    sequence: SurbSupplySequence,
    surb: sink SURB,
): SurbSupplyDisposition =
  # Duplicate, window and capacity checks precede this excerpt.
  session.receivedSurbs.addLast(move(surb))
  session.surbSupplyAcknowledgementBitmap.setBitmapBit(offset)
  while session.surbSupplyAcknowledgementBitmap.bitmapContains(0):
    session.shiftSupplyBitmap()
    inc session.surbSupplyReceiveBase
  if session.receivedSurbs.len > ReplyRefillLowWatermarkSurbs:
    session.nextRefillRequestAt = Opt.none(Moment)
  session.replyCapacityStateChanged.fire()
  SurbSupplyDisposition.Accepted
```

The queue and acknowledgement state intentionally record different facts. Removing a SURB from `receivedSurbs` consumes one return path, but the receive base or bitmap continues recording that its supply sequence arrived. A later retransmission of the same one-use SURB is therefore rejected as a duplicate.

## 6. Reverse Sends Advertise Consumption

The recipient authorizes replacement supply at the moment it removes SURBs from the queue. `takeReceivedSurbs` increments the absolute limit by the number removed:

```nim
proc takeReceivedSurbs*(
    session: TransportSession, count: int
): Result[seq[SURB], string] =
  # Role, availability and sequence-space checks precede this excerpt.
  var surbs = newSeqOfCap[SURB](count)
  for _ in 0 ..< count:
    surbs.add(session.receivedSurbs.popFirst())
  if session.surbSupplyInitialized:
    session.surbSupplyLimit += SurbSupplySequence(count)
  ok(surbs)
```

The recipient branch of `sendStreamFrame` removes a temporary redundancy batch, attaches the resulting current snapshot and then encodes the frame. Consequently, the same reverse frame that consumes SURBs reports the new replacement credit:

```nim
proc sendStreamFrame(
    self: MixTransport,
    session: TransportSession,
    frame: MixTransportFrame,
): Future[Result[void, string]] {.async: (raises: [CancelledError]).} =
  case session.role
  of SessionRole.Initiator:
    # Forward Mix send omitted.
  of SessionRole.Recipient:
    await session.acquireReplySend()
    defer:
      session.releaseReplySend()
    (await self.ensureUnreservedSurbs(session)).isOkOr:
      return err(error)
    var replyBatch = session.takeUnreservedSurbs(
      DefaultReplySurbRedundancy
    ).valueOr:
      return err(error)
    var replyFrame = frame
    session.attachSurbSupplySnapshot(replyFrame)
    let payload = replyFrame.encode().valueOr:
      return err("could not encode " & $frame.kind & " frame: " & error)
    (await self.sendWithSurbRedundancyBatch(replyBatch, payload)).isOkOr:
      return err("could not send " & $frame.kind & " frame: " & error)
    (await self.requestRefill(session)).isOkOr:
      return err(error)
  ok()
```

The reply-send lock keeps concurrent recipient Data, ACK and control sends from selecting the same queued SURBs or encoding a snapshot before another send has finished updating the limit.

## 7. A Low Queue Sends an Urgent Refill Request

Ordinary reverse traffic needs two SURBs and must leave the four-SURB control reserve intact. `ensureUnreservedSurbs` waits until the queue contains at least six SURBs. While the count remains below six, the procedure sends a refill request when its retry deadline is due and otherwise waits for either arriving supply or the next deadline:

```nim
proc ensureUnreservedSurbs(
    self: MixTransport, session: TransportSession
): Future[Result[void, string]] {.async: (raises: [CancelledError]).} =
  while session.receivedSurbCount <
      ReplyControlReserveSurbs + DefaultReplySurbRedundancy:
    session.clearReplyCapacityStateChanged()
    (await self.requestRefill(session)).isOkOr:
      return err(error)
    if session.receivedSurbCount <
        ReplyControlReserveSurbs + DefaultReplySurbRedundancy:
      let waitTime = session.timeUntilNextRefillRequest()
      if waitTime > ZeroDuration:
        discard await session.waitForReplyCapacityStateChange().withTimeout(waitTime)
  ok()
```

`requestRefill` removes two SURBs for the request's own redundancy batch, increments the limit through `takeReceivedSurbs`, attaches the new absolute snapshot and schedules the next permitted request time:

```nim
proc requestRefill(
    self: MixTransport, session: TransportSession
): Future[Result[void, string]] {.async: (raises: [CancelledError]).} =
  if not session.refillRequestDue:
    return ok()

  var replyBatch = session.takeReceivedSurbs(
    DefaultReplySurbRedundancy
  ).valueOr:
    session.scheduleNextRefillRequest(self.refillRequestTimeout)
    if self.proactiveSurbReplenishmentEnabled:
      return ok()
    return err("could not reserve SURBs for refill: " & error)
  var request = MixTransportFrame(
    version: MixTransportVersion,
    sessionId: session.sessionId,
    kind: FrameKind.RefillRequest,
  )
  session.attachSurbSupplySnapshot(request)
  let payload = request.encode().valueOr:
    return err("could not encode RefillRequest: " & error)
  session.scheduleNextRefillRequest(self.refillRequestTimeout)
  (await self.sendWithSurbRedundancyBatch(replyBatch, payload)).isOkOr:
    return err("could not send RefillRequest: " & error)
  ok()
```

The request carries neither a quantity nor a request identifier. On the initiator, `handleReplyFrame` first applies the absolute snapshot and then calls `requestSurbSupply`:

```nim
of FrameKind.RefillRequest:
  if session.role == SessionRole.Initiator and
      session.state == SessionState.Established:
    session.requestSurbSupply()
```

`requestSurbSupply` sets one boolean urgency condition and signals the supplier task. Duplicate requests remain harmless because they repeat absolute state and set the same boolean value. The supplier clears the urgency condition after it has used all currently available credit.

## 8. Unacknowledged Supply Is Retransmitted

`pendingSurbSupply` retains one public serialization, its private credential's identifier and one optional deadline for every unacknowledged supply sequence. Retaining the identifier lets the supplier verify that the matching private credential remains active before retransmitting the public SURB. `takeDueSurbSupplyRetransmission` selects the earliest due entry and clears its deadline before returning it to the asynchronous sender:

```nim
proc takeDueSurbSupplyRetransmission*(
    session: TransportSession,
    now: Moment = Moment.now(),
): Opt[
  tuple[
    sequence: SurbSupplySequence,
    encodedSurb: seq[byte],
    credentialIdentifier: SURBIdentifier,
  ]
] =
  # The earliest due entry is selected above this excerpt.
  let sequence = selectedSequence.valueOr:
    return Opt.none(tuple[sequence: SurbSupplySequence, encodedSurb: seq[byte]])
  session.pendingSurbSupply.withValue(sequence, pending):
    pending.nextRetransmissionAt = Opt.none(Moment)
    return Opt.some((sequence: sequence, encodedSurb: pending.encodedSurb))
  Opt.none(tuple[sequence: SurbSupplySequence, encodedSurb: seq[byte]])
```

Before sending the selected serialization, `runSurbSupplier` removes expired credentials and expired retired-identifier records from `ReplyCredentialStore`. The supplier then looks up `credentialIdentifier`; `get` returns no value when the credential is absent. The supplier removes the pending serialization in that case, because the initiator could no longer recover a reply sent through that SURB:

```nim
let retransmission = session.takeDueSurbSupplyRetransmission()
if retransmission.isSome:
  let value = retransmission.get()
  discard self.replyCredentials.purgeExpired()
  if self.replyCredentials.get(value.credentialIdentifier).isNone:
    session.removePendingSurbSupply(value.sequence)
    continue
  await self.retransmitSurbSupply(
    session, value.sequence, value.encodedSurb
  )
```

When the credential remains active, the supplier sends that single serialized SURB with its original sequence and schedules the next deadline after the attempt. If a reverse snapshot acknowledges the sequence during the send, snapshot application removes the table entry. `scheduleSurbSupplyRetransmission` uses `withValue`, so completion does not recreate an entry that has already been removed.

The retransmitted public SURB still corresponds to the original private credential. No additional credential is created or registered, and retransmission never extends the credential's original TTL.

## 9. Status Probes Repair Complete Starvation

Every recovered frame containing a supply snapshot calls `noteReverseActivity`, which moves the next status-probe deadline forward. If proactive replenishment is enabled and that deadline expires without another reverse snapshot, the initiator calls `sendSurbStatusProbe`.

The probe creates two fresh SURBs for its response and sends them directly inside a forward `SurbStatusProbe` frame:

```nim
proc sendSurbStatusProbe(
    self: MixTransport, session: TransportSession
): Future[void] {.async: (raises: [CancelledError]).} =
  let destination = session.destination.valueOr:
    return
  let prepared = self.createReplySurbs(
    destination, session.sessionId, DefaultReplySurbRedundancy
  ).valueOr:
    return
  let probe = MixTransportFrame(
    version: MixTransportVersion,
    sessionId: session.sessionId,
    kind: FrameKind.SurbStatusProbe,
    surbs: prepared.encoded,
  )
  (await self.sendStreamFrame(session, probe)).isOkOr:
    self.retireReplyCredentials(prepared.credentials)
    return
```

`handleSurbStatusProbe` runs on the recipient. The handler decodes exactly two usable SURBs, attaches the current snapshot to `SurbStatus` and sends the status through those SURBs immediately. The probe SURBs never enter `receivedSurbs` and therefore do not depend on or change ordinary queue credit.

The initiator applies `SurbStatus` through the same snapshot path used by Data, ACK and `RefillRequest`. If the recipient consumed SURBs without successfully reporting the resulting higher limit, the status response exposes that credit and wakes the supplier.

## 10. Session Shutdown Owns Supplier Teardown

`TransportSession.shutdown` wakes the supplier, detaches its task reference, cancels the task and waits for completion before shutting down the session's streams:

```nim
proc shutdown*(
    session: TransportSession
): Future[void] {.async: (raises: []).} =
  session.state = SessionState.Closed
  session.established.fire()
  session.replyCapacityStateChanged.fire()
  session.surbSupplyStateChanged.fire()
  let surbSupplierTask = session.surbSupplierTask
  session.surbSupplierTask = nil
  if not surbSupplierTask.isNil:
    await noCancel surbSupplierTask.cancelAndWait()
  # Stream shutdown follows.
```

The session owns this task because no supply state may outlive the session identifier. Transport teardown subsequently removes the session's reply credentials from `ReplyCredentialStore`.

## 11. Tests Cover Both Hybrid Paths

`tests/test_sessions.nim` exercises the supply state without a live network. The tests verify queue capacity, duplicate suppression, out-of-order receipt, absolute credit growth, bitmap acknowledgements, retained public serialization and retransmission deadlines.

`tests/test_wire.nim` verifies the fixed-size snapshot, numbered individual SURBs, malformed-field rejection and the Data payload bound after snapshot overhead is included.

`tests/test_connect.nim` runs the transport through a live five-node Mix topology. With the default policy, the test waits for proactive numbered supply to fill the recipient's advertised capacity before opening a stream. A second run disables proactive replenishment; the recipient then consumes its bootstrap return paths to send `RefillRequest`, the request wakes the initiator's supplier, and the application request/response exchange completes through the restored supply.
