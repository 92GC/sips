|   SIP-Number | |
|         ---: | :--- |
|        Title | PTB Type Argument Interpolation |
|  Description | Enables type arguments in PTB commands to reference Results from prior commands, allowing publish-and-use in a single transaction. |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

This proposal extends Programmable Transaction Blocks (PTBs) to allow type arguments in `MoveCall` commands to reference `Result` values from prior commands in the same transaction. Currently, type arguments must be fully-resolved string literals at PTB construction time. This limitation prevents "publish-and-use" workflows where a module is deployed and immediately used in the same transaction.

## Motivation

### The Problem: Two-Transaction Minimum for New Coin Types

Sui's type system requires all generic type parameters to be known at compile time. When building a PTB, type arguments must be string literals like `"0xabc123::my_module::MyType"`.

This creates a fundamental limitation: **you cannot use a type from a package you just published in the same PTB**.

```typescript
const ptb = new Transaction();

// Step 1: Publish a new coin module
const [upgradeCap] = ptb.publish({
  modules: compiledModules,
  dependencies: [...]
});
// upgradeCap contains the new package ID, but it's a Result - a runtime value

// Step 2: Try to use the new coin type - FAILS
ptb.moveCall({
  target: `${existingPkg}::escrow::register_coin`,
  typeArguments: [`${???}::my_coin::MY_COIN`], // Can't interpolate Result!
  arguments: [...],
});
```

The `publish` command returns the package ID as a `Result`, but type arguments only accept string literals. There is no way to construct a type string that references the just-published package.

### Real-World Impact

This limitation forces protocols to choose between:

1. **Two-transaction workflows**: Publish in one transaction, use in another (poor UX, race conditions)
2. **Pre-deployed type pools**: Maintain registries of pre-created types that can be acquired on-demand (complex infrastructure, coordination overhead)
3. **Factory patterns with limited flexibility**: Use existing types instead of creating new ones (limits expressiveness)

None of these are ideal. The core issue is that "publish and use atomically" is impossible.

### Who Benefits

- **Token launchpads**: Single-click token creation with immediate liquidity pool setup
- **Prediction markets**: Unique coin types per outcome, created and used atomically
- **Gaming/NFT protocols**: Mint new item types and immediately use them in-game
- **DeFi protocols**: Deploy new pool types and seed initial liquidity in one transaction
- **Any protocol needing dynamic types**: Currently forced into multi-transaction flows or complex workarounds

## Specification

### New PTB Command Variant

Extend the `MoveCall` command to accept a new type argument format:

```typescript
interface TypeArgumentFromResult {
  kind: "Result";
  result: TransactionResult;  // Reference to a prior Publish command
  module: string;             // Module name within the published package
  type: string;               // Type name within the module
  typeParams?: TypeArgument[]; // Optional nested type parameters
}

type TypeArgument = string | TypeArgumentFromResult;
```

### SDK Usage

```typescript
const ptb = new Transaction();

// Publish returns package info including the package ID
const publishResult = ptb.publish({
  modules: compiledModules,
  dependencies: [...]
});

// Use the new type immediately
ptb.moveCall({
  target: `${existingPkg}::escrow::register_coin`,
  typeArguments: [
    ptb.typeFromResult(publishResult, "my_coin", "MY_COIN")
  ],
  arguments: [registry, treasuryCap, ...],
});
```

### Transaction Execution Semantics

1. **Ordering**: Type argument resolution happens **after** prior commands complete but **before** the Move VM executes the current command
2. **Validation**: The runtime verifies that:
   - The referenced `Result` is from a `Publish` command
   - The `Publish` command succeeded
   - The specified module and type exist in the published package
3. **Type String Construction**: The runtime constructs the full type string: `{package_id}::{module}::{type}`
4. **Failure Handling**: If resolution fails (module doesn't exist, type doesn't exist), the transaction aborts

### BCS Encoding

For transaction serialization, add a new variant to the type argument enum:

```rust
enum TypeArgument {
    // Existing: fully resolved type string
    Pure(String),
    // New: reference to a Result
    FromResult {
        result_index: u16,
        module_name: String,
        type_name: String,
        type_params: Vec<TypeArgument>,
    },
}
```

## Rationale

### Why This Approach?

**Option 1: Runtime Type Creation (Rejected)**
Adding dynamic type creation (`coin::create_dynamic()`) would fundamentally change Move's static type system. Move's security guarantees depend on types being known at compile time. This is too invasive.

**Option 2: Deferred Type Resolution / Closures (Rejected)**
Adding closures or callbacks to Move would require language-level changes: new syntax, compiler modifications, VM changes, gas metering for closures. This is a much larger undertaking.

**Option 3: PTB Type Interpolation (This Proposal)**
This is the most surgical approach:
- No Move language changes
- No Move bytecode verifier changes
- Move contracts remain unchanged
- Only the PTB execution layer needs modification
- Sui already does Result-based resolution for object arguments

### Implementation Complexity

The Sui runtime already:
- Tracks `Result` values from prior PTB commands
- Resolves object references at execution time
- Knows the package ID from successful `Publish` commands

Extending this to type arguments is a natural evolution of existing infrastructure.

### No Breaking Changes

Existing PTBs continue to work exactly as before. The new `FromResult` type argument variant is additive.

## Backwards Compatibility

This proposal is purely additive:
- Existing transactions with string-literal type arguments work unchanged
- Existing SDK code works unchanged
- Only new code using `typeFromResult` would use the new capability

SDKs would need updates to support the new syntax, but older SDKs would continue to function for existing use cases.

## Test Cases

### Basic Publish-and-Use

```typescript
const ptb = new Transaction();
const pub = ptb.publish({ modules: coinModule, dependencies: [] });

ptb.moveCall({
  target: "0x2::coin::mint",
  typeArguments: [ptb.typeFromResult(pub, "my_coin", "MY_COIN")],
  arguments: [treasuryCap, amount],
});
```

Expected: Single transaction publishes coin module and mints initial supply.

### Nested Type Parameters

```typescript
ptb.moveCall({
  target: `${pkg}::pool::create`,
  typeArguments: [
    ptb.typeFromResult(pub, "token_a", "TOKEN_A"),
    ptb.typeFromResult(pub, "token_b", "TOKEN_B"),
    `${lpPkg}::lp::LP`
  ],
  arguments: [...],
});
```

Expected: Type arguments can mix Result-based and literal types.

### Invalid Module Reference

```typescript
ptb.moveCall({
  typeArguments: [ptb.typeFromResult(pub, "nonexistent", "TYPE")],
  ...
});
```

Expected: Transaction aborts with clear error: "Module 'nonexistent' not found in package {id}".

## Reference Implementation

A reference implementation would modify:

1. **Transaction format**: Add `FromResult` variant to type argument enum
2. **PTB executor**: Resolve `FromResult` type arguments before Move execution
3. **SDK**: Add `typeFromResult` helper method

The core change is in the PTB executor's command processing loop:

```rust
fn resolve_type_arguments(
    type_args: Vec<TypeArgument>,
    results: &[CommandResult],
) -> Result<Vec<String>, Error> {
    type_args.into_iter().map(|arg| match arg {
        TypeArgument::Pure(s) => Ok(s),
        TypeArgument::FromResult { result_index, module_name, type_name, type_params } => {
            let publish_result = &results[result_index as usize];
            let package_id = publish_result.as_publish()?.package_id;
            let resolved_params = resolve_type_arguments(type_params, results)?;
            Ok(format_type_string(package_id, module_name, type_name, resolved_params))
        }
    }).collect()
}
```

## Security Considerations

1. **No New Capabilities**: This does not grant any new ability to create types at runtime. Types must still be defined in published modules. This only changes *when* the type string is resolved.

2. **Ordering Guarantees**: The `Publish` command must complete successfully before any command can reference its types. The PTB executor already enforces command ordering.

3. **Validation**: Type existence is validated at resolution time. Invalid module/type references abort the transaction cleanly.

4. **No Type Spoofing**: The package ID comes directly from the `Publish` result - callers cannot inject arbitrary addresses.

5. **Gas Metering**: Type resolution is a string operation with minimal overhead. No new gas considerations beyond existing string operations.

## Copyright

Greshamscode 2025
