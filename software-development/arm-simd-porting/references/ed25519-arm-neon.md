# ARM NEON ed25519 / Curve25519 field arithmetic

Porting Firedancer's ed25519 verify/sign from the x86 AVX-512 backend to ARM64
NEON. Field arithmetic (`fd_f25519`) is the hot path and the port target.

## Dispatch: where the ARM fallback lives

`src/ballet/ed25519/fd_f25519.h` picks the field backend:

    #if FD_HAS_AVX512  →  avx512/  (radix-43x6, 6 limbs)
    #else              →  ref/     (fiat-crypto portable 64-bit, 5×51-bit)

On aarch64 `FD_HAS_AVX512` is false, so verify runs the **fiat-crypto scalar
64-bit** path. There is no NEON/SVE backend in-tree — that is the thing to add.

## Baseline (measured on Kunpeng-class aarch64)

Standalone harness linking `libfd_ballet.a` (version-stamped dir — see the
"version-stamped build dirs" pitfall in SKILL.md):

- verify: **42.0 µs/op** → 23,818 verifies/s
- sign:   **15.7 µs/op** → 63,769 signs/s

## The right representation: radix-2^25.5 (10 limbs)

OpenSSL / BoringSSL / libsodium ARM64 curve25519 all use radix-2^25.5: ten limbs,
alternating 26/25 bits, with limb bit-offsets (weights):

    0, 26, 51, 77, 102, 128, 153, 179, 204, 230   (total 255)

even-index limbs are 26-bit, odd-index 25-bit. Field element =
`Σ limb_i · 2^offset_i`. Reduction mod 2^255−19 folds bits ≥255 through
`2^255 ≡ 19` (the "19-fold").

The multiply is a 10×10 schoolbook accumulated with NEON `umlal` (32×32→64 MAC,
4 lanes per 128-bit vector). Expected **2–3×** over the fiat scalar path,
consistent with upstream ARM64 designs. `umlal` schoolbook is the easy part;
the reduction and the frombytes/tobytes bit layout are the hard part.

## Hard-won lesson: do NOT hand-transcribe ref10 field ops from memory

Hand-transcribing the public-domain ref10 `fe_mul` / `fe_frombytes` /
`fe_tobytes` from memory produced two silent bugs:

1. a **1-ulp carry/rounding bug** — the frombytes→tobytes round-trip differed
   by exactly 1 bit in ~0.05% of random inputs. "Almost correct" == completely
   broken for crypto: a tiny fraction of signatures would fail verification, and
   in consensus that is a silent correctness hole, not a perf bug.
2. **coefficient transcription errors** in `fe_mul` (the 19-fold / 2×-factor
   terms for the odd-index limbs) — the output was completely wrong, not just
   1 ulp.

Correctness protocol (the only way to be sure): verify EVERY kernel
(mul/sqr/add/sub/frombytes/tobytes) **byte-for-byte** against the fiat-crypto
reference on ~100k random inputs, plus RFC 8032 test vectors. The field-op
oracle is `fiat_25519_carry_mul` / `fiat_25519_carry_square` /
`fiat_25519_from_bytes` / `fiat_25519_to_bytes` in
`src/third_party/fiat-crypto/curve25519_64.c`. Those functions are
`static FIAT_25519_FIAT_INLINE` — `#include` the `.c` file into the harness
rather than linking it (external linkage will fail).

Best path: start from a **verified** public-domain ref10 source (libsodium
`crypto_scalarmult/curve25519/ref10/` or SUPERCOP) and port it exactly — do not
reconstruct it from memory. If offline (no way to fetch), localize transcription
errors with simple identities before the random sweep:
`mul(x,1)==x`, `mul(x,0)==0`, `mul(2,3)==6`.

## Isolation technique that worked

To tell whether a mismatch is in the field op or the byte I/O, test the
**frombytes→tobytes round-trip alone** against fiat's `from_bytes`→`to_bytes`
canonicalization first. Result taxonomy:

- round-trip clean, mul/sqr differs  → bug is in the field op (coefficients).
- round-trip off by exactly 1 bit in rare cases → carry/rounding bias is wrong
  (the `+ (1<<24)` / `+ (1<<25)` biases in the carry chain).

This round-trip check isolated the 1-ulp bug in minutes; it is the cheapest
first probe for any new field representation.
