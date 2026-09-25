# Issue #4: Implement propose/accept two-step admin handover

## Problem

All four contracts (`compliance`, `asset-token`, `registry`, `dividend`)
stored a single admin address and exposed `transfer_admin(admin, new_admin)`,
which moved the role in one step, driven entirely by the *current* admin.
Because `new_admin` never had to prove it controlled that address, a typo or
a copy-paste mistake would set the admin to an address nobody holds a key
for. Every privileged operation in that contract (KYC decisions, minting,
pausing, registering/deactivating assets, creating distributions) would then
be permanently unreachable, with no recovery path.

## Change

In each contract, `transfer_admin` was replaced with a three-function
propose/accept/cancel flow:

- **`propose_admin(admin, new_admin)`** — current-admin-only. Records
  `new_admin` as a pending proposal. Does **not** change who the admin is.
- **`accept_admin(new_admin)`** — callable only by the proposed successor
  (`new_admin.require_auth()` plus a check that the caller matches the
  stored proposal). This is the only place the admin actually changes.
  Clears the pending proposal and emits `set_admin` with `(old_admin,
  new_admin)`, the same event shape the old `transfer_admin` emitted, so
  existing indexers built for issue #2 keep working.
- **`cancel_admin_proposal(admin)`** — current-admin-only. Clears a pending
  proposal without ever touching the actual admin. Panics with a new
  `NoPendingAdmin` error if nothing is pending.

A new `get_pending_admin()` view returns the address currently proposed, or
`None`.

Because the role never moves in `propose_admin`, a mistyped or unreachable
`new_admin` is harmless: the current admin keeps full control and can simply
re-propose (overwriting the pending value) or cancel. The handover only
completes once the successor proves control of their own address by calling
`accept_admin`.

### Storage / error changes per contract

- `contracts/compliance/src/lib.rs` — added `DataKey::PendingAdmin` (instance
  storage) and `Error::NoPendingAdmin` (`= 7`).
- `contracts/registry/src/lib.rs` — same shape: `DataKey::PendingAdmin`,
  `Error::NoPendingAdmin` (`= 8`).
- `contracts/dividend/src/lib.rs` — same shape: `DataKey::PendingAdmin`,
  `Error::NoPendingAdmin` (`= 12`).
- `contracts/asset-token/src/lib.rs` — this contract keeps its admin as a
  field on `AssetMetadata` rather than a standalone storage key, so the
  pending proposal was added the same way as the existing `guardian` field:
  a new `pending_admin: Option<Address>` field on `AssetMetadata`,
  initialized to `None` in `initialize`. Added `Error::NoPendingAdmin`
  (`= 12`). The unrelated `guardian` field (issue #1) is untouched by any of
  this — a handover does not reset or require a guardian.

All four contracts reuse the existing `require_admin` authorization check
for `propose_admin`/`cancel_admin_proposal` (must be the *current* stored
admin, and must authorize the call), and use `new_admin.require_auth()` plus
an explicit equality check against the stored pending address for
`accept_admin`.

## Tests

Each contract's `test.rs` gained an "admin handover (issue #4)" section
covering the "Done when" criteria from the issue:

- **Propose + accept**: the role stays with the old admin immediately after
  `propose_admin`, moves only after `accept_admin`, and the old admin
  provably loses privileged access afterward (a representative privileged
  call from each contract — `add_to_allowlist`, `mint`, `deactivate_asset`,
  `create_distribution` — is retried with the old admin and asserted to fail
  with `Unauthorized`).
- **Cancel**: the current admin can cancel a pending proposal; the cancelled
  successor can no longer accept (`NoPendingAdmin`); cancelling with nothing
  pending also fails with `NoPendingAdmin`.
- **Unauthorised attempts**: a non-admin cannot propose or cancel
  (`Unauthorized`); an address other than the proposed successor cannot
  accept (`Unauthorized`); accepting with no proposal on file fails
  (`NoPendingAdmin`).
- **Re-proposal**: proposing a second successor overwrites the first, and
  the original proposed address can no longer accept (compliance contract;
  the same behavior holds in all four since they share the same
  `propose_admin` implementation shape).
- **Asset-token specific**: a handover leaves the independent `guardian`
  role (issue #1) untouched.

## Why this satisfies the issue

- A pending admin can be proposed (`propose_admin`) and accepted
  (`accept_admin`) in every contract.
- A proposal can be cancelled by the current admin (`cancel_admin_proposal`),
  restoring the pre-proposal state with no side effects on the actual admin.
- The role only moves inside `accept_admin`, which requires the successor's
  own authorization — `propose_admin` alone can never brick administration,
  even with a mistyped or unreachable address.
- Tests cover propose, accept, cancel, and unauthorised attempts (wrong
  caller proposing/cancelling, wrong caller accepting, accepting/cancelling
  with nothing pending) in all four contracts.
