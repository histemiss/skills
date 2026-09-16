---
name: arm-simd-porting
description: "Use when porting x86 SIMD/crypto code to ARM (aarch64)."
version: 1.0.0
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [arm, aarch64, simd, neon, sve, crypto, intrinsics, porting, cross-arch]
---

# ARM SIMD / Crypto Porting

Techniques for porting x86 SIMD/crypto intrinsics (AVX/SSE/SHA-NI/AES-NI/PCLMUL/GFNI/ADX) to ARM, and for debugging ARM crypto-extension code. Distilled from porting Firedancer (Solana validator) to a HiSilicon Kunpeng-class aarch64 host.

## The #1 gotcha: gcc `-mtune=native` drops crypto features

On some ARM CPUs (observed on HiSilicon Kunpeng-class parts), `-march=native` is fine, but adding **`-mtune=native`** drops `+sha2` / `+sha512` / `+aes` / `+pmull` from the *effective codegen target* — while the **preprocessor still defines** `__ARM_FEATURE_SHA2` / `__ARM_FEATURE_AES` / `__ARM_FEATURE_CRYPTO`. Result: `arm_neon.h` `always_inline` intrinsics fail with **"target specific option mismatch"** even though the feature macros say it's available.

**Fix**: put a function-level target attribute on the crypto function (standard multi-target crypto pattern):

```c
__attribute__((target("+sha2")))    /* vsha256*  → FEAT_SHA2     */
__attribute__((target("+sha3")))    /* vsha512*  → FEAT_SHA3      */
__attribute__((target("+crypto")))  /* aese/aesmc + pmull (aes+sha2 umbrella) */
```

The hardware genuinely has the feature (check `lscpu` flags / `__ARM_FEATURE_*`); this is purely a gcc backend quirk. Confirm the bug first:

```bash
printf '\n' | gcc -march=native -E -dM - | grep -iE 'SHA2|AES|CRYPTO|SVE'   # macros say yes
gcc -march=native -Q --help=target | grep march=                              # target may say no sha2
```

## x86 → ARM intrinsic mapping (quick reference)

| x86 | ARM equivalent | Extension / notes |
|---|---|---|
| SHA-NI `_mm_sha256*` | `vsha256h/h2/su0/su1` | FEAT_SHA2 (`+sha2`) |
| SHA-512 (AVX2 asm) | `vsha512h/h2/su0/su1` | FEAT_SHA3/SHA512 (`+sha3`) |
| AES-NI `aesenc`/`aesenclast` | `aese`/`aesmc` (`aesd`/`aesimc` for decrypt) | FEAT_AES (`+aes`/`+crypto`) |
| PCLMUL `pclmulqdq` | `pmull`/`pmull2` (`vmull_p64`/`vmull_high_p64`) | FEAT_PMULL, **NEON only — there is no SVE PMULL** |
| GFNI `_mm_gf2p8*` | `pmull` (GF(2^8) carry-less) | |
| AVX2 / AVX-512 (bitset/shuffle/lookup) | NEON (128-bit, half width) or SVE1 (variable width) | SVE2 ≠ SVE1; check `__ARM_FEATURE_SVE2` before using SVE2 intrinsics |
| ADX `_mulx_u64`/`_addcarry_u64`/`_subborrow_u64` | `umulh`/`mul` pair (or `__int128`) | |
| BMI `_pdep_u64`/`_pext_u64` | **no direct NEON equiv** (SVE2 `BDEP`/`BEXT` only; many ARM hosts lack SVE2) | fall back to scalar bit loops |

## ARM AES: semantics that are easy to get wrong

`AESE` applies **AddRoundKey FIRST**, then SubBytes, then ShiftRows:

```
AESE(x, k)  = ShiftRows(SubBytes(x XOR k))     // key XOR'd in BEFORE SubBytes
AESMC(x)    = MixColumns(x)
```

Byte order is **natural** — no `rev32`/`rev64`/byte-swap needed (verified against the FIPS-197 test vector). AES-128 round structure (10 rounds):

```c
uint8x16_t s = vld1q_u8(pt);
s = vaeseq_u8(s, rk[0]);            // round 0 key + SubBytes + ShiftRows
for (int r = 1; r < 10; r++) {
  s = vaesmcq_u8(s);                // MixColumns
  s = vaeseq_u8(s, rk[r]);
}
s = veorq_u8(s, rk[10]);            // final round: plain XOR, NOT aese
```

Count: **10 AESE + 9 AESMC + 1 final XOR**. Getting the initial-XOR / extra-MixColumns / final-AESE wrong is the classic bug — produces garbage that looks like a byte-order issue but isn't. See `references/arm-aes-gcm-pmull.md` for the full derivation.

## ARM PMULL GHASH (GCM)

Carry-less multiply for the GF(2^128) GCM field. Natural bit order (bit i = coeff x^i, identity at bit 0), reduction mod `x^128+x^7+x^2+x+1`:

```
P = carryless(X, H)   // 256-bit, via 4× pmull (schoolbook: lo*lo, lo*hi, hi*lo, hi*hi)
Z = P_lo ^ P_hi ^ (P_hi << 1) ^ (P_hi << 2) ^ (P_hi << 7)   // x^1,x^2,x^7
// iterate the fold until Z < 2^128 (2-3 passes; the <<7 overflows)
```

This formula is verified correct (5000/5000 random cases against a bit-by-bit field multiply). **Gotcha (RESOLVED)**: existing GCM code that uses the *reflected* convention (right-shift = multiply-by-x, reduction constant `0xe1<<120`) is a **different GF(2^128) basis** (generator = x^(-1)), *not* a simple full-128-bit bit reversal — but the bridge IS a **per-byte bit reversal**: `spec_mul(a,b) = bitrev8(natural_mul(bitrev8(a), bitrev8(b)))` where `bitrev8(x) = bswap64(rbit64(x))` (NEON `rbit v.16b`). Full bit reversal does NOT bridge them; per-byte bitrev8 does. See `references/arm-aes-gcm-pmull.md`.

## TSO → weak-memory-order (ARM) fixes

x86 lock-free code often relies on **TSO** with only compiler fences (`__asm volatile("" ::: "memory")` + `volatile`). On ARM these are NOT enough — hardware reorders. When a field is a publish flag for adjacent data:

```c
// producer (publish):
__atomic_store_n(p, seq, __ATOMIC_RELEASE);   // → stlr
// consumer (observe):
seq = __atomic_load_n(p, __ATOMIC_ACQUIRE);   // → ldar
```

These compile to single `stlr`/`ldar` on ARM (free on x86 — plain `mov`). Verify the fix with `objdump -d` and grep for `ldar`/`stlr`. Full barriers: `dmb ish` (full), `dmb ishld` (load), `dmb ishst` (store); spin-pause is `yield`.

## Feature detection & verification workflow

- Detect features at build time via the compiler macros (`-march=native -E -dM -`), not `/proc/cpuinfo` alone.
- **Always nail a crypto primitive standalone first** against a known test vector (FIPS-197 for AES, NIST GCM vectors) before integrating into a big codebase — byte-order and round-structure bugs are cheap to fix in isolation.
- Cross-check a SIMD port against the *scalar/reference* implementation with N random inputs (link the reference `.a` and compare); a bit-by-bit reference mirroring the original algorithm is the ground truth for field arithmetic.
- **Byte-order / basis mismatch? Brute-force the transform instead of hand-deriving.** When a port computes the right field op but bytes don't match the reference, the mismatch is usually a bit/byte permutation or a basis change (e.g. GCM natural vs reflected). Write a tiny standalone harness that tests candidate transforms T ∈ {identity, bswap64, bswap128, rbit64, rbit128, swap-halves, and their combos} and reports which satisfies `T(ref(a,b)) == impl(T(a), T(b))`. For GCM the answer was `bitrev8` (per-byte reversal) — full 128-bit reversal and every byte-swap FAILED. This found the bridge in minutes after hours of hand-derivation dead ends.
- Confirm the replacement actually emitted the ARM instructions: `objdump -d <obj> | grep -c 'aese\|aesmc'` etc.
- **Firedancer build dirs are version-stamped**: artifacts land in `build/native/gcc/<cc-version>/lib/...`, NOT the flat `build/native/gcc/lib/` (a stale older-build layout). Linking the flat `libfd_ballet.a` against freshly-edited source silently produces phantom failures (wrong ciphertext, zero `H`, every stress iteration failing) even though the fresh object is correct. Always link the version-stamped lib and confirm `objdump` on the freshly built `.o` shows your new instructions.

## Constant-time field arithmetic (ed25519 / curve25519)

For big-number field arithmetic (ed25519 verify/sign), the x86 AVX-512 backend
is usually guarded by `#if FD_HAS_AVX512 → avx512; else → ref (fiat 64-bit)`,
so ARM silently falls back to fiat-crypto scalar (e.g. 42 µs/verify). The ARM
speedup path is radix-2^25.5 (10 limbs) + NEON `umlal` schoolbook — but the
**reduction and frombytes/tobytes bit layout are the hard part**. Never
hand-transcribe the ref10 field ops from memory: that reliably produces silent
1-ulp carry bugs and coefficient errors ("almost correct" == broken for crypto).
Start from a verified public-domain ref10 source and verify every kernel
byte-for-byte against fiat on 100k random inputs. See
`references/ed25519-arm-neon.md` for the dispatch map, baseline numbers, limb
weights, and the round-trip isolation technique.

## References

- `references/arm-aes-gcm-pmull.md` — full AESE-semantics derivation, PMULL fold-reduction details, and the natural-vs-reflected GCM byte-order notes.
- `references/ed25519-arm-neon.md` — ed25519/curve25519 field-arithmetic port: AVX-512→ref dispatch, fiat baseline, radix-2^25.5 + umlal approach, and the do-not-hand-transcribe lesson.
