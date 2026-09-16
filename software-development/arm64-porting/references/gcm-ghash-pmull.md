# GCM GHASH via PMULL

Accelerating AES-GCM's GHASH (GF(2^128) carry-less multiply) with PMULL.
The AES half of GCM is straightforward (see arm-crypto-intrinsics.md); GHASH
is where the bit-order traps live.

## Verified: natural-order PMULL GHASH

In the NATURAL bit convention (bit i = coefficient of x^i, identity `1` at bit
0, "multiply by x" = LEFT shift, reduction x^128 = x^7+x^2+x+1 = 0x87), the
full field multiply is a 4-PMULL carry-less product + a fold reduction. It is a
correct field multiply (commutative/associative/identity, verified against a
bit-by-bit reference over thousands of random inputs):

```
P = vmull(X0,H0) | vmull(X0,H1)<<64 | vmull(X1,H0)<<64 | vmull(X1,H1)<<128  # 256-bit
Z = P_lo ^ P_hi ^ (P_hi << 1) ^ (P_hi << 2) ^ (P_hi << 7)                    # fold
repeat the fold while Z >= 2^128 (converges in ~2 iterations)
```

where P_hi = high 128 bits, P_lo = low 128 bits. Implement the fold on a 4x64
word accumulator: XOR h into a[0], a[1]; then h<<1 / h<<2 / h<<7 with the carry
bits (h1>>63, h1>>62, h1>>57) into a[2]; zero a[2..3] and loop until a[2]==a[3]==0.
The shifts are 128-bit, so the low word gets h0<<k and the high word gets
(h1<<k)|(h0>>(64-k)).

GCC intrinsic note: `vmull_p64(poly64_t, poly64_t)` returns `poly128_t`, NOT
`poly64x2_t` — a `poly64x2_t` initializer fails to compile ("incompatible types").
Cast the result to `unsigned __int128` and split with `(uint64_t)p` / `(uint64_t)(p>>64)`.
`vmull_p64` multiplies the LOW 64 bits of each operand; feed the high halves as
separate scalars rather than reaching for `vmull_high_p64`.

## The trap: natural vs "reflected" convention

OpenSSL's GCM (and most shipped GCM) uses the REFLECTED convention: bit i =
coefficient of y^i with y = x^(-1), i.e. "multiply by y" = RIGHT shift and the
reduction constant is 0xe1<<120 (= x^(-1) = x^127+x^6+x+1). This is the SAME
field, so results agree, but the bit mapping between the two conventions is a
basis change (x -> x^(-1)) — NOT a simple 128-bit bit reversal, and NOT a simple
bswap64. This is why "bit-reverse the operands, PMULL, bit-reverse back" does
NOT match fd_gcm_gmult_4bit.

OpenSSL additionally byte-swaps each 64-bit word (fd_ulong_bswap per word) of H
and the accumulator before the table-based multiply. So the full bridge is a
basis change composed with a byte swap — two independent transforms, either of
which alone will fail.

## 128-bit bit reversal (`rbit`)

Full 128-bit reversal of [hi:lo] is [rbit64(lo) : rbit64(hi)]:

```c
static inline uint64_t rbit64(uint64_t x){ uint64_t r; __asm__("rbit %0,%1":"=r"(r):"r"(x)); return r; }
/* bitrev128([hi:lo]) = [rbit64(lo) : rbit64(hi)] */
```

Useful for reflection generally, but note the caveat above: FULL 128-bit bit
reversal does NOT bridge GCM's reflected convention — the bridge is per-byte
`bitrev8` (see the RESOLVED section below).

## RESOLVED: the bridge is per-byte bit reversal (bitrev8)

The byte-order bridge was solved in a later session. The missing transform is
**bitrev8** — reverse bits within each byte, no byte reordering, no byte swap:

```
spec_mul(a, b)  =  bitrev8( natural_mul( bitrev8(a), bitrev8(b) ) )
bitrev8(x)       =  bswap64(rbit64(x))    // or NEON `rbit v.16b` on the block
```

Full 128-bit bit reversal does NOT work (tried, 100000/100000 mismatch) — the
bridge is strictly per-byte. Verified: canonical NIST vector
(`0x0388DACE60B6A392F328C2B971B2FE78 · 0x66E94BD4EF8A2C3B884CFA59CA342B2E =
0x5E2EC746917062882C85B0685353DEB7`), 200000 random GHASH cases byte-identical
to `fd_gcm_ghash_4bit`, and 100000 full AES-GCM arm-vs-ref iterations with 0
failures. In Firedancer this ships PMULL GHASH: AES-128-GCM went from ~14.5 MB/s
(4-bit table ref) to ~354 MB/s (~24x).

Working pattern: keep the accumulator in bitrev8-natural space across the whole
GHASH chain; `bitrev8` each 16-byte block on the way in and the final result on
the way out. `H` is stored once as `bitrev8(H_nist)` at init (no `bswap64`).
