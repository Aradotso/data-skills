---
name: ai-data-engineering-rag-pipeline
description: Build production-grade local RAG pipelines with BM25 search, hierarchical chunking, and retrieval evaluation from Nahid's hands-on roadmap
triggers:
  - how do I build a RAG pipeline from scratch
  - implement BM25 baseline search engine
  - create hierarchical chunking for document retrieval
  - evaluate retrieval performance with Recall@10
  - design retrieval contracts and golden datasets
  - set up local RAG with chunk granularity analysis
  - build inverted index for document search
  - implement parent-child metadata linkage for chunks
---

# AI Data Engineering RAG Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build production-grade local RAG (Retrieval-Augmented Generation) pipelines using the architectural patterns and implementations from Nahid Mahmud's AI & Data Engineering Roadmap. The project focuses on baseline search engines, hierarchical chunking strategies, and rigorous evaluation methodologies.

## What This Project Provides

- **Day 01**: Retrieval contract design, governed corpus creation, golden dataset (`questions.jsonl`), and evaluation frameworks
- **Day 02**: BM25 baseline search engine with inverted index, Okapi BM25 ranking, and Recall@10 evaluation
- **Day 03**: RAG pipeline with hierarchical chunking (document, section, paragraph levels), deterministic chunk IDs, parent-child metadata linkage, and failure mode analysis

## Installation & Setup

Clone the repository:

```bash
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap
```

Install dependencies (each day has its own requirements):

```bash
# For Day 02 (BM25)
cd Day_02
pip install -r requirements.txt

# For Day 03 (RAG Pipeline)
cd Day_03
pip install -r requirements.txt
```

Common dependencies across modules:
- `numpy` - numerical operations
- `rank-bm25` - BM25 implementation
- `sentence-transformers` - embeddings (Day 03+)
- `faiss-cpu` - vector search (Day 03+)

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/           # Retrieval contracts & golden datasets
├── Day_02/           # BM25 baseline search engine
│   ├── baseline_bm25.py
│   ├── corpus/       # Document collection
│   └── questions.jsonl
├── Day_03/           # RAG pipeline with hierarchical chunking
│   ├── pipeline.py
│   ├── corpus/       # Multi-domain documents
│   └── questions.jsonl
```

## Day 02: BM25 Baseline Search Engine

### Core Implementation

The BM25 baseline provides a deterministic search engine for retrieval evaluation:

```python
from rank_bm25 import BM25Okapi
import json

# Load corpus
corpus_docs = []
doc_ids = []

# Assuming corpus stored in text files
import os
corpus_dir = "corpus"
for filename in os.listdir(corpus_dir):
    if filename.endswith(".txt"):
        with open(os.path.join(corpus_dir, filename), 'r') as f:
            content = f.read()
            corpus_docs.append(content)
            doc_ids.append(filename)

# Tokenize corpus (simple whitespace tokenization)
tokenized_corpus = [doc.lower().split() for doc in corpus_docs]

# Build BM25 index
bm25 = BM25Okapi(tokenized_corpus)

# Query retrieval
query = "what is machine learning"
tokenized_query = query.lower().split()

# Get top-k scores and documents
doc_scores = bm25.get_scores(tokenized_query)

# Retrieve top 10 documents
import numpy as np
top_n = 10
top_indices = np.argsort(doc_scores)[::-1][:top_n]

results = [(doc_ids[i], corpus_docs[i], doc_scores[i]) for i in top_indices]

for rank, (doc_id, content, score) in enumerate(results, 1):
    print(f"Rank {rank}: {doc_id} (Score: {score:.4f})")
    print(f"Preview: {content[:200]}...\n")
```

### Recall@10 Evaluation

Evaluate retrieval quality against golden dataset:

```python
import json

# Load golden dataset
with open('questions.jsonl', 'r') as f:
    questions = [json.loads(line) for line in f]

def calculate_recall_at_k(bm25_index, tokenized_corpus, doc_ids, questions, k=10):
    """
    Calculate Recall@K for retrieval evaluation
    
    questions format: [{"query": "...", "relevant_docs": ["doc1.txt", "doc2.txt"]}]
    """
    total_recall = 0
    num_queries = len(questions)
    
    for q in questions:
        query = q['query']
        relevant_docs = set(q['relevant_docs'])
        
        # Tokenize and search
        tokenized_query = query.lower().split()
        scores = bm25_index.get_scores(tokenized_query)
        
        # Get top-k
        top_k_indices = np.argsort(scores)[::-1][:k]
        retrieved_docs = set([doc_ids[i] for i in top_k_indices])
        
        # Calculate recall
        relevant_retrieved = len(relevant_docs.intersection(retrieved_docs))
        recall = relevant_retrieved / len(relevant_docs) if relevant_docs else 0
        total_recall += recall
        
        print(f"Query: {query}")
        print(f"Recall@{k}: {recall:.2%}")
        print(f"Retrieved: {retrieved_docs}")
        print(f"Relevant: {relevant_docs}\n")
    
    avg_recall = total_recall / num_queries
    print(f"Average Recall@{k}: {avg_recall:.2%}")
    return avg_recall

# Run evaluation
recall = calculate_recall_at_k(bm25, tokenized_corpus, doc_ids, questions, k=10)
```

## Day 03: Hierarchical Chunking RAG Pipeline

### Chunk Granularity Strategy

Implement multi-level chunking with parent-child metadata:

```python
import hashlib
import uuid

def generate_deterministic_chunk_id(content, parent_id=None, level="document"):
    """
    Generate deterministic chunk ID based on content hash
    """
    content_hash = hashlib.sha256(content.encode('utf-8')).hexdigest()[:16]
    return f"{level}_{content_hash}"

def hierarchical_chunking(document, doc_id):
    """
    Create document, section, and paragraph level chunks
    
    Returns: List of chunk dictionaries with metadata
    """
    chunks = []
    
    # Document-level chunk
    doc_chunk_id = generate_deterministic_chunk_id(document, level="doc")
    chunks.append({
        "chunk_id": doc_chunk_id,
        "content": document,
        "level": "document",
        "parent_id": None,
        "doc_id": doc_id,
        "metadata": {"granularity": "document"}
    })
    
    # Section-level chunking (split by double newline or headers)
    sections = document.split("\n\n")
    for sec_idx, section in enumerate(sections):
        if len(section.strip()) < 50:  # Skip very short sections
            continue
            
        sec_chunk_id = generate_deterministic_chunk_id(section, level="sec")
        chunks.append({
            "chunk_id": sec_chunk_id,
            "content": section,
            "level": "section",
            "parent_id": doc_chunk_id,
            "doc_id": doc_id,
            "metadata": {
                "granularity": "section",
                "section_index": sec_idx
            }
        })
        
        # Paragraph-level chunking
        paragraphs = section.split("\n")
        for para_idx, paragraph in enumerate(paragraphs):
            if len(paragraph.strip()) < 20:  # Skip very short paragraphs
                continue
                
            para_chunk_id = generate_deterministic_chunk_id(paragraph, level="para")
            chunks.append({
                "chunk_id": para_chunk_id,
                "content": paragraph,
                "level": "paragraph",
                "parent_id": sec_chunk_id,
                "doc_id": doc_id,
                "metadata": {
                    "granularity": "paragraph",
                    "section_index": sec_idx,
                    "paragraph_index": para_idx
                }
            })
    
    return chunks

# Example usage
sample_doc = """# Machine Learning Basics

Machine learning is a subset of artificial intelligence that enables systems to learn from data.

## Supervised Learning
Supervised learning uses labeled data to train models. Common algorithms include linear regression and decision trees.

## Unsupervised Learning
Unsupervised learning finds patterns in unlabeled data. Clustering and dimensionality reduction are key techniques."""

chunks = hierarchical_chunking(sample_doc, "ml_basics.txt")

for chunk in chunks:
    print(f"Level: {chunk['level']}, ID: {chunk['chunk_id'][:20]}...")
    print(f"Parent: {chunk['parent_id'][:20] if chunk['parent_id'] else 'None'}...")
    print(f"Content: {chunk['content'][:80]}...\n")
```

### RAG Pipeline with Vector Search

Combine BM25 with semantic search:

```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

class HybridRAGPipeline:
    def __init__(self, embedding_model="all-MiniLM-L6-v2"):
        self.embedding_model = SentenceTransformer(embedding_model)
        self.chunks = []
        self.bm25_index = None
        self.faiss_index = None
        
    def index_documents(self, documents):
        """
        Index documents with both BM25 and vector embeddings
        """
        # Create hierarchical chunks
        all_chunks = []
        for doc_id, doc_content in documents.items():
            chunks = hierarchical_chunking(doc_content, doc_id)
            all_chunks.extend(chunks)
        
        self.chunks = all_chunks
        
        # BM25 indexing
        tokenized_chunks = [chunk['content'].lower().split() for chunk in all_chunks]
        self.bm25_index = BM25Okapi(tokenized_chunks)
        
        # Vector indexing
        chunk_texts = [chunk['content'] for chunk in all_chunks]
        embeddings = self.embedding_model.encode(chunk_texts, show_progress_bar=True)
        
        # Build FAISS index
        dimension = embeddings.shape[1]
        self.faiss_index = faiss.IndexFlatIP(dimension)  # Inner product for cosine similarity
        
        # Normalize embeddings for cosine similarity
        faiss.normalize_L2(embeddings)
        self.faiss_index.add(embeddings)
        
    def retrieve(self, query, top_k=10, alpha=0.5, granularity="paragraph"):
        """
        Hybrid retrieval: alpha * BM25 + (1-alpha) * Vector
        
        Args:
            query: Search query
            top_k: Number of results
            alpha: Weight for BM25 (0-1), (1-alpha) for vector search
            granularity: Filter by chunk level (document, section, paragraph)
        """
        # Filter chunks by granularity
        filtered_indices = [i for i, chunk in enumerate(self.chunks) 
                           if chunk['level'] == granularity]
        
        if not filtered_indices:
            filtered_indices = list(range(len(self.chunks)))
        
        # BM25 scores
        tokenized_query = query.lower().split()
        bm25_scores = self.bm25_index.get_scores(tokenized_query)
        bm25_scores_filtered = bm25_scores[filtered_indices]
        
        # Normalize BM25 scores
        if bm25_scores_filtered.max() > 0:
            bm25_scores_filtered = bm25_scores_filtered / bm25_scores_filtered.max()
        
        # Vector scores
        query_embedding = self.embedding_model.encode([query])
        faiss.normalize_L2(query_embedding)
        
        distances, indices = self.faiss_index.search(query_embedding, len(self.chunks))
        vector_scores = distances[0]
        vector_scores_filtered = vector_scores[filtered_indices]
        
        # Hybrid scoring
        hybrid_scores = alpha * bm25_scores_filtered + (1 - alpha) * vector_scores_filtered
        
        # Get top-k
        top_k_local = np.argsort(hybrid_scores)[::-1][:top_k]
        top_k_global = [filtered_indices[i] for i in top_k_local]
        
        results = []
        for idx in top_k_global:
            results.append({
                "chunk": self.chunks[idx],
                "bm25_score": float(bm25_scores[idx]),
                "vector_score": float(vector_scores[idx]),
                "hybrid_score": float(alpha * bm25_scores[idx] + (1 - alpha) * vector_scores[idx])
            })
        
        return results

# Example usage
pipeline = HybridRAGPipeline()

# Load corpus
documents = {
    "ml_basics.txt": sample_doc,
    "dl_intro.txt": "Deep learning uses neural networks with multiple layers..."
}

pipeline.index_documents(documents)

# Query with different granularities
query = "what is supervised learning"

print("=== Paragraph-level retrieval ===")
results = pipeline.retrieve(query, top_k=5, alpha=0.5, granularity="paragraph")
for i, result in enumerate(results, 1):
    print(f"\nRank {i}:")
    print(f"Content: {result['chunk']['content'][:150]}...")
    print(f"Level: {result['chunk']['level']}")
    print(f"Hybrid Score: {result['hybrid_score']:.4f}")
```

## Retrieval Evaluation Framework

Evaluate chunking strategies against golden dataset:

```python
def evaluate_chunking_strategy(pipeline, questions, granularity="paragraph", k=10):
    """
    Evaluate retrieval performance for specific chunk granularity
    """
    metrics = {
        "recall": [],
        "precision": [],
        "mrr": []  # Mean Reciprocal Rank
    }
    
    for q in questions:
        query = q['query']
        relevant_docs = set(q['relevant_docs'])
        
        # Retrieve with specific granularity
        results = pipeline.retrieve(query, top_k=k, granularity=granularity)
        retrieved_docs = set([r['chunk']['doc_id'] for r in results])
        
        # Recall
        relevant_retrieved = len(relevant_docs.intersection(retrieved_docs))
        recall = relevant_retrieved / len(relevant_docs) if relevant_docs else 0
        metrics['recall'].append(recall)
        
        # Precision
        precision = relevant_retrieved / len(retrieved_docs) if retrieved_docs else 0
        metrics['precision'].append(precision)
        
        # MRR
        for rank, result in enumerate(results, 1):
            if result['chunk']['doc_id'] in relevant_docs:
                metrics['mrr'].append(1.0 / rank)
                break
        else:
            metrics['mrr'].append(0.0)
    
    return {
        "granularity": granularity,
        "avg_recall": np.mean(metrics['recall']),
        "avg_precision": np.mean(metrics['precision']),
        "mrr": np.mean(metrics['mrr'])
    }

# Compare granularities
for granularity in ["document", "section", "paragraph"]:
    eval_results = evaluate_chunking_strategy(
        pipeline, questions, granularity=granularity, k=10
    )
    print(f"\n{granularity.upper()} Level:")
    print(f"Recall@10: {eval_results['avg_recall']:.2%}")
    print(f"Precision@10: {eval_results['avg_precision']:.2%}")
    print(f"MRR: {eval_results['mrr']:.4f}")
```

## Configuration Patterns

### Golden Dataset Format (`questions.jsonl`)

```json
{"query": "what is machine learning", "relevant_docs": ["ml_basics.txt", "ai_intro.txt"]}
{"query": "explain supervised learning algorithms", "relevant_docs": ["ml_basics.txt"]}
{"query": "difference between classification and regression", "relevant_docs": ["ml_basics.txt", "supervised_learning.txt"]}
```

### Chunk Metadata Schema

```python
chunk_schema = {
    "chunk_id": "str (deterministic hash)",
    "content": "str (actual text content)",
    "level": "str (document|section|paragraph)",
    "parent_id": "str|None (parent chunk_id)",
    "doc_id": "str (source document identifier)",
    "metadata": {
        "granularity": "str",
        "section_index": "int (optional)",
        "paragraph_index": "int (optional)",
        "custom_fields": "dict (extensible)"
    }
}
```

## Common Patterns

### Parent-Child Retrieval

Retrieve at fine granularity but return parent context:

```python
def retrieve_with_parent_context(pipeline, query, child_granularity="paragraph", top_k=5):
    """
    Retrieve chunks and include parent section context
    """
    results = pipeline.retrieve(query, top_k=top_k, granularity=child_granularity)
    
    enriched_results = []
    for result in results:
        chunk = result['chunk']
        parent_id = chunk['parent_id']
        
        # Find parent chunk
        parent = next((c for c in pipeline.chunks if c['chunk_id'] == parent_id), None)
        
        enriched_results.append({
            "match": chunk['content'],
            "context": parent['content'] if parent else chunk['content'],
            "score": result['hybrid_score'],
            "level": chunk['level']
        })
    
    return enriched_results
```

### Failure Mode Analysis

Identify queries with poor retrieval:

```python
def analyze_failure_modes(pipeline, questions, threshold=0.3):
    """
    Identify queries where retrieval fails (Recall@10 < threshold)
    """
    failures = []
    
    for q in questions:
        query = q['query']
        relevant_docs = set(q['relevant_docs'])
        
        results = pipeline.retrieve(query, top_k=10)
        retrieved_docs = set([r['chunk']['doc_id'] for r in results])
        
        recall = len(relevant_docs.intersection(retrieved_docs)) / len(relevant_docs)
        
        if recall < threshold:
            failures.append({
                "query": query,
                "recall": recall,
                "expected": relevant_docs,
                "retrieved": retrieved_docs,
                "top_result": results[0]['chunk']['content'][:200] if results else None
            })
    
    return failures

# Analyze and report
failures = analyze_failure_modes(pipeline, questions, threshold=0.3)
print(f"\nFound {len(failures)} failure cases:")
for f in failures[:5]:  # Show first 5
    print(f"\nQuery: {f['query']}")
    print(f"Recall: {f['recall']:.2%}")
    print(f"Expected: {f['expected']}")
    print(f"Retrieved: {f['retrieved']}")
```

## Troubleshooting

### BM25 Returns Low Scores

**Issue**: All BM25 scores are near zero or negative.

**Solution**: Check tokenization and ensure corpus is properly preprocessed:

```python
# Debug tokenization
sample_query = "machine learning"
tokenized = sample_query.lower().split()
print(f"Tokenized query: {tokenized}")

# Check corpus tokens
print(f"First doc tokens: {tokenized_corpus[0][:10]}")

# Verify BM25 parameters (can tune k1 and b)
from rank_bm25 import BM25Okapi
bm25_custom = BM25Okapi(tokenized_corpus, k1=1.5, b=0.75)
```

### Empty Chunks After Hierarchical Split

**Issue**: Some documents produce no paragraph-level chunks.

**Solution**: Adjust minimum length thresholds and splitting logic:

```python
def hierarchical_chunking(document, doc_id, min_section_len=30, min_para_len=10):
    # ... (previous code)
    
    # More lenient filtering
    for sec_idx, section in enumerate(sections):
        if len(section.strip()) < min_section_len:
            continue
        # ... rest of logic
```

### FAISS Index Dimension Mismatch

**Issue**: `Error: dimension mismatch` when searching FAISS index.

**Solution**: Ensure query embeddings match index dimension:

```python
# Check dimensions
print(f"Index dimension: {pipeline.faiss_index.d}")

query_emb = pipeline.embedding_model.encode([query])
print(f"Query embedding shape: {query_emb.shape}")

# Verify model consistency
assert query_emb.shape[1] == pipeline.faiss_index.d, "Dimension mismatch!"
```

### Poor Recall on Golden Dataset

**Issue**: Recall@10 is unexpectedly low.

**Solution**: Validate golden dataset format and tune hybrid weights:

```python
# Validate questions.jsonl
with open('questions.jsonl', 'r') as f:
    for i, line in enumerate(f, 1):
        try:
            q = json.loads(line)
            assert 'query' in q and 'relevant_docs' in q
        except Exception as e:
            print(f"Line {i} error: {e}")

# Experiment with alpha values
for alpha in [0.0, 0.3, 0.5, 0.7, 1.0]:
    results = pipeline.retrieve(query, alpha=alpha)
    print(f"Alpha={alpha}: Top result from {results[0]['chunk']['doc_id']}")
```

## Environment Variables

The project uses local models by default but can be configured via environment:

```bash
# Sentence transformer model
export EMBEDDING_MODEL="all-MiniLM-L6-v2"

# Corpus directory
export CORPUS_DIR="./corpus"

# Golden dataset path
export GOLDEN_DATASET="./questions.jsonl"

# Chunk size limits
export MIN_SECTION_LENGTH="50"
export MIN_PARAGRAPH_LENGTH="20"
```

## Best Practices

1. **Always validate golden datasets** before evaluation runs
2. **Use deterministic chunk IDs** for reproducibility and debugging
3. **Experiment with granularity levels** based on query complexity
4. **Tune hybrid weights (alpha)** for your specific domain
5. **Maintain parent-child linkage** for context expansion
6. **Log failure modes** to identify corpus or chunking issues
7. **Normalize embeddings** before FAISS indexing for cosine similarity

## References

- Project Repository: https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap
- Day 01: Retrieval Contracts & Golden Datasets
- Day 02: BM25 Baseline Implementation
- Day 03: Hierarchical Chunking & RAG Pipeline
