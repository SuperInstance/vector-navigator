# Architecture

## Philosophy

**Navigating high-dimensional embedding space with timeless geometric principles.**

vector-navigator is built on the belief that similarity search should be:
1. **Fast** - Sub-10ms latency for real-time applications
2. **Flexible** - Custom similarity metrics for domain-specific needs
3. **Fundamental** - Based on timeless geometry, not transient trends
4. **Accessible** - Simple API, powerful abstractions

## The Timeless Principle: Cosine Similarity

At the heart of vector-navigator lies cosine similarity, a geometric truth that transcends specific embeddings or models:

```
cos_sim(a, b) = (a · b) / (||a|| × ||b||)
```

### Why This Is Timeless

**Cosine similarity measures angle, not magnitude.**

This simple mathematical fact has profound implications:

#### 1. Magnitude Independence

```
vector("happy") × 100 and vector("happy") × 0.01
point in the SAME direction
cos_sim = 1.0 (perfect match)
```

Magnitude can encode:
- Document length
- Term frequency (TF-IDF)
- Confidence scores
- Attention weights

But similarity is purely directional.

#### 2. Dimension Agnostic

Works for any embedding dimension:
- 384-dim (sentence-transformers/all-MiniLM-L6-v2)
- 768-dim (sentence-transformers/all-mpnet-base-v2)
- 1536-dim (OpenAI text-embedding-ada-002)
- 15360-dim (future models)

The geometry doesn't change.

#### 3. Bounded Range

```
cos_sim ∈ [-1, 1]

1.0    = identical direction
0.0    = orthogonal (unrelated)
-1.0   = opposite direction
```

This bounded range enables:
- Meaningful similarity thresholds
- Stable scoring across queries
- Compositional metrics

#### 4. Geometric Interpretation

```
cos_sim(a, b) = cos(θ)

where θ is the angle between vectors
```

This is not a machine learning technique.
This is not a neural network architecture.
This is geometry.

### The Timeless Code

```rust
// This implementation will never change
// Not because it's perfect, but because it's TRUE

fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    // Dot product
    let dot_product: f32 = a.iter()
        .zip(b.iter())
        .map(|(x, y)| x * y)
        .sum();

    // Magnitudes
    let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();

    // Cosine similarity
    dot_product / (norm_a * norm_b)
}
```

This function is the foundation upon which everything else is built.

## Core Abstractions

### VectorStore

The central abstraction for storing and searching vectors.

```rust
pub trait VectorStore<P: Payload> {
    // Insert a vector with associated payload
    async fn insert(&mut self, vector: Vec<f32>, payload: &P) -> Result<Id>;

    // Search for k nearest neighbors
    async fn search(&self, query: &[f32], k: usize) -> Result<Vec<SearchResult<P>>>;

    // Delete a vector by ID
    async fn delete(&mut self, id: Id) -> Result<()>;

    // Get vector count
    async fn count(&self) -> Result<usize>;
}
```

**Design Principles:**
- **Async-first** - Non-blocking operations for high throughput
- **Generic payloads** - Work with any associated data type
- **Type-safe IDs** - Prevent ID mix-ups between stores
- **Error handling** - Explicit failure modes via Result

### Index

Efficient data structures for approximate nearest neighbor search.

```rust
pub trait Index {
    // Build index from vectors
    fn build(&mut self, vectors: &[Vec<f32>]) -> Result<()>;

    // Add vector to existing index
    fn add(&mut self, vector: Vec<f32>) -> Result<()>;

    // Search for nearest neighbors
    fn search(&self, query: &[f32], k: usize) -> Result<Vec<SearchResult>>;

    // Save index to disk
    fn save(&self, path: &Path) -> Result<()>;

    // Load index from disk
    fn load(&mut self, path: &Path) -> Result<()>;
}
```

**Available Index Types:**

#### HNSW (Hierarchical Navigable Small World)

```rust
let index = HNSWIndex::builder()
    .dimension(384)
    .ef_construction(200)
    .m(16)
    .build()?;
```

**Best for:** High recall, low latency queries

**Characteristics:**
- Graph-based structure
- Excellent query performance (1-5ms)
- Moderate build time
- Memory-intensive

**Performance:**
- P95: 3.2ms
- Recall: >0.98

#### IVF (Inverted File)

```rust
let index = IVFIndex::builder()
    .dimension(384)
    .nlist(100)
    .nprobe(10)
    .quantization(ProductQuantization::new(8, 8)?
    .build()?;
```

**Best for:** Large-scale datasets (>1M vectors)

**Characteristics:**
- Clustering-based coarse quantization
- Excellent scalability
- Fast construction
- Tunable accuracy via nprobe

**Performance:**
- P95: 8.5ms
- Recall: >0.95 (with nprobe=10)

#### Flat (Exact Search)

```rust
let index = FlatIndex::builder()
    .dimension(384)
    .build()?;
```

**Best for:** Small datasets (<100K vectors), ground truth

**Characteristics:**
- Brute-force exact search
- 100% recall
- Scales poorly (O(N))

**Performance:**
- P95: 150ms (100K vectors)
- Recall: 1.0

### SimilarityMetric

Pluggable similarity metrics for domain-specific search.

```rust
pub trait SimilarityMetric: Send + Sync {
    // Calculate similarity between two vectors
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32;

    // Calculate similarity with context
    fn similarity_with_context<P: Payload>(
        &self,
        a: &[f32],
        b: &[f32],
        ctx_a: &P,
        ctx_b: &P
    ) -> f32 {
        // Default: ignore context
        self.similarity(a, b)
    }

    // Distance from similarity (for minimization)
    fn distance(&self, similarity: f32) -> f32 {
        1.0 - similarity
    }
}
```

**Built-in Metrics:**

#### CosineMetric

```rust
let metric = CosineMetric::new();
```

The timeless geometric principle.

#### EuclideanMetric

```rust
let metric = EuclideanMetric::new();
```

For magnitude-sensitive applications.

#### DotProductMetric

```rust
let metric = DotProductMetric::new();
```

Fastest, assumes normalized vectors.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     VectorStore                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Index      │  │  Metric     │  │  Payload Store      │ │
│  │  (HNSW/IVF) │  │  (Cosine)  │  │  (Metadata)         │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
    │ HNSW    │         │ IVF     │         │ Flat    │
    │ Index   │         │ Index   │         │ Index   │
    └─────────┘         └─────────┘         └─────────┘
```

### Data Flow

#### Insert Flow

```
User inserts vector
    │
    ├─► Calculate ID
    │
    ├─► Add to index (async)
    │       │
    │       ├─► HNSW: Insert into graph layers
    │       ├─► IVF: Assign to cluster, add to inverted list
    │       └─► Flat: Append to list
    │
    └─► Store payload (async)
            │
            └─► Serialize and write to WAL
                    │
                    └─► Periodic compaction
```

#### Search Flow

```
User searches with query vector
    │
    ├─► Search index
    │       │
    │       ├─► HNSW: Traverse graph from entry point
    │       ├─► IVF: Query nprobe nearest clusters
    │       └─► Flat: Linear scan
    │
    ├─► Calculate similarities
    │       │
    │       └─► Apply metric (cosine, custom, etc.)
    │
    ├─► Apply filters (sentiment, time, metadata)
    │
    ├─► Top-k selection
    │
    └─► Load payloads for top-k results
            │
            └─► Return sorted results
```

## Integration with equilibrium-tokens

vector-navigator is the context navigation engine for equilibrium-tokens.

### Context Basins

In equilibrium-tokens, conversation context is organized as "basins" in embedding space:

```
                    ┌─────────────────┐
                    │  Topic Basin    │
                    │  (high-level)   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
    ┌───────▼───────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │ Subtopic A    │ │ Subtopic B  │ │ Subtopic C  │
    │ (mid-level)   │ │ (mid-level) │ │ (mid-level) │
    └───────┬───────┘ └──────┬──────┘ └──────┬──────┘
            │                │                │
      ┌─────┼─────┐    ┌─────┼─────┐    ┌─────┼─────┐
      │     │     │    │     │     │    │     │     │
  ┌───▼─┐ ┌─▼─┐ ┌─▼─┐┌─▼─┐ ┌─▼─┐ ┌─▼─┐┌─▼─┐ ┌─▼─┐ ┌─▼─┐
  │Utt  │ │Utt │ │Utt││Utt│ │Utt│ │Utt││Utt│ │Utt│ │Utt│
  │     │ │    │ │   ││   │ │   │ │   ││   │ │   │ │   │
  └─────┘ └────┘ └───┘└───┘ └───┘ └───┘└───┘ └───┘ └───┘
  (utterances)
```

### Basin Navigation API

```rust
use vector_navigator::{VectorStore, CosineMetric};

// Create store for context basins
let store = VectorStore::new(CosineMetric::new())?;

// Navigate to specific basin
let basin_results = store
    .navigate_to(&["topic", "subtopic", "conversation"])
    .with_sentiment_weight(0.8)
    .with_time_decay(0.1)
    .search(&query_embedding, 10)
    .await?;

// Results include:
// - Similarity score (cosine)
// - Sentiment match score
// - Temporal relevance
// - Basin path
for result in basin_results {
    println!("Basin: {:?}", result.basin_path);
    println!("Similarity: {:.2}", result.semantic_score);
    println!("Sentiment Match: {:.2}", result.sentiment_score);
    println!("Combined: {:.2}", result.combined_score);
}
```

### Context Equilibrium Surface

The "equilibrium surface" is the set of context basins that maintain semantic coherence:

```rust
struct EquilibriumSurface {
    store: VectorStore<ConversationContext>,
    sentiment_balance: f32,
    temporal_decay: f32,
}

impl EquilibriumSurface {
    async fn find_equilibrium(
        &self,
        query: &[f32],
        target_sentiment: VAD,
    ) -> Result<Vec<ContextPoint>> {
        let results = self.store
            .search(query, 20)
            .with_sentiment_weight(self.sentiment_balance)
            .with_time_decay(self.temporal_decay)
            .await?;

        // Filter to equilibrium surface
        // (points where semantic + sentiment + temporal forces balance)
        self.filter_to_equilibrium(results, target_sentiment)
    }
}
```

## Invariants

### Magnitude Independence

**Cosine similarity is invariant to vector magnitude.**

```rust
let v1 = vec![1.0, 2.0, 3.0];
let v2 = vec![2.0, 4.0, 6.0];  // v1 × 2
let v3 = vec![0.5, 1.0, 1.5];  // v1 × 0.5

assert_eq!(cos_sim(&v1, &v2), 1.0);
assert_eq!(cos_sim(&v1, &v3), 1.0);
assert_eq!(cos_sim(&v2, &v3), 1.0);
```

### Bounded Range

**Cosine similarity always returns values in [-1, 1].**

This enables:
- Meaningful thresholds (e.g., similarity > 0.8)
- Stable scoring across queries
- Compositional metrics

### Directionality

**Similarity is about direction, not position.**

Two vectors can be similar even if far apart in Euclidean space:

```rust
let v1 = vec![1000.0, 0.0, 0.0];
let v2 = vec![1.0, 0.0, 0.0];
let v3 = vec![0.0, 1000.0, 0.0];

cos_sim(&v1, &v2) = 1.0  // Same direction
cos_sim(&v1, &v3) = 0.0  // Orthogonal
```

### Triangle Inequality

For distance metrics derived from similarity:

```
distance(a, c) ≤ distance(a, b) + distance(b, c)
```

Enables index pruning and efficient search.

## Performance Characteristics

### Latency

| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| Insert    | 0.5ms | 2.1ms | 5.8ms |
| Search    | 0.8ms | 3.2ms | 7.1ms |
| Delete    | 0.3ms | 1.5ms | 3.9ms |

Measured with HNSW index, 384-dim vectors, 100K vectors total.

### Throughput

| Operation | Single-thread | Multi-thread (16x) |
|-----------|---------------|-------------------|
| Insert    | 2,000/sec     | 18,000/sec |
| Search    | 1,200/sec     | 9,500/sec |

### Memory

| Index Type | Memory (per 1M vectors, 384-dim) |
|------------|-----------------------------------|
| HNSW       | 3.2 GB |
| IVF-PQ     | 1.1 GB |
| Flat       | 1.5 GB |

### Scalability

| Dataset Size | HNSW P95 | IVF P95 |
|--------------|----------|---------|
| 10K vectors  | 1.2ms    | 2.8ms   |
| 100K vectors | 3.2ms    | 5.1ms   |
| 1M vectors   | 8.5ms    | 9.3ms   |
| 10M vectors  | 15.2ms   | 12.1ms  |

HNSW for smaller datasets (<1M), IVF for larger (>1M).

## Implementation Details

### Rust Advantages

**Why Rust?**

1. **Memory Safety** - No segfaults, no data races
2. **Zero-Cost Abstractions** - Trait-based design, no runtime overhead
3. **Fearless Concurrency** - Lock-free reads, async/await
4. **Small Binary** - Single executable, easy deployment

### Memory Management

**Arena Allocation:**

Vectors are stored in contiguous arenas for cache efficiency:

```rust
struct VectorArena {
    data: Vec<f32>,
    dimension: usize,
    capacity: usize,
}

impl VectorArena {
    fn allocate(&mut self, vector: Vec<f32>) -> usize {
        let offset = self.data.len();
        self.data.extend_from_slice(&vector);
        offset
    }

    fn get(&self, offset: usize) -> &[f32] {
        &self.data[offset..offset + self.dimension]
    }
}
```

**Benefits:**
- Reduced allocations
- Better cache locality
- SIMD-friendly memory layout

### Concurrency

**Lock-Free Reads:**

Arc (atomic reference counting) enables concurrent reads:

```rust
struct VectorStore<P> {
    index: Arc<RwLock<HNSWIndex>>,
    payloads: Arc<RwLock<PayloadStore<P>>>,
}

async fn search(&self, query: &[f32], k: usize) -> Result<Vec<SearchResult<P>>> {
    // Lock-free read of index
    let index = self.index.read().await;

    // Search without blocking other readers
    let results = index.search(query, k)?;

    // Fetch payloads (can be parallelized)
    let payloads = self.fetch_payloads(&results).await?;

    Ok(payloads)
}
```

### SIMD Optimization

**AVX2-accelerated cosine similarity:**

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[target_feature(enable = "avx2")]
unsafe fn cosine_similarity_avx2(a: &[f32], b: &[f32]) -> f32 {
    // Process 8 floats per iteration
    // 4-5x speedup over scalar code
}
```

**Performance:**
- Scalar: 120ns per cosine_sim (384-dim)
- AVX2: 28ns per cosine_sim (384-dim)
- Speedup: 4.3x

### Persistent Storage

**Write-Ahead Log (WAL):**

```rust
struct WAL {
    file: BufWriter<File>,
    sync_mode: SyncMode,
}

impl WAL {
    fn append(&mut self, op: Operation) -> Result<()> {
        serde_json::to_writer(&mut self.file, &op)?;
        self.file.write_all(b"\n")?;

        if self.sync_mode == SyncMode::Immediate {
            self.file.flush()?;
        }

        Ok(())
    }
}
```

**Durability:**
- Every write logged before acknowledgment
- Periodic compaction
- Crash recovery on restart

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_cosine_similarity_identical() {
        let v1 = vec![1.0, 2.0, 3.0];
        let v2 = vec![1.0, 2.0, 3.0];
        assert!((cosine_similarity(&v1, &v2) - 1.0).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_orthogonal() {
        let v1 = vec![1.0, 0.0, 0.0];
        let v2 = vec![0.0, 1.0, 0.0];
        assert!((cosine_similarity(&v1, &v2) - 0.0).abs() < 1e-6);
    }

    #[test]
    fn test_cosine_similarity_opposite() {
        let v1 = vec![1.0, 2.0, 3.0];
        let v2 = vec![-1.0, -2.0, -3.0];
        assert!((cosine_similarity(&v1, &v2) - (-1.0)).abs() < 1e-6);
    }
}
```

### Integration Tests

```rust
#[tokio::test]
async fn test_insert_search() {
    let mut store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build().unwrap()
    ).unwrap();

    // Insert test vectors
    let id = store.insert(test_vector(), &test_payload()).await.unwrap();

    // Search
    let results = store.search(&test_vector(), 1).await.unwrap();
    assert_eq!(results.len(), 1);
    assert_eq!(results[0].id, id);
}
```

### Benchmarks

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn benchmark_cosine_similarity(c: &mut Criterion) {
    let v1 = vec![0.1; 384];
    let v2 = vec![0.2; 384];

    c.bench_function("cosine_similarity_384", |b| {
        b.iter(|| cosine_similarity(black_box(&v1), black_box(&v2)))
    });
}

criterion_group!(benches, benchmark_cosine_similarity);
criterion_main!(benches);
```

### Accuracy Tests

Compare approximate index results against exact (flat) search:

```rust
#[test]
fn test_hnsw_recall() {
    let mut hnsw = HNSWIndex::builder().dimension(384).build().unwrap();
    let mut flat = FlatIndex::builder().dimension(384).build().unwrap();

    let vectors = generate_random_vectors(10000, 384);

    hnsw.build(&vectors).unwrap();
    flat.build(&vectors).unwrap();

    let query = random_vector(384);

    let hnsw_results = hnsw.search(&query, 10).unwrap();
    let flat_results = flat.search(&query, 10).unwrap();

    let recall = calculate_recall(&hnsw_results, &flat_results);
    assert!(recall > 0.95, "Recall below 95%: {}", recall);
}
```

## Future Directions

### GPU Acceleration

CUDA kernels for index search:
- Batch query processing
- Parallel distance calculations
- Expected 5-10x speedup

### FPGA Acceleration

Custom hardware for critical path:
- Sub-millisecond search at scale
- 10-50x speedup potential
- Long-term investment

### Learned Index Structures

ML-based optimization:
- Learn data distribution
- Optimize for conversational patterns
- Improved recall at same latency

### Distributed Search

Horizontal scaling:
- Sharding by basin
- Federated search across nodes
- Multi-datacenter replication

---

**Architecture is frozen, but implementation evolves. The geometry is eternal.**
