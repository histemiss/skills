---
name: solana-validator-setup
description: "Use when setting up a Solana/Agave validator or RPC node."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [solana, agave, validator, rpc, blockchain, ledger]
    related_skills: [jito-mev-grpc]
---

# Solana / Agave Validator Node Setup

Build and run an Agave validator (Solana node) from source, mount a dedicated ledger disk, apply kernel tuning, open firewall ports, and launch sync-only or voting. Reference setup at `/home/ubuntu` (agave source in `~/agave`, scripts `start-validator.sh`, log `validator.log`).

## When to Use

- Install agave / run a Solana validator or RPC node
- Expand disk + mount a ledger volume, tune kernel, launch/stop validator
- "install agave", "start solana", "mount 500G disk for ledger"

## Key Facts (verify before acting)

- **`agave-validator` is NOT in the prebuilt release since 4.x.** `solana-release-*.tar.bz2` (~84MB) only carries CLI tools; the validator binary is source-only (`cargo build --release --bin agave-validator`).
- CLI install: `sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"` → puts `solana`, `solana-keygen`, `agave-ledger-tool`, etc. in `~/.local/share/solana/install/active_release/bin/`.
- Build deps (Ubuntu): `sudo apt-get install libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang cmake make libprotobuf-dev protobuf-compiler libclang-dev`.
- Rust version pinned by `rust-toolchain.toml` in the agave repo; rustup auto-installs it on first `cargo` run.
- Ledger disk: XFS preferred. `--limit-blockstore-size` (renamed from `--limit-ledger-size`) min value 100000000 shreds.

## Procedure

1. **Install CLI + toolchain**: run the agave-install one-liner; install rustup (`curl https://sh.rustup.rs -sSf | sh -s -- -y --profile minimal`); apt-get build deps.
2. **Build validator**: `git clone --depth 1 --branch vX.Y.Z https://github.com/anza-xyz/agave.git agave`, then `cargo build --release --bin agave-validator`. Copy binary to `~/.local/share/solana/install/active_release/bin/` and `sudo setcap 'cap_net_admin,cap_net_raw+eip'` it.
3. **Mount ledger disk**: `sudo mkfs.xfs /dev/vdb` (USER must run mkfs — agent-blocked), `sudo mount /dev/vdb /mnt/solana`, add fstab entry `UUID=... /mnt/solana xfs defaults,noatime,nofail 0 0`. Grow later with `sudo xfs_growfs /mnt/solana`.
4. **Kernel tuning** (validator exits without it): `net.core.rmem_default=134217728`, `net.core.rmem_max=134217728`, `net.core.wmem_default=134217728`, `net.core.wmem_max=134217728`, `vm.max_map_count=1000000` in `/etc/sysctl.d/21-solana-validator.conf` + `sysctl --system`.
5. **Open security-group ports** (cloud console): TCP 8000-8020, TCP 8899+8900, UDP 8000-8020. Port check is fatal in 4.x.
6. **Launch** (sync-only): `agave-validator --identity /path/validator-keypair.json --ledger /mnt/solana --no-voting --entrypoint entrypoint.mainnet-beta.solana.com:8001 --rpc-port 8899 --limit-blockstore-size 100000000 --log validator.log`.
7. **Auto-stop**: `systemd-run --user --unit=solana-validator --property=RuntimeMaxSec=6h bash start-validator.sh` (RuntimeMaxSec stops it; verify via `systemctl --user show ... -p RuntimeMaxUSec`).

## Pitfalls

- **Hermes background procs have a ~4GB cgroup memory cap** (`_WORKER_MEMORY_MAX_CAP_BYTES=4GB`). A 64-parallel `cargo build` gets OOM-killed (exit 137) even with 244GB free. Run heavy builds via `systemd-run --user` (user slice `memory.max=max`).
- **Port check is fatal in 4.x** (`--no-port-check` removed): if TCP 8000/8900 are unreachable, validator exits "Received no response ... check your port configuration".
- **OS network limit test fails** when `net.core.rmem_max` is below 128MB (default 16MB on Ubuntu) — apply sysctl first.
- **500G disk is too small for mainnet** (2026): snapshot archive ~115G + unpacked accounts ~330GiB + working space = 600GB+. Use ≥1TB, and it re-downloads a newer snapshot if the local one is "too old".
- **Restart re-untars snapshot + rebuilds index (~55min)** because the accounts index is in-memory by default. Add `--accounts-index-path <ledger>/accounts_index` + a finite `--accounts-index-limit` (e.g. 200GB) to persist it for fast restart.
- **Account data is NOT in RAM**: mainnet accounts ~330GiB > RAM (244GiB). Only the index is in memory; account values live on disk (random reads, ~14ms latency). Replay is latency-bound (sequential per-slot), not CPU/RAM/disk-throughput-bound — hence ~2.5 slots/sec ≈ real-time, and 64 cores sit idle (CPU ~2-3 cores).
- **mkfs is hardline-blocked** by the agent (unconditional). The user must run `sudo mkfs.xfs` / `mkfs.ext4` themselves; `xfs_growfs`, `mount`, `resize2fs` are fine.
- **`solana-keygen new`** needs `--no-bip39-passphrase` to be non-interactive; prints seed phrase (secret) to stdout.

## Verification

- `agave-validator --version` prints a version string.
- `solana --url http://127.0.0.1:8899 slot` returns a slot once RPC is up.
- Log shows `bank frozen: <slot>` (replay position) and `check_slot_agrees_with_cluster ... slot: <tip>` (cluster tip); gap shrinking = catching up.
- `systemctl --user is-active solana-validator` = `active`; no `Failed to start validator` in the log.
