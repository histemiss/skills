# ARM AES-GCM + PMULL: derivation notes

Hard-won detail from porting Firedancer's AES-GCM to aarch64. Read alongside the main SKILL.md.

## AESE semantics — how it was nailed down

The wrong mental model ("AESE = SubBytes+ShiftRows, then XOR the round key") produced garbage that looked like a byte-order bug. Brute-forcing 16 byte-order combinations (identity/bswap/rbit/rev32 + inverses) all failed — the problem was never byte order.

The correct semantics, confirmed empirically against the FIPS-197 known-answer vector:

```
AESE(x, k)  = ShiftRows(SubBytes(x XOR k))
AESMC(x)    = MixColumns(x)
```

Key insight: **AddRoundKey happens BEFORE SubBytes inside AESE**. So the round structure is:

```
s0 = AESE(pt, k0)                  // XOR k0, SubBytes, ShiftRows
s1 = AESMC(AESE(s0, k1))           // MixColumns, then XOR k1 + SubBytes + ShiftRows
...
s9 = AESMC(AESE(s8, k9))
out = s9 XOR k10                    // final round has NO SubBytes/MixColumns
```

Byte order is **natural** (no rev32/rev64) — the ARM `aese`/`aesmc` operate directly on the AES state bytes as laid out in the FIPS test vector, unlike the x86 AES-NI path in some codebases that byte-swaps words.

Count check: **10 AESE + 9 AESMC + 1 final XOR** (AES-128). AES-192 → 12 rounds, AES-256 → 14 rounds.

Decrypt: `aesd` = InvShiftRows(InvSubBytes(x XOR k)) (again key XOR'd first), `aesimc` = InvMixColumns.

## PMULL GHASH reduction — the fold formula

GCM GHASH multiplies in GF(2^128) with reduction polynomial `x^128 + x^7 + x^2 + x + 1`.

Using **natural bit order** (bit i of a 128-bit word = coefficient of x^i; the field identity is bit 0 = 1), carry-less multiply of two 128-bit values gives 256 bits. Reduce with the fold:

```
Z = P_lo ^ P_hi ^ (P_hi << 1) ^ (P_hi << 2) ^ (P_hi << 7)
```

where `P_lo`/`P_hi` are the low/high 128-bit halves of the 256-bit product. The `<<1`, `<<2`, `<<7` come from the reduction polynomial's low-order terms (x^7+x^2+x+1). Because the `<<7` can produce another bit at position 128+, iterate the fold 2-3 times until `Z < 2^128`.

Verified 5000/5000 random cases against a bit-by-bit field multiply (`natural_mul`: 128 shifts + conditional XOR). The multiply is commutative/associative with identity at bit 0.

The 256-bit product is computed with 4 NEON `pmull`/`pmull2` (schoolbook: `lo×lo`, `lo×hi`, `hi×lo`, `hi×hi`, XOR the cross terms at the 64-bit boundary). There is **no SVE PMULL** — this is NEON-only.

## natural vs reflected — RESOLVED: bridge is per-byte bit reversal (bitrev8)

Many existing GCM implementations (OpenSSL, Firedancer's `fd_aes_gcm_aesni.S` / `fd_gcm_gmult_4bit`) use the **reflected** convention: right-shift = multiply-by-x, reduction constant `0xe1 << 120`, and they `bswap64` each word at init.

The bridge between natural-order PMULL and this reflected/byte-swapped convention is **`bitrev8`** — reverse the bits *within each byte*, no byte reordering, no byte swap:

```
spec_mul(a, b)  =  bitrev8( natural_mul( bitrev8(a), bitrev8(b) ) )
bitrev8(x)       =  bswap64(rbit64(x))     // RBIT = full 64-bit reversal; +bswap leaves per-byte reversal
```

NEON does `bitrev8` directly with `rbit v.16b` (RBIT is a per-byte bit reversal). The scalar form `bswap64(rbit64(x))` works because `rbit64` = full reversal = `bitrev8 ∘ bswap64`, and `bswap64` is its own inverse.

Distinction that cost hours: **full 128-bit bit reversal (rbit64 both halves + swap) does NOT bridge** the two conventions — tried and failed (100000/100000). Only the *per-byte* reversal (bitrev8, no cross-byte bit movement, no byte swap) bridges them. Do not conflate `rbit64` (full) with `rbit`-per-byte.

Verified end-to-end (Firedancer `bench_arm.c` + standalone):
- Canonical NIST GF(2^128) vector: `X=0x0388DACE60B6A392F328C2B971B2FE78`, `Y=0x66E94BD4EF8A2C3B884CFA59CA342B2E` → `X·Y=0x5E2EC746917062882C85B0685353DEB7` (spec/reflected convention: right-shift, R=`0xe1<<120`, X bits consumed left-first).
- 200000 random GHASH cases byte-identical to `fd_gcm_ghash_4bit`.
- 100000 full AES-GCM arm-vs-ref iterations, 0 failures.

Practical pattern: keep the accumulator in bitrev8-natural space across the whole GHASH chain; `bitrev8` each 16-byte block on the way in and the final result on the way out. Store `H` once as `bitrev8(H_nist)` at init — no `bswap64` of `H`, no Htable. The x86 `fd_aes_gcm_aesni.S` takes an equivalent route instead: byte-reflect (bswap128) the data, then reduce with the reciprocal polynomial (gfpoly = `0xc2<<120`, i.e. x^128+x^127+x^126+x^121+1).

## gcc `-mtune=native` feature-drop — details

Reproduced on gcc 13.3, HiSilicon Kunpeng-class aarch64:

- `gcc -O3 -march=native -c file.c` → compiles, crypto intrinsics fine.
- `gcc -O3 -march=native -mtune=native -c file.c` → **"target specific option mismatch"** on `vsha256h2q_u32` etc.
- The `__ARM_FEATURE_SHA2`/`__ARM_FEATURE_AES` macros are STILL defined under `-mtune=native` — so `#ifdef`-guarded code compiles the intrinsic line, but the function's effective target lost `+sha2`/`+aes`.

Fix is the `__attribute__((target("+sha2")))` etc. on the specific functions (see SKILL.md). This is the same multi-target pattern x86 uses for `target("avx2")` etc., applied to ARM feature bits.

## Postscript: the rest of the Firedancer ARM port (same series)

- `firedancer-dev` (incl. the `bench` TPS tool) builds and runs on aarch64 after
  three fixes: `__NR_epoll_wait` is absent on arm64 (guard with `#ifdef`),
  `bench_cmd_fn` was missing `initialize_snapshot_fds()`, and the TLS event
  service needs `[tiles.event] url = ""`.  The validator boots 45-140 tiles,
  ~80K TPS, 30 min stable (NUMA: blocklist node0 on the shared Kunpeng host).
- ed25519: the ARM path is already the fiat-crypto 64-bit scalar (5x51).  Measured
  field mul on Kunpeng aarch64: fiat 15.6 ns/op, scalar ref10 (radix 2^25.5)
  33.6 ns/op, naive 2-lane NEON `smlal` 20.7 ns/op — so **fiat is already
  near-optimal**; a real NEON win needs Bernstein-Schwabe-style hand scheduling
  and is not worth it.  Don't assume NEON auto-wins for 255-bit field arithmetic.
  (Verified ref10 source: libsodium `crypto_core/ed25519/ref10/fe_25_5/*`; the
  `load_4` helper must return `uint32_t`, an `int32_t` return overflows on
  `in[3]<<24`.)
