# Database Systems: Hash Join Optimization

In this assignment, you will optimize hash join execution in **BabyDB**, a simple in-memory database.
The goal is to speed up multi-table join queries without changing the join order.

## Task Description

BabyDB executes joins using the classic **Volcano iterator model**: each operator pulls data one chunk at a time from its children.
The baseline hash join implementation uses `std::unordered_multimap` and performs significant per-tuple copying.

Your primary task is to optimize hash join execution for a given join order to achieve better performance.
We strongly recommend implementing the **Robust Predicate Transfer (RPT)** algorithm as described in the recent [paper](https://www.vldb.org/pvldb/vol17/p3282-zhao.pdf) (Reference 1).
However, since we are working with an in-memory database, the additional overhead of creating Bloom filters (as used in RPT) might not always lead to substantial performance improvements.
A **20% improvement** would be considered a good achievement.
Therefore, you are encouraged to explore additional or alternative optimization techniques.

You may modify any source file in the `src/` directory, but the benchmark harness (`benchmark/run_job.cpp`) and test cases must remain unchanged.
Your implementation must produce **correct results** — any mismatch in output count invalidates the run.
The testing framework and data loading interfaces are provided in the `benchmark/run_job.cpp` file.

### Benchmark

The benchmark consists of 10 join queries derived from the [Join Order Benchmark (JOB)](https://github.com/gregrahn/join-order-benchmark).
Each query is a multi-way hash join tree over pre-loaded tables.
The harness measures end-to-end execution time (plan construction + execution) and checks output cardinality.

## Getting Started

**Step 1.** Download and extract the test dataset:

```bash
wget https://cloud.tsinghua.edu.cn/seafhttp/files/b3d9f1fe-1c02-4411-9d73-bd771125512a/data.zip
unzip data.zip
```

**Step 2.** Build the project:

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j p2-leaderboard
```

> Use `Release` mode for benchmarking. `Debug` mode enables sanitizers and is useful for correctness testing only.

**Step 3.** Run the benchmark:

```bash
./bin/p2-leaderboard <path-to-data>
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
./bin/p2-leaderboard <path-to-data> 10a 11a
```

**Step 4.** Run correctness tests (inside the `build/` directory):

```bash
make check-tests
```

## Optimization Approaches

You may consider the following techniques (not exhaustive):

### 1. Join Implementation Optimization (Reference 2)

The baseline hash join has several performance bottlenecks:

- **Per-tuple copying**: `UnionTuple` creates a new vector for every output row.
- **Hash table choice**: `std::unordered_multimap` has poor cache locality and high overhead per entry.
- **Memory allocation**: Frequent small allocations from vector resizing.

Consider replacing the hash table with a more cache-friendly structure, reducing copies with columnar or pointer-based approaches, and pre-allocating memory.

### 2. Bloom Filter / Semi-Join Reduction (Reference 3)

When joining a large probe side against a small build side, many probe tuples may find no match.
A **Bloom filter** built from the build-side keys can cheaply reject non-matching probe tuples before they reach the hash table.

In multi-way join trees, this idea generalizes to **Robust Predicate Transfer (RPT)**: push Bloom filters across multiple joins to filter early.
Note that in an in-memory setting, the overhead of building and probing Bloom filters may offset the savings — a 20% improvement from this approach alone would be a good result.

### 3. Hash Partitioning (Reference 4)

Partitioning both build and probe sides by hash value before joining can improve cache locality, especially when the hash table exceeds L2/L3 cache size.
Each partition is joined independently, keeping the working set small.

### 4. Other Ideas

- SIMD-accelerated hashing or probing
- Software prefetching to hide memory latency
- Chunk size tuning
- Parallelism (thread-level or partition-level)

If you determine that approaches other than RPT provide better results for this specific in-memory scenario, you may focus on those optimizations instead.
What's most important is achieving better performance and being able to explain your approach.

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

### Implementation Requirements

Your implementation should:

1. Work correctly with the provided benchmark framework
2. Improve join performance compared to the baseline implementation
3. Be well-documented with comments explaining your optimization strategies

### Report Requirements

You must submit a detailed report that:

1. Follows academic paper standards and structure
2. Explains your optimization approach in detail
3. Provides a performance breakdown analysis for each implemented technique
4. Discusses why certain optimizations were effective or ineffective
5. Includes performance measurements and comparisons with the baseline
6. Analyzes any trade-offs in your approach

The report should be comprehensive enough that another student could implement your approach based solely on your explanation.

## Submission

Submit a zip file containing:

1. Your optimized implementation code
2. A detailed report in PDF format explaining your optimizations
3. Any additional scripts or tools you created for testing/evaluation

## Tips

1. Consider the trade-offs between different optimization strategies
2. Remember that the goal is performance improvement with a clear understanding of why your optimizations work

## References

1. Junyi Zhao, Kai Su, Yifei Yang, Xiangyao Yu, Paraschos Koutris, and Huanchen Zhang. "Debunking the Myth of Join Ordering: Toward Robust SQL Analytics." (2024).
2. Yiming Qiao and Huanchen Zhang. "Data Chunk Compaction in Vectorized Execution." (2024).
3. Harald Lang, Thomas Neumann, Alfons Kemper, Peter Boncz. "Performance-Optimal Filtering: Bloom Overtakes Cuckoo at High Throughput." (2019).
4. Maximilian Bandle, Jana Giceva, Thomas Neumann. "To Partition, or Not to Partition, That is the Join Question in a Real System." (2021).

## Acknowledgments

This assignment is based on [BabyDB](https://github.com/hehezhou/babydb) by [@hehezhou](https://github.com/hehezhou). Thanks for the excellent database framework and benchmark infrastructure.
