---
name: ai-data-engineering-rag-pipeline
description: Build production-grade local RAG pipelines with BM25 baseline search, hierarchical chunking, and evaluation frameworks
triggers:
  - how do I build a local RAG pipeline
  - implement BM25 baseline search engine
  - create hierarchical chunking for RAG
  - evaluate retrieval performance with recall metrics
  - set up a production RAG system
  - build inverted index for document search
  - implement chunking strategies for embeddings
  - design RAG evaluation contracts
---

# AI Data Engineering RAG Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build production-grade local RAG (Retrieval-Augmented Generation) pipelines using the patterns and architectures from the ai-data-engineering-roadmap project. The project provides baseline search implementations, hierarchical chunking strategies, and evaluation frameworks for production RAG systems.

## What This Project Does

The ai-data-engineering-roadmap is a hands-on implementation guide for building RAG pipelines from scratch, covering:

- **Local RAG Pipeline Architecture**: Governed corpus design, golden datasets, and retrieval contracts
- **BM25 Baseline Search**: Okapi BM25 ranking with inverted index for baseline performance
- **Hierarchical Chunking**: Multi-level document segmentation (document, section, paragraph) with parent-child metadata
- **Evaluation Frameworks**: Recall@K metrics, golden query datasets, and deterministic testing

## Installation & Setup

This is a reference implementation project meant to be studied and adapted. Clone the repository:

```bash
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap
```

Each day's module is self-contained with its own dependencies. Install Python dependencies:

```bash
pip install numpy pandas nltk rank_bm25
```

For text processing:

```python
import nltk
nltk.download('punkt')
```

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/  # RAG Pipeline & Retrieval Contracts
├── Day_02/  # BM25 Baseline Search Engine
└── Day_03/  # Hierarchical Chunking Architecture
```

## Day 01: RAG Pipeline & Retrieval Contracts

### Core Concepts

1. **Governed Corpus**: Structured document collection with metadata
2. **Golden Dataset**: `questions.jsonl` with ground-truth query-document pairs
3. **Retrieval Contract**: Formal interface defining input/output schemas
4. **Evaluation Script**: Automated testing against golden queries

### Golden Dataset Format

```python
# questions.jsonl structure
{
    "query_id": "q001",
    "query": "What is kubernetes?",
    "expected_doc_ids": ["doc_k8s_001", "doc_container_orchestration"],
    "domain": "cloud-computing"
}
```

### Retrieval Contract Implementation

```python
from typing import List, Dict, Any
from dataclasses import dataclass

@dataclass
class RetrievalRequest:
    query: str
    top_k: int = 10
    domain_filter: str = None

@dataclass
class RetrievalResult:
    doc_id: str
    score: float
    content: str
    metadata: Dict[str, Any]

class RetrievalContract:
    """Formal contract for RAG retrieval systems"""
    
    def retrieve(self, request: RetrievalRequest) -> List[RetrievalResult]:
        """
        Retrieve top-k documents for a given query
        
        Args:
            request: RetrievalRequest with query and parameters
            
        Returns:
            List of RetrievalResult ordered by relevance score
        """
        raise NotImplementedError
    
    def evaluate(self, golden_dataset: str) -> Dict[str, float]:
        """
        Evaluate against golden dataset
        
        Args:
            golden_dataset: Path to questions.jsonl
            
        Returns:
            Metrics dict with recall@k, precision@k, MRR
        """
        raise NotImplementedError
```

### Evaluation Script Pattern

```python
import json
from pathlib import Path

def evaluate_retrieval(retriever, golden_dataset_path: str, k: int = 10):
    """Evaluate retrieval system against golden queries"""
    
    results = {
        "recall_at_k": [],
        "precision_at_k": [],
        "mrr": []
    }
    
    with open(golden_dataset_path, 'r') as f:
        for line in f:
            query_item = json.loads(line)
            
            # Run retrieval
            request = RetrievalRequest(
                query=query_item["query"],
                top_k=k,
                domain_filter=query_item.get("domain")
            )
            retrieved = retriever.retrieve(request)
            retrieved_ids = [r.doc_id for r in retrieved]
            
            # Calculate metrics
            expected_ids = set(query_item["expected_doc_ids"])
            retrieved_set = set(retrieved_ids[:k])
            
            # Recall@K
            recall = len(expected_ids & retrieved_set) / len(expected_ids)
            results["recall_at_k"].append(recall)
            
            # Precision@K
            precision = len(expected_ids & retrieved_set) / k
            results["precision_at_k"].append(precision)
            
            # MRR
            for i, doc_id in enumerate(retrieved_ids, 1):
                if doc_id in expected_ids:
                    results["mrr"].append(1.0 / i)
                    break
            else:
                results["mrr"].append(0.0)
    
    # Aggregate
    return {
        "recall@{k}": sum(results["recall_at_k"]) / len(results["recall_at_k"]),
        "precision@{k}": sum(results["precision_at_k"]) / len(results["precision_at_k"]),
        "MRR": sum(results["mrr"]) / len(results["mrr"])
    }
```

## Day 02: BM25 Baseline Search Engine

### Core Implementation

BM25 (Okapi BM25) baseline provides a strong traditional IR baseline before moving to neural methods.

```python
from rank_bm25 import BM25Okapi
import nltk
from nltk.tokenize import word_tokenize
from typing import List, Tuple

class BM25Retriever:
    """BM25 baseline search engine with inverted index"""
    
    def __init__(self, corpus: List[Dict[str, str]]):
        """
        Initialize BM25 retriever
        
        Args:
            corpus: List of documents with 'id', 'content', 'metadata'
        """
        self.corpus = corpus
        self.doc_ids = [doc['id'] for doc in corpus]
        
        # Tokenize corpus
        tokenized_corpus = [
            self._tokenize(doc['content']) 
            for doc in corpus
        ]
        
        # Build BM25 index
        self.bm25 = BM25Okapi(tokenized_corpus)
    
    def _tokenize(self, text: str) -> List[str]:
        """Tokenize and preprocess text"""
        tokens = word_tokenize(text.lower())
        # Remove punctuation and short tokens
        return [t for t in tokens if t.isalnum() and len(t) > 2]
    
    def search(self, query: str, top_k: int = 10) -> List[Tuple[str, float]]:
        """
        Search corpus using BM25 ranking
        
        Args:
            query: Search query string
            top_k: Number of results to return
            
        Returns:
            List of (doc_id, score) tuples
        """
        tokenized_query = self._tokenize(query)
        scores = self.bm25.get_scores(tokenized_query)
        
        # Get top-k results
        top_indices = sorted(
            range(len(scores)), 
            key=lambda i: scores[i], 
            reverse=True
        )[:top_k]
        
        return [
            (self.doc_ids[i], scores[i]) 
            for i in top_indices
        ]
    
    def retrieve(self, request: RetrievalRequest) -> List[RetrievalResult]:
        """Implement RetrievalContract interface"""
        search_results = self.search(request.query, request.top_k)
        
        return [
            RetrievalResult(
                doc_id=doc_id,
                score=score,
                content=self.corpus[self.doc_ids.index(doc_id)]['content'],
                metadata=self.corpus[self.doc_ids.index(doc_id)].get('metadata', {})
            )
            for doc_id, score in search_results
        ]
```

### BM25 Baseline Evaluation

```python
# baseline_bm25.py usage pattern
def run_bm25_baseline():
    # Load corpus
    corpus = [
        {
            "id": "doc_001",
            "content": "Kubernetes is a container orchestration platform...",
            "metadata": {"domain": "cloud-computing"}
        },
        {
            "id": "doc_002",
            "content": "Docker containers provide lightweight virtualization...",
            "metadata": {"domain": "containerization"}
        }
    ]
    
    # Initialize BM25 retriever
    retriever = BM25Retriever(corpus)
    
    # Evaluate on golden dataset
    metrics = evaluate_retrieval(
        retriever, 
        "questions.jsonl", 
        k=10
    )
    
    print(f"BM25 Baseline Results:")
    print(f"  Recall@10: {metrics['recall@10']:.3f}")
    print(f"  Precision@10: {metrics['precision@10']:.3f}")
    print(f"  MRR: {metrics['MRR']:.3f}")
    
    return metrics
```

## Day 03: Hierarchical Chunking Architecture

### Chunking Strategy

Multi-level granularity for optimal retrieval: Document → Section → Paragraph

```python
from typing import List, Dict
import hashlib
import re

class ChunkMetadata:
    """Deterministic chunk metadata with parent-child linkage"""
    
    @staticmethod
    def generate_chunk_id(content: str, level: str, parent_id: str = None) -> str:
        """Generate deterministic chunk ID"""
        content_hash = hashlib.md5(content.encode()).hexdigest()[:8]
        if parent_id:
            return f"{parent_id}_{level}_{content_hash}"
        return f"{level}_{content_hash}"

class HierarchicalChunker:
    """Hierarchical document chunking with parent-child metadata"""
    
    def __init__(self, chunk_levels: List[str] = ["document", "section", "paragraph"]):
        self.chunk_levels = chunk_levels
    
    def chunk_document(self, doc_id: str, content: str) -> List[Dict]:
        """
        Chunk document into hierarchical levels
        
        Args:
            doc_id: Unique document identifier
            content: Full document text
            
        Returns:
            List of chunk objects with metadata
        """
        chunks = []
        
        # Level 1: Document
        doc_chunk_id = ChunkMetadata.generate_chunk_id(content, "document")
        chunks.append({
            "chunk_id": doc_chunk_id,
            "doc_id": doc_id,
            "level": "document",
            "content": content,
            "parent_id": None,
            "char_count": len(content)
        })
        
        # Level 2: Sections (split on double newlines or headers)
        sections = re.split(r'\n\n+|(?=^#{1,3}\s)', content, flags=re.MULTILINE)
        sections = [s.strip() for s in sections if s.strip()]
        
        for section in sections:
            section_id = ChunkMetadata.generate_chunk_id(
                section, "section", doc_chunk_id
            )
            chunks.append({
                "chunk_id": section_id,
                "doc_id": doc_id,
                "level": "section",
                "content": section,
                "parent_id": doc_chunk_id,
                "char_count": len(section)
            })
            
            # Level 3: Paragraphs
            paragraphs = [p.strip() for p in section.split('\n') if p.strip()]
            for paragraph in paragraphs:
                if len(paragraph) < 50:  # Skip very short paragraphs
                    continue
                    
                para_id = ChunkMetadata.generate_chunk_id(
                    paragraph, "paragraph", section_id
                )
                chunks.append({
                    "chunk_id": para_id,
                    "doc_id": doc_id,
                    "level": "paragraph",
                    "content": paragraph,
                    "parent_id": section_id,
                    "char_count": len(paragraph)
                })
        
        return chunks

class HierarchicalRetriever:
    """Retrieve with parent-child context expansion"""
    
    def __init__(self, chunks: List[Dict], base_retriever):
        self.chunks = chunks
        self.chunk_index = {c["chunk_id"]: c for c in chunks}
        self.base_retriever = base_retriever
    
    def retrieve_with_context(
        self, 
        query: str, 
        top_k: int = 10,
        expand_to_parent: bool = True
    ) -> List[Dict]:
        """
        Retrieve chunks and optionally expand to parent context
        
        Args:
            query: Search query
            top_k: Number of results
            expand_to_parent: Whether to include parent chunk content
            
        Returns:
            List of chunks with expanded context
        """
        # Base retrieval on paragraph level
        results = self.base_retriever.search(query, top_k)
        
        expanded_results = []
        for chunk_id, score in results:
            chunk = self.chunk_index.get(chunk_id)
            if not chunk:
                continue
            
            result = {
                "chunk_id": chunk_id,
                "score": score,
                "content": chunk["content"],
                "level": chunk["level"]
            }
            
            # Expand to parent context
            if expand_to_parent and chunk["parent_id"]:
                parent = self.chunk_index.get(chunk["parent_id"])
                if parent:
                    result["parent_content"] = parent["content"]
                    result["parent_id"] = parent["chunk_id"]
            
            expanded_results.append(result)
        
        return expanded_results
```

### Chunking Evaluation Pattern

```python
def evaluate_chunking_strategies(corpus: List[Dict], queries: List[str]):
    """Compare different chunking granularities"""
    
    chunker = HierarchicalChunker()
    
    results = {}
    for level in ["document", "section", "paragraph"]:
        # Extract chunks at specific level
        level_chunks = []
        for doc in corpus:
            chunks = chunker.chunk_document(doc["id"], doc["content"])
            level_chunks.extend([c for c in chunks if c["level"] == level])
        
        # Build retriever for this level
        retriever = BM25Retriever([
            {"id": c["chunk_id"], "content": c["content"]}
            for c in level_chunks
        ])
        
        # Evaluate
        metrics = evaluate_retrieval(retriever, "questions.jsonl", k=10)
        results[level] = metrics
    
    # Compare results
    print("Chunking Strategy Comparison:")
    for level, metrics in results.items():
        print(f"\n{level.upper()} Level:")
        print(f"  Recall@10: {metrics['recall@10']:.3f}")
        print(f"  Precision@10: {metrics['precision@10']:.3f}")
    
    return results
```

## Common Patterns

### 1. Building a Complete RAG Pipeline

```python
from pathlib import Path
import json

class RAGPipeline:
    """Production RAG pipeline with evaluation"""
    
    def __init__(self, corpus_path: str, chunk_level: str = "paragraph"):
        # Load corpus
        self.corpus = self._load_corpus(corpus_path)
        
        # Chunk documents
        chunker = HierarchicalChunker()
        self.chunks = []
        for doc in self.corpus:
            chunks = chunker.chunk_document(doc["id"], doc["content"])
            self.chunks.extend([c for c in chunks if c["level"] == chunk_level])
        
        # Build retriever
        self.retriever = BM25Retriever([
            {"id": c["chunk_id"], "content": c["content"], "metadata": c}
            for c in self.chunks
        ])
    
    def _load_corpus(self, path: str) -> List[Dict]:
        """Load corpus from JSON/JSONL"""
        corpus = []
        with open(path, 'r') as f:
            for line in f:
                corpus.append(json.loads(line))
        return corpus
    
    def query(self, text: str, top_k: int = 5) -> List[Dict]:
        """Execute RAG query"""
        request = RetrievalRequest(query=text, top_k=top_k)
        return self.retriever.retrieve(request)
    
    def evaluate(self, golden_dataset: str) -> Dict[str, float]:
        """Run full evaluation"""
        return evaluate_retrieval(self.retriever, golden_dataset, k=10)
```

### 2. Interactive Retrieval Simulation

```python
def interactive_rag_session(pipeline: RAGPipeline):
    """Interactive RAG testing session"""
    
    print("RAG Pipeline Interactive Session")
    print("Type 'quit' to exit\n")
    
    while True:
        query = input("Query: ").strip()
        if query.lower() == 'quit':
            break
        
        if not query:
            continue
        
        # Retrieve
        results = pipeline.query(query, top_k=3)
        
        print(f"\nTop {len(results)} Results:")
        for i, result in enumerate(results, 1):
            print(f"\n[{i}] Score: {result.score:.3f}")
            print(f"Chunk ID: {result.doc_id}")
            print(f"Content: {result.content[:200]}...")
            if "parent_content" in result.metadata:
                print(f"Parent Context: {result.metadata['parent_content'][:100]}...")
        print("\n" + "="*80 + "\n")
```

### 3. Failure Mode Analysis

```python
def analyze_retrieval_failures(pipeline: RAGPipeline, golden_dataset: str):
    """Identify and analyze retrieval failures"""
    
    failures = []
    
    with open(golden_dataset, 'r') as f:
        for line in f:
            query_item = json.loads(line)
            
            results = pipeline.query(query_item["query"], top_k=10)
            retrieved_ids = [r.doc_id for r in results]
            expected_ids = set(query_item["expected_doc_ids"])
            
            # Check for failures
            if not any(doc_id in expected_ids for doc_id in retrieved_ids):
                failures.append({
                    "query": query_item["query"],
                    "expected": list(expected_ids),
                    "retrieved": retrieved_ids[:5],
                    "query_id": query_item["query_id"]
                })
    
    print(f"\nFailure Analysis: {len(failures)} total failures")
    for failure in failures[:5]:  # Show first 5
        print(f"\nQuery: {failure['query']}")
        print(f"Expected: {failure['expected']}")
        print(f"Retrieved: {failure['retrieved']}")
    
    return failures
```

## Configuration

### Environment Variables

```python
import os

# Retrieval parameters
TOP_K = int(os.getenv("RAG_TOP_K", "10"))
CHUNK_LEVEL = os.getenv("RAG_CHUNK_LEVEL", "paragraph")
EXPAND_CONTEXT = os.getenv("RAG_EXPAND_CONTEXT", "true").lower() == "true"

# Corpus paths
CORPUS_PATH = os.getenv("RAG_CORPUS_PATH", "corpus.jsonl")
GOLDEN_DATASET_PATH = os.getenv("RAG_GOLDEN_DATASET", "questions.jsonl")

# BM25 parameters
BM25_K1 = float(os.getenv("BM25_K1", "1.5"))
BM25_B = float(os.getenv("BM25_B", "0.75"))
```

## Troubleshooting

### Low Recall Scores

```python
# Diagnose low recall
def diagnose_low_recall(pipeline: RAGPipeline, query: str, expected_docs: List[str]):
    """Debug why expected docs aren't retrieved"""
    
    # Get top 50 results
    results = pipeline.query(query, top_k=50)
    retrieved_ids = [r.doc_id for r in results]
    
    print(f"Query: {query}")
    print(f"Expected docs: {expected_docs}\n")
    
    for doc_id in expected_docs:
        if doc_id in retrieved_ids:
            rank = retrieved_ids.index(doc_id) + 1
            score = results[retrieved_ids.index(doc_id)].score
            print(f"✓ {doc_id} found at rank {rank} (score: {score:.3f})")
        else:
            print(f"✗ {doc_id} NOT FOUND in top 50")
            # Check if doc exists
            if doc_id in pipeline.retriever.doc_ids:
                print(f"  Document exists but ranked too low")
            else:
                print(f"  Document not in corpus!")
```

### Chunk Size Analysis

```python
def analyze_chunk_sizes(chunks: List[Dict]):
    """Analyze chunk size distribution"""
    
    import statistics
    
    by_level = {}
    for chunk in chunks:
        level = chunk["level"]
        if level not in by_level:
            by_level[level] = []
        by_level[level].append(chunk["char_count"])
    
    print("Chunk Size Analysis:")
    for level, sizes in by_level.items():
        print(f"\n{level.upper()}:")
        print(f"  Count: {len(sizes)}")
        print(f"  Mean: {statistics.mean(sizes):.0f} chars")
        print(f"  Median: {statistics.median(sizes):.0f} chars")
        print(f"  Min: {min(sizes)} chars")
        print(f"  Max: {max(sizes)} chars")
```

### Tokenization Issues

```python
# Handle tokenization errors
try:
    retriever = BM25Retriever(corpus)
except LookupError:
    print("NLTK punkt tokenizer not found. Downloading...")
    import nltk
    nltk.download('punkt')
    retriever = BM25Retriever(corpus)
```

## Best Practices

1. **Golden Dataset Quality**: Ensure `questions.jsonl` has diverse queries and multiple expected documents per query
2. **Chunk Determinism**: Always use deterministic chunk IDs for reproducible evaluation
3. **Baseline First**: Establish BM25 baseline before moving to neural methods
4. **Parent Context**: For small chunks (paragraph level), always link to parent context
5. **Evaluation Metrics**: Track multiple metrics (Recall@K, Precision@K, MRR) not just one
6. **Failure Analysis**: Regularly analyze retrieval failures to identify systematic issues

This skill provides the foundation for building and evaluating production RAG systems with strong baseline performance and structured evaluation frameworks.
