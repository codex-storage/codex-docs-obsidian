---
tags:
  - logos-storage/block-exchange
  - logos-storage/download
  - logos-storage/peers
  - logos-storage/performance
related:
  - "[[Libp2p connections - connect vs dial]]"
  - "[[Closing libp2p connections and streams]]"
  - "[[New Logos Storage Download Flow]]"
  - "[[New Logos Storage Block Exchange Protocol]]"
  - "[[Block Exchange Peer Stores]]"
---

# Block Exchange Peer Performance and BDP

> [!info] Source baseline
> This note describes `logos-storage-nim` `master` at commit `81f0d053`
> (2026-07-20).

Logos Storage sends several block batches concurrently to a peer. It therefore
needs to answer two different questions:

1. How many requests may be in flight to this peer?
2. Which peer should receive the next batch?

The first answer is the peer's adaptive **pipeline depth**. It is informed by a
bandwidth-delay-product calculation and adjusted through active probing.

The second answer is a weighted **peer-selection score**. It considers unused
pipeline capacity, throughput, request latency, and timeout history.

These are related mechanisms, but they are not one “BDP score.”

```text
successful batch responses
  -> latency samples + cumulative delivered bytes
     -> average request latency
     -> rolling throughput
        -> BDP depth estimate
        -> adaptive pipeline depth

eligible peers below their pipeline depth
  -> capacity + throughput + latency + timeout score
     -> peer selected for the next batch
```

## Where performance state lives

Performance statistics belong to the node-wide `PeerContext`:

```nim
type PeerContext* = ref object of RootObj
  id*: PeerId
  stats*: PeerPerfStats
  wantListBusy*: bool

proc new*(T: type PeerContext, id: PeerId): PeerContext =
  PeerContext(id: id, stats: PeerPerfStats.new())
```

`PeerContext` is stored in `BlockExcEngine.peers`, so measurements are shared
by all downloads using that peer. They do not belong to one
`DownloadContext.swarm`.

Consequently:

- one successful download can teach the engine how to pipeline later
  downloads to the same peer;
- simultaneous downloads share the same peer capacity counter;
- when the peer leaves and its `PeerContext` is evicted, these statistics are
  discarded;
- a later reconnection creates a fresh `PeerContext` with default statistics.

See [[Block Exchange Peer Stores]] for the distinction between node-wide peer
state and per-download swarm state.

## The measurements come from successful `WantBlocks` batches

The outer engine request method timestamps the entire operation:

```nim
let
  requestStartTime = Moment.now()
  requestResult = await self.requestWantBlocks(
    peer.id, BlockRange(treeCid: treeCid, ranges: ranges)
  )
  rttMicros = (Moment.now() - requestStartTime).microseconds.uint64
```

After the response has been decoded, validated, and stored, it records the
successful delivery:

```nim
storage_block_exchange_blocks_received.inc(validCount.int64)

download.ctx.swarm.recordBatchSuccess(peer, rttMicros, totalBytes)
```

The swarm resets this download's failure/timeout counters and forwards the
measurement to the node-wide peer statistics:

```nim
proc recordBatchSuccess*(
    swarm: Swarm, peer: PeerContext, rttMicros: uint64, totalBytes: uint64
) =
  swarm.peers.withValue(peer.id, swarmPeer):
    swarmPeer[].resetFailures()
    swarmPeer[].touch()
  peer.stats.recordRequest(rttMicros, totalBytes)
```

Only bytes from blocks that passed validation and were stored contribute to
`totalBytes`.

## “RTT” is actually complete batch-request latency

The variable is named `rttMicros`, but its timer surrounds:

```text
request construction
  -> write binary WantBlocks request
  -> network delivery
  -> provider lookup and response construction
  -> response transfer
  -> response decoding
```

It is therefore not a transport-level round-trip time. It is closer to
end-to-end batch response latency.

This matters when interpreting the formulas below. Classical BDP uses network
RTT. Logos Storage uses the duration of a complete application request, which
also contains transfer time and provider processing time.

## The statistics object

The relevant defaults are:

```nim
const
  RttSampleCount* = 16
  MinRequestsPerPeer* = 2
  MaxRequestsPerPeer* = 32
  DefaultRequestsPerPeer* = 2
  DefaultPipelineDepth* = 2
  MinThroughputDuration* = 100.milliseconds
  ThroughputWindow* = 3.seconds
  ProbeIntervalBatches* = 16
  ProbeWindowBatches* = 16
  GainThresholdPct* = 8
  LossThresholdPct* = 20
  MaxProbeBackoffShift* = 4
```

The object stores latency samples, timestamped cumulative-byte samples, and
the adaptive-depth controller:

```nim
type PeerPerfStats* = object
  rttSamples: Deque[uint64]
  throughputSamples: Deque[ThroughputSample]
  totalBytesDelivered: uint64
  currentDepth: int
  lastDepthChangeTime: Moment
  probeMode: ProbeMode
  probeBaselineBps: uint64
  probeStartTotalBytes: uint64
  probeStartTime: Moment
  batchesSinceProbe: int
  batchesInProbeWindow: int
  consecutiveReverts: int
```

A new peer begins with depth two:

```nim
proc new*(T: type PeerPerfStats): PeerPerfStats =
  PeerPerfStats(
    rttSamples: initDeque[uint64](RttSampleCount),
    throughputSamples: initDeque[ThroughputSample](),
    totalBytesDelivered: 0,
    currentDepth: DefaultRequestsPerPeer,
    lastDepthChangeTime: Moment.now(),
    probeMode: Stable,
    batchesSinceProbe: 0,
    consecutiveReverts: 0,
  )
```

## The last 16 latency samples

Every successful batch appends its request duration. Once the deque already
contains 16 entries, the oldest is removed:

```nim
proc recordRequest*(self: var PeerPerfStats, rttMicros: uint64, bytes: uint64) =
  if self.rttSamples.len >= RttSampleCount:
    discard self.rttSamples.popFirst()
  self.rttSamples.addLast(rttMicros)
```

The reported latency is their arithmetic mean:

```nim
proc avgRttMicros*(self: PeerPerfStats): Option[uint64] =
  if self.rttSamples.len == 0:
    return none(uint64)

  var total: uint64 = 0
  for sample in self.rttSamples:
    total += sample

  some(total div self.rttSamples.len.uint64)
```

After the peer has enough history:

```text
average latency =
  sum(last 16 successful batch durations) / 16
```

The number 16 is a tuning choice, not a protocol requirement:

- several observations smooth individual spikes;
- a bounded recent history can still react to changed conditions;
- storage and averaging remain cheap.

There is no source comment deriving exactly 16. Latency retention is based on
sample count, not age. Sixteen busy-peer samples may cover a fraction of a
second, while sixteen samples from a quiet peer may cover minutes.

## Throughput samples are cumulative counters

On every successful batch, the implementation adds the number of valid bytes
to a running total and records a timestamped snapshot:

```nim
let now = Moment.now()
self.totalBytesDelivered += bytes
self.throughputSamples.addLast(
  ThroughputSample(time: now, cumBytes: self.totalBytesDelivered)
)
self.trimThroughputWindow(now)
```

A sample does not say “this request achieved X bytes per second.” It says:

```text
At time T, this peer had successfully delivered N cumulative bytes.
```

Throughput is computed from the difference between two cumulative snapshots:

```text
newest cumulative bytes - oldest cumulative bytes
-------------------------------------------------
newest timestamp - oldest timestamp
```

This naturally measures aggregate delivery when several requests overlap.

## What a rolling three-second window means

Before calculating throughput, entries more than three seconds older than the
current time are removed:

```nim
proc trimThroughputWindow(self: var PeerPerfStats, now: Moment) =
  while self.throughputSamples.len > 0 and
      (now - self.throughputSamples[0].time) > ThroughputWindow:
    discard self.throughputSamples.popFirst()
```

The boundary is always:

```text
now - 3 seconds
```

It moves whenever `now` moves, hence “rolling.” There are no fixed buckets
such as `10–13 seconds`, then `13–16 seconds`.

### Rolling-window example

Suppose the peer has these samples:

| Time | Cumulative valid bytes |
|---:|---:|
| 10.0 s | 1 MiB |
| 11.0 s | 3 MiB |
| 12.5 s | 7 MiB |

At `now = 12.5 s`, the active interval begins at `9.5 s`. All three samples
remain:

```text
throughput =
  (7 MiB - 1 MiB) / (12.5 s - 10.0 s)
  = 6 MiB / 2.5 s
  = 2.4 MiB/s
```

At `now = 13.5 s`, the active interval begins at `10.5 s`. The `10.0 s`
sample is discarded:

```text
throughput =
  (7 MiB - 3 MiB) / (12.5 s - 11.0 s)
  = 4 MiB / 1.5 s
  ≈ 2.67 MiB/s
```

If no new batch succeeds, the remaining samples eventually age out and
throughput becomes unavailable.

The code also requires at least two samples spanning at least 100 ms:

```nim
proc avgThroughputBps(self: var PeerPerfStats, now: Moment): Option[uint64] =
  self.trimThroughputWindow(now)
  if self.throughputSamples.len < 2:
    return none(uint64)

  let
    first = self.throughputSamples[0]
    last = self.throughputSamples[self.throughputSamples.len - 1]
    duration = last.time - first.time

  if duration < MinThroughputDuration:
    return none(uint64)

  let
    delta = last.cumBytes - first.cumBytes
    secs = duration.nanoseconds.float64 / 1_000_000_000.0
  some((delta.float64 / secs).uint64)
```

This avoids calculating a highly unstable rate from one response or a very
short interval.

## Why the latency and throughput histories differ

| Measurement | Retention rule |
|---|---|
| Request duration called RTT | last 16 successful batches, regardless of age |
| Throughput | successful-batch snapshots no more than approximately 3 seconds old |

The latency average is count-based. Throughput is time-based.

This means the two values may describe different periods. For example, after a
quiet interval, the old latency average may still exist while the three-second
throughput estimate has expired.

## Classical bandwidth-delay product

The bandwidth-delay product estimates how many bytes must be in flight to keep
a path busy:

```text
BDP bytes = throughput bytes/second × round-trip time seconds
```

If a path carries 20 MiB/s and acknowledgement or response feedback takes
0.2 seconds:

```text
20 MiB/s × 0.2 s = 4 MiB
```

Approximately 4 MiB must be in flight to prevent the sender from becoming idle
while it waits for feedback.

Logos Storage sends data in batches, so it converts bytes in flight to request
count:

```text
pipeline depth = ceil(BDP bytes / batch bytes)
```

## The Logos Storage BDP calculation

The implementation is:

```nim
proc computeBdpDepth(self: var PeerPerfStats, batchBytes: uint64, now: Moment): int =
  if batchBytes == 0:
    return DefaultPipelineDepth

  let rttMicrosOpt = self.avgRttMicros()
  if rttMicrosOpt.isNone:
    return DefaultRequestsPerPeer

  let throughputOpt = self.avgThroughputBps(now)
  if throughputOpt.isNone:
    return DefaultRequestsPerPeer

  let
    rttMicros = rttMicrosOpt.get()
    throughput = throughputOpt.get()
    rttSecs = rttMicros.float64 / 1_000_000.0
    bdpBytes = throughput.float64 * rttSecs
    depth = ceil(bdpBytes / batchBytes.float64).int

  clamp(depth, MinRequestsPerPeer, MaxRequestsPerPeer)
```

The result is always between 2 and 32.

### BDP-depth example

Assume:

```text
rolling throughput       = 20 MiB/s
average request duration = 200 ms = 0.2 s
batch size               = 1 MiB
```

Then:

```text
BDP bytes = 20 MiB/s × 0.2 s = 4 MiB

depth = ceil(4 MiB / 1 MiB)
      = 4 concurrent batch requests
```

The intended interpretation is that four batches in flight should supply
enough data to cover the observed feedback delay.

Because the measured “RTT” includes response transfer and processing, this is
not a pure classical BDP calculation. It is an application-level heuristic
using the same mathematical shape.

## Calculated BDP does not directly increase the depth

The actual capacity is `currentDepth`. It starts at two. In stable mode, the
BDP calculation can lower it after a grace interval:

```nim
let
  bdpDepth = self.computeBdpDepth(batchBytes, now)
  gracePassed = (now - self.lastDepthChangeTime) >= ThroughputWindow

if bdpDepth < self.currentDepth and gracePassed:
  self.currentDepth = max(MinRequestsPerPeer, bdpDepth)
  self.lastDepthChangeTime = now
```

It does not immediately increase `currentDepth` to a larger BDP estimate.
Upward movement is deliberately experimental.

## Increasing concurrency by probing

After a stable peer has completed enough batches, the controller raises the
depth by one:

```nim
let effectiveInterval =
  ProbeIntervalBatches * (1 shl min(self.consecutiveReverts, MaxProbeBackoffShift))

if self.batchesSinceProbe >= effectiveInterval and
    self.currentDepth < MaxRequestsPerPeer:
  let baseline = self.avgThroughputBps(now)
  if baseline.isSome:
    self.probeBaselineBps = baseline.get()
    self.probeStartTotalBytes = self.totalBytesDelivered
    self.probeStartTime = now
    self.probeMode = Probing
    self.batchesInProbeWindow = 0
    self.currentDepth = self.currentDepth + 1
```

Normally:

```text
16 successful batches
  -> temporarily increase depth by one
  -> observe 16 more successful batches
  -> compare throughput with the baseline
```

The evaluation is:

```nim
if deltaPct >= GainThresholdPct:
  self.consecutiveReverts = 0
elif deltaPct <= -LossThresholdPct:
  self.consecutiveReverts = 0
  self.currentDepth = max(MinRequestsPerPeer, self.currentDepth - 2)
else:
  self.consecutiveReverts += 1
  self.currentDepth = max(MinRequestsPerPeer, self.currentDepth - 1)
```

Therefore:

- throughput improved by at least 8%: keep the extra slot;
- throughput fell by at least 20%: remove the probe slot and one additional
  slot;
- anything in between: undo the probe and increase the delay before the next
  probe.

### Probe example

Suppose the peer currently has depth two and baseline throughput 10 MiB/s:

```text
after 16 stable batches:
  depth 2 -> probe depth 3
```

After the 16-batch probe:

| Observed throughput | Change | Result |
|---:|---:|---|
| 11 MiB/s | +10% | keep depth 3 |
| 9.5 MiB/s | −5% | return to depth 2 |
| 7.5 MiB/s | −25% | reduce by two, clamped to depth 2 |

Repeated neutral or inconclusive probes back off exponentially:

```text
16, 32, 64, 128, then 256 batches between probes
```

The shift is capped at four, so the interval does not grow beyond
`16 × 2⁴ = 256` batches.

## The node-wide in-flight counter

`PeerInFlightTracker` holds unfinished request futures by peer:

```nim
type PeerInFlightTracker* = ref object
  peerInFlight*: Table[PeerId, seq[Future[void]]]

proc track*(self: PeerInFlightTracker, peerId: PeerId, fut: Future[void]) =
  self.peerInFlight.mgetOrPut(peerId, @[]).add(fut)
```

Counting also removes finished futures:

```nim
proc count*(self: PeerInFlightTracker, peerId: PeerId): int =
  self.peerInFlight.withValue(peerId, lst):
    var live: seq[Future[void]]
    for fut in lst[]:
      if not fut.finished:
        live.add(fut)
    # replace or remove the old list
    return live.len
  return 0
```

The tracker belongs to `DownloadManager`, so its count includes requests from
all active downloads using the peer.

## Pipeline capacity is checked before scoring

The swarm first filters candidates by advertised availability and by capacity:

```nim
for peerId in candidates:
  let peer = peers.get(peerId)
  if peer.isNil:
    discard swarm.removePeer(peerId)
    continue
  if tracker.count(peerId) < peer.optimalPipelineDepth(batchBytes):
    peerCtxs.add(peer)
```

For example:

```text
peer pipeline depth = 4
unfinished futures  = 3  -> eligible

peer pipeline depth = 4
unfinished futures  = 4  -> at capacity, not scored
```

If every otherwise suitable peer is full, selection returns
`pskAtCapacity`. The worker requeues the batch and tries again later.

## New peers are deliberately sampled

Before scoring established peers, `selectByBDP` looks for candidates without a
throughput estimate:

```nim
var untriedPeers: seq[PeerContext]
for peer in peers:
  if peer.stats.throughputBps().isNone:
    let
      pipelineDepth = peer.optimalPipelineDepth(batchBytes)
      currentLoad = tracker.count(peer.id)
    if currentLoad < pipelineDepth:
      untriedPeers.add(peer)
```

If any exist, it chooses the one with the smallest current load. Without this
step, an unknown peer could never collect measurements because established
peers would always have a more informative score.

## Exploration prevents permanent lock-in

When all candidates have measurements, there is a 20% probability of choosing
a random peer with capacity:

```nim
let exploreRoll = rand(1.0)
if exploreRoll < ExplorationProbability:
  # choose a random peer with capacity
```

This gives slower or previously unlucky peers opportunities to demonstrate
changed performance.

## The peer-selection score

For ordinary exploitation, each candidate receives a weighted score:

```nim
const
  WeightCapacity* = 0.30
  WeightThroughput* = 0.25
  WeightRtt* = 0.25
  WeightPenalty* = 0.20
```

Every component is normalized to `[0, 1]`, where zero is best. The actual
implementation is:

```nim
let
  pipelineDepth = self.optimalPipelineDepth(batchBytes)
  capacityRatio =
    if currentLoad >= pipelineDepth:
      WorstRatio
    elif pipelineDepth > 0:
      currentLoad.float / pipelineDepth.float
    else:
      WorstRatio

  throughputRatio =
    if self.stats.throughputBps().isSome:
      let bps = self.stats.throughputBps().get().float
      if bps <= 0:
        WorstRatio
      else:
        clamp(WorstRatio - bps / RefMaxBps, BestRatio, WorstRatio)
    else:
      FallbackThroughputRatio

  rttRatio =
    if self.stats.avgRttMicros().isSome:
      clamp(
        self.stats.avgRttMicros().get().float / RefMaxRttMicros,
        BestRatio,
        WorstRatio,
      )
    else:
      FallbackRttRatio

  penaltyRatio = clamp(penalty / RefMaxPenalty, BestRatio, WorstRatio)
```

The final score is:

```nim
WeightCapacity * capacityRatio +
  WeightThroughput * throughputRatio +
  WeightRtt * rttRatio +
  WeightPenalty * penaltyRatio
```

Lower is better.

The reference points are saturation points, not measured maxima:

```nim
RefMaxBps = 100 MiB/s
RefMaxRttMicros = 500 ms
RefMaxPenalty = 15
```

A throughput at or above 100 MiB/s gets the best throughput component. A
request duration at or above 500 ms gets the worst latency component.

## Timeout penalty

Each consecutive timeout since the last successful batch adds:

```nim
TimeoutPenaltyWeight = 3.0
```

The score normalizes against 15:

```text
one timeout:   3 / 15 = 0.2
three timeouts: 9 / 15 = 0.6
five timeouts: 15 / 15 = 1.0
```

Any successful batch resets that download's failure and timeout counters. Five
timeouts without an intervening success also match the default threshold at
which a peer is removed from that download's swarm.

## Complete score example

Consider two eligible peers for a 1 MiB batch.

Peer A:

```text
pipeline depth = 4
current load   = 1
throughput     = 20 MiB/s
latency        = 200 ms
timeouts       = 1
```

Its normalized components are:

```text
capacity   = 1 / 4             = 0.25
throughput = 1 - 20 / 100      = 0.80
latency    = 200 / 500         = 0.40
penalty    = (1 × 3) / 15      = 0.20
```

Its score is:

```text
0.30 × 0.25 +
0.25 × 0.80 +
0.25 × 0.40 +
0.20 × 0.20
= 0.415
```

Peer B:

```text
pipeline depth = 2
current load   = 1
throughput     = 50 MiB/s
latency        = 400 ms
timeouts       = 0
```

Its components are:

```text
capacity   = 1 / 2         = 0.50
throughput = 1 - 50 / 100  = 0.50
latency    = 400 / 500     = 0.80
penalty    = 0             = 0.00
```

Its score is:

```text
0.30 × 0.50 +
0.25 × 0.50 +
0.25 × 0.80 +
0.20 × 0.00
= 0.475
```

Peer A wins because `0.415 < 0.475`, even though Peer B has higher measured
throughput. Peer A has more spare pipeline capacity and lower response latency.

## Adaptive batch timeout uses the same measurements

Latency and throughput also determine the timeout:

```nim
let
  transferTimeMicros = (batchBytes * 1_000_000) div throughput
  totalTimeMicros = transferTimeMicros + rttMicros
  timeoutMicros = (totalTimeMicros.float * TimeoutSafetyFactor).uint64
```

In formula form:

```text
timeout =
  3 × (batch bytes / throughput + average request latency)
```

The result is clamped to 5–45 seconds. With missing measurements, it falls
back to 30 seconds.

Again, the “RTT” term already includes a complete batch transfer, so adding a
separate transfer-time estimate may be conservative. The timeout is an
application heuristic rather than a direct network model.

## How the mechanisms fit into batch scheduling

```text
Scheduler.take()
  -> batch in current presence window
  -> Swarm.peersWithRange()
  -> discard disconnected PeerContexts
  -> discard peers at pipeline capacity
  -> prefer unmeasured peer, sometimes explore randomly,
     otherwise select lowest weighted score
  -> mark PendingBatch
  -> start sendWantBlocksRequest future
  -> PeerInFlightTracker.track(peer, future)
  -> do not await; schedule another batch

successful response
  -> validate and store blocks
  -> record latency + valid bytes
  -> update pipeline controller
  -> future completes and no longer counts as in flight
```

This is why multiple batches can be sent concurrently to the same peer without
unbounded concurrency.

## Practical caveats

1. `rttMicros` measures full request time, not pure RTT.
2. Throughput counts only successfully validated and stored bytes.
3. Latency history may outlive the three-second throughput history.
4. The BDP formula can lower capacity, but upward growth happens through
   probing.
5. The score is bypassed while trying unmeasured peers and during 20% random
   exploration.
6. Capacity accounting is shared across downloads.
7. The constants 16, 3 seconds, 8%, 20%, and the score weights are tuning
   choices; the source does not document experimental derivations for them.

## Short mental model

```text
Last 16 successful requests
  -> smoothed application request latency

Valid bytes delivered during roughly the last 3 seconds
  -> recent aggregate throughput

throughput × latency / batch size
  -> BDP depth estimate

careful +1 concurrency probes
  -> actual adaptive pipeline depth

capacity + throughput + latency + timeouts
  -> next peer to receive a batch
```

## Source map

- `storage/blockexchange/peers/peerstats.nim`
  - latency and throughput histories
  - BDP depth
  - adaptive probing
- `storage/blockexchange/peers/peercontext.nim`
  - pipeline depth wrapper
  - batch timeout
  - weighted score
- `storage/blockexchange/engine/swarm.nim`
  - capacity filtering
  - exploration and peer selection
  - successful-batch recording
- `storage/blockexchange/engine/peertracker.nim`
  - node-wide unfinished-future counts
- `storage/blockexchange/engine/engine.nim`
  - request timing
  - valid-byte accounting
  - tracking concurrent request futures
