---
simd: '0597'
title: 'Loader V3: Truncate'
authors:
    - frank
    - cavey
    - Joe C (Anza)
category: Standard
type: Core
status: Draft
created: 2026-09-05
feature: (fill in with feature key and github tracking issues once accepted)
---

## Summary

This SIMD proposes the addition of a `Truncate` instruction to Loader v3, reducing the programdata buffer size to the size of the ELF and reclaiming all excess rent stored in the programdata buffer account.

## Motivation

There are several programs that have over-allocated buffer space. As of SIMD-0186, the programdata buffer size is used in calculating `loaded_accounts_data_cost`. As of SIMD-0437, the rent required for the programdata buffers has also been strictly reduced.  We would like to allow devs to reclaim the excess rent.

## New Terminology

No new terminology is introduced by this proposal.

## Detailed Design

We propose adding the following variant to the `UpgradeableLoaderInstruction` enum.

```rust
enum UpgradeableLoaderInstruction {
  // ...

  /// Truncate a programdata or buffer account to the minimum size.
  ///
  /// # Account references
  ///   0. `[writable]` The ProgramData or Buffer account.
  ///   1. `[writable]` The ProgramData account's associated Program account.
  ///   2. `[signer]` The upgrade authority on the ProgramData/Buffer account.
  ///   3. `[writable]` The recipient account, optional, that will receive excess lamports
  ///      from the ProgramData/Buffer account. No account provided means
  ///      all lamports remain in the account.
  ///   4. `[]` System program (`solana_sdk::system_program::id()`), optional, used to transfer
  ///      lamports from the ProgramData/Buffer account to the recipient.
  Truncate,
}
```

### Workflow

We can calculate the minimum necessary ELF size by doing something like this.

```
elf_end = max(
    e_ehsize,
    e_phoff + e_phnum * e_phentsize,
    e_shoff + e_shnum * e_shentsize,
    max(p_offset + p_filesz),
    max(sh_offset + sh_size for sections excluding SHT_NOBITS)
)
extra_padding = account_size - elf_end
```

And then the new Loader v3 `Truncate` handler can do:

```rust
let padding = find_padding(program_account)?;
let current_len = program_data_account.get_data().len();
let new_len = current_len.saturating_sub(padding);

// this should not occur
if new_len > current_len {
    return Err(InstructionError::InvalidAccountData);
}

// resize if necessary
if new_len != current_len {
    program_data_account.set_data_length(new_len)?;

    let executable = Executable::<InvokeContext>::load(
        programdata,
        Arc::new(deployment_program_runtime_environment),
    )?;

    // just revert if we messed up the sizing somehow
    executable.verify::<RequisiteVerifier>()?;
}

// reclaim rent if recipient is provided
if let Some(recipient) = recipient_maybe {
    let new_rent_requirement = 1.max(rent.minimum_balance(new_len));
    let excess = program_data_account.get_lamports().saturating_sub(new_rent_requirement);

    program_data_account.checked_sub_lamports(excess)?;
    recipient.checked_add_lamports(excess)?;
}
```

### Rent Reclamation

Even if the programdata buffer is already at its minimum size there is still potentially excess rent as a result of SIMD-0437.  The excess rent calculation should then use `Rent::minimum_balance` to calculate the minimum required in any `Truncate` invocation with a provided `recipient` account, regardless of whether the programdata buffer was resized, and sweep any stored lamports in excess of the minimum balance to the `recipient`.

## Alternatives Considered

SIMD-0433. This proposal supersedes 0433 for the following reasons:

- A dev may want to just reclaim excess rent without doing a program upgrade.
- A dev can just add a `Truncate` instruction in their upgrade workflow if they want to do an upgrade and then size to fit.

## Impact

Program devs will now be able to much more easily reclaim excess rent from programdata buffer accounts. As of SIMD-0437 this instruction is a significantly more attractive proposal.

## Security Considerations

If we did not verify the program ELF after resizing this could result in invalid / malformed ELFs being persisted to ledger.  We should treat this operation equivalently to `Deploy` / `Upgrade` and perform all requisite checks.

## Backwards Compatibility

This change adds a new instruction to Loader v3 and therefore requires a feature gate to enable.  Current program dev workflows are unchanged as this proposal does not alter the semantics of any current instructions.

Tooling, e.g. `solana program` CLI, is backwards compatible with this change but requires updates to access the new `Truncate` instruction.