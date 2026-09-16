---
name: arm64-porting
description: Use when porting x86 code (SIMD, crypto, TSO) to ARM64.
---

# ARM64 Porting (x86 SIMD / crypto / memory model)

Migrating performance-critical x86 code to ARM64. Covers SIMD intrinsics,
crypto extensions, and the TSO memory-model assumptions that silently break on
weakly-ordered ARM. Prefer building on the real aarch64 target and running its
unit tests rather than cross-compiling blind.

## Workflow

1. **Inventory x86 SIMD usage** — grep the tree for intrinsic families:
   - AVX-512: `_mm512_`, `__m512`, `__AVX512`
   - AVX2: `_mm256_`, `__m256`
   - SSE: `_mm_`, `__m128`
   - SHA-NI: `_mm_sha256`, `_mm_sha1`
   - AES-NI/PCLMUL: `_mm_aes`, `_mm_clmulepi64`
   - GFNI: `_mm_gf2p8`, `_mm512_gf2p8`
   - BMI/ADX: `_mulx`, `_addcarry`, `_subborrow`, `_pdep`, `_pext`
2. **Check what actually dispatches on ARM** — read the build `Local.mk` /
   `#if FD_HAS_*` guards. Many codebases already carry ARM paths that either
   fail to compile on the specific CPU or silently fall back to scalar; those
   are the cheapest wins (fix, don't rewrite).
3. **Map to ARM equivalents** — see `references/arm-crypto-intrinsics.md`.
4. **Build on the real ARM target, then run the unit tests** — a clean build is
   not correctness. Validate crypto against test vectors (FIPS-197 AES, NIST GCM).

## Pitfalls (each cost real debugging time)

- **`-mtune=native` drops crypto features on some ARM CPUs** (observed on
  HiSilicon Kunpeng-class parts with gcc 13): `-march=native -mtune=native`
  together silently drop `+sha2`/`+sha512`/`+aes`/`+pmull` from codegen — while
  `-march=native` alone keeps them and `-E -dM` still shows the macros. Symptom:
  `error: inlining failed in call to 'always_inline' 'vsha256h2q_u32': target
  specific option mismatch`. Fix: `__attribute__((target("+sha2")))` / `("+sha3")`
  / `("+crypto")` on the affected function. Verify with a standalone compile.
- **ARM AESE does AddRoundKey FIRST**: `AESE(x,k) = ShiftRows(SubBytes(x XOR k))`,
  not SubBytes+ShiftRows+AddRoundKey. Getting this wrong yields ciphertext that
  looks byte-order-broken (not a simple permutation). See the reference for the
  exact round structure.
- **ARM AES byte order is natural (FIPS-197 column-major), not rev32** — load
  keys/data with plain `vld1q_u8`. Use a scalar FIPS-197 key schedule; do NOT
  reuse a `u32[]`-based key expansion (little-endian storage) without rev32.
- **TSO assumptions** — x86's total-store-order lets `volatile` + a compiler
  fence publish data; ARM reorders, so flag publish/read needs release/acquire.
  See `references/tso-memory-model.md`.
- **aarch64 syscall table differs from x86-64** — some legacy syscalls are
  absent on arm64 (e.g. `__NR_epoll_wait`; arm64 folded `epoll_wait` into
  `__NR_epoll_pwait`).  Any code that references a raw `__NR_*` number (e.g.
  reading `/proc/<pid>/syscall` to detect a blocked waiter, or direct
  `syscall(__NR_...)`) fails to compile on arm64.  Guard legacy `__NR_*`
  references with `#ifdef` — the fix is build-only, not a runtime behavior
  change.
- **Firedancer ed25519 on ARM uses fiat-crypto 64-bit scalar (not SIMD)** —
  `fd_f25519.h` dispatches `#if FD_HAS_AVX512 → avx512 backend, #else →
  ref (fiat curve25519_64.c, 5x51-bit limbs)`.  On ARM the 5x51 scalar mul
  is ~15 ns/op and ed25519 verify ~42 us/op.  The speedup path is a
  radix-2^25.5 (10x26-bit) NEON backend using `vmlal` (32x32->64 MAC) —
  but note the SCALAR radix-2^25.5 (ref10) is ~2x SLOWER than fiat on ARM
  (~34 ns/op vs 15 ns/op); the win comes only from NEON vectorization of
  the 10 dot-products.  Verified reference: libsodium
  `crypto_core/ed25519/ref10/fe_25_5/{fe.h}` + `include/sodium/private/
  ed25519_ref10_fe_25_5.h` (frombytes/tobytes/reduce + mul/sq/add/sub).
  Gotcha: hand-written `load_4` must use `uint32_t` (an `int32_t` return
  overflows on `in[3]<<24`, silently corrupting the high limb — this cost
  a long 1-ulp debugging session).
  MEASURED (Kunpeng aarch64): fiat 5x51 scalar mul ~15.6 ns/op, scalar
  ref10 ~33.6 ns/op, a naive 2-lane NEON `smlal` ref10 mul ~20.7 ns/op.
  So on this CPU the existing fiat 64-bit path is already near-optimal;
  beating it needs the Bernstein-Schwabe-style hand-scheduled NEON (4-lane
  smlal/smlal2, no lane-sum overhead), a deep effort with uncertain payoff —
  do NOT assume NEON auto-wins for 255-bit field arithmetic.

## Reference files

- `references/arm-crypto-intrinsics.md` — ARM AES/SHA/PMULL crypto-extension
  semantics, byte order, round structure, target attributes, verified vectors.
- `references/tso-memory-model.md` — x86 TSO → ARM acquire/release migration
  pattern + how to recognize and verify it.
- `references/gcm-ghash-pmull.md` — verified natural-order PMULL GHASH reduction
  formula, the natural-vs-reflected convention trap, and the RESOLVED bridge
  (per-byte `bitrev8`) to OpenSSL's reflected GCM.
- `references/firedancer-bench-arm.md` — running `firedancer-dev bench` (validator
  TPS benchmark) on aarch64: config schema gotchas (`[runtime]`, `max_page_size`,
  `provider="socket"`), hugetlbfs setup, the NUMA workspace-placement issue
  (`blocklist_cores`), the two boot-time fixes (`initialize_snapshot_fds` in
  `bench_cmd_fn`, `[tiles.event] url = ""`), and the tile-count scaling ceiling
  (dedup 64-input cap, bencho tempo `bad lazy`) that caps verify at 63 and
  practical throughput at ~80k TPS.
