# Flanger, the Railway cleaner

Flanger lowers the memory bill of databases on [Railway](https://railway.com) by dropping page cache
the database isn't using. It ships as ready-made Railway templates for Postgres, MySQL, MariaDB and
MongoDB.

| Database | Deploy |
|---|---|
| Postgres 18 | https://railway.com/deploy/postgres-flanger |
| MySQL 8.4 | https://railway.com/deploy/mysql-flanger |
| MariaDB 11.8 | https://railway.com/deploy/mariadb-flanger |
| MongoDB 7.0 | https://railway.com/deploy/mongodb-flanger |

This repository holds documentation and benchmark results only. Flanger's source code is not public;
the templates run published container images (`ghcr.io/housek/flanger-<engine>`).

## The problem: Railway bills page cache

Railway bills a service's memory as Linux counts it for the container, and that count includes the
page cache: file data the kernel keeps in memory after it was read or written. A database's data
files land there whenever they are read. After a backup, a `pg_dump`, a bulk import or a large report,
that cache can stay at several times what the database actually needs, and you keep paying for it.

The database's own cache (Postgres `shared_buffers`, the InnoDB buffer pool, the WiredTiger cache) is
separate, and Flanger never touches it.

## What Flanger does

Flanger wraps the database's start command in the container. It watches which cached file data the
database re-reads and drops the rest with `fadvise(DONTNEED)`, at a pace bounded by the disk's
measured latency. It backs off when what it dropped comes back. It only drops clean page cache, so a
wrong guess costs a re-read from disk, never data. There are no settings to tune.

## Results

Measured in containers on a desktop, not on Railway; full method and numbers in
[BENCHMARKS.md](BENCHMARKS.md).

- **After a spike** (a logical dump in these tests), billed memory was 14-35% lower, depending on the
  database. Postgres saved the most: 3,036 MB down to 1,963 MB after a dump of a 1.5 GB database.
- **During steady load**, the saving was small (about 1-4%), because little of an actively used cache
  can be dropped.
- **Speed:** with 10 interleaved runs per arm, neither pgbench TPC-B nor MongoDB YCSB E showed a
  measurable difference. Shorter comparisons of other workloads showed differences within the
  run-to-run spread, which was large on this machine.
- **On Railway**, a mostly idle Postgres deployment held at 160 MB with about 1 MB of page cache over 24
  hours.

## Overhead

| | Measured |
|---|---|
| Flanger's own memory | 6-7 MB |
| CPU | about 0.1% of one core on an idle database; scanning is capped at 1% of the container's CPU |
| Extra disk reads | re-reads of what it dropped, held to a budget set by the disk's measured latency |

## When not to use it

- **Your cache is mostly hot.** If the database re-reads nearly all its data all the time, there is
  little cold cache to drop, and little to save.
- **Your platform bills a fixed size.** Only usage-billed memory benefits; on fixed-size plans there is
  nothing to save.
- **Most of your memory is the database's own cache.** MySQL and MariaDB templates keep a 1 GB InnoDB
  buffer pool, which Flanger cannot shrink; on small databases that limits savings.
- **Throughput matters more to you than memory, and you haven't measured your workload.** Test first.
  `FLANGER_OFF=1` turns Flanger off without redeploying a different image.

## Reporting an issue

Open an issue in this repository with these lines from your service's Railway logs:

- the `event="start"` and `event="roots"` lines from the start of the log
- the last few `event="summary"` lines
- any `event="warning"` or `event="backoff"` lines near the problem

They contain versions, sizes, rates and data-folder paths, and no passwords or query contents, but
read them before sharing.
