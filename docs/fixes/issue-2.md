# Issue #2: Emit event on admin handover in all four contracts

## Problem

None of the four contracts (`compliance`, `asset-token`, `registry`,
`dividend`) had any way to change the admin after `initialize` — the admin
address was set once and then read-only for the life of the contract. Since
admin is the single most security-relevant piece of state (it gates minting,
compliance changes, KYC decisions, asset deactivation, and distribution
creation across the whole platform), the lack of a rotation path was itself a
gap: operators had no way to rotate a compromised or retiring admin key
without redeploying, and — the specific ask in the issue — even if such a
path existed, off-chain indexers would have had no event to observe the
transition.

## Change

Added a `transfer_admin(admin: Address, new_admin: Address)` function to each
of the four contracts, following the existing `require_admin`/`bump` patterns
already used by every other admin-gated call in these contracts:

- `contracts/compliance/src/lib.rs` — `transfer_admin` calls the existing
  `require_admin` helper (current admin must auth and match storage), writes
  `new_admin` into `DataKey::Admin`, refreshes the instance TTL via
  `bump_instance`, and publishes the event.
- `contracts/registry/src/lib.rs` — same shape, using the contract's local
  `require_admin`/`bump` helpers.
- `contracts/dividend/src/lib.rs` — same shape, using the contract's local
  `require_admin`/`bump` helpers.
- `contracts/asset-token/src/lib.rs` — admin lives inside the `AssetMetadata`
  struct rather than a standalone storage key here, so `transfer_admin` loads
  metadata via `require_admin` (which already returns it), overwrites
  `meta.admin`, and rewrites the whole metadata record. This is independent
  of the optional `guardian` role added for issue #1 — transferring admin
  does not touch or clear the guardian.

Every variant emits the same event shape:

```rust
env.events()
    .publish((symbol_short!("set_admin"), admin), new_admin);
```

The old admin is part of the topic (consistent with how other events in
these contracts index the relevant address, e.g. `suspend`/`removed` in
compliance), and the new admin is the payload — so an indexer gets both
addresses involved in the handover directly off the event, without a
separate storage read.

In every contract, only the **current** admin can call `transfer_admin` — it
reuses the same `require_admin` authorization check as every other
admin-gated mutation, so a handover requires the outgoing admin's signature
just like minting or configuration changes do today.

## Docs

Updated each contract's docs page under `docs/` with:

- A `transfer_admin(admin, new_admin)` entry in the "Functions" section.
- A `set_admin` row in the "Events" table, documenting the `(old_admin) →
  new_admin` shape.

Files touched: `docs/compliance.md`, `docs/asset-token.md`,
`docs/registry.md`, `docs/dividend.md`.

## Why this satisfies the issue

- An event (`set_admin`) is now published whenever admin changes, in all
  four contracts.
- It carries both the previous admin (topic) and the new admin (payload),
  so indexers don't need to diff two `get_admin()` reads to notice or
  reconstruct a handover.
- Each contract's docs page now documents the function and the event.
