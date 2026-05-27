# Fork Notes — rustpush (presencesync branch)

This branch (`presencesync`) tracks **`thisiscam/findmy-export-support`** (not
upstream `master`). That branch, maintained by the export-findmy author
(Cambridge Yang), exposes internal APIs needed for headless FindMy key export.

## Upstream tracking

| Remote | Branch | Role |
|--------|--------|------|
| `upstream` | OpenBubbles/rustpush `master` | Original project (iMessage Push) |
| `thisiscam` | thisiscam/rustpush `findmy-export-support` | **Our actual base** — FindMy API exposure |
| `origin` | bexelbie/rustpush `presencesync` | This fork |

We rebase on `thisiscam/findmy-export-support`, not on upstream `master`.

## Our commit stack (on top of `thisiscam/findmy-export-support`)

### 1. `fix: provide explicit RFC 3394 IV for AES key wrap/unwrap`

**Problem:** OpenSSL 0.10.80+ changed default IV handling for the AES key
wrap cipher. Passing `None` as the IV previously used the RFC 3394 default
(`0xA6` × 8), but newer versions require it explicitly. Without this,
key wrap/unwrap silently produces incorrect results and iCloud Keychain
escrow decryption fails.

**Fix:** Pass `Some(&[0xA6u8; 8])` explicitly to both `Crypter::new()`
(wrap) and `decrypt()` (unwrap) in `rfc6637_wrap_key` / `rfc6637_unwrap_key`.

**Upstream PR candidate:** Yes — bug fix, required for anyone using
OpenSSL ≥0.10.80. Affects all callers.
**PR target:** `OpenBubbles/rustpush` `master` (thisiscam gets it on rebase).

### 2. `feat: make sharing circle fields pub for external consumers`

**Problem:** `MemberSharingCircle.owner` and
`SharingCircleSecret.sharing_circle_identifier` are private, but external
tools need them to identify sharing circle ownership and match secrets to
the correct circles.

**Fix:** Make both fields `pub`.

**Upstream PR candidate:** Yes — simple visibility change, consistent with
the other `pub` fields already on these structs. Needed by any tool
implementing the getShare API.
**PR target:** `thisiscam/rustpush` `findmy-export-support` branch.

### 3. `feat: add as_bytes() accessors to WildRootKey and CircleSecretKey`

**Problem:** External tools need raw key bytes to serialize into plist
format (FindMy.py compatibility). The key types wrap `Vec<u8>` but don't
expose it.

**Fix:** Add `pub fn as_bytes(&self) -> &[u8]` to both types.

**Upstream PR candidate:** Yes — trivial accessor, no security implication
(callers already have the key object).
**PR target:** `thisiscam/rustpush` `findmy-export-support` branch.

### 4. `feat: expose shared beacon attribute fetch`

**Problem:** `rustpush` already knows how to fetch and decrypt shared item
`beaconAttributes` via the SearchParty `itemsharing/getShare` endpoint, but
that logic is private to `FindMyClient` and requires the APS/IDS/state client
stack. Headless exporters already have the CloudKit sharing records and
secrets, but cannot reuse the existing name/emoji/serial extraction path.

**Fix:** Add a public `fetch_shared_beacon_details()` helper that accepts the
existing anisette client, token provider, OS config, sharing circle, join
token, and circle shared secret, then returns decrypted `SharedBeaconDetails`
with `BeaconAttributes`.

**Upstream PR candidate:** Yes — focused API exposure that reuses existing
request and decryption behavior without changing existing callers.
**PR target:** `thisiscam/rustpush` `findmy-export-support` branch.

### 5. `chore: update apple-private-apis submodule to presencesync branch`

Points the submodule at our `apple-private-apis` fork which has its own
commit stack (see `apple-private-apis/FORK.md`).

**Upstream PR candidate:** No — submodule pointer is fork-specific.

## Summary

All code changes (commits 1–4) are individually PR-able to
`thisiscam/rustpush` on the `findmy-export-support` branch. They are
small, focused, and don't change behavior for existing callers.
