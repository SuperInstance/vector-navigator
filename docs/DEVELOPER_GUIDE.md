# Developer Guide

Guide for contributing to and developing vector-navigator.

## Table of Contents

1. [Development Setup](#development-setup)
2. [Project Structure](#project-structure)
3. [Testing Strategy](#testing-strategy)
4. [Benchmarking Methodology](#benchmarking-methodology)
5. [Contributing Guidelines](#contributing-guidelines)
6. [Release Process](#release-process)
7. [Code Style Guidelines](#code-style-guidelines)

## Development Setup

### Prerequisites

- Rust 1.70 or later
- Git
- Linux/macOS/WSL2 (Windows support via WSL2)

### Clone Repository

```bash
git clone https://github.com/equilibrium-tokens/vector-navigator.git
cd vector-navigator
```

### Install Development Tools

```bash
# Install Rust tools
rustup component add clippy rustfmt
cargo install cargo-watch
cargo install cargo-expand

# Install Python tools (for Python bindings)
cd bindings/python
pip install -r requirements-dev.txt
```

### Build Project

```bash
# Debug build (fast compilation)
cargo build

# Release build (optimized)
cargo build --release

# Run tests
cargo test

# Run clippy (linter)
cargo clippy -- -D warnings

# Format code
cargo fmt
```

### Development Workflow

```bash
# Watch for changes and rebuild
cargo watch -x build

# Watch and run tests
cargo watch -x test

# Watch and run clippy
cargo watch -x clippy
```

## Project Structure

```
vector-navigator/
├── Cargo.toml                 # Main package manifest
├── README.md                  # Project overview
├── LICENSE-APACHE             # Apache 2.0 license
├── LICENSE-MIT                # MIT license
│
├── docs/                      # Documentation
│   ├── ARCHITECTURE.md        # Architecture design
│   ├── USER_GUIDE.md          # User guide
│   ├── DEVELOPER_GUIDE.md     # This file
│   └── API.md                 # API reference
│
├── src/                       # Rust source code
│   ├── lib.rs                 # Library root
│   │
│   ├── store.rs               # VectorStore trait and impl
│   ├── index.rs               # Index traits and base types
│   ├── metric.rs              # SimilarityMetric trait and impls
│   ├── payload.rs             # Payload handling
│   ├── wal.rs                 # Write-ahead log
│   │
│   ├── index/                 # Index implementations
│   │   ├── mod.rs
│   │   ├── hnsw.rs            # HNSW index
│   │   ├── ivf.rs             # IVF index
│   │   ├── flat.rs            # Flat (exact) index
│   │   └── quantization.rs    # Product quantization
│   │
│   ├── metric/                # Metric implementations
│   │   ├── mod.rs
│   │   ├── cosine.rs
│   │   ├── euclidean.rs
│   │   └── dot_product.rs
│   │
│   └── types/                 # Core types
│       ├── mod.rs
│       ├── vector.rs          # Vector types
│       ├── id.rs              # ID types
│       └── result.rs          # Result types
│
├── benches/                   # Benchmarks
│   ├── cosine_sim.rs          # Cosine similarity benchmarks
│   ├── hnsw_search.rs         # HNSW search benchmarks
│   ├── ivf_search.rs          # IVF search benchmarks
│   └── insert_benchmark.rs    # Insert performance
│
├── tests/                     # Integration tests
│   ├── common/
│   │   └── mod.rs             # Test utilities
│   ├── test_store.rs          # Store tests
│   ├── test_hnsw.rs           # HNSW tests
│   ├── test_ivf.rs            # IVF tests
│   └── test_custom_metric.rs  # Custom metric tests
│
├── examples/                  # Example programs
│   ├── basic_search.rs        # Basic search example
│   ├── custom_metric.rs       # Custom metric example
│   ├── batch_insert.rs        # Batch insertion example
│   └── rag_system.rs          # RAG system example
│
└── bindings/                  # Language bindings
    ├── python/                # Python bindings
    │   ├── Cargo.toml         # Python package manifest
    │   ├── src/
    │   │   └── lib.rs         # PyO3 bindings
    │   ├── python/
    │   │   └── vector_navigator/
    │   │       ├── __init__.py
    │   │       └── types.py
    │   ├── tests/
    │   │   └── test_python.py
    │   └── setup.py
    │
    └── go/                    # Go bindings
        ├── vector_navigator.go
        ├── types.go
        └── tests/
            └── vector_navigator_test.go
```

### Key Modules

**store.rs**
- `VectorStore` trait
- `VectorStoreImpl` concrete implementation
- Batch operations
- Concurrency management

**index/hnsw.rs**
- `HNSWIndex` implementation
- Graph construction
- Layer management
- Neighbor selection

**index/ivf.rs**
- `IVFIndex` implementation
- Clustering (k-means)
- Inverted file management
- Quantization support

**metric.rs**
- `SimilarityMetric` trait
- Built-in metrics
- Context-aware metrics

## Testing Strategy

### Unit Tests

**Location:** Next to source code in `src/`

**Example:**

```rust
// src/metric/cosine.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_cosine_similarity_identical() {
        let metric = CosineMetric::new();
        let v1 = vec![1.0, 2.0, 3.0];
        let v2 = vec![1.0, 2.0, 3.0];

        let sim = metric.similarity(&v1, &v2);
        assert!((sim - 1.0).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_orthogonal() {
        let metric = CosineMetric::new();
        let v1 = vec![1.0, 0.0, 0.0];
        let v2 = vec![0.0, 1.0, 0.0];

        let sim = metric.similarity(&v1, &v2);
        assert!((sim - 0.0).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_opposite() {
        let metric = CosineMetric::new();
        let v1 = vec![1.0, 2.0, 3.0];
        let v2 = vec![-1.0, -2.0, -3.0];

        let sim = metric.similarity(&v1, &v2);
        assert!((sim - (-1.0)).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_magnitude_independence() {
        let metric = CosineMetric::new();
        let v1 = vec![1.0, 2.0, 3.0];
        let v2 = vec![2.0, 4.0, 6.0];  // v1 * 2
        let v3 = vec![0.5, 1.0, 1.5];  // v1 * 0.5

        let sim12 = metric.similarity(&v1, &v2);
        let sim13 = metric.similarity(&v1, &v3);
        let sim23 = metric.similarity(&v2, &v3);

        assert!((sim12 - 1.0).abs() < 1e-6);
        assert!((sim13 - 1.0).abs() < 1e-6);
        assert!((sim23 - 1.0).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_range() {
        let metric = CosineMetric::new();
        let v1 = vec![1.0, 2.0, 3.0];

        // Test with many random vectors
        for _ in 0..100 {
            let v2: Vec<f32> = (0..3).map(|_| rand::random()).collect();
            let sim = metric.similarity(&v1, &v2);
            assert!(sim >= -1.0 && sim <= 1.0, "similarity out of range: {}", sim);
        }
    }
}
```

**Run unit tests:**

```bash
# Run all unit tests
cargo test --lib

# Run specific test
cargo test test_cosine_similarity_identical

# Run with output
cargo test -- --nocapture

# Run tests in parallel
cargo test -- --test-threads=4
```

### Integration Tests

**Location:** `tests/`

**Example:**

```rust
// tests/test_store.rs

use vector_navigator::{VectorStore, CosineMetric, HNSWIndex};
use std::time::SystemTime;

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
struct TestPayload {
    id: String,
    data: String,
}

#[tokio::test]
async fn test_insert_search() {
    let mut store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build().unwrap()
    ).unwrap();

    // Insert test vector
    let vector = vec![0.1; 384];
    let payload = TestPayload {
        id: "test1".to_string(),
        data: "test data".to_string(),
    };

    let id = store.insert(vector.clone(), &payload).await.unwrap();

    // Search
    let results = store.search(&vector, 1).await.unwrap();

    assert_eq!(results.len(), 1);
    assert_eq!(results[0].id, id);
    assert_eq!(results[0].payload.id, "test1");
}

#[tokio::test]
async fn test_batch_insert() {
    let mut store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build().unwrap()
    ).unwrap();

    let vectors: Vec<Vec<f32>> = (0..100)
        .map(|_| (0..384).map(|_| rand::random()).collect())
        .collect();

    let payloads: Vec<TestPayload> = (0..100)
        .map(|i| TestPayload {
            id: format!("test{}", i),
            data: format!("data{}", i),
        })
        .collect();

    // Batch insert
    for (vec, payload) in vectors.iter().zip(payloads.iter()) {
        store.insert(vec.clone(), payload).await.unwrap();
    }

    // Verify count
    let count = store.count().await.unwrap();
    assert_eq!(count, 100);
}

#[tokio::test]
async fn test_delete() {
    let mut store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build().unwrap()
    ).unwrap();

    let vector = vec![0.1; 384];
    let payload = TestPayload {
        id: "test1".to_string(),
        data: "test data".to_string(),
    };

    let id = store.insert(vector.clone(), &payload).await.unwrap();

    // Delete
    store.delete(id).await.unwrap();

    // Verify deletion
    let count = store.count().await.unwrap();
    assert_eq!(count, 0);
}
```

**Run integration tests:**

```bash
# Run all integration tests
cargo test --test '*'

# Run specific test file
cargo test --test test_store

# Run specific test
cargo test test_insert_search
```

### Accuracy Tests

**Compare approximate index against exact (flat) search:**

```rust
// tests/test_accuracy.rs

use vector_navigator::{VectorStore, CosineMetric, HNSWIndex, FlatIndex};

fn calculate_recall<P: std::fmt::Debug + Clone>(
    approx_results: &[(usize, f32, P)],
    exact_results: &[(usize, f32, P)],
    k: usize,
) -> f32 {
    let exact_ids: std::collections::HashSet<usize> = exact_results
        .iter()
        .take(k)
        .map(|(id, _, _)| *id)
        .collect();

    let recall = approx_results
        .iter()
        .take(k)
        .filter(|(id, _, _)| exact_ids.contains(id))
        .count() as f32 / k as f32;

    recall
}

#[tokio::test]
async fn test_hnsw_recall() {
    let dimension = 384;
    let num_vectors = 10_000;

    // Generate random vectors
    let vectors: Vec<Vec<f32>> = (0..num_vectors)
        .map(|_| (0..dimension).map(|_| rand::random()).collect())
        .collect();

    // Build HNSW index
    let mut hnsw_store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder()
            .dimension(dimension)
            .ef_construction(200)
            .build()
            .unwrap()
    ).unwrap();

    for (i, vec) in vectors.iter().enumerate() {
        hnsw_store.insert(vec.clone(), &i).await.unwrap();
    }

    // Build Flat index (exact)
    let mut flat_store = VectorStore::new(
        CosineMetric::new(),
        FlatIndex::builder().dimension(dimension).build().unwrap()
    ).unwrap();

    for (i, vec) in vectors.iter().enumerate() {
        flat_store.insert(vec.clone(), &i).await.unwrap();
    }

    // Test queries
    let num_queries = 100;
    let mut total_recall = 0.0;

    for _ in 0..num_queries {
        let query: Vec<f32> = (0..dimension).map(|_| rand::random()).collect();

        let hnsw_results = hnsw_store.search(&query, 10).await.unwrap();
        let flat_results = flat_store.search(&query, 10).await.unwrap();

        let recall = calculate_recall(
            &hnsw_results.into_iter().map(|r| (r.id, r.score, ())).collect(),
            &flat_results.into_iter().map(|r| (r.id, r.score, ())).collect(),
            10
        );

        total_recall += recall;
    }

    let avg_recall = total_recall / num_queries as f32;

    println!("Average recall: {:.2}", avg_recall);
    assert!(avg_recall > 0.95, "Recall below 95%: {:.2}", avg_recall);
}
```

### Property-Based Tests

**Use proptest for randomized testing:**

```toml
[dev-dependencies]
proptest = "1.0"
```

```rust
// tests/test_properties.rs

use proptest::prelude::*;

proptest! {
    #[test]
    fn test_cosine_similarity_properties(
        v1 in prop::collection::vec(-1.0..1.0, 1..1000),
        v2 in prop::collection::vec(-1.0..1.0, 1..1000),
    ) {
        use vector_navigator::CosineMetric;

        // Ensure vectors have same length
        if v1.len() != v2.len() {
            return Ok(());
        }

        let metric = CosineMetric::new();
        let sim = metric.similarity(&v1, &v2);

        // Property 1: Range [-1, 1]
        prop_assert!(sim >= -1.0 && sim <= 1.0);

        // Property 2: Symmetry
        let sim_rev = metric.similarity(&v2, &v1);
        prop_assert!((sim - sim_rev).abs() < 1e-6);

        // Property 3: Reflexivity (if vectors are equal)
        if v1 == v2 {
            prop_assert!((sim - 1.0).abs() < 1e-6);
        }
    }
}
```

**Run property tests:**

```bash
cargo test test_properties -- --nocapture
```

## Benchmarking Methodology

### Benchmark Framework

**Use Criterion.rs for benchmarks:**

```toml
[dev-dependencies]
criterion = "0.5"
```

```rust
// benches/cosine_sim.rs

use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};
use vector_navigator::CosineMetric;

fn bench_cosine_similarity(c: &mut Criterion) {
    let metric = CosineMetric::new();

    let dimensions = vec![384, 768, 1536];

    for dim in dimensions {
        let v1: Vec<f32> = (0..dim).map(|i| i as f32 / dim as f32).collect();
        let v2: Vec<f32> = (0..dim).map(|i| (i as f32 + 0.5) / dim as f32).collect();

        c.bench_with_input(BenchmarkId::new("cosine_similarity", dim), &dim, |b, _| {
            b.iter(|| {
                metric.similarity(black_box(&v1), black_box(&v2))
            })
        });
    }
}

criterion_group!(benches, bench_cosine_similarity);
criterion_main!(benches);
```

**Run benchmarks:**

```bash
# Run all benchmarks
cargo bench

# Run specific benchmark
cargo bench cosine_similarity

# Generate plots
cargo bench -- --save-baseline main
cargo bench -- --baseline main
```

### Performance Benchmarks

**HNSW Search Benchmark:**

```rust
// benches/hnsw_search.rs

use criterion::{criterion_group, criterion_main, Criterion};
use vector_navigator::{VectorStore, CosineMetric, HNSWIndex};
use std::time::Duration;

fn bench_hnsw_search(c: &mut Criterion) {
    let dimension = 384;
    let num_vectors = 100_000;

    let mut group = c.benchmark_group("hnsw_search");
    group.measurement_time(Duration::from_secs(10));
    group.sample_size(100);

    // Build index
    let mut store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder()
            .dimension(dimension)
            .build()
            .unwrap()
    ).unwrap();

    let vectors: Vec<Vec<f32>> = (0..num_vectors)
        .map(|_| (0..dimension).map(|_| rand::random()).collect())
        .collect();

    for vec in &vectors {
        store.insert(vec.clone(), &()).await.unwrap();
    }

    // Benchmark search
    let query: Vec<f32> = (0..dimension).map(|_| rand::random()).collect();

    group.bench_function("search_10", |b| {
        b.iter(|| {
            black_box(store.search(black_box(&query), 10))
        })
    });

    group.bench_function("search_100", |b| {
        b.iter(|| {
            black_box(store.search(black_box(&query), 100))
        })
    });

    group.finish();
}

criterion_group!(benches, bench_hnsw_search);
criterion_main!(benches);
```

### Latency Percentiles

**Measure P50, P95, P99 latencies:**

```rust
// benches/latency.rs

use std::time::Instant;

fn measure_latency_percentiles<F>(mut f: F, iterations: usize) -> (f64, f64, f64)
where
    F: FnMut() -> (),
{
    let mut latencies: Vec<f64> = Vec::with_capacity(iterations);

    for _ in 0..iterations {
        let start = Instant::now();
        f();
        let elapsed = start.elapsed().as_secs_f64() * 1000.0; // Convert to ms
        latencies.push(elapsed);
    }

    latencies.sort_by(|a, b| a.partial_cmp(b).unwrap());

    let p50 = latencies[iterations / 2];
    let p95 = latencies[(iterations * 95) / 100];
    let p99 = latencies[(iterations * 99) / 100];

    (p50, p95, p99)
}

#[test]
fn test_search_latency() {
    let store = setup_store().await;

    let query = random_vector(384);

    let (p50, p95, p99) = measure_latency_percentiles(
        || {
            tokio::runtime::Runtime::new()
                .unwrap()
                .block_on(store.search(&query, 10))
                .unwrap();
        },
        1000
    );

    println!("P50: {:.2}ms, P95: {:.2}ms, P99: {:.2}ms", p50, p95, p99);

    assert!(p50 < 1.0, "P50 latency too high: {:.2}ms", p50);
    assert!(p95 < 5.0, "P95 latency too high: {:.2}ms", p95);
    assert!(p99 < 10.0, "P99 latency too high: {:.2}ms", p99);
}
```

### Memory Profiling

**Measure memory usage:**

```rust
// benches/memory.rs

use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

struct MemTracker;

static ALLOCATED: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for MemTracker {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let size = layout.size();
        ALLOCATED.fetch_add(size, Ordering::SeqCst);
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        let size = layout.size();
        ALLOCATED.fetch_sub(size, Ordering::SeqCst);
        System.dealloc(ptr, layout)
    }
}

#[global_allocator]
static GLOBAL: MemTracker = MemTracker;

#[test]
fn test_memory_usage() {
    let before = ALLOCATED.load(Ordering::SeqCst);

    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build().unwrap()
    ).unwrap();

    // Insert 100K vectors
    for _ in 0..100_000 {
        let vec = vec![0.1; 384];
        tokio::runtime::Runtime::new()
            .unwrap()
            .block_on(store.insert(vec, &()))
            .unwrap();
    }

    let after = ALLOCATED.load(Ordering::SeqCst);
    let memory_mb = (after - before) as f64 / 1024.0 / 1024.0;

    println!("Memory usage: {:.2} MB", memory_mb);

    assert!(memory_mb < 4000.0, "Memory usage too high: {:.2} MB", memory_mb);
}
```

## Contributing Guidelines

### Code Review Process

1. **Fork repository** and create feature branch
   ```bash
   git checkout -b feature/my-feature
   ```

2. **Make changes** and write tests
   ```bash
   # Write code
   # Write tests
   cargo test
   ```

3. **Format code**
   ```bash
   cargo fmt
   ```

4. **Run linter**
   ```bash
   cargo clippy -- -D warnings
   ```

5. **Commit changes**
   ```bash
   git add .
   git commit -m "Add my feature"
   ```

6. **Push to fork**
   ```bash
   git push origin feature/my-feature
   ```

7. **Create pull request** on GitHub

### Pull Request Checklist

- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Code formatted (`cargo fmt`)
- [ ] No clippy warnings (`cargo clippy`)
- [ ] All tests pass (`cargo test`)
- [ ] Benchmarks run (if performance changes)
- [ ] Commit messages follow conventions

### Commit Message Conventions

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Test additions/changes
- `chore`: Build process or tooling changes

**Example:**

```
feat(hnsw): add ef_search parameter tuning

Allow runtime tuning of ef_search parameter for better
latency/recall trade-offs.

Closes #123
```

### Code Review Guidelines

**For reviewers:**
1. Check code style and clarity
2. Verify tests are comprehensive
3. Ensure documentation is updated
4. Test the changes locally if possible
5. Provide constructive feedback

**For authors:**
1. Address all review comments
2. Update tests and documentation
3. Request re-review when ready
4. Be patient and respectful

## Release Process

### Version Bumping

**Semantic versioning: MAJOR.MINOR.PATCH**

- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes (backward compatible)

**Update version in:**

1. `Cargo.toml`
   ```toml
   [package]
   version = "0.2.0"
   ```

2. `bindings/python/setup.py`
   ```python
   version = "0.2.0"
   ```

3. `bindings/python/python/vector_navigator/__init__.py`
   ```python
   __version__ = "0.2.0"
   ```

### Release Checklist

1. **Update version numbers** (see above)

2. **Update CHANGELOG.md**
   ```markdown
   ## [0.2.0] - 2026-01-15

   ### Added
   - IVF index implementation
   - Custom similarity metrics API
   - Python bindings

   ### Changed
   - Improved HNSW build performance by 30%

   ### Fixed
   - Memory leak in WAL rotation
   ```

3. **Run full test suite**
   ```bash
   cargo test --all
   ```

4. **Run benchmarks**
   ```bash
   cargo bench
   ```

5. **Create git tag**
   ```bash
   git tag -a v0.2.0 -m "Release v0.2.0"
   git push origin v0.2.0
   ```

6. **Publish to crates.io**
   ```bash
   cargo publish
   ```

7. **Publish Python package**
   ```bash
   cd bindings/python
   python setup.py sdist bdist_wheel
   twine upload dist/*
   ```

8. **Create GitHub release**
   - Go to GitHub Releases
   - Click "Draft a new release"
   - Choose tag (e.g., v0.2.0)
   - Add release notes from CHANGELOG
   - Publish release

### Release Notes Template

```markdown
# vector-navigator v0.2.0

## Highlights

- IVF index for large-scale datasets (>1M vectors)
- Custom similarity metrics for domain-specific search
- Python bindings for easy integration

## What's Changed

### Added
- IVF index implementation by @author
- Custom similarity metrics API by @author
- Python bindings via PyO3 by @author

### Changed
- Improved HNSW build performance by 30%
- Reduced memory usage by 20%

### Fixed
- Memory leak in WAL rotation (#42)
- Race condition in concurrent search (#38)

## Migration Guide

If upgrading from v0.1.0:

1. Update Cargo.toml: `vector-navigator = "0.2"`
2. No breaking changes for basic usage
3. Custom metrics now require explicit trait implementation

## Downloads

- [Cargo](https://crates.io/crates/vector-navigator/0.2.0)
- [Python](https://pypi.org/project/vector-navigator/0.2.0/)
- [GitHub](https://github.com/equilibrium-tokens/vector-navigator/releases/tag/v0.2.0)

## Contributors

@contributor1 @contributor2 @contributor3
```

## Code Style Guidelines

### Rust Style

**Follow official Rust guidelines:**
- Use `rustfmt` for formatting
- Follow [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- Use `clippy` for linting

**Naming conventions:**

```rust
// Types: PascalCase
struct VectorStore { }
enum IndexType { }
trait SimilarityMetric { }

// Functions: snake_case
fn search_vector() { }
async fn insert_vector() { }

// Constants: SCREAMING_SNAKE_CASE
const MAX_VECTORS: usize = 1_000_000;

// Generics: T, U, etc. (descriptive names preferred)
fn insert<P: Payload>(payload: P) { }
fn build_index<M: SimilarityMetric>(metric: M) { }
```

**Error handling:**

```rust
// Use Result for fallible operations
pub fn search(&self, query: &[f32]) -> Result<Vec<SearchResult>, SearchError> {
    if query.len() != self.dimension {
        return Err(SearchError::InvalidDimension {
            expected: self.dimension,
            got: query.len(),
        });
    }
    // ...
}

// Define custom error types
#[derive(Debug, thiserror::Error)]
pub enum SearchError {
    #[error("Invalid dimension: expected {expected}, got {got}")]
    InvalidDimension { expected: usize, got: usize },

    #[error("Index not built")]
    IndexNotBuilt,

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

**Documentation:**

```rust
/// Search for k nearest neighbors to the query vector.
///
/// # Arguments
///
/// * `query` - The query vector to search for
/// * `k` - Number of nearest neighbors to return
///
/// # Returns
///
/// A vector of search results sorted by similarity (highest first).
///
/// # Errors
///
/// Returns `SearchError::InvalidDimension` if query dimension doesn't match
/// the index dimension.
///
/// # Examples
///
/// ```rust
/// let results = store.search(&query, 10).await?;
/// for result in results {
///     println!("Score: {:.2}", result.score);
/// }
/// ```
///
/// # Performance
///
/// - P50 latency: <1ms
/// - P95 latency: <5ms
/// - P99 latency: <10ms
pub async fn search(
    &self,
    query: &[f32],
    k: usize,
) -> Result<Vec<SearchResult<P>>, SearchError> {
    // ...
}
```

### Testing Style

**Test naming:**

```rust
#[test]
fn test_cosine_similarity_identical() {
    // test_<function>_<scenario>
}

#[tokio::test]
async fn test_insert_search_delete_flow() {
    // test_<workflow>
}

#[test]
#[should_panic(expected = "dimension mismatch")]
fn test_search_invalid_dimension() {
    // test_<error_condition>
}
```

**Test structure:**

```rust
#[test]
fn test_feature_x() {
    // Arrange
    let input = setup_test_data();

    // Act
    let result = function_under_test(input);

    // Assert
    assert_eq!(result.expected, result.actual);
}
```

### Documentation Style

**README.md:**
- Start with one-line description
- Include quick start example
- Link to full documentation
- Use badges for status

**API docs:**
- Document all public APIs
- Include examples
- Explain performance characteristics
- Note panics and errors

**Architecture docs:**
- Explain design decisions
- Include diagrams
- Discuss trade-offs
- Provide rationale

---

**Thank you for contributing to vector-navigator! The grammar is eternal.**
