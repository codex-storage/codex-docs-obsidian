---
related:
  - "[[Mix Transport Implementation Walk Through]]"
  - "[[Mix Transport Design Specification]]"
  - "[[Mix Transport Implementation Walk Through - Wire Format Foundation]]"
  - "[[Mix Transport SURB Replenishment Strategy]]"
  - "[[Mix Transport Implementation Walk Through - SURB Replenishment]]"
---
This phase adds the initiator-side state needed to receive replies through SURBs and connects the state to `MixTransport.handleRawSurbReply`. The store is used by the `Connect` handshake described in [[Mix Transport Implementation Walk Through - Connect Handshake]] and by every later SURB supplied to the session recipient.

The implementation is in `libp2p_mix_transport/reply_credentials.nim`, with focused tests in `tests/test_reply_credentials.nim`.

## Why the Initiator Stores Credentials

When the initiator asks Mix to create a SURB, Mix returns two related values:

- a public `SURB`, which the initiator sends to the recipient;
- a private `ReplyCredential`, which remains on the initiator and is required to recover the eventual reply.

The recipient never receives the credential. The recipient stores the public SURB in its session queue and may later select that SURB as part of a temporary redundancy batch for one reverse frame. When the resulting `RawSurbReply` reaches the initiator, its `SURBIdentifier` selects the matching private credential.

Mix understands the cryptographic contents of the credential, but MixTransport owns its lifetime. The credential belongs to a transport session, expires according to transport policy and must be removed when its SURB is used or its session ends.

## Why Credentials Are Independent

SURBs are supplied and stored individually. The recipient decides which `N` SURBs to combine only when sending a reverse frame. The initiator cannot know that temporary selection in advance, so the reply credential store does not represent redundancy batches.

Recovering one redundant copy consumes only the credential matching that copy. Another copy can still arrive through a different SURB and be recovered with its own credential. The decoded transport frame then suppresses the repeated logical effect: Data has a stream sequence, ACK is an absolute snapshot, handshake responses act only on pending state, and `RefillRequest` carries absolute supply state.

This model may retain the credential for a lost redundant copy until expiry even after another copy succeeded. The TTL and credential capacity bound that cost. The advantage is that SURBs remain independent throughout supply and storage, and no persistent grouping has to be transmitted or reconstructed.

## Indexed Entries

Each stored entry contains one credential together with its transport ownership and lifetime:

```nim
StoredReplyCredential* = object
  credential*: ReplyCredential
  sessionId*: PeerId
  expiresAt*: Moment
```

The store indexes these entries by `SURBIdentifier`:

```nim
credentials: Table[SURBIdentifier, StoredReplyCredential]
```

A raw reply therefore requires one lookup to obtain the opaque Mix credential, the transport session that owns the reply and the credential deadline. The receive path is:

```text
RawSurbReply.identifier
        ↓
ReplyCredentialStore.get
        ↓
Mix.recoverReply(stored.credential, rawReply)
        ↓ success
ReplyCredentialStore.consume(reply.identifier)
        ↓
dispatch the recovered transport frame to stored.sessionId
```

`ReplyCredentialStore.recoverReply` returns `Opt.none` when the identifier is unknown. Before recovery, the transport separately checks whether the identifier is retired. A genuinely unknown identifier produces `RawSurbReplyDisposition.Unhandled`, leaving the reply available to Mix's embedded connection path. A retired identifier belongs to a SURB that this transport already consumed or cancelled, so the transport returns `Handled` without attempting recovery or fallback.

When the identifier is active, the transport owns the reply and returns `Handled` even if recovery fails. A Sphinx recovery failure leaves that credential active because a later packet using the same SURB identifier may be valid. If Sphinx recovery succeeds but the recovered Mix payload is malformed, the matching credential is consumed because another packet using that one-time SURB cannot produce a different plaintext. Other credentials remain independent even if the recipient happened to use their SURBs for redundant copies of the same malformed frame.

## Atomic Registration

`createReplySurbs` creates several public SURBs and collects their private credentials before exposing either collection. It then registers all collected credentials with one call:

```nim
self.replyCredentials.add(sessionId, prepared.credentials).isOkOr:
  return err("could not register reply credentials: " & error)
```

The store validates the complete input before inserting anything:

```nim
proc add*(
    store: ReplyCredentialStore,
    sessionId: PeerId,
    credentials: openArray[ReplyCredential],
    now: Moment = Moment.now(),
): Result[void, string]
```

`add` rejects an empty input, an empty identifier, a duplicate inside the input, an identifier already registered as an active credential, an identifier that remains retired from earlier use or an addition that would exceed the configured capacity. Rejecting an identifier while its tombstone is active prevents the raw reply handler from mistaking a new reply for a repeated packet from the earlier SURB.

Only after every check succeeds does `add` insert the individual entries. Failed registration therefore leaves none of the proposed credentials in the store.

The corresponding public SURBs cross the wire separately, as described in [[Mix Transport Implementation Walk Through - Wire Format Foundation]]. `Connect` carries bootstrap SURBs, `OpenStream` and `SurbStatusProbe` carry response-specific SURBs, and `SurbSupply` carries numbered session supply. None of these frames encodes redundancy-batch membership.

## Capacity Without Eviction

The store checks the complete proposed addition against its remaining capacity:

```nim
if credentials.len > store.maxCredentials - store.credentials.len:
  return err("reply credential store is at capacity")
```

The store does not evict an active credential because that credential may be the only way to recover an unrelated reply. The caller can apply backpressure or fail the operation without damaging another session. The subtraction form avoids overflowing an addition such as `currentLen + newLen`.

## Expiry

Every credential receives its own deadline when it is registered:

```nim
StoredReplyCredential(
  credential: credential,
  sessionId: sessionId,
  expiresAt: now + store.ttl,
)
```

Credentials registered by the same call currently receive the same deadline because `add` uses one `now` value, but the store does not rely on that equality. Later proactive supply operations can register additional credentials for the same session at different times.

`get` checks the selected credential's deadline directly. Once the deadline is reached, the credential is no longer returned even if a cleanup sweep has not physically removed its table entry. An expired reply therefore cannot be accepted merely because the process has been idle and no sweep has run.

`purgeExpired` performs physical cleanup for active credentials and retired identifiers. The procedure first collects expired identifiers and then removes them because a Nim `Table` must not be modified while it is being iterated.

The default TTL is thirty minutes, matching the existing Mix credential lifetime while the transport policy is developed. The transport will eventually ensure that this lifetime does not exceed the usable lifetime of the underlying Mix return path.

## Consuming One Credential

Successful recovery calls:

```nim
store.consume(reply.identifier)
```

`consume` removes only the selected credential and records its identifier as retired until that credential's original expiry time. A repeated packet using the same SURB therefore remains attributable to the transport even though its private credential is no longer available. `MixTransport.handleRawSurbReply` detects the tombstone and returns `Handled` immediately.

Consumption is idempotent. Repeating `consume` for an identifier that is no longer active leaves the store unchanged. Session teardown can therefore retire an entry without coordinating with normal reply completion.

## Retired Identifiers

Retired identifiers are short-lived tombstones, not credentials. A tombstone contains only a `SURBIdentifier` and an expiry time and cannot recover another reply.

`isRetiredIdentifier` is a pure query. The function returns true only when the identifier is present and its deadline has not passed; the function never deletes an entry. Cleanup happens explicitly through `purgeExpired`, so callers do not have to expect an `is*` operation to mutate the store.

The store keeps only a limited number of retired identifiers so that repeated-reply tracking cannot consume memory without limit. This limit applies only to tombstones; active reply credentials have a separate capacity and are never removed to make room for a tombstone.

When the tombstone table is full, the store compares the new identifier's expiry time with the expiry times already stored. The store retains the identifiers that remain valid for the longest time and discards the candidate whose expiry comes first. The discarded candidate may be either an existing tombstone or the new identifier. Evicting a tombstone only removes additional repeated-packet suppression; eviction does not remove a credential needed to recover an outstanding reply.

`removeSession(sessionId)` uses the same retirement rule during cancellation or teardown. The procedure first purges entries that are already expired, then removes every still-active credential owned by the selected session and retires each identifier until its individual deadline. Credentials belonging to one session can have different deadlines because bootstrap, stream-opening, numbered-supply and status-probe operations create them at different times.

## Tests

The focused tests create real Mix `ReplyCredential` values by generating a minimal one-hop return path and calling `createSURB`. To exercise successful recovery, the helper builds a normal empty-codec Mix reply, pads it to the fixed Sphinx message size with `addPadding`, sends it through the SURB with `useSURB`, processes the return hop and passes the resulting `RawSurbReply` to the store. This path avoids constructing opaque credentials by assigning their private cryptographic fields.

The tests establish the essential behavior:

- consuming one credential leaves other independently registered credentials active;
- a full store rejects a new atomic addition while preserving active credentials;
- retired identifiers are independently bounded, become semantically inactive at their deadline and remain stored until an explicit purge;
- removing one session retires its active identifiers while preserving another session's credentials;
- an expired credential is invisible before `purgeExpired` removes its table entry;
- an unknown identifier remains available to Mix's embedded fallback path;
- every successfully recovered redundant copy consumes only its matching credential;
- Sphinx corruption retains the matching credential for a potentially valid later packet;
- a cryptographically recovered but malformed Mix payload consumes only the matching credential.

`MixTransport` owns one `ReplyCredentialStore`, clears the store during shutdown and uses it from the registered raw-reply callback. Recovered bytes are decoded as a `MixTransportFrame`, and the frame's `sessionId` must match `StoredReplyCredential.sessionId` before the frame is dispatched to the live session.
