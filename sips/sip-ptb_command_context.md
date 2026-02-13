|   SIP-Number | |
|         ---: | :--- |
|        Title | PTB Command Context & Scoped Execution |
|  Description | Expose PTB command metadata and argument provenance to Move; add scoped execution for isolation |
|       Author | Greshamscode, @92GC |
|       Editor | |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-02-04 |
| Comments-URI | |
|       Status | |

## Abstract

Four additions to PTBs:
1. **Command Context** — Any command in a scope can read metadata (package, module, function) of every other command in that scope.
2. **Argument Provenance** — Track which command produced each argument, enabling origin verification.
3. **Scoped Execution** — Partition PTB commands into isolated groups that share internal context but hide it from the outside.
4. **Scope Witness** — Proof that two commands executed in the same scope.

## Motivation

Composing Move protocols today requires hot potato wrappers for every integration. A DAO executing "vault spend → DEX swap → vault deposit" needs a custom wrapper action for each DEX. New DEX = new wrapper code.

With command context + argument provenance, the deposit function can verify at runtime that the coin it received came from an approved package — no wrapper needed:

```move
public fun deposit_from_approved<CoinType>(ctx: &TxContext, coin: Coin<CoinType>) {
    let source = ptb_context::argument_source(ctx, 0);
    assert!(source.is_some(), ENotFromPTBResult);
    let cmd = ptb_context::command_at(ctx, source.borrow().command_index());
    assert!(is_approved(cmd.package_id), EUnauthorizedSource);
}
```

Scopes solve the flipside: without them, a protocol could inspect context and refuse execution when it sees a competitor in the same PTB. Scopes partition commands so each group only sees its own context, preventing censorship while keeping the transaction atomic.

## Specification

### 1. Command Context

```move
module sui::ptb_context {
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
        package_id: address,    // MoveCall/Upgrade only, @0x0 otherwise
        module_name: String,    // MoveCall only
        function_name: String,  // MoveCall only
    }

    /// Current command's index in its scope
    public native fun current_index(ctx: &TxContext): u16;

    /// Total commands in current scope
    public native fun total_commands(ctx: &TxContext): u16;

    /// Get info for any command in the current scope
    public native fun command_at(ctx: &TxContext, index: u16): Option<CommandInfo>;

    /// Scope nesting depth. Top-level PTB and explicit scope both return 1
    /// (indistinguishable by design — prevents contracts from detecting scope usage)
    public native fun scope_depth(ctx: &TxContext): u8;
}
```

All commands within a scope can see all other commands in that scope. Visibility does not cross scope boundaries unless `inherit_context` is set.

### 2. Argument Provenance

```move
module sui::ptb_context {
    public enum ArgumentSource has copy, drop {
        Input { input_index: u16 },
        Result { command_index: u16, result_index: u16 },
        NestedResult { command_index: u16, result_index: u16 },
    }

    /// Source of the argument at the given index (excluding implicit &TxContext)
    public native fun argument_source(ctx: &TxContext, arg_index: u8): Option<ArgumentSource>;

    /// Check if argument came from a specific command
    public fun argument_is_from_command(ctx: &TxContext, arg_index: u8, command_index: u16): bool {
        let source = argument_source(ctx, arg_index);
        if (source.is_none()) return false;
        match (*source.borrow()) {
            ArgumentSource::Result { command_index: idx, .. } => idx == command_index,
            ArgumentSource::NestedResult { command_index: idx, .. } => idx == command_index,
            _ => false,
        }
    }
}
```

### 3. Scoped Execution

New PTB command variant:

```rust
enum Command {
    // ... existing variants ...
    Scope {
        commands: Vec<Command>,
        inherit_context: bool,
    },
}
```

- `current_index()` resets to 0 inside a scope
- `command_at()` returns only scope-internal commands (unless `inherit_context: true`)
- Argument provenance indices are scope-relative
- Results flow out via normal `Result(scope_idx)` references
- Scopes nest; `scope_depth` increments accordingly

SDK usage:
```typescript
ptb.scope({ inheritContext: false }, (scope) => {
    const pub = scope.publish({ modules, deps });
    scope.moveCall({ target: `...`, typeArguments: [scope.typeFromResult(pub, "mod", "Type")] });
});
```

### 4. Scope Witness

```move
module sui::ptb_context {
    public struct ScopeWitness has drop {
        scope_id: u256,
        creator_index: u16,
        scope_depth: u8,
    }

    public native fun create_scope_witness(ctx: &mut TxContext): ScopeWitness;
    public native fun in_same_scope(ctx: &TxContext, witness: &ScopeWitness): bool;
    public native fun current_scope_id(ctx: &TxContext): u256;
}
```

## Rationale

**Full scope visibility (not past-only):** Restricting to past commands would still allow ordering games. Full visibility within a scope is simpler and lets contracts verify the complete execution context they're part of. Scopes are the isolation boundary, not command ordering.

**Scopes, not `is_scoped()`:** Exposing whether a call is scoped would let protocols refuse non-scoped calls, defeating the purpose. `scope_depth()` returns 1 for both top-level and explicit scopes — indistinguishable.

**No parameter exposure:** Too expensive (arbitrary BCS), type-unsafe, and argument provenance covers the important cases.

**Exposure vs hiding is intentional:** Context lets governance protocols verify callers. Scopes let DeFi protocols stay composable. The user picks the boundary per call.

## Backwards Compatibility

Purely additive. Existing PTBs work unchanged. `ptb_context` functions return sensible defaults pre-feature. `argument_source` returns `None` for pre-feature commands.

## Security Considerations

- Read-only — commands can inspect context but not modify it
- All commands in a scope see each other; scopes are the privacy boundary
- Package IDs are Original IDs (immutable, not upgraded addresses)
- Provenance is VM-tracked and unforgeable
- Scope witnesses are VM-generated — cannot be spoofed
- Max scope depth limit prevents gas exhaustion via deep nesting
- Contracts should validate values, not just their provenance

## Open Questions

1. Max scope depth? (suggest 64)
2. Should scopes have independent gas budgets?
3. Can outer PTB reference inner scope Results by index?
4. Does inner scope failure abort the entire PTB? (suggest yes)
5. Should `typeFromResult` work across scope boundaries?
6. Should `&TxContext` count as arg 0 or be excluded from indexing?

## Test Cases

To be developed.

## Reference Implementation

To be developed.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
