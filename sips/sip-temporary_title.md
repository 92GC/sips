|   SIP-Number | |
|         ---: | :--- |
|        Title | On-Chain Commit Liveness Counters |
|  Description | Extends the system Clock with consensus-maintained commit-gap statistics and monotonic stall counters for trustless circuit breakers. |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

This proposal extends the system `Clock` with commit-gap statistics maintained automatically by the network. It exposes an exponential moving average (EMA) of normal commit intervals and monotonic counters for gaps that exceed the baseline by significant multiples.

Contracts can snapshot a counter when an operation begins and compare it later. If the counter changed, the contract knows that a material network stall occurred during the operation, even if the network subsequently recovered.

## Motivation

Contracts cannot currently determine whether Sui experienced a significant stall during a market, auction, oracle window, or governance process. This is particularly dangerous for short-window TWAPs: a market may appear to span two hours of wall-clock time while only being live for a small part of that interval.

Checkpoint timestamps are not the correct primitive because checkpoint construction and execution are asynchronous. Consensus commit timestamps directly measure the intervals relevant to execution.

Historical range queries, sliding-window maxima, and median calculations would require additional history, indexing, and crash-recovery machinery. A monotonic incident counter provides the circuit-breaker property directly with constant-size state.

## Specification

The system `Clock` is extended with read-only accessors equivalent to:

```move
module sui::clock {
    /// Most recently observed interval between consecutive consensus commits.
    public fun last_commit_gap_ms(clock: &Clock): u64;

    /// Protocol-maintained EMA of normal commit intervals.
    public fun commit_gap_ema_ms(clock: &Clock): u64;

    /// Number of commit gaps that exceeded 10x the prior EMA baseline.
    public fun stall_count_10x(clock: &Clock): u64;

    /// Number of commit gaps that exceeded 100x the prior EMA baseline.
    public fun stall_count_100x(clock: &Clock): u64;
}
```

The exact storage and native-function boundary is an implementation detail. The values must be consensus-maintained and identical for all validators.

For each new consensus commit, the protocol:

1. Calculates the gap from the preceding commit timestamp.
2. Compares the gap with the EMA value from before the current observation.
3. Increments the applicable stall counters.
4. Updates the EMA with the new observation.
5. Persists the resulting state in the system `Clock`.

The EMA coefficient, initialization rule, integer rounding, and a protocol-defined minimum absolute gap must be specified and versioned in protocol configuration. The absolute floor prevents ordinary startup variance or very small baselines from being classified as stalls.

Counters are monotonic. A 100x incident also increments the 10x counter.

## Example

A market snapshots the counter when it opens:

```move
market.start_stall_count = clock::stall_count_100x(clock);
```

At settlement:

```move
let stalled = clock::stall_count_100x(clock) != market.start_stall_count;
if (stalled) {
    // Apply the market's circuit-breaker policy.
};
```

Because the counter is monotonic, the incident remains detectable after normal block production resumes.

## Rationale

- **Commit timestamps:** Measure consensus execution intervals without checkpoint-construction delay.
- **Clock extension:** Keeps liveness data in the existing system time primitive.
- **EMA baseline:** Adapts to durable changes in normal network cadence.
- **Pre-update comparison:** A large gap is recorded before it can influence the EMA, so smoothing cannot hide the incident.
- **Monotonic counters:** Allow arbitrary overlapping markets to take independent snapshots without retaining per-market history in the protocol.
- **Constant-size state:** Avoids historical scans, sliding deques, segment trees, and validator state bloat.
- **No keeper:** The network updates the values automatically; contracts and bots only read them.

## Crash Recovery

The previous commit timestamp, EMA, and counters are consensus state. After a validator or network restart, the first successful commit compares its timestamp with the last persisted commit timestamp. A restart-sized gap is therefore recorded automatically.

No off-chain cranker, archival database, or reconstruction of a sliding history window is required.

## Backwards Compatibility

This proposal is additive. Existing `Clock` behavior and public functions remain unchanged.

## Test Cases

- Normal commit intervals update the EMA without incrementing counters.
- A gap greater than 10x but less than 100x increments only the 10x counter.
- A gap greater than 100x increments both counters.
- The triggering gap is compared against the EMA before that gap is incorporated.
- A restart-sized gap is recorded on the first subsequent commit.
- Counter values remain monotonic across epochs and protocol upgrades.
- EMA initialization and integer rounding are deterministic.

## Reference Implementation

To be developed.

## Security Considerations

- Only the protocol may update the liveness fields.
- Commit timestamps must use the same consensus-defined monotonicity rules as the system clock.
- Arithmetic must define overflow, saturation, and rounding behavior.
- Changes to EMA or threshold parameters must be protocol-versioned.
- The counters report that a timing anomaly occurred; individual applications remain responsible for choosing the appropriate response.

## Copyright

Greshamscode 2025
