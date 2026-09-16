# Firedancer `bench` (validator TPS benchmark) on aarch64

`firedancer-dev bench` is Firedancer's equivalent of `solana-bench-tps`. It runs
the full validator topology plus load-generator tiles: `benchg` generates +
ed25519-signs transactions, `benchs` sends them over QUIC (or raw UDP with
`--no-quic`), `bencho` fetches the latest blockhash over RPC and orchestrates,
then reports TPS.

## Build

`make -j firedancer-dev` links on aarch64 after the `__NR_epoll_wait` fix (see
the syscall-table pitfall in SKILL.md). Binary lands at
`build/native/gcc/<cc-version>/bin/firedancer-dev` (version-stamped dir — do not
link the flat `build/native/gcc/lib/`, see SKILL.md).

## Running needs root

Tile workspaces come from hugetlbfs, and the default net provider is XDP:

```bash
sudo sysctl -w vm.nr_hugepages=32768      # reserve 2MiB pages (64GiB)
sudo chmod 777 /dev/hugepages             # or run the binary under sudo
sudo ./build/native/gcc/<cc-ver>/bin/firedancer-dev bench \
  --config <cfg>.toml --log-level-stderr NOTICE
```

In Hermes: put `SUDO_PASSWORD` in `~/.hermes/.env` to enable passwordless sudo,
but note the terminal tool reads `.env` at startup — a **session restart** is
needed for it to take effect — and it only auto-injects the password when
`sudo` is the FIRST token of the command (wrapping it in `timeout` or a pipe
breaks detection).

## Config schema gotchas (each cost iterations)

Configs for `firedancer-dev` live in `src/app/firedancer/config/` — NOT
`src/app/fdctl/config/` (that is frankendancer). The firedancer-dev default is
`src/app/firedancer/config/default.toml`.

- Section is `[runtime]`, NOT `[runtime.limits]`. The committed
  `firedancer-dev/config/minimal.toml` is stale (old schema) and fails to parse.
- `[hugetlbfs] max_page_size = "huge"` forces 2MiB pages; the default
  `"gigantic"` (1GiB) almost always fails ENOMEM on cloud/VM hosts.
- Shrink memory: `[accounts] max_accounts` defaults to `1_300_000_000` (~130GiB
  index). Drop `[development.bench] larger_max_cost_per_block` and
  `larger_shred_limits_per_block` (both balloon the workspace footprint).
- `[net] provider = "socket"` avoids XDP (no root NIC privilege). Default `"auto"`.
- Tile counts have hard minimums (e.g. `sign_tile_count >= 2`); the validator
  names the exact constraint in its error.

A known-good minimal config is committed at
`src/app/firedancer/config/bench-arm-minimal.toml`.

## NUMA / hugepage allocation — RESOLVED with blocklist_cores

- NUMA detection on ARM is CORRECT: `fd_numa_node_cnt()` returns the right count
  and `fd_numa_node_idx()` maps CPUs correctly (verified on a 4-node Kunpeng:
  cpus 0-79→node0, 80→1, 160→2, 240→3).
- The ~50GiB workspace is allocated per-NUMA-node in proportion to tile
  placement. With `affinity="auto"` on a shared host whose node 0 was exhausted
  (389MB free vs nodes 1-3 with 111GB each), the `configure` stage died with
  ENOMEM. **Fix:** `[layout] blocklist_cores = "0-79"` (blocklist node0's cores)
  so auto-affinity places tiles — and the workspace — on the free nodes. The
  validator then boots all 45 tiles on aarch64.
- Snapshot-pool boot bug — RESOLVED: after all 45 tiles boot, `snapzp` dies with
  `fcntl(snapshot pool fd 20xxxx) failed (EBADF), was the snapshot pool
  initialized on boot?`. Snapshot-pool fds live at a fixed base
  `FD_SNAP_FD_BASE = 200000` (`fd_backup.h`) and are opened by
  `initialize_snapshot_fds()` in `src/app/shared/commands/run/run.c`. The
  single-process dev path `run_firedancer_threaded` (`commands/dev.c`) calls it,
  but the bench path `bench_cmd_fn` (`src/app/shared_dev/commands/bench/bench.c`)
  is an INLINE COPY of that function that dropped the `initialize_snapshot_fds()`
  call (it only calls `initialize_accdb_fd()`). Fix: add
  `if( FD_LIKELY( config->is_firedancer ) ) initialize_snapshot_fds( config );`
  right after `initialize_accdb_fd( config )` in `bench_cmd_fn`. This is a
  genuine bench-path bug — NOT ARM-specific and NOT in the snapshot tile itself.
- Event-tile TLS — RESOLVED: next, the `event` tile dies with "TLS requested for
  event service (https:// URL) but this build does not include OpenSSL". The
  default `[tiles.event] url = "https://events-in.firedancer.io"` needs
  TLS/OpenSSL. Fix: set `[tiles.event] url = ""` (empty disables event
  reporting) in the bench config.
- RESULT: with those two fixes the full validator topology runs on aarch64 — all
  45 tiles boot and the consensus engine actually executes: `replay` becomes
  leader and produces blocks, `pack` packs ~16.7k microblocks/slot (~22M CUs),
  `runtime` settles fees, `repair` recovers 4128 shreds via FEC, `txsend` emits
  vote txns. Ran stable 90s+ (killed by test timeout, not a crash).

## Scaling the tile count — framework limits (not ARM bugs)

Throughput scales with `verify_tile_count` (ed25519 verification dominates
bench ingest): 4 verify → ~47k TPS, 32 verify → ~80k TPS (28.1k tx/block,
37.5M CUs of a 48M CU/block budget). Block rate is FIXED by slot time (~2.85
blocks/s), so TPS grows by packing more tx per block, not faster blocks.
Pushing further hits two hard framework caps, both worth knowing before
re-tuning:

- **`dedup` tile hard 64-input limit** — `fd_dedup_in_ctx_t in[64]` in
  `fd_dedup_tile.c`. Each verify tile wires one `verify_dedup` input plus one
  `replay_out`, so `verify_tile_count <= 63`. `verify_tile_count = 64` fails
  boot: `ERR dedup:0 fd_dedup_tile.c(262)[unprivileged_init]: FAIL:
  tile->in_cnt <= sizeof(ctx->in)/sizeof(ctx->in[0])`.
- **`bencho` tile "bad lazy" at high event counts** — at ~136 tiles (verify=60)
  bencho dies with `ERR bencho:0 fd_stem.c(390)[stem_run1]: bad lazy 289 33`.
  Root cause: `fd_tempo_lazy_default(cr_max=128)` returns `1 + (9*128)>>2 = 289`
  ns, and `fd_tempo_async_min(lazy=289, event_cnt=33, tick_per_ns)` returns 0
  when `tick_per_ns * 289 / 33 < 1` (the "unreasonably small async_min" floor).
  On this Kunpeng the cycle-counter tick period measures small enough that the
  floor trips once bencho's metric fan-in (`event_cnt`) grows past ~32. This is
  a tempo/tick-calibration limit, not a config error — if chasing it, review
  `fd_tempo_tick_per_ns` / `fd_tempo_tick_per_ns_dev` (firedancer-dev uses the
  `_dev` variant) for ARM cycle-counter calibration.

Practical ceiling on the 320-core/4-NUMA aarch64 host (node 0 blocklisted, ~240
cores usable) is ~80k TPS at verify=32. Higher tile counts need source changes,
not config.

## Stability on ARM (weak-memory-order check)

30 minutes of sustained high load (5 bench runs, ~4,950 blocks, verify=32
config) produced 0 errors, 0 crashes, 0 stalls, and ~28k tx/block with no
throughput drift. No TSO/weak-ordering race manifested under this benchmark.
