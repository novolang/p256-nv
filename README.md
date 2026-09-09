# p256-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The NIST P-256 curve — secp256r1, prime256v1 — as much of it as
Bluetooth LE Secure Connections needs: a key pair from random bytes the
caller supplies, ECDH down to the shared X coordinate, and the
public-key validation that stands between a pairing peripheral and the
invalid-curve attack.

It is deliberately not a general elliptic-curve library.  There is no
ECDSA here, no other curve, and no SEC1 point compression.  A package
that has one job and no dependencies is one a firmware author can read
in an afternoon, and BLE pairing is that job.

## Adding it, and checking it

```bash
novo pkg add p256-nv        # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test tests/p256_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constant fails with `not implemented:
p256.<fn>`, which is what an interface package looks like from the
outside.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use p256

// The 32 bytes come from the host — an nRF52 RNG peripheral,
// /dev/urandom, a DRBG.  This package never asks for them itself.
fn pair(entropy: [u8], peer_x: [u8], peer_y: [u8]) -> ?[u8]
    match p256.key_pair_from_random(entropy)
        Err(_) => None
        Ok(me) =>
            match p256.public_key_from_bytes(peer_x, peer_y)
                Err(_)   => None
                Ok(peer) =>
                    match p256.ecdh(me.secret, peer)
                        Err(_)  => None
                        Ok(dh)  => Some(dh)
```

`dh` is the 32-byte DHKey that Core Vol 3 Part H § 2.3.5.6.1 hands to
f5.  f5 itself lives in [smp-nv](../smp-nv), for the reason below.

## The layer, and why

`core`.  Every function is arithmetic over bytes the caller already
holds: nothing is read, nothing is written, no clock is consulted, and —
the one that decides the layer — **no randomness is drawn**.  A key pair
needs entropy, and entropy is a `[rand]` effect that belongs to the
host; so `key_pair_from_random` takes the bytes as an argument and this
package's whole surface stays `[]`.  That is what lets the same code run
on a peripheral's pairing path with no operating system under it.

`tests/embedded_probe.nv` is the claim in a form that either builds or
does not.  **It does not build today.**  The `Error` trait is not in the
prelude at `@tier(embedded)`, so `Result<T, KeyError>` cannot be spelled
for a device at all — and neither can `Result<T, Str>`, which the
compiler's own hint recommends.  That is an open toolchain defect
(`result-is-unusable-at-tier-embedded-no-error-trait`), not a property
of this package, and the signatures below keep the `Result` rather than
retreating to `?T`: a package whose refusals carry no reason would be
routing around the defect instead of recording it.

## The load-bearing interface

`KeyError`, and specifically its `NotOnCurve` variant.

```novo
pub enum KeyError
    BadLength(got: Int, want: Int)
    ScalarOutOfRange
    NotOnCurve
    PointAtInfinity
```

Everything else here follows from one decision: **`ecdh` validates the
peer's key itself and refuses, rather than offering validation as a
separate call a caller may skip.**  Scalar multiplication against a
point that is not on the curve leaks the local private key over a
handful of pairings — CVE-2018-5383, which is the Bluetooth pairing
vulnerability that this check is the answer to — and an API that lets a
caller forget is an API that will be forgotten.

So the split is: `public_key_from_bytes` checks the widths and nothing
else, because a PDU parser wants to know it has 64 bytes before it
spends a field inversion; `validate` answers the curve question on
demand for a caller that wants it early; and `ecdh` asks it again
regardless.  Asking twice costs one curve-equation evaluation and is
worth it.

## Where f4, f5, f6 and g2 live, and why not here

In [smp-nv](../smp-nv), not here and not in
[crypto-nv](../crypto-nv).  Three shelves, one rule — **a function goes
where its specification is**:

- **crypto-nv** takes the primitives that any protocol could call and
  that a standards body outside Bluetooth defines: AES-128 (FIPS 197),
  AES-CMAC (RFC 4493), SHA-256 (FIPS 180-4).  Nothing about them is
  Bluetooth's.
- **p256-nv** takes the curve (FIPS 186-4 D.1.2.3).  Also not
  Bluetooth's: the same arithmetic serves TLS and JWT ES256, which is
  why the interfaces grid has `jwt-nv` depending on this package too.
- **smp-nv** takes f4, f5, f6, g2, h6, h7, c1, s1 and ah.  Every one of
  them is defined in Core Vol 3 Part H § 2.2 and used by nothing outside
  Bluetooth pairing.  They are AES-CMAC compositions with
  Bluetooth-specific key IDs, salts and byte orders; putting them in a
  general crypto package would be filing the Bluetooth spec under
  cryptography.

The dividing question is not "is it cryptography" — all of it is — but
"would a reader who has never opened the Bluetooth Core Specification
have any use for this function".  For AES-CMAC, yes.  For `f5`, no.

## The reference implementation

`crypto-ble-ecc`'s `prims/p256.nv`, ported into this package's shape,
with RustCrypto's `p256` crate as the second opinion and FIPS 186-4
D.1.2.3 / RFC 6090 as the source of the test vectors.

**Read this before implementing.**  The reference is a *field
arithmetic* layer and nothing above it: `fp_add`, `fp_sub`, `fp_mul`,
`fp_sqr`, `fp_inv` and Solinas reduction, verified against the curve
equation and the λ-derivation identity.  It has **no point operations at
all** — `jp_double`, `jp_add`, `jp_scalar_mul` and `jp_to_affine` are a
TODO block in that file, and its own note records that an earlier
attempt produced 2·G coordinates that did not match RFC 6090 while every
field-arithmetic check passed.  So every function in this interface is
above the line the reference reaches, and the implementation lane
inherits a known-open bug rather than a working port.  Budget for it.

## Status

| item | implemented |
| --- | --- |
| `p256.SCALAR_BYTES`, `.COORDINATE_BYTES`, `.PUBLIC_KEY_BYTES`, `.DHKEY_BYTES` | yes — they are constants |
| `p256.secret_key_from_bytes` | no |
| `p256.public_key_from_bytes` | no |
| `p256.public_key_bytes` | no |
| `p256.secret_key_bytes` | no |
| `p256.key_pair_from_secret` | no |
| `p256.key_pair_from_random` | no |
| `p256.validate` | no |
| `p256.ecdh` | no |
| `p256.KeyError.message` | no |
