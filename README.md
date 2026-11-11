# Holochain builds with Flowsta's patches

This repository builds the [Holochain](https://github.com/holochain/holochain) conductor from an upstream release tag with a small set of patches applied. It exists for Flowsta's apps, which bundle a conductor.

It holds no copy of Holochain's source. Each build checks out the upstream tag at a pinned commit, applies the patch files in `patches/<tag>/`, and runs the same build command that upstream uses for its own release binaries. Everything that differs from the official binary is in those patch files.

## What is built

| Upstream release | Commit | Patches |
|---|---|---|
| `holochain-0.7.1` | `fdaa678786cd5618c36a7290aad69a5f1511d8a9` | 6 |

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

### 0002: the scheduler skips a store with nothing scheduled

The conductor's scheduler runs every 100 ms. On each tick, for every running cell, it runs three statements against the `ScheduledFunction` table (expired persisted functions, live functions, and a delete of started ephemeral ones), whether or not anything is scheduled. With 20 cells that is 600 statements a second, and an idle conductor spends 16 to 19 % of a core on them.

The patch asks each DHT store once per tick whether the table holds any row at all (`SELECT EXISTS (SELECT 1 FROM ScheduledFunction)`). Cells of one DNA share a store, so the cells of a store with no rows are left out of that tick; all three statements find nothing on an empty table, so nothing else changes. A store that has any scheduled function, for any agent, is handled exactly as before. If the check fails, the tick runs as before.

Files changed: `crates/holochain/src/conductor/conductor.rs`, `crates/holochain/src/conductor/cell.rs`, `crates/holochain_state/src/dht_store/reads.rs`, `crates/holochain_data/src/dht/db_operations/scheduled_function.rs`, `crates/holochain_data/src/dht/inner/scheduled_function.rs` (with a test).

### 0003: publish waits for a remote peer

In 0.7.1 a publish that finds no peer to send to is an error, and the ops stay unpublished so that they go out as soon as a peer appears (holochain/holochain#5919). On a conductor with no remote peer at all, which is how an app runs for a person with one device, that means no op is ever marked as published: every commit starts a publish round that lists every op the agent has authored and tries each one again, and so does the publish loop every one to five minutes.

The patch checks, before a publish round, whether the space's peer store holds any reachable agent that is not one of this conductor's own cells. If there is none, the round is skipped and the loop keeps running, so the ops are published within one to five minutes of a peer appearing. With any remote peer present, nothing changes. If the peer store cannot be read, the round runs as before.

Files changed: `crates/holochain/src/core/queue_consumer/publish_dht_ops_consumer.rs`.

### 0004: chain queries filter in SQL, and the init check is remembered

In 0.7.1 every source chain query reads and decodes every action of the agent's chain and only then applies its filter. Two things run such a query all the time: the check, before every zome call, of whether the zomes' `init` has run; and the HDK's `query` host function, whatever filter a zome gives it. Each therefore costs time in proportion to the chain's length (holochain/holochain#6060).

This patch is the fix proposed there, taken from [lightningrodlabs/holochain@ff9cf800c](https://github.com/lightningrodlabs/holochain/commit/ff9cf800c76aebe1c84b6ec40dab6c8bc0bc78bf) by Eric Harris-Braun, without its changelog line and with one import hunk adjusted to the 0.7.1 release:

- a query's action types and sequence range are applied in SQL, so it reads only the rows that can match; the in-memory filter still has the last word, so results are unchanged;
- a cell remembers that `init` has completed, so later zome calls do not ask.

Files changed: `crates/holochain/src/conductor/cell.rs`, `crates/holochain_state/src/source_chain.rs`, `crates/holochain_state/src/dht_store/reads.rs`, `crates/holochain_data/src/dht/db_operations/action.rs`, `crates/holochain_data/src/dht/inner/action.rs`, `crates/holochain_data/src/dht.rs` (with upstream's tests).

### 0005: capability grant reads start from the grants

Each zome call looks up the caller's capability grants with a join of `CapGrant` and `Action`. SQLite runs that join by walking every action of the author and probing `CapGrant` for each, so this read also grows with the chain.

The patch writes the join as `CROSS JOIN`, which in SQLite fixes the join order: the few grant rows are read first and each looks up its action by hash. The rows returned and their order are the same.

Files changed: `crates/holochain_data/src/dht/inner/cap_grant.rs`.

### 0006: the presence check takes any number of op ids

Before fetching the ops a peer offers, a conductor asks its database which of them it already holds (`check_op_hashes_present`). The statement binds every op id three times, and the bundled SQLite takes at most 32,766 bound values in one statement, so an offer of more than 10,922 ids fails with "too many SQL variables" and the whole gossip exchange is dropped (holochain/holochain#6062).

The usual way around that is to cap what one exchange may offer (`maxGossipOpBytes`). But Kitsune2 sends the ops of one sector and time slice as one batch, and skips a batch that does not fit the cap; a batch larger than the cap is then never sent. One agent's activity ops all sit in one sector, so an agent that writes a few thousand actions in one time slice makes such a batch, and a second device of the same person never receives that history.

The patch runs the check in chunks of 10,000 ids, so an exchange of any size is answered and the cap is no longer needed. The rows returned are the same.

Files changed: `crates/holochain_data/src/dht/db_operations/sync_queries.rs`, `crates/holochain_data/src/dht/inner/sync_queries.rs` (a test).

### What 0003 to 0005 measure together

One agent with no peers writing 8,000 entries, each with one link (Linux x86_64, encrypted databases):

| | All 8,000 | The first 1,000 | The last 1,000 |
|---|---|---|---|
| Patches 0001 and 0002 | 1,763 s | 61 s | 405 s |
| with 0003 | 1,331 s | 67 s | 300 s |
| with 0003 and 0004 | 528 s | 46 s | 84 s |
| with 0003, 0004 and 0005 | 549 s | 78 s | 65 s |

With all five the time per thousand no longer grows with the chain (61 to 78 s throughout; the runs were made on a desktop in use, so totals vary by run). Time spent in database statements for the same number of writes, early in the run against late: 10.6 s against 63.3 s with 0003, and 2.9 s against 3.8 s with all five.

## Building

The `Upstream tests` workflow, run by hand before a release tag, runs upstream's own tests of `holochain_data` and `holochain_state` and the conductor's tests of the patched areas on the patched source.

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
