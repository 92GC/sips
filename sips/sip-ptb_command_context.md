|   SIP-Number | |
|         ---: | :--- |
|        Title | PTB Command Context & Scoped Execution |
|  Description | Expose PTB command history to Move and enable isolated execution scopes within transactions |
|       Author | Greshamscode, @92GC |
|       Editor | |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-02-04 |
| Comments-URI | |
|       Status | |

## Abstract

Three related enhancements to PTBs:
1. **Command Context** - Expose previous command info (package, module, function) to Move
2. **Scoped Execution** - Isolate groups of commands that share internal context but not external
3. **Scope Witness** - Cryptographic proof that commands belong to same scope

## Motivation

Move functions are blind to their PTB context:
- Cannot verify which package/function called them
- Cannot see PTB command history
- No way to create isolated "sub-transactions" within atomic PTB

**Use cases:**
- Verifiable dynamic dispatch without application-level hot potatoes
- Publish-and-use with type interpolation (see SIP: PTB Type Argument Interpolation)
- Isolated token launches where inner commands can't leak to outer PTB
- Cross-protocol trust verification
- **Preventing AMM censorship of arbitrage** (see below)

### AMM Censorship Problem

Without scopes, AMMs could inspect PTB context and refuse to execute if they detect arbitrage:

```move
// Malicious AMM could do this:
public fun swap(ctx: &TxContext, ...) {
    let prev = ptb_context::previous_command(ctx);
    let next = ptb_context::command_at(ctx, ptb_context::current_index(ctx) + 1);

    // Refuse if sandwich detected
    if (is_competing_amm(prev) || is_competing_amm(next)) {
        abort ENoArbitrageAllowed
    };
}
```

**Consequences:**
- **Liquidity fragmentation** - AMMs can't be freely composed, fragmenting global liquidity
- **Validator MEV** - Validators see full PTB before execution, can front-run/censor arbitrage
- **Rent extraction** - AMMs become gatekeepers, extracting value from traders

**Scopes solve this:**
```typescript
// User wraps each AMM call in isolated scope
ptb.scope({ inheritContext: false }, (s) => s.moveCall({ target: "amm_a::swap" }));
ptb.scope({ inheritContext: false }, (s) => s.moveCall({ target: "amm_b::swap" }));
// Neither AMM can see the other exists in this PTB
```

This preserves:
- Atomic execution (both swaps succeed or both fail)
- Composability (any AMM can be combined)
- MEV resistance (pattern not visible to contracts)

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
        package_id: address,    // MoveCall/Upgrade only
        module_name: String,    // MoveCall only
        function_name: String,  // MoveCall only
    }

    /// Current command's index in PTB (or scope)
    public native fun current_index(ctx: &TxContext): u16;

    /// Total commands in current scope
    public native fun total_commands(ctx: &TxContext): u16;

    /// Get command info at index (must be < current_index)
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

### Part 2: Scoped Execution

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
- Results can flow out via normal `Result(scope_idx)` references
- Scopes can nest (scope_depth increments)

**SDK:**
```typescript
ptb.scope({ inheritContext: false }, (scope) => {
    const pub = scope.publish({ modules, deps });
    scope.moveCall({ target: `...`, typeArguments: [scope.typeFromResult(pub, "mod", "Type")] });
});
```

### Part 3: Scope Witness

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

**Why scopes?**
- Publish-and-use requires isolation (new types shouldn't leak)
- Composability with boundaries (protocol A calls protocol B without exposing internals)
- **Prevent AMM/protocol censorship** - contracts can't refuse execution based on surrounding PTB context
- **Preserve global liquidity** - AMMs remain freely composable for arbitrage
- **MEV resistance** - arbitrage patterns hidden from contracts (validators still see, but can't selectively abort)
- Gas accounting per scope (future: parallel execution hints)

**Why not expose parameters?**
- Too expensive (arbitrary BCS data)
- Type safety issues (how to represent in Move?)
- Security risk (parameter inspection enables new attack vectors)

**Why not expose `is_scoped()`?**
- Would allow contracts to detect if caller used Scope command
- Defeats isolation purpose - AMMs could refuse non-scoped calls
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

## Security Considerations

**Command Context:**
- Read-only - cannot modify history
- Only past commands visible (not future)
- Package ID is immutable (Original ID, not upgraded address)

**Scopes:**
- Inner scope cannot access outer scope's Results unless explicitly passed
- `inherit_context: false` provides strict isolation
- Scope witness cannot be forged (VM-generated scope_id)

**Attack vectors to consider:**
- Scope escape via object references (mitigated: objects still owned normally)
- Context spoofing (mitigated: native functions, not user-controllable)
- Gas exhaustion via deep nesting (mitigated: max scope depth limit)

**Design tension: Exposure vs Hiding**

Part 1 (Command Context) and Part 2 (Scopes) create intentional tension:
- Context exposure enables verifiable dispatch (good for security)
- Scopes enable context hiding (good for composability/MEV resistance)

Resolution: **User controls the boundary**
- Protocols that WANT caller verification use Part 1 (governance, vaults)
- Protocols that SHOULD NOT discriminate are called within scopes (AMMs, orderbooks)
- User decides isolation level per-call via `inheritContext` flag

## Test Cases

To be developed:
- Command history accuracy across command types
- Scope isolation verification
- Nested scope behavior
- Scope witness validity checks
- inherit_context: true vs false behavior

## Reference Implementation

To be developed.

## Open Questions

1. **Max scope depth?** Suggest 64 (sufficient for most use cases, limits complexity)
2. **Scope gas limits?** Should scopes have independent gas budgets?
3. **Result visibility?** Can outer PTB reference inner scope Results by index?
4. **Error semantics?** Does inner scope failure abort entire PTB? (suggest yes - atomic)
5. **Type interpolation interaction?** Should `typeFromResult` work across scope boundaries?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
