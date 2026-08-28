---
related:
  - "[[Mix Transport Design Specification]]"
  - "[[libp2p MIX Architecture and API]]"
  - "[[Sphinx SURBs implementation in the libp2p MIX protocol]]"
---

# Mix Transport - Pluggable Integration Model

## Direction

The new `MixTransport` will be an optional layer built on the stateless Mix API. It will not replace the embedded request/reply implementation during the initial integration.

Without `MixTransport`, existing callers continue to use `toConnection`, the internal `SurbStore`, destination read behaviours and automatic reply recovery exactly as they do today.

With `MixTransport`, the transport registers a handler for its own service and a raw SURB reply handler. It owns the sessions, virtual connections, framing, credentials and received SURBs described in [[Mix Transport Design Specification]]. Other Mix services continue to use the embedded defaults.

## Current Position

The additive Mix API is mostly present:

- `send` sends an opaque payload to a named service;
- `createSurb` returns a public `SURB` and an opaque `ReplyCredential`;
- `sendWithSurb` sends an opaque payload through a caller-supplied SURB;
- `recoverReply` recovers the original payload with a caller-owned credential;
- service-scoped `MixDeliveryHandler` registration delivers ordinary messages without invoking the embedded exit handling;
- `RawSurbReplyHandler` registration exposes encrypted replies before embedded credential lookup.

The embedded transport and its state remain in `MixProtocol` as the compatibility implementation. This is intentional for the migration period and does not change the ownership model of the new transport.

## Remaining Mix Work

The raw-reply handler must report whether it recognized the SURB identifier. When `MixTransport` owns the identifier, it handles the reply and the embedded path must not see it. When the identifier is unknown to `MixTransport`, Mix must try the existing internal `SurbStore`. A simple `Handled` / `Unhandled` result is sufficient. A reply with a recognized identifier but failed recovery is still `Handled`; it must not fall through to unrelated legacy credentials.

Exit-equals-destination support should become unconditional rather than depending on `libp2p_mix_experimental_exit_is_dest`. Removing that build flag does not require deleting forward mode or the embedded transport.

Before beginning `MixTransport`, add focused integration coverage for the plug-in boundary:

- `send` delivers the exact service and payload to the registered service handler;
- an unregistered service follows the existing embedded path;
- a handled raw reply does not reach the embedded store;
- an unhandled raw reply falls back to the embedded reply machinery;
- unregistering handlers restores the defaults.

Once those points are complete, the Mix API is ready for the new transport repository. Removing the embedded connection API, internal `SurbStore`, forwarding mode or old tests is not a prerequisite and should be considered separately after the new transport is proven.
