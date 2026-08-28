---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Reply Credential Store]]"
---

# Mix Transport Implementation Walk Through - Session Registry

This note describes the transport-owned registry used by the implemented `Connect` handshake, stream registry, received-SURB supply and refill state. `SessionStore` provides both the frame-routing lookup by pseudonym and the `connect(destination)` reuse lookup by real destination.

The implementation is in `libp2p_mix_transport/sessions.nim`. Its focused tests are in `tests/test_sessions.nim`.

## What a Transport Session Represents

A `TransportSession` represents a long-lived relationship between two MixTransport endpoints. Once the session is established, the endpoints can open multiple virtual streams through it. Closing one of those streams does not close the session; the session remains available until MixTransport explicitly drops the peer.

Each session has a random `sessionId` represented as a valid libp2p `PeerId`. The initiator generates this identifier before sending `Connect`. Both endpoints then include the same `sessionId` in transport frames so that an incoming frame can be routed to the correct local session.

Although `sessionId` has the `PeerId` type, it is a session pseudonym rather than the initiator's authenticated libp2p identity. The recipient learns this pseudonym through Mix, but the anonymity provided by Mix prevents it from learning which authenticated libp2p peer initiated the session.

## What Each Endpoint Knows

The two endpoints store different information because they have different views of the anonymous connection.

On the initiator:

- `destination` is the real `PeerId` of the remote Mix node selected by the initiator.
- `sessionId` is the random pseudonym generated for the transport session.
- `peerId` returns `destination`, because higher-level protocols on the initiator already know which remote node they asked MixTransport to connect to.

On the recipient:

- `destination` is absent because this field would have to identify the anonymous initiator, whose authenticated `PeerId` is not revealed by Mix.
- `sessionId` is the pseudonym received in the initiator's `Connect` frame.
- `peerId` returns `sessionId`, because the pseudonym is the only peer identifier that the recipient can expose to higher-level protocols for this session.

This distinction lets higher-level protocols use a `PeerId` on both endpoints without claiming that the recipient has learned the initiator's real identity. The recipient must therefore treat the pseudonym as local transport identity only. It must not pass it to `Switch.connect`, `Switch.dial`, or an ordinary libp2p peer store, because those APIs expect an authenticated libp2p identity.

## Role and Establishment State

Every session records a role and an establishment state:

```nim
SessionRole = Initiator | Recipient
SessionState = Pending | Established
```

The role records which endpoint created the local session state. An initiator session was created by the endpoint that started the handshake. A recipient session was created by the endpoint that received the corresponding `Connect` frame.

Both roles initially use `Pending`, but the state has a different immediate meaning on each endpoint. On the initiator, `Pending` means that `Connect` has not yet produced a valid `ConnectAck`. On the recipient, it means that the incoming `Connect` is being accepted and its acknowledgement is being prepared. The recipient calls `establish` after submitting `ConnectAck`; the initiator calls it after recovering a valid matching acknowledgement.

`Established` means that the transport relationship is ready to carry virtual streams. The state belongs to `TransportSession` because it describes the relationship between the two transport endpoints, rather than the lifetime of any individual stream opened through that relationship.

The type keeps the state that must survive individual stream operations:

```nim
TransportSession* = ref object
  sessionId: PeerId
  destination: Opt[PeerId]
  role: SessionRole
  state: SessionState
  established: AsyncEvent
  receivedSurbGroups: Deque[seq[SURB]]
  receivedSurbGroupsAvailable: AsyncEvent
  replySendLock: AsyncLock
  pendingRefillBatchId: Opt[uint64]
  nextRefillBatchId: uint64
  streams: Table[uint64, TransportStream]
  nextOutboundStreamId: uint64
```

The `established` event is how `connect` waits for `ConnectAck` without polling. The received-SURB queue and refill fields belong here because reply capacity is shared by all reverse traffic in the session, rather than by one virtual stream. The stream table routes a frame carrying `streamId` after the session has first been selected by `sessionId`.

## How Sessions Are Found

`SessionStore` keeps two tables because the transport needs to answer two different questions:

```text
bySessionId:   session pseudonym -> local TransportSession
byDestination: real destination -> local initiator TransportSession
```

The first table is used when a transport frame arrives. Every frame carries a `sessionId`, so MixTransport looks up that pseudonym in `bySessionId` to find the local session that should process the frame. Both initiator and recipient sessions are stored in this table because both endpoints receive frames carrying the session pseudonym.

The second table is used when the local application asks MixTransport to connect to a destination. For an initiator session, the store maps the destination's real `PeerId` to the existing session. A later `connect(destination)` call can therefore reuse that session instead of creating a second session with a new pseudonym.

The two lookup operations make that distinction explicit:

```nim
proc get*(store: SessionStore, sessionId: PeerId): Opt[TransportSession] =
  store.bySessionId.withValue(sessionId, session):
    return Opt.some(session[])
  Opt.none(TransportSession)

proc getByDestination*(
    store: SessionStore, destination: PeerId
): Opt[TransportSession] =
  store.byDestination.withValue(destination, session):
    return Opt.some(session[])
  Opt.none(TransportSession)
```

For example, `handleReplyFrame` calls `get(frame.sessionId)` because the sender placed the pseudonym in the frame. In contrast, `connect(destination)` calls `getByDestination(destination)` because its caller knows the real node it wants to reach.

Recipient sessions are not stored in `byDestination`. The recipient does not know the anonymous initiator's authenticated `PeerId`, so it has no real destination value that could serve as a key in this table. It finds a recipient session only through the `sessionId` carried by incoming frames.

## Adding a Session Without Leaving Partial State

`addInitiatorSession(destination, sessionId)` must add one session to both tables. Before changing either table, it checks that:

- `destination` and `sessionId` are not empty;
- `byDestination` does not already contain a session for `destination`; and
- `bySessionId` does not already contain a session for `sessionId`.

Only after all checks succeed does the function create the `TransportSession` and insert the same object into both tables. If either identifier is already in use, the function returns an error before performing either insertion. This prevents a failed registration from leaving one table changed while the other still describes the previous state, and it preserves the session that was registered first.

The successful registration ends with both keys referring to the same `ref object`:

```nim
let session = TransportSession(
  sessionId: sessionId,
  destination: Opt.some(destination),
  role: SessionRole.Initiator,
  state: SessionState.Pending,
  established: newAsyncEvent(),
  receivedSurbGroups: initDeque[seq[SURB]](),
  receivedSurbGroupsAvailable: newAsyncEvent(),
  replySendLock: newAsyncLock(),
  pendingRefillBatchId: Opt.none(uint64),
  nextRefillBatchId: 1,
  streams: initTable[uint64, TransportStream](),
  nextOutboundStreamId: 1,
)
store.bySessionId[sessionId] = session
store.byDestination[destination] = session
```

The recipient constructor uses the same session-owned resources, but stores `destination: Opt.none(PeerId)`, uses role `Recipient`, starts outbound stream allocation at `2`, and inserts the session only into `bySessionId`.

`addRecipientSession(sessionId)` changes only `bySessionId`. It rejects an empty pseudonym or one that is already registered, then creates a pending recipient session. It does not accept a destination because the recipient has no authenticated identity for the anonymous initiator.

## Establishing and Removing a Session

`establish(session)` changes the session state from `Pending` to `Established`. It does not move the session or change either lookup table, so the same session remains reachable by all identifiers under which it was registered.

`remove(sessionId)` begins with the lookup available on both endpoints: the session pseudonym. If no session has that pseudonym, it returns `none` and changes nothing.

When it finds an initiator session, it removes the entry from `bySessionId` and also removes the entry keyed by the real destination from `byDestination`. When it finds a recipient session, it removes only the `bySessionId` entry because no destination entry exists for that role. The function returns the removed session so that later teardown code can cancel its tasks and release the resources attached to it.

Calling `remove` again with the same `sessionId` is harmless: the first call has already removed the session, so the second call returns `none`.

`MixTransport` owns one `SessionStore`. A current `TransportSession` also owns its stream table, received SURB groups, refill batch identifier, SURB-availability event and per-session reply-send lock. Per-stream retransmission payloads live in each `TransportStream`. During shutdown, `stop` first cancels handler and flow tasks, then clears the reply credential and session stores. Coordinated runtime session teardown is still incomplete because the disconnect and reset frames are not handled yet.

## What the Tests Demonstrate

The first test creates an initiator session and verifies both views of its identity. Looking it up by `sessionId` returns the session used to route transport frames, while looking it up by the real destination returns the same object used to satisfy future `connect(destination)` calls. The test also calls `establish` and verifies the transition from `Pending` to `Established`.

The second test creates a recipient session. It verifies that the destination is absent, that `peerId` exposes the session pseudonym, and that the session can be found by `sessionId` but not through the destination table.

The third test first registers an initiator session. It then tries to reuse that destination with another pseudonym and to reuse that pseudonym for a recipient session. Both operations must fail, the store must still contain exactly one session, and both lookups must still return the original object. This test protects the rule that a rejected registration cannot partially modify the store or replace an established mapping.

The fourth test removes an initiator session by its `sessionId`. It verifies that the session disappears from both tables and that repeating the removal returns `none` without changing any other state.

## How the Handshake Uses the Registry

The implemented `Connect` handshake now uses these operations directly. The initiating MixTransport generates a fresh session pseudonym and calls `addInitiatorSession(destination, sessionId)` before sending `Connect`. Keeping the session in `Pending` state gives the returning `ConnectAck` handler a stable object to find through the `sessionId` carried by the acknowledgement.

The recipient's `Connect` handler reads the pseudonym from the frame and calls `addRecipientSession(sessionId)`. It attaches the public SURB groups received from the initiator to that session, uses the first group to send `ConnectAck`, and marks the recipient session established after at least one acknowledgement copy has been submitted successfully.

The recipient establishes the session only after the acknowledgement has been accepted for sending:

```nim
let session = self.sessions.addRecipientSession(frame.sessionId).valueOr:
  return
session.addReceivedSurbGroups(decodedGroups).isOkOr:
  discard self.sessions.remove(frame.sessionId)
  return

var replyGroup = session.takeReceivedSurbGroup().valueOr:
  discard self.sessions.remove(frame.sessionId)
  return

(await self.sendWithSurbGroup(replyGroup, acknowledgement)).isOkOr:
  discard self.sessions.remove(frame.sessionId)
  return
session.establish()
```

This order matters. If the recipient cannot decode or store the supplied SURBs, cannot reserve a group for the acknowledgement, or cannot submit any redundant acknowledgement, it removes the new session instead of retaining local state that the initiator can never confirm.

When the initiator recovers and validates `ConnectAck`, it looks up the pending session by `sessionId` and marks that same object established. A later `connect(destination)` call finds it through `byDestination` and returns the existing session instead of starting another handshake. The complete exchange is described in [[Mix Transport Implementation Walk Through - Connect Handshake]].
