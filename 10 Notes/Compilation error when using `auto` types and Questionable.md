[![hackmd-github-sync-badge](https://hackmd.io/onqV1RP7R9KOkYpx_iRuzQ/badge)](https://hackmd.io/onqV1RP7R9KOkYpx_iRuzQ)
So here is a piece of code (test):

```nim
test "Should retrieve block expiration information":
  proc unpack(
	beIter: auto,
  ): Future[seq[BlockExpiration]] {.async: (raises: [CancelledError]).} =
    var expirations = newSeq[BlockExpiration](0)
    without iter =? (await beIter), err:
	  info "Failed to get BlockExpiration async iterator, returning empty sequence",
	    err = err.msg
	  return expirations
	# ...
```

Compiling this on Ubuntu (see also [[How to build Codex on Ubuntu 24 (VM)]]) fails with the following error:

```bash
stack trace: (most recent call last)
/home/codex/code/nim-codex/vendor/questionable/questionable/withoutresult.nim(27, 10) without
/home/codex/code/nim-codex/tests/codex/stores/testrepostore.nim(59, 1) template/generic instantiation of `asyncchecksuite` from here
/home/codex/code/nim-codex/tests/codex/stores/testrepostore.nim(294, 3) template/generic instantiation of `test` from here
/home/codex/code/nim-codex/vendor/nim-unittest2/unittest2.nim(1128, 24) template/generic instantiation of `failingOnExceptions` from here
/home/codex/code/nim-codex/vendor/nim-unittest2/unittest2.nim(1132, 26) template/generic instantiation of `failingOnExceptions` from here
/home/codex/code/nim-codex/vendor/asynctest/asynctest/private/suite.nim(34, 5) template/generic instantiation of `runAsync` from here
/home/codex/code/nim-codex/vendor/asynctest/asynctest/private/chronos/unittest2/runasync.nim(4, 26) template/generic instantiation of `async` from here
/home/codex/code/nim-codex/tests/codex/stores/testrepostore.nim(327, 21) template/generic instantiation of `unpack` from here
/home/codex/code/nim-codex/tests/codex/stores/testrepostore.nim(306, 14) template/generic instantiation of `setResult` from here
/home/codex/code/nim-codex/tests/codex/stores/testrepostore.nim(299, 39) Error: expected an identifier, got 
err(typeof(result))
```

Not two, not obvious ways to fix it:

**Solution 1: change the name of the `err` variable**

```nim
test "Should retrieve block expiration information":
  proc unpack(
	beIter: auto,
  ): Future[seq[BlockExpiration]] {.async: (raises: [CancelledError]).} =
    var expirations = newSeq[BlockExpiration](0)
    without iter =? (await beIter), error:
	  info "Failed to get BlockExpiration async iterator, returning empty sequence",
	    err = error.msg
	  return expirations
	# ...
```

**Solution 2: use explicit type instead of `auto`**

```nim
test "Should retrieve block expiration information":
  proc unpack(
	beIter: Future[?!SafeAsyncIter[BlockExpiration]].Raising([CancelledError]),
  ): Future[seq[BlockExpiration]] {.async: (raises: [CancelledError]).} =
    var expirations = newSeq[BlockExpiration](0)
    without iter =? (await beIter), err:
	  info "Failed to get BlockExpiration async iterator, returning empty sequence",
	    err = err.msg
	  return expirations
	# ...
```

