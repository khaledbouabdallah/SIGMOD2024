# SIGMOD 2024 Programming Contest: Hybrid Vector Search

A C++ solution to the [ACM SIGMOD 2024 Programming Contest](https://github.com/transactionalblog/sigmod-contest-2024) on hybrid vector search, built between March and May 2024 as a graded team project at Université Paris Cité.

## The problem

Each data point is a 100-dimensional vector with a categorical attribute `C` and a timestamp `T`. For every query, return the 100 nearest neighbours (squared Euclidean distance) among the points that satisfy the query's constraints. There are four query types:

| Type | Constraint |
|---|---|
| 0 | None: vector search only |
| 1 | `C` equals a given value |
| 2 | `T` falls in a given range |
| 3 | Both `C` and the `T` range |

Answers are scored on recall against the exact result, under a time limit.

## Approach

We implemented several query strategies behind one `Query` interface and compared them. The strategy is selected with `queryType` in `src/main.cpp`.

| `queryType` | Strategy | Source |
|---|---|---|
| 0 | Sequential scan | `QuerySeqScan` |
| 1 | Sequential scan over the matching category/timestamp range | `QuerySeqScanRange` |
| 2, 3 | Incremental versions of the two scans | `QuerySeqScanIncremental`, `QuerySeqScanRangeIncremental` |
| 4, 5 | SAX-based filtering within the range | `QueryRangeSAXFilter`, `QueryRangeSAXOnly` |
| 6 | SAX lookaround | `QuerySAXLookaround` |
| 7 | IVF: k-means clusters, probe the `nprob` closest | `QueryIVF` |

The main ideas:

- **Pre-sorted indices.** Points are sorted by category and timestamp, so a constraint becomes a contiguous index range instead of a per-point check.
- **IVF with constraint-aware pruning.** Points are clustered with k-means, adapted from the [DKM](https://github.com/genbattle/dkm) library to work on our data structures. Each cluster keeps its timestamp bounds and category set, so clusters that cannot contain a valid answer are skipped, and only the matching range inside each probed cluster is scanned.
- **SAX / iSAX.** Vectors are summarized with PAA and SAX codes, with a trie index, to filter candidates cheaply.
- **Multithreading.** Queries are distributed across worker threads (pthreads) from a shared queue.

`src/main.cpp` currently runs a grid over the number of clusters `k` and `nprob`, and prints recall and run time for each combination.

## Build and run

Requires CMake 3.10+ and a C++14 compiler.

```bash
./run.sh
```

This builds the `TER` executable with `-O3 -march=native -ffast-math` and runs it.

The repository only includes the small dummy dataset. The input and reference paths are set at the top of `main()` in `src/main.cpp`; by default they point to the contest's 1M release files (`data/contest-data-release-1m.bin` and `data/contest-queries-release-1m.bin`), which must be downloaded from the contest repository into `data/`.

`submit.sh` packages a run with ReproZip, the format the contest required for submissions.

## Project layout

```
src/, include/   C++ sources and headers
tests/           k-means and SAX tests, profiling output
notebooks/       Python helpers for reading the binary formats
data/            dummy dataset and reference outputs
dk.cpp           the DKM k-means header our implementation is adapted from
```

Other branches: `baseline` (first sequential version), `IVF_Numba` (Python/Numba IVF prototype) and `IVFT` (threaded clustering).

## Team

Contributors, from the commit history: janie, Khaled Bouabdallah, Mohamed Bouanane and Jb Hochet.
