# p256-nv

P-256 is an elliptic curve standardised by NIST in
[FIPS 186-4](https://csrc.nist.gov/pubs/fips/186-4/final) appendix D.1.2.3, and
known elsewhere as secp256r1 and prime256v1. This package implements as much of
it as Bluetooth LE Secure Connections needs: a key pair from random bytes the
caller supplies, an Elliptic Curve Diffie-Hellman exchange down to the shared X
coordinate, and the public-key validation that stands between a pairing
peripheral and the invalid-curve attack.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What P-256 is

An elliptic curve over a prime field is a set of points satisfying an equation,
here `y² = x³ − 3x + b`, together with an addition rule. Multiplying a point by
an integer means adding it to itself that many times, and recovering the integer
from the result is the hard problem the security rests on.

A P-256 *secret key* is an integer below the curve order `n`. A *public key* is
the curve's fixed base point multiplied by that integer, written as its two
32-byte coordinates. An ECDH exchange multiplies the peer's public key by your
own secret key. Both sides arrive at the same point, and its X coordinate is the
shared secret.

| Value | Size |
| --- | --- |
| Secret key (scalar) | 32 bytes |
| Coordinate | 32 bytes |
| Public key (X and Y) | 64 bytes |
| Shared secret (DHKey) | 32 bytes |

Coordinates and scalars are big-endian.

## Install

```
novo pkg add p256-nv
```

## Example

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

`dh` is the 32-byte DHKey that the Bluetooth Core Specification, Volume 3
Part H section 2.3.5.6.1, hands to the function `f5`. `f5` itself lives in
smp-nv, for the reason under "Related packages".

Build and test with:

```
novo pkg build              # type- and effect-check the package
novo test tests/p256_tests.nv
```

Today `novo test` fails on purpose: every test reaches a
`not implemented: p256.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

One module.

| Module | Contents |
| --- | --- |
| `p256` | The whole surface: the four size constants; the key types `PublicKey`, `SecretKey` and `KeyPair`; the constructors and accessors; the standalone curve check `validate`; the exchange `ecdh`; and the error type `KeyError`. |

No function in this package draws randomness, reads a clock, or performs input
or output. The entropy a key pair needs arrives as an argument, which lets the
same code run on a peripheral's pairing path with no operating system under it.

## How to choose an entry point

**A packet parser calls `public_key_from_bytes`.** It checks the widths and
nothing else, so a caller can find out it has 64 bytes before spending a field
inversion.

**A program that stores a peer's key calls `validate`.** It answers the curve
question on demand, so a key can be rejected at registration rather than at
first use.

**Everything else calls `key_pair_from_random` and then `ecdh`.**

## The rules a user needs

1. **`ecdh` validates the peer's key itself and refuses.** Scalar
   multiplication against a point that is not on the curve leaks the local
   private key over a handful of pairings. That is CVE-2018-5383, the Bluetooth
   pairing vulnerability this check answers. Validation is not an optional call
   a caller might skip: `ecdh` performs it whether or not `validate` was called
   first. Asking twice costs one curve-equation evaluation.
2. **A scalar of zero, or at or above the curve order `n`, is refused** with
   `ScalarOutOfRange`. Neither names a point on the curve.
3. **A coordinate pair that does not satisfy `y² = x³ − 3x + b` is refused**
   with `NotOnCurve`.
4. **The point at infinity is refused** with `PointAtInfinity`. It has no affine
   coordinates and is not a legal public key.
5. **A byte list of any other width is refused** with `BadLength(got, want)`.
6. **`secret_key_from_bytes` is the only way to make a `SecretKey`**, so a value
   of that type is always a usable scalar. The same holds for the other
   constructors and their types.

Every refusal is about the arguments. None of them means the arithmetic went
wrong.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device with no
heap allocator, and `tests/embedded_probe.nv` is that claim as a program. Every
function here is arithmetic over bytes the caller already holds, so the claim is
the right one to make.

**The probe does not build today.** The `Error` trait is not available in code
compiled for such a device, so `Result<T, KeyError>` cannot be written there at
all, and neither can `Result<T, Str>`, which the compiler's own hint recommends.
That is an open toolchain defect, filed against the compiler, and not a property
of this package. The signatures keep the `Result` rather than retreating to
`?T`. The defect is recorded so that it gets fixed.

## Timing behaviour

No timing claim is made for this interface release. The implementation release
will state which operations are constant-time with respect to the secret scalar.

The shared secret is compared by the caller, not here. A program that checks one
against a stored value needs a constant-time comparison, and the one on the
registry is `digest.ct_eq` in
[crypto-nv](https://novo-lang.org/packages/crypto-nv).

## What is not included

- **ECDSA.** This package is the curve and the exchange. Signing and verifying
  would need a hash, which is why the package has no dependencies.
- **Any other curve.** For Curve25519, see
  [x25519-nv](https://novo-lang.org/packages/x25519-nv) and
  [ed25519-nv](https://novo-lang.org/packages/ed25519-nv).
- **SEC1 point compression.** Public keys here are the two full coordinates,
  which is the form LE Secure Connections puts on the wire.
- **A random number generator.** Draw the bytes yourself, from a hardware
  peripheral, from `/dev/urandom` or from a DRBG, and pass them in.
- **The Bluetooth pairing functions `f4`, `f5`, `f6`, `g2`, `h6`, `h7`, `c1`,
  `s1` and `ah`.** See "Related packages".

## Related packages

Three packages divide this ground, and the rule is that a function goes where
its specification is.

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) holds the primitives any
  protocol could call and that a standards body outside Bluetooth defines:
  AES-128 (FIPS 197), AES-CMAC (RFC 4493), SHA-256 (FIPS 180-4). Nothing about
  them is Bluetooth's.
- **p256-nv** holds the curve (FIPS 186-4 D.1.2.3). Also not Bluetooth's: the
  same arithmetic serves TLS and the JWT algorithm ES256, which is why a JWT
  package depends on this one too.
- **smp-nv** holds `f4`, `f5`, `f6`, `g2`, `h6`, `h7`, `c1`, `s1` and `ah`.
  Every one of them is defined in the Bluetooth Core Specification, Volume 3
  Part H section 2.2, and used by nothing outside Bluetooth pairing. They are
  AES-CMAC compositions with Bluetooth-specific key identifiers, salts and byte
  orders.

The dividing question is not whether a function is cryptography, since all of it
is, but whether a reader who has never opened the Bluetooth Core Specification
would have a use for it. For AES-CMAC, yes. For `f5`, no.

## Test vectors

FIPS 186-4 D.1.2.3 and [RFC 6090](https://www.rfc-editor.org/rfc/rfc6090) are
the sources of the test vectors. RustCrypto's `p256` crate is the second check.

The implementation will be ported from `crypto-ble-ecc`'s `prims/p256.nv`.
**Read this before implementing.** That reference is a *field arithmetic* layer
and nothing above it: `fp_add`, `fp_sub`, `fp_mul`, `fp_sqr`, `fp_inv` and
Solinas reduction, verified against the curve equation and the λ-derivation
identity. It has no point operations at all. `jp_double`, `jp_add`,
`jp_scalar_mul` and `jp_to_affine` are a TODO block in that file, and its own
note records that an earlier attempt produced coordinates for 2·G that did not
match RFC 6090 while every field-arithmetic check passed.

Every function in this interface therefore sits above the line the reference
reaches, and the implementation inherits a known-open defect rather than a
working port. Allow time for it.

## Implementation status

| Item | Implemented |
| --- | --- |
| `p256.SCALAR_BYTES`, `.COORDINATE_BYTES`, `.PUBLIC_KEY_BYTES`, `.DHKEY_BYTES` | yes (they are constants) |
| `p256.PublicKey`, `.SecretKey`, `.KeyPair`, `.KeyError` | declared |
| `p256.secret_key_from_bytes` | no |
| `p256.public_key_from_bytes` | no |
| `p256.public_key_bytes` | no |
| `p256.secret_key_bytes` | no |
| `p256.key_pair_from_secret` | no |
| `p256.key_pair_from_random` | no |
| `p256.validate` | no |
| `p256.ecdh` | no |
| `p256.KeyError.message` | no |
| The microcontroller probe | does not build — the toolchain defect above |

## Licence

Apache-2.0. See `LICENSE`.
