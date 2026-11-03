# Benchmark results, 2026 Nov 01 GMT

Flanger 0.20.0 against the same image with `FLANGER_OFF=1`.

## Method

- A desktop (AMD Ryzen AI MAX+ 395, NVMe SSD, btrfs with zstd compression), rootless podman, 4 GB container limit. Other work ran on the host, so results are noisy.
- Each scenario: load data, 30 s pause, 180 s run with 8 clients, 60 s idle, a logical dump, 120 s wait. Two repeats, with the arm order swapped between them.
- Own caches kept small so the page cache matters: `shared_buffers=256MB`, InnoDB buffer pool 256 MB, WiredTiger cache 0.25 GB.
- Data: pgbench scale 100; sysbench 8 tables of 1M rows; YCSB 2M records, zipfian.
- Billed memory is `memory.current` read from the host. Run memory is the mean over the second half of the run; after dump is the last sample 120 s after the dump.
- Flanger's own memory was not captured in this round (harness bug).

## Results

Each cell: repeat 1 / repeat 2.

| Workload | tps off | tps on | Median change | p99 ms off | p99 ms on | Run MB off | Run MB on | After dump MB off | After dump MB on | Evicted MB | Re-read MB |
|---|---|---|---|---|---|---|---|---|---|---|---|
| pg-tpcb | 2801 / 2632 | 2746 / 2757 | +1.3% | 6.94 / 10.64 | 6.45 / 6.41 | 3082 / 3027 | 2950 / 3164 | 3066 / 3066 | 1676 / 1957 | 1934 | 152.5 |
| pg-select-uniform | 228344 / 216357 | 226172 / 208971 | -2.1% | 0.07 / 0.08 | 0.07 / 0.08 | 2183 / 3012 | 2711 / 2737 | 1887 / 2988 | 1685 / 1929 | 639 | 0.7 |
| mysql-rw | 727 / 706 | 691 / 710 | -2.3% | 61.08 / 63.32 | 65.65 / 62.19 | 777 / 655 | 665 / 760 | 780 / 604 | 556 / 663 | 113 | 5.5 |
| mariadb-rw | 1033 / 908 | 1036 / 1005 | +5.1% | 39.65 / 46.63 | 39.65 / 40.37 | 691 / 595 | 618 / 636 | 690 / 572 | 491 / 546 | 341 | 211.5 |
| mysql-ro | 6438 / 5513 | 5983 / 5983 | +0.1% | 3.02 / 3.25 | 3.43 / 2.91 | 664 / 681 | 585 / 614 | 591 / 531 | 549 / 579 | 150 | 5.4 |
| mariadb-ro | 6156 / 6135 | 6752 / 6092 | +4.5% | 8.74 / 5.77 | 2.66 / 6.09 | 625 / 601 | 509 / 514 | 550 / 547 | 488 / 503 | 122 | 0.0 |
| mongo-ycsb-a | 49589 / 37563 | 44827 / 31978 | -11.9% | 0.54 / 0.75 | 0.66 / 0.91 | 4295 / 4293 | 4292 / 4293 | 4121 / 4186 | 3098 / 2208 | 1913 | 598.2 |
| mongo-ycsb-b | 56149 / 51778 | 55084 / 50155 | -2.5% | 0.46 / 0.52 | 0.48 / 0.53 | 4295 / 4295 | 4294 / 4295 | 4185 / 4181 | 3841 / 3940 | 675 | 337.9 |
| mongo-ycsb-c | 62406 / 59506 | 55345 / 56744 | -8.1% | 0.41 / 0.43 | 0.47 / 0.46 | 3063 / 2875 | 2866 / 3053 | 3091 / 2932 | 2519 / 2707 | 412 | 16.6 |
| mongo-ycsb-e | 12629 / 11948 | 9742 / 10864 | -16.2% | 2.02 / 1.88 | 2.36 / 2.10 | 3020 / 3000 | 3097 / 2966 | 3106 / 3089 | 2427 / 2823 | 612 | 36.7 |

## Reading these

- Repeats differ by up to about 25% even without Flanger, so small gaps mean nothing.
- Postgres, MySQL and MariaDB: no slowdown distinguishable from noise.
- MongoDB: YCSB C and E were slower with Flanger on both repeats. A hit the 4 GB limit, so its figures are distorted. Cause under investigation.
- Savings show after a spike. During steady load billed memory is about the same, since little of an active cache is evicted. MySQL saves little or nothing.
- Re-read figures use the corrected counter, capped by disk reads, and fedora's compressed disk understates them.

## MongoDB follow-up, 2026 Nov 02 GMT

Two repeats were not enough: the first table's 8-16% MongoDB gaps did not hold. YCSB E, 8 GB limit, 10 strictly interleaved runs per arm, Flanger 0.20.1:

| | Median ops/s | Mean ops/s | p99 ms | Memory after dump MB |
|---|---|---|---|---|
| No Flanger | 12,524 | 13,040 | 1.79 | 3,083 |
| Flanger | 12,402 | 13,016 | 1.79 | 2,786 |

Both arms moved between about 11.5k and 15.6k ops/s together over time, so that swing is the host, not Flanger. Old against fixed hook, 6 runs each, also showed no difference. YCSB C still varies too much to read (48k to 80k in every arm, 5 repeats).

## Postgres TPC-B follow-up, 2026 Nov 02 GMT

5 side-by-side runs had suggested 3-5% lower throughput. 10 strictly interleaved runs per arm, Flanger 0.20.1:

| | Mean tps | Median tps | Median p99 ms | Memory after dump MB |
|---|---|---|---|---|
| No Flanger | 2,692 | 2,683 | 9.77 | 3,036 |
| Flanger | 2,687 | 2,706 | 7.27 | 1,963 |

Paired differences ranged from -8.6% to +7.1% (median +0.5%, Flanger ahead in 6 of 10). No measurable cost.

The filesystem change in 0.21.0 was checked against 0.20.1 the same way (5 runs each): pgbench TPC-B 2,763 against 2,764 tps, MySQL read-write 697 against 697 tps.

## Self-tuning horizon, 2026 Nov 02 GMT (not shipped)

double the idle horizon when more than half of a window's evictions came back, step back down otherwise. Against 0.21.0, 5 interleaved runs each, 8 GB limit:

| Workload | Throughput, paired median | Re-read MB | Memory after dump MB |
|---|---|---|---|
| MariaDB read-write | +0.9% (-16% to +20%) | 223 to 135 | 497 to 534 |
| MongoDB YCSB B | -0.7% (-20% to +5%) | 214 to 265 | 4,365 to 4,609 |
| pgbench TPC-B | +0.9% (-3% to +12%) | 80 to 26 | 1,820 to 1,916 |

Fewer re-reads on two of three, 5-7% more memory kept after a dump, no speed gain. The re-reads it avoided did not cost anything measurable, so it would raise the bill for nothing. Kept unmerged.
