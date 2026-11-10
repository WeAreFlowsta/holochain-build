# Holochain builds with Flowsta's patches

This repository builds the [Holochain](https://github.com/holochain/holochain) conductor from an upstream release tag with a small set of patches applied. It exists for Flowsta's apps, which bundle a conductor.

It holds no copy of Holochain's source. Each build checks out the upstream tag at a pinned commit, applies the patch files in `patches/<tag>/`, and runs the same build command that upstream uses for its own release binaries. Everything that differs from the official binary is in those patch files.

## What is built

| Upstream release | Commit | Patches |
|---|---|---|
| `holochain-0.7.1` | `fdaa678786cd5618c36a7290aad69a5f1511d8a9` |2 |

Only the conductor (`holochain`) is built here. The key store (`lair-keystore`) is used as upstream ships it.

## The rules for a patch

A patch goes in only when all of these hold:

1. It fixes a problem that people using the apps meet, and that an app cannot work around.
2. It is small and contained.
3. It changes nothing another conductor can see: no change to the network protocol, to validation, or to hashes.
4. A database written by a patched build still opens with the official binary of the same release, and the other way round.
5. It is reported upstream, or is a backport of an upstream fix, so that it has an end. When an upstream release carries the fix, the patch is dropped.

## Patches for holochain-0.7.1

### 0001: narrow-arc time-slice reads use an index

Kitsune2 keeps a model of the DHT in 512 sectors. When a conductor creates a network space, and again every 15 minutes, it asks the database two questions for each sector: the earliest authored timestamp in the sector (`earliest_authored_timestamp_in_arc`), and the op hashes in a time slice of the sector (`op_hashes_in_time_slice`).

In 0.7.1 both statements read every action in the store each time they run, because there is no index on an op's storage location. The cost of one round therefore grows with the size of the whole store, 1,000 to 2,000 times over.

The patch:

- adds one index, `ChainOp(storage_center_loc)`, created when a DHT database is opened (`CREATE INDEX IF NOT EXISTS`, outside the migrations, so the official binary still opens the database and ignores the index);
- answers the two reads through that index when the arc is narrow (under 1/64 of the location space and not wrapping), which covers the per-sector reads;
- leaves every other arc on the unchanged 0.7.1 statement, because for wide arcs a scan is the faster plan.

The statements return the same rows. Rows with equal timestamps may come back in a different order; 0.7.1 does not define that order, and Kitsune2 combines the hashes with XOR, which does not depend on it.

Files changed: `crates/holochain_data/src/lib.rs`, `crates/holochain_data/src/dht/inner/sync_queries.rs` (with tests).

### 0002: the scheduler tick reads before it deletes

The conductor's scheduler runs every 100 ms. On each tick, for every cell, it deletes that agent's ephemeral scheduled functions that have started (`DELETE FROM ScheduledFunction WHERE ephemeral = 1 AND author = ? AND start_at <= ?`), whether or not there are any. A delete takes the database's write lock even when it removes nothing, and it measured about 1 ms each: with 20 cells and nothing scheduled, that is 11 seconds of database time every minute, and it is most of an idle conductor's CPU.

The patch asks first, with a read, whether there is a row the delete would remove (`SELECT EXISTS` with the same condition, about 0.07 ms), and runs the delete only when there is. The delete itself is unchanged, so a cell that does schedule functions behaves as before.

Files changed: `crates/holochain_data/src/dht/db_operations/scheduled_function.rs`, `crates/holochain_data/src/dht/inner/scheduled_function.rs` (with a test).

## Building

Run the `Build` workflow by hand for a test build: it uploads the binary and its SHA-256 as workflow artifacts. Pushing a tag named `<upstream tag>-flowsta.<n>` (for example `holochain-0.7.1-flowsta.1`) builds every platform and publishes a release with `SHA256SUMS.txt`.

To reproduce a build locally:

```bash
. ./upstream.env
git clone https://github.com/holochain/holochain && cd holochain
git checkout "$HOLOCHAIN_COMMIT"
for p in ../patches/"$HOLOCHAIN_TAG"/*.patch; do git apply "$p"; done
cargo test --package holochain_data --features test-utils,encryption
cargo build --release --package holochain --bin holochain
```

## Licenses

Holochain is the work of the Holochain Foundation and its contributors, under the licenses in the upstream repository (the conductor is CAL-1.0; `holochain_data` is Apache-2.0). Each patch here is offered under the license of the files it changes. The binaries built here are Holochain with those patches and are distributed under Holochain's own license terms.
