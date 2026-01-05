|   SIP-Number | |
|         ---: | :--- |
|        Title | Permissionless Package Introspection |
|  Description | Adds native functions to sui::package for querying package metadata without holding the UpgradeCap |
|       Author | Greshamscode, @92GC |
|       Editor | <Leave this blank; it will be assigned by a SIP Editor> |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-06-25 |
| Comments-URI | |
|       Status | |
|     Requires | |

## Abstract

Add native functions to `sui::package` that allow querying package metadata—upgrade policy, version, immutability, and currently executing package ID—without holding the `UpgradeCap`.

## Motivation

All existing package introspection requires the `UpgradeCap`:

```move
// Current API - requires UpgradeCap
public fun upgrade_policy(cap: &UpgradeCap): u8;
public fun version(cap: &UpgradeCap): u64;
```

Third-party developers keep their `UpgradeCap`, making it impossible to:

1. **Verify third-party package security**: A DAO can't enforce "only allow immutable or additive-only modules" because there's no way to check `is_immutable(package_id)` without the cap.

2. **Pin execution to audited versions**: No way to answer "which physical package version is executing right now?" for governance version pinning. The existing `type_name::defining_id<T>()` only works for types first defined in the current version.

## Specification

```move
module sui::package {
    /// Physical Object ID of the currently executing package
    public native fun current_package_id(): address;

    /// Upgrade policy for any package (0=COMPATIBLE, 128=ADDITIVE, 192=DEP_ONLY, 255=IMMUTABLE)
    public native fun policy_for(package_id: address): u8;

    /// True if package is immutable
    public native fun is_immutable(package_id: address): bool;

    /// Current version of any package
    public native fun version_for(package_id: address): u64;
}
```

## Rationale

- **Why `sui::package`?** Package upgrades are Sui-specific. This module already contains all upgrade types (`UpgradeCap`, etc.).
- **Why new function names?** `policy_for` vs `upgrade_policy` distinguishes "permissionless query" from "I hold the cap".
- **Why native?** Move VM doesn't expose ledger state; native functions bridge package metadata into execution context.

## Backwards Compatibility

Purely additive. Existing `UpgradeCap` functions unchanged.

## Test Cases

To be developed.

## Reference Implementation

To be developed.

## Security Considerations

- **Read-only**: No power to upgrade, only observe
- **Increases security**: Enables trustless verification of third-party packages

## Copyright

Greshamscode 2025
