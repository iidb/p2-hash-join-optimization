# Database Systems: Hash Join Optimization

In this assignment, you will optimize hash join execution in **BabyDB**, a simple in-memory database.
The goal is to speed up multi-table join queries without changing the join order.

## Task Description

BabyDB executes joins using the classic **Volcano iterator model**: each operator pulls data one chunk at a time from its children.
The baseline hash join implementation uses `std::unordered_multimap` and performs significant per-tuple copying.

Your task is to optimize the hash join pipeline so that the provided benchmark queries run faster.
You may modify any source file in the `src/` directory, but the benchmark harness (`benchmark/run_job.cpp`) and test cases must remain unchanged.
Your implementation must produce **correct results** — any mismatch in output count invalidates the run.

### Benchmark

The benchmark consists of 10 join queries derived from the [Join Order Benchmark (JOB)](https://github.com/gregrahn/join-order-benchmark).
Each query is a multi-way hash join tree over pre-loaded tables.
The harness measures end-to-end execution time (plan construction + execution) and checks output cardinality.

## Getting Started

**Step 1.** Download and extract the test dataset:

```bash
# Download data.zip from the release page
wget https://github.com/iidb/p2-hash-join-optimization/releases/download/v1.0/data.zip
unzip data.zip
```

This creates a `reduced_data/` directory containing `10a.data` through `19a.data`.

**Step 2.** Build the project:

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc) p2-leaderboard
```

> Use `Release` mode for benchmarking. `Debug` mode enables sanitizers and is useful for correctness testing only.

**Step 3.** Run the benchmark:

```bash
./bin/p2-leaderboard reduced_data
```

Example output:

```
10a: correct time: 142ms
11a: correct time: 87ms
...
Total time: 1205ms
```

You can also run individual test cases:

```bash
./bin/p2-leaderboard reduced_data 10a 11a
```

**Step 4.** Run correctness tests:

```bash
make check-tests
```

## Optimization Approaches

You may consider the following techniques (not exhaustive):

### 1. Join Implementation Optimization

The baseline hash join has several performance bottlenecks:

- **Per-tuple copying**: `UnionTuple` creates a new vector for every output row.
- **Hash table choice**: `std::unordered_multimap` has poor cache locality and high overhead per entry.
- **Memory allocation**: Frequent small allocations from vector resizing.

Consider replacing the hash table with a more cache-friendly structure, reducing copies with columnar or pointer-based approaches, and pre-allocating memory.

### 2. Bloom Filter / Semi-Join Reduction

When joining a large probe side against a small build side, many probe tuples may find no match.
A **Bloom filter** built from the build-side keys can cheaply reject non-matching probe tuples before they reach the hash table.

In multi-way join trees, this idea generalizes to **Robust Predicate Transfer (RPT)**: push Bloom filters across multiple joins to filter early.
Note that in an in-memory setting, the overhead of building and probing Bloom filters may offset the savings — a 20% improvement from this approach alone would be a good result.

### 3. Hash Partitioning

Partitioning both build and probe sides by hash value before joining can improve cache locality, especially when the hash table exceeds L2/L3 cache size.
Each partition is joined independently, keeping the working set small.

### 4. Other Ideas

- SIMD-accelerated hashing or probing
- Software prefetching to hide memory latency
- Chunk size tuning
- Parallelism (thread-level or partition-level)

## Project Structure

```
.
├── CMakeLists.txt
├── README.md
├── src/
│   ├── babydb.cpp                        # BabyDB instance + OptimizeJoinPlan()
│   ├── include/
│   │   ├── babydb.hpp
│   │   ├── common/                       # Types, config, macros
│   │   ├── concurrency/                  # Transaction support
│   │   ├── execution/
│   │   │   ├── operator.hpp              # Base Operator class (Volcano model)
│   │   │   ├── hash_join_operator.hpp    # ★ Primary optimization target
│   │   │   ├── seq_scan_operator.hpp
│   │   │   └── ...                       # Other operators
│   │   └── storage/                      # Catalog, Table, Index
│   ├── execution/
│   │   ├── hash_join_operator.cpp        # ★ Primary optimization target
│   │   └── ...
│   ├── concurrency/
│   └── storage/
├── benchmark/
│   └── run_job.cpp                       # Benchmark harness (do NOT modify)
├── test/                                 # Correctness tests
└── third_party/                          # GoogleTest
```

### Key Files

| File | Description |
|------|-------------|
| `src/execution/hash_join_operator.cpp` | Baseline hash join — start here |
| `src/include/execution/hash_join_operator.hpp` | Hash join operator header |
| `src/babydb.cpp` → `OptimizeJoinPlan()` | Hook for global join plan optimization (currently empty) |
| `src/include/common/config.hpp` | `CHUNK_SUGGEST_SIZE` and other config |
| `src/include/execution/operator.hpp` | Base `Operator` class with `Next()` / `Init()` / `Check()` |
| `src/include/common/typedefs.hpp` | Core types: `data_t` (int64), `Tuple`, `Schema`, `Chunk` |

## Grading

| Component | Points |
|-----------|--------|
| Performance improvement (1 point per test case) | 10 |
| Report quality | 10 |
| **Total** | **20** |

### Performance Scoring

Each test case is worth 1 point. You earn the point if your implementation runs faster than the baseline threshold for that test case.

### Report Requirements

Submit a report (PDF) that:

1. Explains your optimization approach in detail
2. Provides performance measurements before and after each optimization
3. Analyzes why each optimization was effective or ineffective
4. Discusses trade-offs in your design

The report should be detailed enough that another student could reproduce your approach.

## Submission

Submit a zip file containing:

1. Your complete source code
2. Your report in PDF format

## References

1. Junyi Zhao, Kai Su, Yifei Yang, Xiangyao Yu, Paraschos Koutris, and Huanchen Zhang. "Debunking the Myth of Join Ordering: Toward Robust SQL Analytics." (2024).
2. Yiming Qiao and Huanchen Zhang. "Data Chunk Compaction in Vectorized Execution." (2024).
3. Harald Lang, Thomas Neumann, Alfons Kemper, Peter Boncz. "Performance-Optimal Filtering: Bloom Overtakes Cuckoo at High Throughput." (2019).
4. Maximilian Bandle, Jana Giceva, Thomas Neumann. "To Partition, or Not to Partition, That is the Join Question in a Real System." (2021).

## Acknowledgments

This assignment is based on [BabyDB](https://github.com/hehezhou/babydb) by [@hehezhou](https://github.com/hehezhou). Thanks for the excellent database framework and benchmark infrastructure.
