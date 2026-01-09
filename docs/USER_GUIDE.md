# User Guide

Complete guide to installing, configuring, and using vector-navigator.

## Table of Contents

1. [Installation](#installation)
2. [Basic Usage](#basic-usage)
3. [Advanced Usage](#advanced-usage)
4. [Configuration and Tuning](#configuration-and-tuning)
5. [Performance Optimization](#performance-optimization)
6. [Troubleshooting](#troubleshooting)
7. [Examples](#examples)

## Installation

### Rust (Core Library)

**Requirements:**
- Rust 1.70 or later
- Cargo (comes with Rust)

**Add to Cargo.toml:**

```toml
[dependencies]
vector-navigator = "0.1"
tokio = { version = "1.0", features = ["full"] }
```

**Install from source:**

```bash
git clone https://github.com/equilibrium-tokens/vector-navigator.git
cd vector-navigator
cargo build --release
```

### Python Bindings

**Install via pip:**

```bash
pip install vector-navigator
```

**From source:**

```bash
cd vector-navigator/bindings/python
pip install -e .
```

**Python version:** 3.8 or later

### Go Bindings

**Install via Go:**

```bash
go get github.com/equilibrium-tokens/vector-navigator-go
```

**Go version:** 1.18 or later

## Basic Usage

### Creating a VectorStore

```rust
use vector_navigator::{VectorStore, CosineMetric, HNSWIndex};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a new vector store
    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder()
            .dimension(384)  // Embedding dimension
            .build()?
    )?;

    Ok(())
}
```

### Inserting Vectors

```rust
use vector_navigator::ConversationContext;
use std::time::SystemTime;

// Define your payload type
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Document {
    id: String,
    text: String,
    category: String,
}

// Insert a vector with payload
let vector = vec![0.1, 0.2, 0.3, /* ... 381 more ... */];
let payload = Document {
    id: "doc1".to_string(),
    text: "The quick brown fox jumps over the lazy dog".to_string(),
    category: "animals".to_string(),
};

let id = store.insert(vector, &payload).await?;
println!("Inserted document with ID: {}", id);
```

### Searching Vectors

```rust
// Search for 10 most similar vectors
let query = vec![0.15, 0.25, 0.35, /* ... */];
let results = store.search(&query, 10).await?;

// Process results
for result in results {
    println!("Score: {:.4}", result.score);
    println!("Document: {}", result.payload.text);
    println!("Category: {}", result.payload.category);
    println!("---");
}
```

### Deleting Vectors

```rust
// Delete a vector by ID
store.delete(id).await?;
```

### Getting Store Statistics

```rust
// Get number of vectors in store
let count = store.count().await?;
println!("Total vectors: {}", count);
```

## Advanced Usage

### Custom Similarity Metrics

Define domain-specific similarity by implementing the `SimilarityMetric` trait:

```rust
use vector_navigator::SimilarityMetric;

struct SentimentWeightedMetric {
    semantic_weight: f32,
    sentiment_weight: f32,
}

impl SimilarityMetric for SentimentWeightedMetric {
    fn similarity(&self, a: &[f32], b: &[f32]) -> f32 {
        // Standard cosine similarity
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
        let semantic = self.similarity(a, b);

        // Access sentiment from payload
        let sentiment_a = ctx_a.sentiment();
        let sentiment_b = ctx_b.sentiment();

        let sentiment_match = cosine_similarity_3d(
            sentiment_a.valence,
            sentiment_a.arousal,
            sentiment_a.dominance,
            sentiment_b.valence,
            sentiment_b.arousal,
            sentiment_b.dominance,
        );

        // Combine semantic and sentiment
        semantic * self.semantic_weight
            + sentiment_match * self.sentiment_weight
    }
}

// Use the custom metric
let store = VectorStore::new(
    SentimentWeightedMetric {
        semantic_weight: 0.7,
        sentiment_weight: 0.3,
    },
    HNSWIndex::builder().dimension(384).build()?
)?;
```

### Sentiment-Weighted Search

For equilibrium-tokens, use built-in sentiment weighting:

```rust
use vector_navigator::{VAD, SentimentWeight};

// Search with sentiment weighting
let results = store
    .search(&query, 10)
    .with_sentiment_weight(0.8)  // High sentiment importance
    .with_target_sentiment(VAD {
        valence: 0.8,   // Positive
        arousal: 0.3,   // Calm
        dominance: 0.7, // Confident
    })
    .await?;
```

**Understanding VAD (Valence-Arousal-Dominance):**

- **Valence**: Positive/negative emotion (-1.0 to 1.0)
- **Arousal**: Intensity/calmness (-1.0 to 1.0)
- **Dominance**: Control/submission (-1.0 to 1.0)

### Hierarchical Navigation

Organize and navigate context basins hierarchically:

```rust
use vector_navigator::BasinNavigator;

// Create hierarchical structure
let navigator = BasinNavigator::new(store);

// Define basin hierarchy
navigator
    .create_basin(&["conversation", "technical", "rust"])
    .with_sentiment(VAD { valence: 0.6, arousal: 0.4, dominance: 0.8 })
    .await?;

// Insert into specific basin
navigator
    .insert_into_basin(
        &["conversation", "technical", "rust"],
        embedding,
        &payload
    )
    .await?;

// Search within basin hierarchy
let results = navigator
    .search_in_basin(&["conversation", "technical"], &query, 10)
    .with_depth(2)  // Search this basin and children
    .await?;
```

**Basin Path Examples:**

```
["conversation"]                    # Top-level: all conversations
["conversation", "technical"]       # Mid-level: technical discussions
["conversation", "technical", "rust"] # Low-level: Rust-specific
```

### Time-Decayed Search

Weight recent context more heavily:

```rust
use std::time::SystemTime;

let now = SystemTime::now();

let results = store
    .search(&query, 10)
    .with_time_decay(0.1)  // 10% decay per hour
    .with_current_time(now)
    .await?;
```

**Decay Formula:**

```
time_weight = exp(-decay_rate × hours_elapsed)
final_score = semantic_score × time_weight
```

### Batch Operations

Insert multiple vectors efficiently:

```rust
use vector_navigator::BatchInserter;

let batcher = BatchInserter::new(store.clone())
    .with_batch_size(1000)
    .with_concurrency(4);

// Prepare batch
let vectors_and_payloads = vec![
    (vec1, payload1),
    (vec2, payload2),
    // ... more vectors
];

// Insert batch
batcher.insert_all(vectors_and_payloads).await?;
```

### Filtering Results

Filter search results by metadata:

```rust
let results = store
    .search(&query, 100)  // Get more candidates
    .await?;

// Apply filters
let filtered: Vec<_> = results
    .into_iter()
    .filter(|r| r.payload.category == "technical")
    .filter(|r| r.score > 0.8)
    .take(10)  // Keep top 10 after filtering
    .collect();
```

### Combining Metrics

Chain multiple transformations:

```rust
let results = store
    .search(&query, 20)
    .with_sentiment_weight(0.7)
    .with_time_decay(0.05)
    .with_metadata_filter(|p| p.category == "rust")
    .await?;
```

## Configuration and Tuning

### Index Selection

**HNSW (Hierarchical Navigable Small World)**

```rust
let index = HNSWIndex::builder()
    .dimension(384)
    .m(16)                    // Connections per node (default: 16)
    .ef_construction(200)     // Index build accuracy (default: 200)
    .ef_search(100)           // Search accuracy (default: 100)
    .build()?;
```

**When to use HNSW:**
- Dataset size < 1M vectors
- Low latency critical (<5ms)
- High recall required (>0.95)
- Sufficient memory available

**Tuning parameters:**
- `m`: Higher = better recall, more memory, slower build
- `ef_construction`: Higher = better recall, slower build
- `ef_search`: Higher = better recall, slower search

**IVF (Inverted File)**

```rust
use vector_navigator::ProductQuantization;

let index = IVFIndex::builder()
    .dimension(384)
    .nlist(100)              // Number of clusters
    .nprobe(10)              // Clusters to query
    .quantization(ProductQuantization::new(8, 8)?)  // PQ parameters
    .build()?;
```

**When to use IVF:**
- Dataset size > 1M vectors
- Memory-constrained
- Fast index construction required
- Moderate latency acceptable (<10ms)

**Tuning parameters:**
- `nlist`: Should be ~√N (where N = number of vectors)
- `nprobe`: Higher = better recall, slower search
- Quantization: Reduces memory at cost of accuracy

**Flat (Exact Search)**

```rust
let index = FlatIndex::builder()
    .dimension(384)
    .build()?;
```

**When to use Flat:**
- Dataset size < 100K vectors
- 100% recall required
- Benchmarking ground truth

### Performance Tuning

**Memory Configuration**

```rust
use vector_navigator::MemoryConfig;

let config = MemoryConfig::builder()
    .arena_size(1_000_000_000)  // 1GB arena
    .enable_mlock(true)         // Lock memory (prevent swapping)
    .build();

let store = VectorStore::with_config(config, metric, index)?;
```

**Concurrency Configuration**

```rust
use vector_navigator::ConcurrencyConfig;

let config = ConcurrencyConfig::builder()
    .max_search_threads(8)
    .max_insert_threads(4)
    .build();

let store = VectorStore::with_concurrency(config, metric, index)?;
```

**WAL (Write-Ahead Log) Configuration**

```rust
use vector_navigator::{WALConfig, SyncMode};

let wal_config = WALConfig::builder()
    .path("/path/to/wal")
    .sync_mode(SyncMode::Periodic)
    .sync_interval_ms(100)
    .rotation_size_mb(100)
    .build();

let store = VectorStore::with_wal(wal_config, metric, index)?;
```

**Sync modes:**
- `Immediate`: Flush every write (safest, slowest)
- `Periodic`: Flush every N ms (balanced)
- `None`: No flush (fastest, risk of data loss)

### Choosing Similarity Metrics

**Cosine Similarity** (default)

```rust
let metric = CosineMetric::new();
```

**Use when:**
- Magnitude doesn't matter
- Directional similarity is key
- Working with normalized embeddings
- **Most semantic search use cases**

**Euclidean Distance**

```rust
let metric = EuclideanMetric::new();
```

**Use when:**
- Magnitude matters
- Working with unnormalized vectors
- Physical/geometric interpretation needed

**Dot Product**

```rust
let metric = DotProductMetric::new();
```

**Use when:**
- Vectors are pre-normalized
- Maximum speed required
- Careful about vector scaling

## Performance Optimization

### Pre-normalization

Normalize vectors once for faster cosine similarity:

```rust
fn normalize(vec: &mut [f32]) {
    let norm: f32 = vec.iter().map(|x| x * x).sum::<f32>().sqrt();
    for v in vec.iter_mut() {
        *v /= norm;
    }
}

// Before insertion
let mut vector = /* ... */;
normalize(&mut vector);
store.insert(vector, &payload).await?;
```

**Then use DotProductMetric (fastest):**

```rust
let metric = DotProductMetric::new();
```

### Batch Insertion

Insert vectors in batches for better performance:

```rust
// Bad: Individual inserts
for (vec, payload) in vectors {
    store.insert(vec, &payload).await?;  // Slow
}

// Good: Batch insert
batcher.insert_all(vectors).await?;  // Fast
```

### Parallel Search

Search multiple queries in parallel:

```rust
use futures::future::join_all;

let queries = vec![query1, query2, query3];

let results = join_all(
    queries.iter().map(|q| store.search(q, 10))
).await;
```

### Index Warming

Warm up index before production use:

```rust
// Run some test queries
let dummy_queries = generate_random_queries(100);

for query in dummy_queries {
    store.search(&query, 10).await?;
}

// Index is now "warm" and in CPU cache
```

### Memory Prefetching

Enable prefetching for better cache utilization:

```rust
let config = SearchConfig::builder()
    .enable_prefetch(true)
    .prefetch_distance(4)
    .build();

let results = store
    .search_with_config(&query, 10, config)
    .await?;
```

## Troubleshooting

### High Latency

**Symptom:** Search takes >10ms

**Diagnosis:**

```rust
let start = std::time::Instant::now();
let results = store.search(&query, 10).await?;
let elapsed = start.elapsed();
println!("Search took: {:?}", elapsed);
```

**Solutions:**

1. **Check index type:**
   ```rust
   // For large datasets (>1M), use IVF instead of HNSW
   let index = IVFIndex::builder()
       .dimension(384)
       .nlist(1000)
       .nprobe(10)
       .build()?;
   ```

2. **Reduce ef_search (HNSW):**
   ```rust
   let index = HNSWIndex::builder()
       .dimension(384)
       .ef_search(50)  // Reduce from 100
       .build()?;
   ```

3. **Enable SIMD:**
   ```bash
   RUSTFLAGS="-C target-cpu=native" cargo build --release
   ```

4. **Check memory pressure:**
   ```bash
   # Ensure enough RAM
   free -h

   # Disable swap if possible
   sudo swapoff -a
   ```

### Low Recall

**Symptom:** Missing relevant results

**Diagnosis:**

```rust
// Compare against exact search
let mut exact_store = VectorStore::new(
    CosineMetric::new(),
    FlatIndex::builder().dimension(384).build()?
)?;

// Insert same vectors into both stores
// ...

let approx_results = store.search(&query, 10).await?;
let exact_results = exact_store.search(&query, 10).await?;

let recall = calculate_recall(&approx_results, &exact_results);
println!("Recall: {:.2}", recall);
```

**Solutions:**

1. **Increase ef_search (HNSW):**
   ```rust
   let index = HNSWIndex::builder()
       .dimension(384)
       .ef_search(200)  // Increase from 100
       .build()?;
   ```

2. **Increase nprobe (IVF):**
   ```rust
   let index = IVFIndex::builder()
       .dimension(384)
       .nprobe(20)  // Increase from 10
       .build()?;
   ```

3. **Disable quantization (IVF):**
   ```rust
   let index = IVFIndex::builder()
       .dimension(384)
       .quantization(None)  // No compression
       .build()?;
   ```

4. **Use HNSW instead of IVF:**
   ```rust
   // HNSW typically has higher recall
   let index = HNSWIndex::builder()
       .dimension(384)
       .build()?;
   ```

### Out of Memory

**Symptom:** Process killed, OOM error

**Solutions:**

1. **Use IVF with quantization:**
   ```rust
   use vector_navigator::ProductQuantization;

   let index = IVFIndex::builder()
       .dimension(384)
       .quantization(ProductQuantization::new(8, 8)?)  // 8x compression
       .build()?;
   ```

2. **Reduce HNSW parameters:**
   ```rust
   let index = HNSWIndex::builder()
       .dimension(384)
       .m(8)  // Reduce from 16
       .ef_construction(100)  // Reduce from 200
       .build()?;
   ```

3. **Enable memory limit:**
   ```rust
   use vector_navigator::MemoryConfig;

   let config = MemoryConfig::builder()
       .max_memory_bytes(4_000_000_000)  // 4GB limit
       .build();

   let store = VectorStore::with_config(config, metric, index)?;
   ```

### Slow Insertion

**Symptom:** Insert takes >5ms per vector

**Solutions:**

1. **Use batch insertion:**
   ```rust
   batcher.insert_all(vectors).await?;
   ```

2. **Disable WAL (if acceptable):**
   ```rust
   let wal_config = WALConfig::builder()
       .sync_mode(SyncMode::None)
       .build();
   ```

3. **Reduce ef_construction (HNSW):**
   ```rust
   let index = HNSWIndex::builder()
       .dimension(384)
       .ef_construction(100)  // Reduce from 200
       .build()?;
   ```

## Examples

### Example 1: Semantic Document Search

```rust
use vector_navigator::{VectorStore, CosineMetric, HNSWIndex};
use sentence_transformers::{SentenceTransformer, Device};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load embedding model
    let model = SentenceTransformer::load("all-MiniLM-L6-v2", Device::Cpu)?;

    // Create vector store
    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Index documents
    let documents = vec![
        "Rust is a systems programming language",
        "Python is great for data science",
        "Go excels at concurrency",
    ];

    for doc in &documents {
        let embedding = model.encode(doc)?;
        store.insert(embedding, &doc.to_string()).await?;
    }

    // Search
    let query = "programming language for systems";
    let query_embedding = model.encode(query)?;
    let results = store.search(&query_embedding, 2).await?;

    for result in results {
        println!("Score: {:.2}, Text: {}", result.score, result.payload);
    }

    Ok(())
}
```

### Example 2: Conversation Context Navigation

```rust
use vector_navigator::{VectorStore, CosineMetric, VAD, BasinNavigator};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    let navigator = BasinNavigator::new(store);

    // Create conversation basins
    navigator.create_basin(&["conversation", "technical"]).await?;
    navigator.create_basin(&["conversation", "casual"]).await?;

    // Insert with context
    navigator.insert_into_basin(
        &["conversation", "technical"],
        embedding,
        &ConversationTurn {
            text: "The mutex ensures thread safety".to_string(),
            sentiment: VAD {
                valence: 0.3,
                arousal: 0.2,
                dominance: 0.6,
            },
            timestamp: SystemTime::now(),
        }
    ).await?;

    // Search in technical conversations
    let results = navigator
        .search_in_basin(&["conversation", "technical"], &query, 5)
        .with_sentiment_weight(0.8)
        .await?;

    Ok(())
}
```

### Example 3: Real-time RAG System

```rust
use vector_navigator::{VectorStore, CosineMetric, IVFIndex};
use tokio::sync::mpsc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStore::new(
        CosineMetric::new(),
        IVFIndex::builder()
            .dimension(1536)
            .nlist(100)
            .build()?
    )?;

    // Real-time document ingestion
    let (doc_tx, mut doc_rx) = mpsc::channel(100);

    // Document processor task
    let store_clone = store.clone();
    tokio::spawn(async move {
        while let Some(doc) = doc_rx.recv().await {
            let embedding = embed_document(&doc).await?;
            store_clone.insert(embedding, &doc).await?;
        }
        Ok::<(), Box<dyn std::error::Error>>(())
    });

    // Query handler
    let query = "What is the capital of France?";
    let query_embedding = embed_query(query).await?;

    let results = store
        .search(&query_embedding, 5)
        .with_time_decay(0.01)
        .await?;

    for result in results {
        println!("Relevant: {:.2} - {}", result.score, result.payload.text);
    }

    Ok(())
}
```

### Example 4: Multi-Vector Search (Hybrid)

```rust
use vector_navigator::{VectorStore, CosineMetric};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder().dimension(384).build()?
    )?;

    // Multi-vector query (e.g., query + expansion)
    let query1 = "rust programming";
    let query2 = "systems language";

    let embedding1 = embed(query1).await?;
    let embedding2 = embed(query2).await?;

    // Search both
    let results1 = store.search(&embedding1, 20).await?;
    let results2 = store.search(&embedding2, 20).await?;

    // Reciprocal rank fusion
    let fused = reciprocal_rank_fusion(&[results1, results2], k=60);

    for result in fused.into_iter().take(10) {
        println!("Score: {:.2}, Text: {}", result.score, result.payload.text);
    }

    Ok(())
}

fn reciprocal_rank_fusion(results: &[Vec<SearchResult>], k: usize) -> Vec<SearchResult> {
    use std::collections::HashMap;

    let mut scores: HashMap<String, f32> = HashMap::new();

    for result_list in results {
        for (rank, result) in result_list.iter().enumerate() {
            let rrf_score = 1.0 / (k as f32 + rank as f32 + 1.0);
            *scores.entry(result.payload.id.clone()).or_insert(0.0) += rrf_score;
        }
    }

    // Sort by score
    let mut scored: Vec<_> = scores.into_iter().collect();
    scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    // Convert back to SearchResult (simplified)
    scored.into_iter()
        .map(|(id, score)| SearchResult { id, score, /* ... */ })
        .collect()
}
```

## Best Practices

1. **Always normalize vectors** if using cosine similarity
2. **Batch insertions** for better performance
3. **Use appropriate index type** for your dataset size
4. **Monitor latency and recall** in production
5. **Set memory limits** to prevent OOM
6. **Use WAL** for durability
7. **Warm up index** before production traffic
8. **Profile before optimizing** - measure first

---

**For API details, see [API.md](API.md). For architecture, see [ARCHITECTURE.md](ARCHITECTURE.md).**
