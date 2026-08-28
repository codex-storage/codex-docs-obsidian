---
related:
  - "[[Digging into Nim's refc]]"
  - "[[Nim]]"
---
`Option` and `Result` can be distinguished by two different use-cases:

- `Option[T]`: a value may be absent; there is no error to explain
- `Result[T, E]`: either a value or an error value explaining failure

Use `Option` for normal absence: an optional configuration value, a cache miss, or a lookup with no match. Use `Result` when callers need to handle or report why an operation failed.

## `std/options`

```nim
import std/options

proc findUser(id: int): Option[string] =
  if id == 42: some("Ada")
  else: none(string)

let user = findUser(42)

if user.isSome:
  echo user.get

let name = user.get("anonymous")
```

- `some(value)` and `none(T)` construct an option.
- `.isSome` / `.isNone` inspect it.
- `.get` without an argument is unsafe for `none`: it raises a `Defect`.
- `.get(default)` returns the default when the option is empty.

## `nim-results`

The generic form is `Result[T, E]`:

```nim
import results

proc parsePort(text: string): Result[int, string] =
  try:
    Result[int, string].ok(parseInt(text))
  except ValueError:
    Result[int, string].err("not a port")
```

Inspect it with `.isOk` / `.isErr`; a failed result carries its error value.

## Project convention: `questionable/results`

Logos Storage commonly imports `pkg/questionable/results`, a convenience layer over `nim-results`:

```nim
proc loadThing(): ?!string =
  success "value"
  # or: failure "could not load it"
```

`?!T` is shorthand for `Result[T, ref CatchableError]`.

Two common patterns:

```nim
let key = parseKey(text).valueOr:
  return failure("Invalid key: " & $error)

let info = ?pubInfoFromJson(entry)
```

`valueOr:` unwraps a success; on failure it executes the block, binding the failure as `error`. Prefix `?` propagates a failure from a `?!...` expression by immediately returning it from the current `?!...` proc.
