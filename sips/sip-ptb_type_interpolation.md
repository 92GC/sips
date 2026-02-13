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

PTB type args only accept string literals. If you publish a package, you don't know the package ID until the tx lands — so you can't use your new types as generics in the same transaction. This adds a `FromResult` variant to the type argument encoding and `typeFromResult()` to the SDK. No Move or VM changes — the executor already resolves Result references for objects, this extends that to type args.

## Motivation

You cannot use a type from a package you just published in the same PTB:

```typescript
const ptb = new Transaction();
const [upgradeCap] = ptb.publish({ modules, dependencies });

// FAILS - type arguments only accept string literals
ptb.moveCall({
  target: `${pkg}::escrow::register_coin`,
  typeArguments: [`${???}::my_coin::MY_COIN`], // Can't interpolate Result!
});
```

This forces two-transaction workflows for token launches, prediction markets, and any protocol creating types dynamically.

## Specification

Add a new type argument variant that references a `Publish` result:

```typescript
// SDK usage
const pub = ptb.publish({ modules, dependencies });
ptb.moveCall({
  typeArguments: [ptb.typeFromResult(pub, "my_coin", "MY_COIN")],
});
```

```rust
// BCS encoding
enum TypeArgument {
    Pure(String),
    FromResult {
        result_index: u16,
        module_name: String,
        type_name: String,
        type_params: Vec<TypeArgument>,
    },
}
```

The runtime resolves `FromResult` after the `Publish` completes, constructing `{package_id}::{module}::{type}`.

## Rationale

This is surgical—no Move language or bytecode verifier changes. The PTB executor already resolves `Result` references for objects; extending this to type arguments is natural.

Alternatives rejected:
- **Runtime type creation**: Too invasive to Move's static type system
- **Closures/deferred execution**: Requires language-level changes

## Backwards Compatibility

Purely additive. Existing PTBs work unchanged.

## Test Cases

To be developed.

## Reference Implementation

To be developed.

## Security Considerations

- **No new capabilities**: Types must still be defined in published modules
- **Ordering enforced**: `Publish` must succeed before types can be referenced
- **Validation**: Invalid module/type references abort cleanly

## Copyright

Greshamscode 2025
