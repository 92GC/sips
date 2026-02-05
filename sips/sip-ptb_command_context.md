|   SIP-Number | |
|         ---: | :--- |
|        Title | PTB Command Context & Scoped Execution |
|  Description | Expose PTB command history and argument provenance to Move, enabling dynamic dispatch and isolated scopes |
|       Author | Greshamscode, @92GC |
|       Editor | |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-02-04 |
| Comments-URI | |
|       Status | |

## Abstract

Four related enhancements to PTBs:
1. **Command Context** - Expose previous command info (package, module, function) to Move
2. **Argument Provenance** - Track which command produced each argument passed to a function
3. **Scoped Execution** - Isolate groups of commands that share internal context but not external
4. **Scope Witness** - Cryptographic proof that commands belong to same scope

## Motivation

### The Hot Potato Problem

Today, composing Move protocols requires **application-level hot potatoes** - wrapper actions that must be pre-built for every external protocol integration.

Example: A DAO wants to execute "spend USDC → swap on Cetus → deposit result":

```
Current approach:
  Intent stages: [VaultSpend → CetusSwapWrapper → VaultDeposit]
                              ↑
                    Must pre-build this action
                    Type-checks coin flow at compile time
                    New protocol = new wrapper
```

This creates:
- **Action explosion** - Every protocol integration needs custom wrapper code
- **Upgrade burden** - Protocol upgrades require action updates
- **Reduced composability** - Can't use protocols without pre-built wrappers

### Dynamic Dispatch: The Solution

With PTB Command Context + Argument Provenance, DAOs can verify constraints at runtime:

```
New approach:
  Intent stages: [VaultSpend → <any approved protocol> → VaultDeposit]
                              ↑
                    No wrapper needed!
                    Runtime verification via ptb_context
```

The deposit action verifies:
1. "The coin I'm receiving came from command N" (argument provenance)
2. "Command N was a call to an approved package" (command context)

```move
public fun deposit_from_approved_source<CoinType>(
    ctx: &TxContext,
    coin: Coin<CoinType>,
) {
    // Verify coin came from a PTB Result (not raw Input)
    let source_idx = ptb_context::argument_source(ctx, 1); // arg 1 = coin
    assert!(source_idx.is_some(), ENotFromPTBResult);

    // Verify source command was approved protocol
    let source_cmd = ptb_context::command_at(ctx, *source_idx.borrow());
    assert!(is_approved_defi_protocol(source_cmd.package_id), EUnauthorizedSource);

    // Proceed with deposit...
}
```

**Result**: No more wrapper actions. Integrate any protocol by adding it to an approved list.

### Secondary Benefits

**Scoped Execution** prevents protocol censorship:

Without scopes, protocols could inspect PTB context and refuse to execute:
```move
// Malicious AMM could refuse arbitrage:
if (is_competing_amm(previous_command(ctx))) abort ENoArbitrageAllowed;
```

Scopes hide context between isolated groups of commands, preserving:
- Atomic execution (all-or-nothing)
- Free composability (any protocol combination)
- MEV resistance (patterns hidden from contracts)

## Specification

### Part 1: Command Context

```move
module sui::ptb_context {
    /// Command types in PTB - defined as enum for type safety
    public enum CommandType has copy, drop {
        MoveCall,
        TransferObjects,
        SplitCoins,
        MergeCoins,
        Publish,
        MakeMoveVec,
        Upgrade,
        Scope,
    }

    public struct CommandInfo has copy, drop {
        index: u16,
        command_type: CommandType,
        package_id: address,    // MoveCall/Upgrade only, @0x0 for others
        module_name: String,    // MoveCall only, empty for others
        function_name: String,  // MoveCall only, empty for others
    }

    /// Current command's index in PTB (or scope)
    public native fun current_index(ctx: &TxContext): u16;

    /// Total commands in current scope
    public native fun total_commands(ctx: &TxContext): u16;

    /// Get command info at index (must be < current_index, past only)
    public native fun command_at(ctx: &TxContext, index: u16): Option<CommandInfo>;

    /// Convenience: previous command
    public fun previous_command(ctx: &TxContext): Option<CommandInfo> {
        let idx = current_index(ctx);
        if (idx == 0) option::none()
        else command_at(ctx, idx - 1)
    }

    /// Scope nesting depth (1 = top-level PTB or explicit scope, increments for nested)
    /// NOTE: Top-level PTB is indistinguishable from explicit scope - both return 1
    /// This prevents contracts from detecting whether caller used Scope command
    public native fun scope_depth(ctx: &TxContext): u8;
}
```

### Part 2: Argument Provenance

```move
module sui::ptb_context {
    /// Argument source types
    public enum ArgumentSource has copy, drop {
        /// Value came from transaction Input (provided by sender)
        Input { input_index: u16 },
        /// Value came from a previous command's Result
        Result { command_index: u16, result_index: u16 },
        /// Value came from nested Result (e.g., Result(5, 0) for first return of command 5)
        NestedResult { command_index: u16, result_index: u16 },
    }

    /// Get the source of argument at given index for current command
    /// arg_index 0 = first argument (excluding &TxContext which is implicit)
    /// Returns None if argument tracking unavailable
    public native fun argument_source(ctx: &TxContext, arg_index: u8): Option<ArgumentSource>;

    /// Convenience: check if argument came from a specific command's result
    public fun argument_is_from_command(ctx: &TxContext, arg_index: u8, command_index: u16): bool {
        let source = argument_source(ctx, arg_index);
        if (source.is_none()) return false;

        match (*source.borrow()) {
            ArgumentSource::Result { command_index: idx, .. } => idx == command_index,
            ArgumentSource::NestedResult { command_index: idx, .. } => idx == command_index,
            _ => false,
        }
    }

    /// Convenience: get command index that produced this argument (if Result-based)
    public fun argument_command_index(ctx: &TxContext, arg_index: u8): Option<u16> {
        let source = argument_source(ctx, arg_index);
        if (source.is_none()) return option::none();

        match (*source.borrow()) {
            ArgumentSource::Result { command_index, .. } => option::some(command_index),
            ArgumentSource::NestedResult { command_index, .. } => option::some(command_index),
            _ => option::none(),
        }
    }
}
```

### Part 3: Scoped Execution

New `Command` variant:

```rust
enum Command {
    // ... existing ...
    Scope {
        commands: Vec<Command>,
        inherit_context: bool,  // Can inner see outer command history?
    },
}
```

**Semantics:**
- Commands inside scope share internal context
- `current_index()` resets to 0 inside scope
- `command_at()` only returns scope-internal commands (unless `inherit_context`)
- `argument_source()` returns indices relative to current scope
- Results can flow out via normal `Result(scope_idx)` references
- Scopes can nest (scope_depth increments)

**SDK:**
```typescript
ptb.scope({ inheritContext: false }, (scope) => {
    const pub = scope.publish({ modules, deps });
    scope.moveCall({ target: `...`, typeArguments: [scope.typeFromResult(pub, "mod", "Type")] });
});
```

### Part 4: Scope Witness

```move
module sui::ptb_context {
    /// Witness proving commands are in same scope
    public struct ScopeWitness has drop {
        scope_id: u256,
        creator_index: u16,
        scope_depth: u8,
    }

    /// Create witness (marks current command as scope anchor)
    public native fun create_scope_witness(ctx: &mut TxContext): ScopeWitness;

    /// Verify caller is in same scope as witness creator
    public native fun in_same_scope(ctx: &TxContext, witness: &ScopeWitness): bool;

    /// Get scope_id of current scope (unique per scope instance)
    public native fun current_scope_id(ctx: &TxContext): u256;
}
```

## Rationale

**Why expose command history?**
- Enables trustless verification of caller identity
- No opt-in required (unlike hot potato patterns)
- Read-only - cannot enforce ordering, just verify

**Why argument provenance?**
- Completes the verification story: know WHAT called you AND WHERE values came from
- Enables "accept coin only if it came from approved protocol" pattern
- Type safety preserved by Move's type system; provenance adds origin verification

**Why scopes?**
- Publish-and-use requires isolation (new types shouldn't leak)
- Composability with boundaries (protocol A calls protocol B without exposing internals)
- **Prevent AMM/protocol censorship** - contracts can't refuse execution based on surrounding PTB context
- **Preserve global liquidity** - AMMs remain freely composable for arbitrage
- **MEV resistance** - arbitrage patterns hidden from contracts (validators still see, but can't selectively abort)
- Gas accounting per scope (future: parallel execution hints)

**Why not expose future commands?**
- Would allow contracts to enforce specific PTB structure
- Breaks composability (can't add cleanup commands after)
- Creates ordering games between protocols

**Why not expose parameters?**
- Too expensive (arbitrary BCS data)
- Type safety issues (how to represent in Move?)
- Security risk (parameter inspection enables new attack vectors)
- Argument provenance provides sufficient information for most use cases

**Why not expose `is_scoped()`?**
- Would allow contracts to detect if caller used Scope command
- Defeats isolation purpose - protocols could refuse non-scoped calls
- Instead, `scope_depth()` returns 1 for both top-level PTB and explicit scope

**Alternatives rejected:**
- Full call stack introspection - too invasive, breaks function isolation assumptions
- Mutable context - allows state smuggling between commands

## Backwards Compatibility

Purely additive:
- New native functions in `sui::ptb_context`
- New `Scope` command variant
- Existing PTBs work unchanged
- `ptb_context` functions return sensible defaults for non-scoped execution
- `argument_source` returns `None` for commands executed before this feature

## Security Considerations

**Command Context:**
- Read-only - cannot modify history
- Only past commands visible (not future)
- Package ID is immutable (Original ID, not upgraded address)

**Argument Provenance:**
- Read-only tracking of PTB execution flow
- Cannot forge provenance (VM-tracked)
- Useful for allowlist-based verification, not for preventing all attacks
- Contracts should still validate values themselves, not just their source

**Scopes:**
- Inner scope cannot access outer scope's Results unless explicitly passed
- `inherit_context: false` provides strict isolation
- Scope witness cannot be forged (VM-generated scope_id)
- Argument provenance indices are scope-relative (cannot reference out-of-scope commands)

**Attack vectors to consider:**
- Scope escape via object references (mitigated: objects still owned normally)
- Context spoofing (mitigated: native functions, not user-controllable)
- Gas exhaustion via deep nesting (mitigated: max scope depth limit)
- Provenance-based allowlist bypass (mitigated: contracts should validate values, not just source)

**Design tension: Exposure vs Hiding**

Parts 1-2 (Context + Provenance) and Part 3 (Scopes) create intentional tension:
- Context exposure enables verifiable dispatch (good for governance, security)
- Scopes enable context hiding (good for composability/MEV resistance)

Resolution: **User controls the boundary**
- Protocols that WANT caller verification use Parts 1-2 (governance, vaults, DAOs)
- Protocols that SHOULD NOT discriminate are called within scopes (AMMs, orderbooks)
- User decides isolation level per-call via `inheritContext` flag

## Test Cases

To be developed:
- Command history accuracy across command types
- Argument provenance tracking for all argument sources (Input, Result, NestedResult)
- Scope isolation verification
- Nested scope behavior
- Scope witness validity checks
- inherit_context: true vs false behavior
- Cross-scope argument provenance (should not leak)

## Reference Implementation

To be developed.

## Open Questions

1. **Max scope depth?** Suggest 64 (sufficient for most use cases, limits complexity)
2. **Scope gas limits?** Should scopes have independent gas budgets?
3. **Result visibility?** Can outer PTB reference inner scope Results by index?
4. **Error semantics?** Does inner scope failure abort entire PTB? (suggest yes - atomic)
5. **Type interpolation interaction?** Should `typeFromResult` work across scope boundaries?
6. **Argument indexing?** Should `&TxContext` count as arg 0, or be excluded from indexing?
7. **Nested result granularity?** Should `argument_source` distinguish between tuple elements?

## Related SIPs

- PTB Type Argument Interpolation (enables `typeFromResult` for publish-and-use)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
