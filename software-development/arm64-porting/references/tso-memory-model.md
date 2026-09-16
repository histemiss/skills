# x86 TSO → ARM Memory Model

x86 has a total-store-order (TSO) model: stores are globally ordered, so a
`volatile` store plus a compiler fence is enough to publish data (write payload
bytes, then write the flag; a reader that sees the flag also sees the payload).

ARM is weakly ordered: the hardware may reorder loads and stores. A
publish-flag pattern needs explicit release/acquire on ARM.

## The migration pattern

```c
/* producer — was: volatile store + compiler fence (x86-TSO-only) */
__atomic_store_n(&flag, seq, __ATOMIC_RELEASE);          /* stlr on ARM */

/* consumer — was: volatile load + compiler fence (x86-TSO-only) */
ulong seq = __atomic_load_n(&flag, __ATOMIC_ACQUIRE);    /* ldar on ARM */
```

Compiles to single `stlr`/`ldar` on ARM (with LSE) and a plain `mov` on x86 —
zero overhead on x86, correct on both. So the fix can be applied universally
(no `#ifdef __aarch64__`) and costs nothing on x86.

## Recognizing the pattern

Grep for `FD_COMPILER_MFENCE` / `__sync_synchronize` / raw `volatile` around a
lock-free publish/read. If a "flag" field guards a data region (producer writes
data then flag; consumer reads flag then data), it needs acquire/release. A
comment saying "acts as an implicit compiler fence" or "YMMV on non-x86" is a
red flag that the author assumed TSO.

## Verification

```sh
objdump -d foo.o | grep -E 'ldar|stlr'   # confirm acquire/release emitted
```

Multi-threaded tests (mcache/fseq-style) are the real correctness check — a
structure can pass single-threaded tests and still be racy on ARM.
