---
related:
  - "[[Nim Option and Result]]"
  - "[[Digging into Nim's refc]]"
  - "[[Nim]]"
---
Nim separates the callee's ownership contract from the caller's decision at a particular call site:

- A `sink T` parameter says that the callee may consume the argument.
- `move(x)` says that this call must transfer the value out of `x`; afterward `x` is reset to its default state.

## Passing to a `sink` parameter

```nim
proc consume(value: sink Buffer) =
  store = move(value)
```

With `consume(buffer)`, the compiler performs last-use analysis. If `buffer` is not used afterward, it can be moved; if it must remain usable, the compiler passes a copy instead.

With `consume(move(buffer))`, the move is explicit: `buffer` is reset regardless of later references. Nim permits later access, but it observes the default value rather than the old one.

## Passing `move(x)` to an ordinary parameter

```nim
proc inspect(value: Buffer) =
  echo value

inspect(move(buffer))
```

`buffer` is still moved and reset. The moved value becomes the argument used during the call; this does not necessarily create another copy. However, an ordinary parameter does not let the callee consume the parameter as owned storage. If the callee retains it elsewhere, that operation generally requires a copy.

## Quick reference

| Call | Effect |
| --- | --- |
| `ordinary(x)` | `x` is preserved. |
| `ordinary(move(x))` | `x` is definitely reset; no extra copy is necessarily made, but the ordinary parameter is not an ownership-taking interface. |
| `consume(x)` for `sink T` | The compiler moves `x` when it can prove this is its last use; otherwise it copies. |
| `consume(move(x))` for `sink T` | `x` is definitely reset and ownership is explicitly transferred to the sink parameter. |

`sink` provides affine, at-most-once consumption inside the callee; it does not make a type non-copyable. For one-use values such as SURBs, the application must also avoid making duplicates and remove a value from its owning pool before sending it.
