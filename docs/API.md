# API Reference

Complete API reference for vector-navigator.

## Table of Contents

1. [Core Traits](#core-traits)
2. [VectorStore](#vectorstore)
3. [Index Types](#index-types)
4. [Similarity Metrics](#similarity-metrics)
5. [Configuration](#configuration)
6. [Types](#types)
7. [Examples](#examples)

## Core Traits

### VectorStore

The main abstraction for storing and searching vectors.

```rust
pub trait VectorStore<P: Payload>: Send + Sync {
    /// Insert a vector with associated payload
    async fn insert(&mut self, vector: Vec<f32>, payload: &P) -> Result<Id, StoreError>;

    /// Search for k nearest neighbors
    async fn search(&self, query: &[f32], k: usize) -> Result<Vec<SearchResult<P>>, StoreError>;

    /// Delete a vector by ID
    async fn delete(&mut self, id: Id) -> Result<(), StoreError>;

    /// Get total vector count
    async fn count(&self) -> Result<usize, StoreError>;

    /// Clear all vectors
    async fn clear(&mut self) -> Result<(), StoreError>;
}
```

**Methods:**

#### `insert`

```rust
async fn insert(&mut self, vector: Vec<f32>, payload: &P) -> Result<Id, StoreError>
```

Insert a vector with associated payload into the store.

**Parameters:**
- `vector: Vec<f32>` - The vector to insert (must match index dimension)
- `payload: &P` - Associated metadata/data

**Returns:**
- `Result<Id, StoreError>` - Unique identifier for inserted vector

**Errors:**
- `StoreError::InvalidDimension` - Vector dimension doesn't match index
- `StoreError::IndexFull` - Index at capacity
- `StoreError::SerializationError` - Failed to serialize payload

**Example:**

```rust
let vector = vec![0.1, 0.2, 0.3, /* ... */];
let payload = Document {
    id: "doc1".to_string(),
    text: "Example document".to_string(),
};

let id = store.insert(vector, &payload).await?;
```

#### `search`

```rust
async fn search(&self, query: &[f32], k: usize) -> Result<Vec<SearchResult<P>>, StoreError>
```

Search for k nearest neighbors to the query vector.

**Parameters:**
- `query: &[f32]` - Query vector (must match index dimension)
- `k: usize` - Number of results to return

**Returns:**
- `Result<Vec<SearchResult<P>>, StoreError>` - Vector of search results, sorted by similarity (highest first)

**Errors:**
- `StoreError::InvalidDimension` - Query dimension doesn't match index
- `StoreError::IndexNotBuilt` - Index hasn't been built yet
- `StoreError::SearchFailed` - Search operation failed

**Example:**

```rust
let query = vec![0.15, 0.25, 0.35, /* ... */];
let results = store.search(&query, 10).await?;

for result in results {
    println!("Score: {:.4}", result.score);
    println!("Text: {}", result.payload.text);
}
```

#### `delete`

```rust
async fn delete(&mut self, id: Id) -> Result<(), StoreError>
```

Delete a vector from the store by ID.

**Parameters:**
- `id: Id` - Unique identifier of vector to delete

**Returns:**
- `Result<(), StoreError>` - Success or error

**Errors:**
- `StoreError::NotFound` - ID not found in store
- `StoreError::DeleteFailed` - Deletion operation failed

**Example:**

```rust
store.delete(id).await?;
```

#### `count`

```rust
async fn count(&self) -> Result<usize, StoreError>
```

Get total number of vectors in the store.

**Returns:**
- `Result<usize, StoreError>` - Vector count

**Example:**

```rust
let count = store.count().await?;
println!("Total vectors: {}", count);
```

#### `clear`

```rust
async fn clear(&mut self) -> Result<(), StoreError>
```

Remove all vectors from the store.

**Returns:**
- `Result<(), StoreError>` - Success or error

**Example:**

```rust
store.clear().await?;
```

### SimilarityMetric

Trait for custom similarity metrics.

```rust
pub trait SimilarityMetric: Send + Sync {
    /// Calculate similarity between two vectors
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32;

    /// Calculate similarity with context (optional)
    fn similarity_with_context<P: Payload>(
        &self,
        a: &[f32],
        b: &[f32],
        ctx_a: &P,
        ctx_b: &P,
    ) -> f32 {
        // Default: ignore context
        self.similarity(a, b)
    }

    /// Convert similarity to distance (for minimization)
    fn distance(&self, similarity: f32) -> f32 {
        1.0 - similarity
    }
}
```

**Methods:**

#### `similarity`

```rust
fn similarity(&self, a: &[f32], b: &[f32]) -> f32
```

Calculate similarity between two vectors.

**Parameters:**
- `a: &[f32]` - First vector
- `b: &[f32]` - Second vector (same length as a)

**Returns:**
- `f32` - Similarity score (typically in [-1, 1] or [0, 1])

**Example:**

```rust
struct CustomMetric;

impl SimilarityMetric for CustomMetric {
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32 {
        // Custom similarity calculation
        let dot_product: f32 = a.iter()
            .zip(b.iter())
            .map(|(x, y)| x * y)
            .sum();
        dot_product
    }
}
```

#### `similarity_with_context`

```rust
fn similarity_with_context<P: Payload>(
    &self,
    a: &[f32],
    b: &[f32],
    ctx_a: &P,
    ctx_b: &P,
) -> f32
```

Calculate similarity with additional context (payload information).

**Parameters:**
- `a: &[f32]` - First vector
- `b: &[f32]` - Second vector
- `ctx_a: &P` - Context for first vector
- `ctx_b: &P` - Context for second vector

**Returns:**
- `f32` - Similarity score incorporating context

**Example:**

```rust
impl SimilarityMetric for SentimentMetric {
    fn similarity_with_context<P: Payload>(
        &self,
        a: &[f32],
        b: &[f32],
        ctx_a: &P,
        ctx_b: &P,
    ) -> f32 {
        let semantic = self.similarity(a, b);

        let sentiment_a = ctx_a.sentiment();
        let sentiment_b = ctx_b.sentiment();

        let sentiment_match = cosine_similarity_3d(
            sentiment_a.valence, sentiment_a.arousal, sentiment_a.dominance,
            sentiment_b.valence, sentiment_b.arousal, sentiment_b.dominance,
        );

        // Combine semantic and sentiment
        0.7 * semantic + 0.3 * sentiment_match
    }
}
```

## VectorStore

Concrete implementation of VectorStore trait.

### Constructor

```rust
pub fn new<P: Payload>(
    metric: impl SimilarityMetric + 'static,
    index: impl Index + 'static,
) -> Result<VectorStoreImpl<P>, StoreError>
```

Create a new vector store.

**Parameters:**
- `metric: impl SimilarityMetric` - Similarity metric to use
- `index: impl Index` - Index implementation

**Returns:**
- `Result<VectorStoreImpl<P>, StoreError>` - New vector store or error

**Example:**

```rust
let store = VectorStoreImpl::new(
    CosineMetric::new(),
    HNSWIndex::builder().dimension(384).build()?
)?;
```

### Builder Pattern

```rust
let store = VectorStoreImpl::builder()
    .metric(CosineMetric::new())
    .index(HNSWIndex::builder().dimension(384).build()?)
    .max_vectors(1_000_000)
    .wal_config(WALConfig::default())
    .build()?;
```

### Search Options

```rust
let results = store
    .search(&query, 10)
    .with_sentiment_weight(0.8)
    .with_time_decay(0.1)
    .await?;
```

## Index Types

### HNSWIndex

Hierarchical Navigable Small World index for high-performance approximate nearest neighbor search.

**Builder:**

```rust
let index = HNSWIndex::builder()
    .dimension(384)              // Required: vector dimension
    .m(16)                       // Optional: connections per node (default: 16)
    .ef_construction(200)        // Optional: build-time accuracy (default: 200)
    .ef_search(100)              // Optional: search-time accuracy (default: 100)
    .build()?;
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `dimension` | `usize` | **Required** | Vector dimension |
| `m` | `usize` | 16 | Number of bi-directional links for each node |
| `ef_construction` | `usize` | 200 | Size of dynamic candidate list for construction |
| `ef_search` | `usize` | 100 | Size of dynamic candidate list for search |

**Performance Characteristics:**

- **Build time:** O(N log N) with moderate constant
- **Search time:** O(log N) average
- **Memory:** O(N × m × dimension)
- **Recall:** >0.98 with proper tuning

**When to use:**
- Dataset size < 1M vectors
- Low latency critical (<5ms)
- High recall required (>0.95)

**Example:**

```rust
let index = HNSWIndex::builder()
    .dimension(384)
    .m(16)
    .ef_construction(200)
    .ef_search(100)
    .build()?;

let store = VectorStoreImpl::new(CosineMetric::new(), index)?;
```

### IVFIndex

Inverted File index with product quantization for large-scale vector search.

**Builder:**

```rust
use vector_navigator::ProductQuantization;

let index = IVFIndex::builder()
    .dimension(384)              // Required: vector dimension
    .nlist(100)                  // Optional: number of clusters (default: √N)
    .nprobe(10)                  // Optional: clusters to query (default: 10)
    .quantization(ProductQuantization::new(8, 8)?)  // Optional: PQ
    .build()?;
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `dimension` | `usize` | **Required** | Vector dimension |
| `nlist` | `usize` | √N | Number of clusters (Voronoi cells) |
| `nprobe` | `usize` | 10 | Number of clusters to search |
| `quantization` | `Option<PQ>` | None | Product quantization config |

**Performance Characteristics:**

- **Build time:** O(N × nlist × dimension)
- **Search time:** O(nprobe × N/nlist × dimension)
- **Memory:** O(N × dimension / compression_ratio)
- **Recall:** >0.95 with nprobe=10

**When to use:**
- Dataset size > 1M vectors
- Memory-constrained environments
- Fast index construction required

**Example:**

```rust
let index = IVFIndex::builder()
    .dimension(384)
    .nlist(100)
    .nprobe(10)
    .quantization(ProductQuantization::new(8, 8)?)  // 8x compression
    .build()?;

let store = VectorStoreImpl::new(CosineMetric::new(), index)?;
```

### FlatIndex

Exact (brute-force) search for ground truth and small datasets.

**Builder:**

```rust
let index = FlatIndex::builder()
    .dimension(384)              // Required: vector dimension
    .build()?;
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `dimension` | `usize` | **Required** | Vector dimension |

**Performance Characteristics:**

- **Build time:** O(N)
- **Search time:** O(N × dimension)
- **Memory:** O(N × dimension)
- **Recall:** 1.0 (exact)

**When to use:**
- Dataset size < 100K vectors
- 100% recall required
- Benchmarking ground truth

**Example:**

```rust
let index = FlatIndex::builder()
    .dimension(384)
    .build()?;

let store = VectorStoreImpl::new(CosineMetric::new(), index)?;
```

## Similarity Metrics

### CosineMetric

Cosine similarity: measures angle between vectors (magnitude-independent).

```rust
let metric = CosineMetric::new();
```

**Formula:**

```
cos_sim(a, b) = (a · b) / (||a|| × ||b||)
```

**Range:** [-1, 1]
- 1.0 = identical direction
- 0.0 = orthogonal
- -1.0 = opposite direction

**Example:**

```rust
let metric = CosineMetric::new();
let similarity = metric.similarity(&vec1, &vec2);
```

### EuclideanMetric

Euclidean distance: measures straight-line distance between vectors.

```rust
let metric = EuclideanMetric::new();
```

**Formula:**

```
euclidean_dist(a, b) = sqrt(Σ(a[i] - b[i])²)
```

**Range:** [0, ∞)
- 0 = identical
- Larger values = more distant

**Example:**

```rust
let metric = EuclideanMetric::new();
let similarity = metric.similarity(&vec1, &vec2);
```

### DotProductMetric

Dot product: fastest for pre-normalized vectors.

```rust
let metric = DotProductMetric::new();
```

**Formula:**

```
dot_product(a, b) = Σ(a[i] × b[i])
```

**Range:** (-∞, ∞)
- Assumes vectors are normalized
- Equal to cosine similarity for unit vectors

**Example:**

```rust
let metric = DotProductMetric::new();
let similarity = metric.similarity(&vec1, &vec2);
```

### Custom Metrics

Implement custom similarity by implementing the `SimilarityMetric` trait.

**Example: Sentiment-Weighted Similarity**

```rust
use vector_navigator::SimilarityMetric;

pub struct SentimentWeightedMetric {
    pub semantic_weight: f32,
    pub sentiment_weight: f32,
}

impl SimilarityMetric for SentimentWeightedMetric {
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32 {
        // Calculate cosine similarity
        let dot_product: f32 = a.iter()
            .zip(b.iter())
            .map(|(x, y)| x * y)
            .sum();

        let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
        let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();

        dot_product / (norm_a * norm_b)
    }

    fn similarity_with_context<P: Payload>(
        &self,
        a: &[f32],
        b: &[f32],
        ctx_a: &P,
        ctx_b: &P,
    ) -> f32 {
        // Get semantic similarity
        let semantic = self.similarity(a, b);

        // Get sentiment from payloads
        let sentiment_a = ctx_a.sentiment();
        let sentiment_b = ctx_b.sentiment();

        // Calculate sentiment match (3D cosine similarity)
        let va = sentiment_a.valence;
        let aa = sentiment_a.arousal;
        let da = sentiment_a.dominance;
        let vb = sentiment_b.valence;
        let ab = sentiment_b.arousal;
        let db = sentiment_b.dominance;

        let dot = va * vb + aa * ab + da * db;
        let norm_a = (va * va + aa * aa + da * da).sqrt();
        let norm_b = (vb * vb + ab * ab + db * db).sqrt();
        let sentiment_match = dot / (norm_a * norm_b);

        // Combine semantic and sentiment
        semantic * self.semantic_weight + sentiment_match * self.sentiment_weight
    }
}
```

**Usage:**

```rust
let metric = SentimentWeightedMetric {
    semantic_weight: 0.7,
    sentiment_weight: 0.3,
};

let store = VectorStoreImpl::new(metric, index)?;
```

## Configuration

### WALConfig

Configure Write-Ahead Log for durability.

```rust
use vector_navigator::{WALConfig, SyncMode};

let wal_config = WALConfig::builder()
    .path("/path/to/wal")              // WAL file path
    .sync_mode(SyncMode::Periodic)     // Sync strategy
    .sync_interval_ms(100)             // Sync interval (for Periodic)
    .rotation_size_mb(100)             // Rotation size
    .build()?;
```

**Sync Modes:**

- `SyncMode::Immediate` - Flush every write (safest, slowest)
- `SyncMode::Periodic` - Flush every N ms (balanced)
- `SyncMode::None` - No flush (fastest, risk of data loss)

### MemoryConfig

Configure memory limits and allocation.

```rust
use vector_navigator::MemoryConfig;

let mem_config = MemoryConfig::builder()
    .arena_size(1_000_000_000)         // 1GB arena
    .max_memory_bytes(4_000_000_000)   // 4GB limit
    .enable_mlock(true)                // Lock memory
    .build()?;
```

### ConcurrencyConfig

Configure threading and concurrency.

```rust
use vector_navigator::ConcurrencyConfig;

let conc_config = ConcurrencyConfig::builder()
    .max_search_threads(8)             // Max threads for search
    .max_insert_threads(4)             // Max threads for insert
    .enable_work_stealing(true)        // Work-stealing scheduler
    .build()?;
```

### SearchConfig

Configure search behavior.

```rust
use vector_navigator::SearchConfig;

let search_config = SearchConfig::builder()
    .enable_prefetch(true)             // Enable memory prefetch
    .prefetch_distance(4)              // Prefetch distance
    .enable SIMD(true)                 // Enable SIMD optimizations
    .build()?;
```

## Types

### Id

Unique identifier for vectors in the store.

```rust
pub struct Id(/* private */);
```

**Methods:**

```rust
// Generate new ID
let id = Id::new();

// Convert to/from string
let id_str = id.to_string();
let id = Id::from_str(id_str)?;

// Get UUID bytes
let bytes = id.as_bytes();
```

### SearchResult

Result from a search operation.

```rust
pub struct SearchResult<P> {
    pub id: Id,
    pub score: f32,
    pub payload: P,
}
```

**Fields:**

- `id: Id` - Unique identifier of the result
- `score: f32` - Similarity score (higher = more similar)
- `payload: P` - Associated payload data

### VAD

Valence-Arousal-Dominance sentiment representation.

```rust
pub struct VAD {
    pub valence: f32,     // Positive/negative (-1 to 1)
    pub arousal: f32,     // Intensity/calmness (-1 to 1)
    pub dominance: f32,   // Control/submission (-1 to 1)
}
```

**Example:**

```rust
let sentiment = VAD {
    valence: 0.8,   // Very positive
    arousal: 0.3,   // Calm
    dominance: 0.7, // Confident
};
```

### StoreError

Error type for store operations.

```rust
pub enum StoreError {
    InvalidDimension { expected: usize, got: usize },
    IndexNotBuilt,
    IndexFull,
    NotFound,
    SerializationError,
    IoError(std::io::Error),
    // ... more variants
}
```

## Examples

### Basic Search

```rust
use vector_navigator::{VectorStoreImpl, CosineMetric, HNSWIndex};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create store
    let store = VectorStoreImpl::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Insert vectors
    let vec1 = vec![0.1, 0.2, 0.3, /* ... */];
    let payload1 = "Document 1".to_string();
    store.insert(vec1, &payload1).await?;

    let vec2 = vec![0.2, 0.3, 0.4, /* ... */];
    let payload2 = "Document 2".to_string();
    store.insert(vec2, &payload2).await?;

    // Search
    let query = vec![0.15, 0.25, 0.35, /* ... */];
    let results = store.search(&query, 2).await?;

    for result in results {
        println!("Score: {:.2}, Text: {}", result.score, result.payload);
    }

    Ok(())
}
```

### Custom Metric

```rust
use vector_navigator::{VectorStoreImpl, SimilarityMetric, HNSWIndex};

struct MyMetric;

impl SimilarityMetric for MyMetric {
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32 {
        // Custom similarity logic
        let mut score = 0.0;
        for (x, y) in a.iter().zip(b.iter()) {
            score += (x - y).abs();  // Manhattan distance variant
        }
        1.0 / (1.0 + score)  // Convert to similarity
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStoreImpl::new(
        MyMetric,
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Use store...

    Ok(())
}
```

### Sentiment-Weighted Search

```rust
use vector_navigator::{VectorStoreImpl, CosineMetric, HNSWIndex, VAD};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStoreImpl::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Insert with sentiment
    let vector = vec![0.1; 384];
    let sentiment = VAD {
        valence: 0.8,
        arousal: 0.3,
        dominance: 0.7,
    };
    store.insert(vector, &sentiment).await?;

    // Search with sentiment weighting
    let query = vec![0.1; 384];
    let target_sentiment = VAD {
        valence: 0.7,
        arousal: 0.4,
        dominance: 0.6,
    };

    let results = store
        .search(&query, 10)
        .with_sentiment_weight(0.8)
        .with_target_sentiment(target_sentiment)
        .await?;

    for result in results {
        println!("Score: {:.2}", result.score);
    }

    Ok(())
}
```

### Batch Insertion

```rust
use vector_navigator::{VectorStoreImpl, CosineMetric, HNSWIndex, BatchInserter};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStoreImpl::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Create batch inserter
    let batcher = BatchInserter::new(store)
        .with_batch_size(1000)
        .with_concurrency(4);

    // Prepare batch
    let vectors: Vec<Vec<f32>> = (0..10000)
        .map(|_| vec![0.1; 384])
        .collect();

    let payloads: Vec<String> = (0..10000)
        .map(|i| format!("Document {}", i))
        .collect();

    let batch: Vec<_> = vectors.into_iter()
        .zip(payloads.into_iter())
        .collect();

    // Insert batch
    batcher.insert_all(batch).await?;

    println!("Inserted 10,000 vectors");

    Ok(())
}
```

### Error Handling

```rust
use vector_navigator::{VectorStoreImpl, CosineMetric, HNSWIndex, StoreError};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStoreImpl::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Insert with error handling
    let result = store.insert(vec![0.1; 384], &"test".to_string()).await;

    match result {
        Ok(id) => println!("Inserted with ID: {}", id),
        Err(StoreError::InvalidDimension { expected, got }) => {
            eprintln!("Dimension mismatch: expected {}, got {}", expected, got);
        }
        Err(StoreError::IndexFull) => {
            eprintln!("Index is full, cannot insert more vectors");
        }
        Err(e) => {
            eprintln!("Error inserting vector: {}", e);
        }
    }

    Ok(())
}
```

### Configuration Example

```rust
use vector_navigator::{
    VectorStoreImpl, CosineMetric, HNSWIndex,
    WALConfig, MemoryConfig, ConcurrencyConfig, SyncMode
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Configure WAL
    let wal_config = WALConfig::builder()
        .path("./data/wal")
        .sync_mode(SyncMode::Periodic)
        .sync_interval_ms(100)
        .build()?;

    // Configure memory
    let mem_config = MemoryConfig::builder()
        .max_memory_bytes(4_000_000_000)
        .enable_mlock(true)
        .build()?;

    // Configure concurrency
    let conc_config = ConcurrencyConfig::builder()
        .max_search_threads(8)
        .max_insert_threads(4)
        .build()?;

    // Create store with full configuration
    let store = VectorStoreImpl::builder()
        .metric(CosineMetric::new())
        .index(HNSWIndex::builder().dimension(384).build()?)
        .wal_config(wal_config)
        .memory_config(mem_config)
        .concurrency_config(conc_config)
        .build()?;

    // Use store...

    Ok(())
}
```

---

**For usage examples, see [examples/](../examples/). For architecture, see [ARCHITECTURE.md](ARCHITECTURE.md).**
