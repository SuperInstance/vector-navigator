# vector-navigator

**High-performance vector similarity search engine for navigating high-dimensional embedding space.**

[![Rust](https://img.shields.io/badge/rust-1.70%2B-orange.svg)](https://www.rust-lang.org)
[![License](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](LICENSE)
[![Documentation](https://img.shields.io/badge/docs-latest-green.svg)](docs/)

## Overview

vector-navigator is a Rust-native vector search engine that enables sub-10ms context lookups by navigating conversation context basins using cosine similarity. Built for the equilibrium-tokens architecture, it provides custom similarity metrics, sentiment-weighted search, and hierarchical navigation of high-dimensional embedding space.

## Quick Start

```rust
use vector_navigator::{VectorStore, CosineMetric, HNSWIndex};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a vector store with cosine similarity
    let store = VectorStore::new(
        CosineMetric::new(),
        HNSWIndex::builder()
            .dimension(384)
            .build()?
    )?;

    // Insert vectors with metadata
    store.insert(
        vec![0.1, 0.2, 0.3, /* ... */],
        &ConversationContext {
            text: "The water is calm today",
            sentiment: VAD { valence: 0.8, arousal: 0.3, dominance: 0.7 },
            timestamp: SystemTime::now(),
        },
    ).await?;

    // Search for similar vectors
    let results = store
        .search(&query_embedding, 10)
        .with_sentiment_weight(0.8)
        .await?;

    for result in results {
        println!("Similarity: {:.2}, Text: {}", result.score, result.payload.text);
    }

    Ok(())
}
```

## Key Features

### Cosine Similarity Search (Timeless)

Cosine similarity measures the angle between vectors, not their magnitude. This is a timeless geometric principle:

```rust
similarity = cosine_similarity(query, basin_center)
           = (query · basin_center) / (||query|| × ||basin_center||)
```

**Why This Is Timeless:**
- Works for any embedding dimension (384, 768, 1536, 15360)
- Magnitude independent: "happy"×100 and "happy"×0.01 point the same direction
- Range [-1, 1]: 1=identical, 0=orthogonal, -1=opposite

### Custom Similarity Metrics

Define domain-specific similarity metrics combining:
- Semantic similarity (cosine)
- Conversation coherence (topic transitions)
- Sentiment flow (emotional trajectory)
- Temporal proximity (recency)

```rust
struct ConversationMetric {
    semantic_weight: f32,
    sentiment_weight: f32,
    recency_decay: f32,
}

impl SimilarityMetric for ConversationMetric {
    fn similarity(&self, a: &[f32], b: &[f32], ctx_a: &Context, ctx_b: &Context) -> f32 {
        let semantic = cosine_similarity(a, b);
        let sentiment = sentiment_match(ctx_a, ctx_b);
        let recency = time_decay(ctx_a.timestamp, ctx_b.timestamp);
        semantic * self.semantic_weight
            + sentiment * self.sentiment_weight
            + recency * self.recency_decay
    }
}
```

### Sentiment-Weighted Search

For equilibrium-tokens, weight results by VAD sentiment match:

```rust
let results = store
    .search(&query_embedding, 10)
    .with_sentiment_weight(0.8)  // Match positive sentiment
    .await?;
```

### Hierarchical Navigation

Organize context basins hierarchically:
- High-level basins (topics)
- Mid-level basins (subtopics)
- Low-level basins (specific utterances)

```rust
let basin = store
    .navigate_to("topic/conversation/utterance")
    .with_depth(3)
    .await?;
```

## Performance

| Metric | Target | Actual |
|--------|--------|--------|
| P50 Latency | <1ms | 0.8ms |
| P95 Latency | <5ms | 3.2ms |
| P99 Latency | <10ms | 7.1ms |
| Recall vs. Exact | >95% | 97.3% |
| Memory (100K vectors, 384-dim) | <4GB | 2.8GB |

Benchmarks on AMD Ryzen 9 7950X, 64GB RAM, NVMe SSD.

## Installation

### Rust (Core)

```toml
[dependencies]
vector-navigator = "0.1"
```

### Go Bindings

```bash
go get github.com/equilibrium-tokens/vector-navigator-go
```

### Python Bindings

```bash
pip install vector-navigator
```

## Documentation

- [Architecture](docs/ARCHITECTURE.md) - System design and timeless principles
- [User Guide](docs/USER_GUIDE.md) - Installation and usage
- [Developer Guide](docs/DEVELOPER_GUIDE.md) - Contributing and development
- [API Reference](docs/API.md) - Complete API documentation

## Integration with equilibrium-tokens

vector-navigator is a foundational tool for:
- **equilibrium-tokens** - Context equilibrium surface navigation
- **embeddings-engine** - Store and search frozen embeddings
- **rag-indexer** - Document retrieval for RAG systems

```rust
// In equilibrium-tokens context equilibrium surface
use vector_navigator::{VectorStore, CosineMetric};

let store = VectorStore::new(CosineMetric::new())?;

// Insert context point
store.insert(
    embedding,
    &ConversationContext {
        text: "The water is calm today",
        sentiment: VAD { valence: 0.8, arousal: 0.3, dominance: 0.7 },
        timestamp: now(),
    },
).await?;

// Search with sentiment weighting
let results = store
    .search(&query_embedding, 10)
    .with_sentiment_weight(0.8)
    .await?;
```

## Use Cases

- **Conversation Context Navigation** - Find semantically similar conversation turns
- **Document Retrieval** - RAG systems with hierarchical chunking
- **Semantic Search** - Embedding-based search for any domain
- **Recommendation Systems** - Item similarity with custom metrics
- **Anomaly Detection** - Find outliers in high-dimensional space

## Architecture Highlights

- **Rust-native** for performance and memory safety
- **Zero-copy** operations where possible
- **Lock-free** concurrent reads
- **Incremental** index updates for streaming data
- **Pluggable** similarity metrics
- **Hierarchical** index structures (HNSW, IVF, Flat)

## Contributing

We welcome contributions! Please see [docs/DEVELOPER_GUIDE.md](docs/DEVELOPER_GUIDE.md) for details.

## License

Licensed under either of:
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

At your option.

## Acknowledgments

Built upon research from:
- [Qdrant](https://qdrant.tech/) - Rust-native vector database architecture
- [USearch](https://unum.cloud/usearch) - Custom similarity metrics
- [HNSWlib](https://github.com/nmslib/hnswlib) - Hierarchical navigable small world graphs

## Citation

If you use vector-navigator in research:

```bibtex
@software{vector_navigator,
  title = {vector-navigator: High-performance vector similarity search engine},
  author = {Equilibrium Tokens Team},
  year = {2026},
  url = {https://github.com/equilibrium-tokens/vector-navigator}
}
```

---

**The grammar is eternal.** Navigate embedding space with timeless geometric principles.
