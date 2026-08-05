# Import Benchmark

## Why

This tool is used to benchmark the performance of **ethrex**.  
We aim to execute the same set of blocks on the same hardware to ensure consistent 
performance comparisons. Doing this on a running node is difficult because of variations 
in hardware, peer count, block content, and system load.

To achieve consistent results, we run the same blocks multiple times on the same machine 
using the `import-bench` subcommand.

## Setup

To run this benchmark, you will need:

- An **ethrex** database containing the blockchain state (required for realistic
 database performance testing), located at:  
  `~/.local/share/ethrex_NETWORK_bench/ethrex`
- The database **must have completed snapshot generation** (`flatkeyvalue` generation).  
  *(On mainnet, this process takes about 8 hours.)*
- A `chain.rlp` file containing the blocks you want to test, located at:  
  `~/.local/share/ethrex_NETWORK_bench/chain.rlp`
- It is recommended that the file contains **at least 1,000 blocks**, 
which can be generated using the `export` subcommand in ethrex.

### Recommended procedure

1. Run an ethrex node until it fully syncs and generates the snapshots.  
2. Shut down the node and copy the database and the last block number.  
3. Restart the node and let it advance by *X* additional blocks.  
4. Stop the node again and run:  
   ```bash
   ethrex export --first <block_num> --last <block_num + X> ~/.local/share/ethrex_NETWORK_bench/chain.rlp
   ```

## Run

The Makefile includes the following command:

```
run-bench: ## Runs a benchmark for the current PR.
```

Parameters:
  - BENCH_ID: Identifier for the log file, saved as bench-BENCH_ID.log
  - NETWORK: Network to access (e.g., hoodi, mainnet)

Example:

`make run-bench BENCH_ID=1 NETWORK=mainnet`

## View Output

You can view and compare benchmark results with:
`python3 parse_bench.py <bench_file_1> <bench_file_2> <...>`

## RocksDB `max_bytes_for_level_base` A/B

Compares compaction write amplification on the four state CFs
(`account_trie_nodes`, `storage_trie_nodes`, `account_flatkeyvalue`,
`storage_flatkeyvalue`) when changing `--rocksdb.max-bytes-for-level-base`.

**Primary metric:** Sum `W-Amp` / `Write(GB)` in the cfstats dump at end of
`import-bench` (logged after a compaction settle). Do not use Int for a single
end-of-run dump. Ggas/s from `parse_bench.py` is secondary.

**Write volume:** prefer **≥5k–10k** mainnet blocks in `chain.rlp`. With ~1k
blocks, treat results as directional only.

### Example (same frozen DB copy each run)

Match `benchmark.sh`: wipe the temp datadir, copy the frozen DB, then import.
`--network mainnet` makes the effective path `$TEMP/mainnet` (ethrex migrates an
unsuffixed copy under `$TEMP` automatically when needed). Confirm the open log
line includes the intended `max_bytes_for_level_base`.

```bash
TEMP=~/.local/share/temp
CHAIN=~/.local/share/ethrex_mainnet_bench/chain.rlp
FROZEN=~/.local/share/ethrex_mainnet_bench/ethrex

run_one() {
  local level_base=$1
  rm -rf "$TEMP"
  cp -a "$FROZEN" "$TEMP"
  ethrex --datadir "$TEMP" --network mainnet \
    --rocksdb.max-bytes-for-level-base "$level_base" \
    import-bench "$CHAIN"
}

# 256 MiB (pre-fix default behavior)
run_one $((256*1024*1024))
# 2 GiB (current default)
run_one $((2*1024*1024*1024))
# 4 GiB (Tuning Guide upper bound)
run_one $((4*1024*1024*1024))
```

Repeat each value ≥3 times on a quiet machine with identical
`--rocksdb.block-cache-size`. Always use a fresh frozen-DB copy per run.
After import, wait for `RocksDB state-CF compaction settled` (or a settle
timeout warning), then compare Sum `W-Amp` / `Write(GB)` in the four
`rocksdb state CF compaction stats` log records.

### Decision rule

1. Discard a value that clearly worsens stalls or mean Ggas/s beyond noise.
2. Prefer the lowest average state-CF **Sum** W-Amp (and compaction Write(GB)).
3. If 2 GiB and 4 GiB are within noise, keep **2 GiB**.
4. If 256 MiB is clearly worse, that confirms the level-base fix; do not revert.
