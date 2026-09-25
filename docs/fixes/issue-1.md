# Issue #1: Optional guardian role for pause-only emergency control

## Problem

`AssetTokenContract::pause` required `admin.require_auth()` via
`require_admin`, so only the full admin key could ever freeze transfers and
mints in an emergency. Since the admin key also controls minting, valuation
updates, and compliance configuration, delegating "hit the emergency brake"
authority meant handing over the whole keyring — not something teams want to
do for an on-call responder or a multisig-lite hot key.

## Change

`contracts/asset-token/src/lib.rs`:

- Added an optional `guardian: Option<Address>` field to `AssetMetadata`.
  It's set to `None` in `initialize`, so every token starts with no guardian
  — the role is absent by default, matching the issue's requirement.
- Added `set_guardian(env, admin, guardian: Option<Address>)`, admin-only
  (via the existing `require_admin` helper), to assign or clear the
  guardian after the fact. Passing `None` removes the role.
- Reworked `pause` to take a generic `caller: Address` instead of `admin`.
  It authenticates the caller, then allows the call through if the caller
  is either the admin or the currently-set guardian; anyone else gets
  `Error::Unauthorized` (`#3`), the same error already used for other
  admin-gated calls.
- Left `unpause`, `mint`, and `mint_batch` untouched — they still go
  through `require_admin`, which only accepts the admin address. The
  guardian has no path to any of them.
- `get_metadata` already returns the full `AssetMetadata` struct, so the
  new `guardian` field is automatically visible off-chain without adding a
  separate getter.

This keeps the emergency-pause path delegable while every other privileged
action (unpause, mint, mint_batch, update_valuation, set_compliance) stays
admin-only, satisfying "permitted to pause but not unpause or mint."

## Tests

Added to `contracts/asset-token/src/test.rs`:

- `test_guardian_absent_by_default` — a freshly initialized token has
  `guardian == None`.
- `test_pause_by_stranger_reverts` — an address that is neither admin nor
  guardian cannot pause.
- `test_guardian_can_pause` — after `set_guardian`, the guardian can pause
  and the metadata reflects `paused == true`.
- `test_guardian_cannot_unpause` — a guardian calling `unpause` panics with
  `Error::Unauthorized`.
- `test_guardian_cannot_mint` — a guardian calling `mint` panics with
  `Error::Unauthorized`.
- `test_set_guardian_by_non_admin_reverts` — only the admin may assign a
  guardian.
- `test_cleared_guardian_loses_pause_rights` — calling `set_guardian` with
  `None` revokes the former guardian's ability to pause.
- `test_admin_still_pauses_with_guardian_set` — adding a guardian doesn't
  take pause rights away from the admin.

Together these cover the issue's "done when" checklist: guardian can pause,
guardian cannot unpause or mint, the role is optional/absent by default,
and each permission boundary has a dedicated test.
