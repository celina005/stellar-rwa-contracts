# Issue #3: Prove and document divergent compliance/asset-token admins

## Problem

`scripts/deploy.sh` initializes the compliance contract and the asset-token
contract with the same `$ADMIN_ADDR`. Nothing in the codebase stated whether
that was a hard requirement of the contracts or just a convenience default for
the sample deployment, and there was no test demonstrating that the two
admins can actually be different addresses. A real issuer may want compliance
(KYC/jurisdiction decisions) administered by a dedicated compliance officer,
separate from whoever runs the asset token itself.

## Investigation

Reading both contracts confirmed they already support divergent admins with
no code change needed:

- `contracts/compliance/src/lib.rs` stores its admin under its own
  `DataKey::Admin` instance-storage key, set once in `initialize` and checked
  only by that contract's own `require_admin`.
- `contracts/asset-token/src/lib.rs` stores its admin as the `admin` field of
  its own `AssetMetadata`, set once in `initialize` and checked only by that
  contract's own `require_admin`.
- The only interaction between the two contracts is the asset token calling
  the compliance contract's read-only `is_allowed(address)` on transfers and
  mints. Nothing reads, stores, or compares the other contract's admin.

So the two admins were already fully independent — the gap was that this
fact was unproven (no test) and undocumented (nothing said the shared address
in `deploy.sh` was a choice rather than a constraint).

## Change

No behavioral code change was needed or made, since the contracts already
support this. The changes are documentation plus a proof test:

- `contracts/compliance/src/lib.rs` — added a module-doc section ("Admin is
  independent of the asset-token admin") stating explicitly that this
  contract's admin is self-contained state, unrelated to any asset-token
  admin, and that `scripts/deploy.sh` reusing one address is a convenience
  default, not a requirement.
- `contracts/asset-token/src/lib.rs` — added the mirroring module-doc section,
  noting that `AssetMetadata.admin` and the linked compliance contract's admin
  are tracked in separate storage and never compared; only
  `compliance_contract`'s `is_allowed` result is ever consulted.
- `scripts/deploy.sh` — added a comment above the network/identity setup
  explaining that initializing both contracts with `$ADMIN_ADDR` is a
  single-operator demo convenience, and showing the one-line change
  (`initialize --admin "$COMPLIANCE_OFFICER_ADDR"`) an operator would make to
  split the roles.
- `docs/compliance.md` — added an "Admin independence from the asset-token
  admin (issue #3)" section spelling out the same point for readers of the
  contract docs, with the two-line snippet showing distinct admins, and a
  pointer to the new test.
- `contracts/asset-token/src/test.rs` — added
  `test_compliance_admin_diverges_from_asset_admin`, a new integration-style
  test (this file already cross-registers the compliance contract for
  existing tests, so it's the natural home for a test spanning both
  contracts). It:
  1. Generates two distinct addresses, `asset_admin` and
     `compliance_officer`, and asserts they differ.
  2. Initializes the compliance contract with `compliance_officer` and
     initializes the asset token with `asset_admin`, pointed at that
     compliance contract.
  3. Asserts `token.get_metadata().admin` and `compliance.get_admin()` are
     the two distinct addresses.
  4. Has `compliance_officer` alone drive KYC: approve, transfer, suspend —
     proving compliance administration works without any involvement from
     `asset_admin`, and that a suspended holder's transfer is correctly
     rejected (`SenderNotCompliant`).
  5. Has `asset_admin` alone mint new tokens — proving asset-token
     administration keeps working independent of the compliance officer.
  6. Proves the negative direction too: `asset_admin` calling
     `compliance.try_add_to_allowlist` is rejected
     (`compliance::Error::Unauthorized`), and `compliance_officer` calling
     `token.try_mint` is rejected (`asset_token::Error::Unauthorized`) — so
     neither admin has any implicit authority over the other's contract.

## Why this satisfies the issue

- Divergent admins are now proven to work by a test
  (`test_compliance_admin_diverges_from_asset_admin`), which fails if a
  future change accidentally couples the two admins together.
- The deployment script's use of one address for both is now documented, in
  three places (`scripts/deploy.sh`, `docs/compliance.md`, and the module
  docs of both contracts), as a default for the sample deployment — not a
  requirement — with the exact CLI change needed to use separate addresses.
