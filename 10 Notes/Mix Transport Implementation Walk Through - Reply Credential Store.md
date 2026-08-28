---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Wire Format Foundation]]"
---

# Mix Transport Implementation Walk Through - Reply Credential Store

This phase adds the initiator-side state needed to receive replies through SURBs and connects it to `MixTransport.handleRawSurbReply`. The store is now used by the `Connect` handshake described in [[Mix Transport Implementation Walk Through - Connect Handshake]].

The implementation is in `libp2p_mix_transport/reply_credentials.nim`, with focused tests in `tests/test_reply_credentials.nim`.

## Why the Initiator Stores Credentials

When the initiator asks Mix to create a SURB, Mix returns two related values:

- a public `SURB`, which the initiator sends to the destination;
- a private `ReplyCredential`, which remains on the initiator and is required to recover the eventual reply.

The destination never receives the credential. It only consumes the public SURB when sending a return frame. When the resulting `RawSurbReply` reaches the initiator, its `SURBIdentifier` selects the matching credential.

Mix understands the cryptographic contents of the credential, but MixTransport owns its lifetime. The credential exists because a transport operation is awaiting a reply, belongs to a transport session, expires according to transport policy, and must be removed together with the redundant credentials created for the same logical reply.

## Redundancy Groups

```nim
ReplyCredentialGroup* = ref object
  sessionId: PeerId
  identifiers: HashSet[SURBIdentifier]
  expiresAt: Moment
```

One return frame may be sent through several redundant SURBs. Each SURB has its own credential and identifier, but all of them represent one logical reply opportunity.

`ReplyCredentialGroup` records that relationship. The `sessionId` tells the future raw-reply path which transport session owns the reply. `identifiers` lets the store remove every redundant credential together. `expiresAt` gives the whole group one lifetime, so a partially expired group cannot be represented.

The group is a `ref object` because every indexed credential must point to the same group identity. Consuming the group through any one entry must affect the group seen through all other entries.

## Indexed Entries

```nim
StoredReplyCredential* = object
  credential*: ReplyCredential
  group*: ReplyCredentialGroup
```

The store is indexed by `SURBIdentifier`:

```nim
credentials: Table[SURBIdentifier, StoredReplyCredential]
```

A raw reply therefore needs one table lookup to obtain both the opaque Mix credential and the transport group that owns it. The receive path now uses them as follows:

```text
RawSurbReply.identifier
        ↓
ReplyCredentialStore.get
        ↓
Mix.recoverReply(stored.credential, rawReply)
        ↓ success
ReplyCredentialStore.consume(stored.group)
        ↓
dispatch the recovered transport frame to stored.group.sessionId
```

`ReplyCredentialStore.recoverReply` returns `Opt.none` when the identifier is unknown. Before recovery, the transport separately checks whether the identifier is retired. A genuinely unknown identifier produces `RawSurbReplyDisposition.Unhandled`, leaving the reply available to Mix's embedded connection path. A retired identifier belongs to a reply group that this transport already completed or cancelled, so the transport returns `Handled` without recovery or fallback.

When the identifier is known, the transport owns the reply and returns `Handled` even if recovery fails. A Sphinx recovery failure leaves the group intact because another redundant packet may be valid. If Sphinx recovery succeeds but the recovered Mix payload is malformed, the group is consumed: every redundant SURB carries the same logical payload, so another copy cannot repair its encoding.

## Atomic Group Registration

```nim
store.addGroup(sessionId, credentials)
```

`addGroup` validates the complete input before inserting anything. It rejects an empty group, an empty identifier, a duplicate within the new group, an identifier already present as an active credential, an identifier that remains retired from an earlier group, or a group that would exceed the configured capacity. Rejecting an identifier while its tombstone is active prevents the raw reply handler from mistaking a new reply for a late packet from the earlier group.

Only after every check succeeds does it create the shared group and insert its credentials. A failed registration therefore cannot leave half a redundancy group in the store.

The public SURBs corresponding to these credentials are handled separately by the wire boundary described in [[Mix Transport Implementation Walk Through - Wire Format Foundation]]. `createReplyGroups` registers each private credential group locally and places only the serialized public SURBs in `Connect`, `OpenStream` or `Refill`.

## Capacity Without Eviction

```nim
if credentials.len > store.maxCredentials - store.credentials.len:
  return err("reply credential store is at capacity")
```

The store refuses a new group when it lacks capacity. It does not evict an existing credential because that credential may be the only way to recover an unrelated in-flight reply. The caller can then wait, apply backpressure, or fail the operation without damaging another session.

The subtraction form avoids overflowing an addition such as `currentLen + newLen`.

## Expiry

Each group receives one deadline when it is registered:

```nim
expiresAt: now + store.ttl
```

`get` checks this deadline directly. Once the deadline is reached, the credential is no longer returned even if a cleanup sweep has not physically removed its table entry yet. This separates correctness from cleanup timing: an expired reply cannot be accepted merely because the process has been idle and no sweep has run.

`purgeExpired` performs physical cleanup for active credentials and retired identifiers. It first collects expired entries and then removes them because a Nim `Table` must not be modified while it is being iterated.

The default TTL is currently thirty minutes, matching the existing Mix credential lifetime while the transport policy is developed. The transport will eventually ensure that this lifetime does not exceed Mix's replay-tag lifetime.

## Group Consumption

```nim
store.consume(group)
```

After the initiator successfully recovers one reply, `consume` removes every credential in the redundancy group and records each identifier as retired until the group's original expiry time. A later redundant packet therefore remains attributable to the completed group even though its private credential is no longer available. `MixTransport.handleRawSurbReply` detects that retired identifier and returns `Handled` immediately.

Consumption is idempotent. A redundant reply that was already in flight may reach the receive path after the winning copy, and teardown may race with normal completion. Repeating `consume` in either case is harmless.

## Retired Identifiers

Retired identifiers are short-lived tombstones, not credentials. They contain only a `SURBIdentifier` and the expiry time inherited from the corresponding `ReplyCredentialGroup`. They do not allow another reply to be recovered.

`isRetiredIdentifier` is a pure query. It returns true only when the identifier is present and its deadline has not passed; it never deletes an entry. Cleanup happens explicitly through `purgeExpired`, so a caller does not have to expect an `is*` operation to mutate the store.

The store keeps only a limited number of retired identifiers so that late-reply tracking cannot consume memory without limit. This limit applies only to retired identifiers; active reply credentials have a separate capacity and are never removed to make room for a tombstone.

When the retired-identifier table is full, the store compares the new identifier's expiry time with the expiry times of the identifiers already stored. It keeps the identifiers that will remain valid for the longest time and discards the one whose expiry time comes first. The discarded identifier may therefore be either an existing tombstone or the new identifier. For example, if the new identifier expires sooner than every identifier already stored, the store simply does not add it.

This policy uses the available capacity for the identifiers that can suppress late packets for the longest remaining period. Evicting a tombstone only removes that additional late-packet suppression; it does not remove a credential needed to recover an outstanding reply.

`removeSession(sessionId)` uses the same retirement rule during cancellation or teardown. It first purges entries that are already expired, then removes the still-active credentials owned by the selected session and retires their identifiers until each group's deadline. Groups belonging to the same transport session need not share a deadline because SURB refill can create them at different times.

## Tests

The tests create real Mix `ReplyCredential` values by generating a minimal one-hop return path and calling `createSURB`. To exercise successful recovery, the helper builds a normal empty-codec Mix reply, pads it to the fixed Sphinx message size with `addPadding`, sends it through the SURB with `useSURB`, processes the return hop, and passes the resulting `RawSurbReply` to the store. Mix does not add fragmentation metadata or a sender-derived sequence number; messages that do not fit in one Sphinx packet are rejected. This avoids constructing an opaque credential or recovered reply by assigning private cryptographic fields.

The tests establish the essential behavior:

- consuming the group through one successful reply removes both redundant credentials, retires both identifiers, and can safely be repeated;
- a full store rejects a new group while preserving the credentials already in flight;
- retired identifiers are independently bounded, become semantically inactive at their deadline, and remain stored until an explicit purge;
- removing one session retires its active identifiers while preserving another session's credentials;
- an expired credential is invisible before `purgeExpired` removes its table entry;
- an unknown identifier remains available to Mix's embedded fallback path;
- a valid reply returns its transport `sessionId` and payload, then consumes the group;
- Sphinx corruption retains the group for another redundant copy;
- a cryptographically recovered but malformed Mix payload consumes the group.

`MixTransport` owns one `ReplyCredentialStore`, clears it during shutdown, and uses it from its registered raw-reply callback. Recovered bytes are decoded as a `MixTransportFrame`, and the frame's `sessionId` must match the session recorded by the credential group before the frame is dispatched to the live session.
