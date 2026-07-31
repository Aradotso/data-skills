---
name: ai-data-engineering-rag-pipeline
description: Build production-grade local RAG pipelines with BM25 search, hierarchical chunking, and retrieval evaluation frameworks
triggers:
  - how do I build a local RAG pipeline
  - implement BM25 search for document retrieval
  - create hierarchical chunking for RAG
  - evaluate retrieval performance with recall metrics
  - set up a document retrieval system
  - build a baseline search engine
  - implement chunking strategies for embeddings
  - evaluate RAG pipeline performance
---

# AI Data Engineering RAG Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides a production-grade framework for building local Retrieval-Augmented Generation (RAG) pipelines from scratch. It focuses on foundational search and retrieval techniques including BM25 baseline search, hierarchical chunking architectures, and systematic evaluation using recall metrics.

**Key Components:**
- **Day 01**: Retrieval contracts, golden datasets, and evaluation frameworks
- **Day 02**: BM25 baseline search engine with Okapi ranking
- **Day 03**: Hierarchical chunking with parent-child metadata linkage

## Installation

```bash
# Clone the repository
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap

# Install dependencies (typical Python setup)
pip install -r requirements.txt
```

Common dependencies for RAG pipelines:
```bash
pip install rank-bm25 numpy pandas jsonlines
```

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/           # Retrieval contracts & golden datasets
│   ├── questions.jsonl
│   └── evaluation.py
├── Day_02/           # BM25 baseline search
│   └── baseline_bm25.py
├── Day_03/           # Hierarchical chunking
│   └── pipeline.py
```

## Day 01: Retrieval Contracts & Evaluation

### Creating Golden Datasets

Golden datasets define query-document pairs for evaluation:

```python
import jsonlines

# Structure for questions.jsonl
golden_dataset = [
    {
        "question_id": "q001",
        "query": "What is the capital of France?",
        "relevant_doc_ids": ["doc_123", "doc_456"],
        "context": "geography"
    },
    {
        "question_id": "q002",
        "query": "How does photosynthesis work?",
        "relevant_doc_ids": ["doc_789"],
        "context": "biology"
    }
]

# Write golden dataset
with jsonlines.open('questions.jsonl', 'w') as writer:
    writer.write_all(golden_dataset)
```

### Evaluation Script Pattern

```python
import jsonlines

def load_golden_dataset(filepath):
    """Load questions and relevant documents"""
    with jsonlines.open(filepath) as reader:
        return list(reader)

def evaluate_retrieval(retrieved_docs, relevant_docs, k=10):
    """Calculate Recall@K metric"""
    retrieved_set = set(retrieved_docs[:k])
    relevant_set = set(relevant_docs)
    
    if not relevant_set:
        return 0.0
    
    hits = len(retrieved_set.intersection(relevant_set))
    recall = hits / len(relevant_set)
    return recall

# Usage
golden_data = load_golden_dataset('questions.jsonl')
recalls = []

for item in golden_data:
    query = item['query']
    relevant = item['relevant_doc_ids']
    
    # Your retrieval function here
    retrieved = your_search_function(query)
    
    recall = evaluate_retrieval(retrieved, relevant, k=10)
    recalls.append(recall)

avg_recall = sum(recalls) / len(recalls)
print(f"Average Recall@10: {avg_recall:.3f}")
```

## Day 02: BM25 Baseline Search Engine

### Implementing BM25 Search

```python
from rank_bm25 import BM25Okapi
import numpy as np

class BM25SearchEngine:
    def __init__(self, corpus):
        """
        Initialize BM25 search engine
        
        Args:
            corpus: List of documents (strings)
        """
        self.corpus = corpus
        self.tokenized_corpus = [doc.lower().split() for doc in corpus]
        self.bm25 = BM25Okapi(self.tokenized_corpus)
    
    def search(self, query, top_k=10):
        """
        Search corpus using BM25
        
        Args:
            query: Search query string
            top_k: Number of results to return
            
        Returns:
            List of (doc_index, score) tuples
        """
        tokenized_query = query.lower().split()
        scores = self.bm25.get_scores(tokenized_query)
        
        # Get top K indices
        top_indices = np.argsort(scores)[::-1][:top_k]
        
        results = [
            (idx, scores[idx]) 
            for idx in top_indices
        ]
        return results

# Usage example
corpus = [
    "The quick brown fox jumps over the lazy dog",
    "A fast auburn fox leaps above an idle canine",
    "Python is a high-level programming language",
    "Machine learning models require training data"
]

engine = BM25SearchEngine(corpus)
results = engine.search("fox jumps", top_k=2)

for doc_idx, score in results:
    print(f"Score: {score:.3f} | Doc: {corpus[doc_idx]}")
```

### BM25 with Document IDs

```python
class BM25WithIDs:
    def __init__(self, documents):
        """
        Args:
            documents: List of dicts with 'id' and 'text' keys
        """
        self.documents = documents
        self.doc_ids = [doc['id'] for doc in documents]
        self.corpus = [doc['text'] for doc in documents]
        
        tokenized = [text.lower().split() for text in self.corpus]
        self.bm25 = BM25Okapi(tokenized)
    
    def search(self, query, top_k=10):
        """Returns list of document IDs ranked by relevance"""
        tokenized_query = query.lower().split()
        scores = self.bm25.get_scores(tokenized_query)
        
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [self.doc_ids[idx] for idx in top_indices]

# Usage
docs = [
    {"id": "doc_001", "text": "Cloud computing enables scalable infrastructure"},
    {"id": "doc_002", "text": "Cybersecurity protects against digital threats"},
    {"id": "doc_003", "text": "DevSecOps integrates security into development"}
]

search_engine = BM25WithIDs(docs)
results = search_engine.search("security development", top_k=3)
print("Top results:", results)
```

### Recall@K Evaluation with BM25

```python
def evaluate_bm25_recall(search_engine, golden_dataset, k=10):
    """Evaluate BM25 search using Recall@K"""
    recalls = []
    
    for item in golden_dataset:
        query = item['query']
        relevant_ids = set(item['relevant_doc_ids'])
        
        # Get top K results
        retrieved_ids = set(search_engine.search(query, top_k=k))
        
        # Calculate recall
        hits = len(retrieved_ids.intersection(relevant_ids))
        recall = hits / len(relevant_ids) if relevant_ids else 0
        recalls.append(recall)
    
    return {
        'mean_recall': np.mean(recalls),
        'median_recall': np.median(recalls),
        'min_recall': np.min(recalls),
        'max_recall': np.max(recalls)
    }

# Example
metrics = evaluate_bm25_recall(search_engine, golden_data, k=10)
print(f"Mean Recall@10: {metrics['mean_recall']:.3f}")
```

## Day 03: Hierarchical Chunking Architecture

### Document Chunking Strategy

```python
from typing import List, Dict
import hashlib

class HierarchicalChunker:
    def __init__(self):
        self.chunk_levels = {
            'document': [],
            'section': [],
            'paragraph': []
        }
    
    def generate_chunk_id(self, text: str, level: str, parent_id: str = None) -> str:
        """Generate deterministic chunk ID"""
        hash_input = f"{level}:{text[:100]}"
        if parent_id:
            hash_input = f"{parent_id}:{hash_input}"
        
        return hashlib.md5(hash_input.encode()).hexdigest()[:12]
    
    def chunk_document(self, document: Dict) -> List[Dict]:
        """
        Create hierarchical chunks from document
        
        Args:
            document: Dict with 'id', 'title', 'sections'
            
        Returns:
            List of chunk dictionaries with metadata
        """
        chunks = []
        doc_id = document['id']
        
        # Document-level chunk
        doc_chunk = {
            'chunk_id': self.generate_chunk_id(document['title'], 'document'),
            'level': 'document',
            'text': document['title'],
            'parent_id': None,
            'doc_id': doc_id,
            'metadata': {
                'title': document['title']
            }
        }
        chunks.append(doc_chunk)
        
        # Section-level chunks
        for section in document.get('sections', []):
            section_chunk = {
                'chunk_id': self.generate_chunk_id(section['heading'], 'section', doc_chunk['chunk_id']),
                'level': 'section',
                'text': f"{section['heading']}\n{section['content']}",
                'parent_id': doc_chunk['chunk_id'],
                'doc_id': doc_id,
                'metadata': {
                    'heading': section['heading']
                }
            }
            chunks.append(section_chunk)
            
            # Paragraph-level chunks
            paragraphs = section['content'].split('\n\n')
            for para in paragraphs:
                if para.strip():
                    para_chunk = {
                        'chunk_id': self.generate_chunk_id(para, 'paragraph', section_chunk['chunk_id']),
                        'level': 'paragraph',
                        'text': para,
                        'parent_id': section_chunk['chunk_id'],
                        'doc_id': doc_id,
                        'metadata': {
                            'section': section['heading']
                        }
                    }
                    chunks.append(para_chunk)
        
        return chunks

# Usage
chunker = HierarchicalChunker()

document = {
    'id': 'doc_cloud_001',
    'title': 'Introduction to Cloud Computing',
    'sections': [
        {
            'heading': 'What is Cloud Computing',
            'content': 'Cloud computing is the delivery of computing services.\n\nIt includes servers, storage, and databases.'
        },
        {
            'heading': 'Benefits',
            'content': 'Cost reduction and scalability.\n\nImproved collaboration and flexibility.'
        }
    ]
}

chunks = chunker.chunk_document(document)
for chunk in chunks:
    print(f"Level: {chunk['level']}, ID: {chunk['chunk_id']}, Parent: {chunk['parent_id']}")
```

### Multi-Level Retrieval Pipeline

```python
class HierarchicalRAGPipeline:
    def __init__(self, chunks: List[Dict]):
        """
        Initialize pipeline with hierarchical chunks
        
        Args:
            chunks: List of chunk dictionaries
        """
        self.chunks = chunks
        self.chunk_by_id = {c['chunk_id']: c for c in chunks}
        
        # Create BM25 index per level
        self.indices = {}
        for level in ['document', 'section', 'paragraph']:
            level_chunks = [c for c in chunks if c['level'] == level]
            if level_chunks:
                self.indices[level] = BM25WithIDs([
                    {'id': c['chunk_id'], 'text': c['text']}
                    for c in level_chunks
                ])
    
    def retrieve(self, query: str, level: str = 'paragraph', k: int = 5):
        """Retrieve chunks at specified granularity"""
        if level not in self.indices:
            return []
        
        chunk_ids = self.indices[level].search(query, top_k=k)
        return [self.chunk_by_id[cid] for cid in chunk_ids]
    
    def retrieve_with_context(self, query: str, k: int = 5):
        """Retrieve paragraphs with parent section context"""
        para_chunks = self.retrieve(query, level='paragraph', k=k)
        
        enriched = []
        for chunk in para_chunks:
            parent_id = chunk['parent_id']
            parent = self.chunk_by_id.get(parent_id, {})
            
            enriched.append({
                'text': chunk['text'],
                'metadata': chunk['metadata'],
                'context': parent.get('text', '')
            })
        
        return enriched

# Usage
all_chunks = chunker.chunk_document(document)
pipeline = HierarchicalRAGPipeline(all_chunks)

# Basic retrieval
results = pipeline.retrieve("cloud computing benefits", level='section', k=2)
for result in results:
    print(f"Section: {result['text'][:100]}")

# Retrieval with context
contextual_results = pipeline.retrieve_with_context("scalability", k=3)
for result in contextual_results:
    print(f"Text: {result['text']}")
    print(f"Context: {result['context'][:100]}")
```

### Chunk Granularity Evaluation

```python
def evaluate_chunk_granularity(pipeline, golden_dataset, levels=['document', 'section', 'paragraph']):
    """Compare retrieval performance across chunk levels"""
    results = {}
    
    for level in levels:
        recalls = []
        
        for item in golden_dataset:
            query = item['query']
            relevant_ids = set(item['relevant_doc_ids'])
            
            # Retrieve at this level
            chunks = pipeline.retrieve(query, level=level, k=10)
            retrieved_doc_ids = set([c['doc_id'] for c in chunks])
            
            # Calculate recall
            hits = len(retrieved_doc_ids.intersection(relevant_ids))
            recall = hits / len(relevant_ids) if relevant_ids else 0
            recalls.append(recall)
        
        results[level] = {
            'mean_recall': np.mean(recalls),
            'std_recall': np.std(recalls)
        }
    
    return results

# Compare granularities
granularity_results = evaluate_chunk_granularity(pipeline, golden_data)
for level, metrics in granularity_results.items():
    print(f"{level.capitalize()}: Recall@10 = {metrics['mean_recall']:.3f} ± {metrics['std_recall']:.3f}")
```

## Configuration Patterns

### Environment Variables

```python
import os

# Configuration
CONFIG = {
    'corpus_path': os.getenv('CORPUS_PATH', './data/corpus.jsonl'),
    'golden_dataset_path': os.getenv('GOLDEN_DATASET_PATH', './data/questions.jsonl'),
    'chunk_size': int(os.getenv('CHUNK_SIZE', '512')),
    'chunk_overlap': int(os.getenv('CHUNK_OVERLAP', '50')),
    'top_k': int(os.getenv('TOP_K', '10')),
    'bm25_k1': float(os.getenv('BM25_K1', '1.5')),
    'bm25_b': float(os.getenv('BM25_B', '0.75'))
}
```

### Loading Corpus Data

```python
import jsonlines

def load_corpus(filepath):
    """Load document corpus from JSONL file"""
    with jsonlines.open(filepath) as reader:
        return list(reader)

def load_multi_domain_corpus(base_path):
    """Load corpus from multiple domain files"""
    domains = ['cloud', 'cybersecurity', 'devops', 'ai']
    corpus = []
    
    for domain in domains:
        filepath = f"{base_path}/{domain}_docs.jsonl"
        if os.path.exists(filepath):
            with jsonlines.open(filepath) as reader:
                docs = list(reader)
                for doc in docs:
                    doc['domain'] = domain
                corpus.extend(docs)
    
    return corpus
```

## Common Patterns

### End-to-End RAG Pipeline

```python
def build_rag_pipeline(corpus_path, golden_dataset_path):
    """Complete RAG pipeline from corpus to evaluation"""
    
    # 1. Load data
    corpus = load_corpus(corpus_path)
    golden_data = load_golden_dataset(golden_dataset_path)
    
    # 2. Create hierarchical chunks
    chunker = HierarchicalChunker()
    all_chunks = []
    for doc in corpus:
        chunks = chunker.chunk_document(doc)
        all_chunks.extend(chunks)
    
    # 3. Build retrieval pipeline
    pipeline = HierarchicalRAGPipeline(all_chunks)
    
    # 4. Evaluate
    results = evaluate_chunk_granularity(pipeline, golden_data)
    
    return pipeline, results

# Usage
pipeline, eval_results = build_rag_pipeline(
    'data/corpus.jsonl',
    'data/questions.jsonl'
)
```

### Batch Processing

```python
def batch_retrieve(pipeline, queries, level='paragraph', k=5):
    """Process multiple queries efficiently"""
    results = []
    
    for query in queries:
        chunks = pipeline.retrieve(query, level=level, k=k)
        results.append({
            'query': query,
            'results': chunks
        })
    
    return results

# Usage
queries = ["cloud security", "DevOps practices", "AI training data"]
batch_results = batch_retrieve(pipeline, queries, k=3)
```

## Troubleshooting

### Low Recall Issues

```python
# Diagnose retrieval failures
def diagnose_retrieval(pipeline, query, relevant_doc_ids):
    """Analyze why relevant docs weren't retrieved"""
    
    # Try different levels
    for level in ['document', 'section', 'paragraph']:
        chunks = pipeline.retrieve(query, level=level, k=20)
        retrieved_docs = set([c['doc_id'] for c in chunks])
        
        missing = set(relevant_doc_ids) - retrieved_docs
        print(f"\nLevel: {level}")
        print(f"Retrieved: {len(retrieved_docs)} docs")
        print(f"Missing: {missing}")
        
        if chunks:
            print(f"Top result: {chunks[0]['text'][:100]}")
```

### Token Mismatch

```python
# Ensure consistent tokenization
def normalize_text(text):
    """Standardize text preprocessing"""
    text = text.lower()
    text = text.strip()
    # Add domain-specific normalization
    return text

# Use in both indexing and querying
class NormalizedBM25:
    def __init__(self, documents):
        self.documents = documents
        normalized_texts = [normalize_text(doc['text']) for doc in documents]
        tokenized = [text.split() for text in normalized_texts]
        self.bm25 = BM25Okapi(tokenized)
```

### Memory Optimization

```python
# For large corpora, use generators
def chunk_corpus_generator(corpus, chunker):
    """Generate chunks without loading all in memory"""
    for doc in corpus:
        chunks = chunker.chunk_document(doc)
        for chunk in chunks:
            yield chunk

# Process in batches
def process_in_batches(corpus, batch_size=1000):
    chunker = HierarchicalChunker()
    
    for i in range(0, len(corpus), batch_size):
        batch = corpus[i:i+batch_size]
        chunks = list(chunk_corpus_generator(batch, chunker))
        # Process batch
        yield chunks
```

This skill provides the foundational knowledge to build, evaluate, and optimize local RAG pipelines using BM25 search and hierarchical chunking strategies.
