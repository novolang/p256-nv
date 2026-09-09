# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.1 — 2026-09-09

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `PublicKey`, `SecretKey`, `KeyPair` and the `KeyError` enum that says
  why a key was refused.
- `secret_key_from_bytes`, `public_key_from_bytes`, `public_key_bytes`,
  `secret_key_bytes`, `key_pair_from_secret`, `key_pair_from_random`,
  `validate` and `ecdh` — the P-256 surface LE Secure Connections needs
  and nothing else.
- Every function is `[]`. Entropy arrives as an argument to
  `key_pair_from_random`, which is what keeps the package `core` and
  what lets the same code run on a pairing peripheral.

Two things a reader should know before depending on it. The
implementation it will be ported from — `crypto-ble-ecc`'s
`prims/p256.nv` — is field arithmetic only and has no point operations
at all, so `0.1.0` is a bigger step than a port. And
`tests/embedded_probe.nv` does not build today: the `Error` trait is
absent from the prelude at `@tier(embedded)`, so `Result<T, KeyError>`
cannot be spelled for a device. That is an open toolchain defect
(`result-is-unusable-at-tier-embedded-no-error-trait`) and the
signatures keep the `Result` rather than retreating around it.
