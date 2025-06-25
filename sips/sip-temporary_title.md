|   SIP-Number | |
|         ---: | :--- |
|        Title | Onchain Checkpoint Oracle |
|  Description | Introduces a sui::checkpoint module to expose historical checkpoint data, enabling contracts to detect network liveness failures. |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

This proposal introduces a new, read-only system module, sui::checkpoint, to expose historical checkpoint sequence numbers and their corresponding consensus timestamps to Move smart contracts. This module functions as a native "liveness oracle," enabling developers to build robust, time-sensitive applications that can detect and react to network stalls or significant degradation in liveness.

## Motivation

Time-sensitive logic in DeFi and governance is vulnerable to exploits caused by the divergence between on-chain consensus time (sui::clock) and real-world time during network stalls. Contracts currently have no native way to measure the chain's operational progress against the passage of consensus time. This proposal provides a secure, on-chain primitive to bridge this gap, allowing contracts to verify that the network has been progressing at a normal rate, thus enhancing the security and reliability of the Sui ecosystem.

## Specification

This SIP introduces a new native module, sui::checkpoint, with the following public functions:
```
module sui::checkpoint {
    /// Returns the sequence number of the most recently finalized checkpoint.
    public native fun last_finalized_sequence_number(): u64;

    /// Returns the consensus timestamp in milliseconds for the checkpoint with the
    /// given `sequence_number`.
    /// Returns `Some(timestamp)` if the checkpoint is within the protocol-defined
    /// accessible history window, and `None` otherwise.
    public native fun get_timestamp_ms(sequence_number: u64): option::Option<u64>;

    /// Returns the sequence number of the oldest checkpoint available via `get_timestamp_ms`.
    /// This allows contracts to discover the bounds of accessible history.
    public native fun accessible_history_window_start(): u64;
}
```

## Rationale

This functionality must be implemented as a native module within the Sui runtime. The Move VM's security model intentionally prevents smart contracts from arbitrarily accessing the ledger's historical state, which is necessary for this feature. A native implementation is the only approach that is secure, performant, and avoids on-chain state bloat. It leverages the efficient, indexed database of checkpoints that Sui nodes already maintain for their own operation.

## Backwards Compatibility

This proposal is a purely additive and non-breaking change. It introduces a new, self-contained module and does not alter any existing modules, functions, or data structures. No existing smart contracts will be affected. 

## Test Cases

To be developed following initial review of the proposal.

## Reference Implementation

A reference implementation will be developed by core engineers if the SIP is approved.

## Security Considerations

1. Denial-of-Service (DoS) via Resource Exhaustion: A malicious actor could repeatedly call get_timestamp_ms for old checkpoints, consuming excessive validator resources.

  - Mitigation: The gas cost for get_timestamp_ms must include a variable component that scales with the age of the requested checkpoint (current_checkpoint - requested_checkpoint), making deep historical queries economically infeasible for attackers.

2. Validator State Bloat: Requiring validators to store an infinite history of checkpoint metadata is unsustainable and would increase centralization pressure.

  - Mitigation: The protocol must define and enforce a fixed-size, sliding window for the on-chain accessible history (e.g., the last 2-3 million checkpoints). Data older than this window would be pruned and only accessible via off-chain archival solutions.

## Copyright

Greshamscode 2025
