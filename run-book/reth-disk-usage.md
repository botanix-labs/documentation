### Who this is for

Node operators whose disk usage has ballooned due to the reth snapshots and want to safely shrink the reth database on disk.

To prevent further snapshots being created, simply disable snapshots by geting rid of these flags on reth startup

```sh
--sync.enable_state_sync
--sync.enable_historical_sync
```

To take it a step further i.e cleaning the snapshot table and shrink db size

#### **NB: The following operations are fully offline. Plan a maintenance window.**

### Stop the node & take a backup

```
docker-compose --env-file .bitcoin.env down

mkdir -p data/reth.backup
rsync -aH --info=progress2 data/reth/ data/reth.backup/
```

### Remove the snapshots table
Download the Botanix DB cleaner and run it against your reth data directory

```sh
wget https://storage.googleapis.com/botanix-artifact-registry/releases/botanix-db-cleaner/botanix-db-cleaner
chmod +x botanix-db-cleaner

sudo ./botanix-db-cleaner --db-path data/reth/db --entity snapshots
```

### Compact the reth database
Reth uses [libmdbx](https://libmdbx.dqdkfa.ru/intro.html). The safest way to shrink the file is to create a compacted copy, then swap directories.

```sh
git clone https://github.com/paradigmxyz/reth
cd reth
make db-tools
```

Now run `mdbx_copy` with compaction:
```sh
sudo mkdir -p data/reth.compact/db
sudo ./db-tools/mdbx_copy -c data/reth/db data/reth.compact/db/mdbx.dat
```

This reads your current DB at `data/reth/db` and writes a compacted MDBX file to `data/reth.compact/db/mdbx.dat`

### Swap directories & restore static assets
Replace the old data directory with the compacted one and put back `static-files` reth expects.
```sh
sudo mv data/reth data/reth.before-compact

sudo mkdir -p data/reth/db
sudo mv data/reth.compact/db/mdbx.dat data/reth/db/

sudo mkdir -p data/reth
cp -r data/reth.before-compact/static_files/ data/reth/
```

### Bring services back on
```
docker-compose --env-file .bitcoin.env up -d
```

If all looks good, logs come up clean and you're satisfied with stability, remove the old directory later to reclaim space:

```sh
sudo rm -rf data/reth.before-compact data/reth.compact
```