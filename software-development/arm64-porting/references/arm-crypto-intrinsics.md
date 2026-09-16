# ARM Crypto Extension Intrinsics

## AES (FEAT_AES) — `aese`, `aesmc`, `aesd`, `aesimc`

Semantics (verified against the FIPS-197 AES-128 vector, key `000102..0f`,
plaintext `001122..ff` → ciphertext `69c4e0d8...c55a`):

- `AESE(x, k) = ShiftRows(SubBytes(x XOR k))` — **AddRoundKey FIRST**, then
  SubBytes, then ShiftRows. It does NOT MixColumns.
- `AESMC(x) = MixColumns(x)`.
- `AESD` / `AESIMC` are the decrypt / inverse-MixColumns counterparts.

Byte order is NATURAL (FIPS-197 column-major). Load state and round keys with
`vld1q_u8`, store with `vst1q_u8`. No `vrev32q_u8` needed — if results look
"byte-swapped", the round structure is wrong, not the byte order.

AES-Nr encryption (Nr = 10/12/14):

```c
uint8x16_t s = vld1q_u8(in);
s = vaeseq_u8(s, vld1q_u8(rk[0]));
for (uint r = 1; r < nr; r++) {
  s = vaesmcq_u8(s);
  s = vaeseq_u8(s, vld1q_u8(rk[r]));
}
s = veorq_u8(s, vld1q_u8(rk[nr]));   /* final AddRoundKey, NOT aese */
vst1q_u8(out, s);
```

That is **Nr `aese` + (Nr-1) `aesmc` + 1 final `eor`** (11 round keys, indexed
0..Nr). Common bugs: an extra initial `eor` of rk[0], or a trailing `aesmc`
after the last round (giving Nr instead of Nr-1 MixColumns).

Key expansion: use a scalar FIPS-197 schedule producing round keys in NATURAL
byte order (byte 0 = MSB of each word). A `u32[]`-based schedule stores each
word little-endian and needs a `rev32` per word before `aese` will accept it.

## SHA (FEAT_SHA2 / FEAT_SHA3)

- SHA-256: `vsha256hq_u32`, `vsha256h2q_u32`, `vsha256su0q_u32`,
  `vsha256su1q_u32` — gated by `+sha2`.
- SHA-512: `vsha512hq_u64`, `vsha512h2q_u64`, `vsha512su0q_u64`,
  `vsha512su1q_u64` — gated by `+sha3` (NOT `+sha512`; check the
  `#pragma GCC target` guard in arm_neon.h).
- SHA-3 / SM3 / SM4 are separate feature bits.

## PMULL (FEAT_PMULL) — carry-less GF(2^n)

`vmull_p64(a, b)` = 64x64→128 carry-less multiply; `vmull_high_p64` for the
high 64-bit halves. Used to accelerate GCM GHASH (GF(2^128)) and Reed-Solomon
GF(2^8) arithmetic. Gated by `+pmull` (or `+crypto`).

## gcc `-mtune=native` feature-drop bug

On some ARM CPUs (HiSilicon Kunpeng-class), `-march=native -mtune=native`
together drop `+sha2`/`+sha512`/`+aes`/`+pmull` from codegen. The preprocessor
(`-E -dM`) still defines `__ARM_FEATURE_SHA2` etc., so the `#if FD_HAS_*` code
paths compile but the `always_inline` intrinsics then fail with "target specific
option mismatch".

Diagnosis:

```sh
gcc -march=native -Q --help=target | grep march=            # shows dropped features
gcc -march=native -E -dM - | grep -E 'SHA2|SHA3|AES|PMULL'  # macros still present
```

Fix: `__attribute__((target("+sha2")))` / `("+sha3")` / `("+crypto")` on the
function using the intrinsics. Feature-detect with `__ARM_FEATURE_AES` /
`__ARM_FEATURE_SHA2` / `__ARM_FEATURE_SHA512` for the dispatch guard.

## Feature detection flags

When adding a new ARM backend, add a `FD_HAS_ARM_*` flag to the build's
native-config probe (e.g. `native_config.sh` emit_feature via `__ARM_FEATURE_AES`)
AND to any cross/machine configs that hardcode `-mcpu=...+crypto`, so the
backend is compiled consistently in both native and cross builds.
