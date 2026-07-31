---
name: ai-data-engineering-rag-pipeline
description: Production-grade local RAG pipeline with BM25 search, hierarchical chunking, and retrieval evaluation frameworks
triggers:
  - build a RAG pipeline with BM25 search
  - implement hierarchical chunking for documents
  - evaluate retrieval performance with recall metrics
  - create a local RAG system with chunk evaluation
  - set up document retrieval with inverted index
  - build a production RAG architecture
  - implement retrieval contracts and golden datasets
  - evaluate chunking strategies for RAG
---

# AI Data Engineering RAG Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

The `ai-data-engineering-roadmap` project provides production-grade implementations for building local Retrieval-Augmented Generation (RAG) pipelines from scratch. It includes:

- **BM25 Baseline Search Engine**: Okapi BM25 ranking with inverted index
- **Hierarchical Chunking**: Multi-level document segmentation (document, section, paragraph)
- **Retrieval Evaluation**: Recall@K metrics with golden datasets
- **Retrieval Contracts**: Schema validation for queries and results
- **Parent-Child Metadata Linkage**: Deterministic chunk ID generation

This project is ideal for learning production RAG architectures, chunk granularity evaluation, and retrieval system benchmarking.

## Installation

```bash
# Clone the repository
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap

# Install dependencies (if requirements.txt exists)
pip install -r requirements.txt

# Or install common dependencies manually
pip install numpy scikit-learn nltk
```

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/          # Local RAG Pipeline & Retrieval Contracts
│   ├── corpus/      # Governed corpus documents
│   ├── questions.jsonl  # Golden dataset
│   └── evaluate.py  # Evaluation script
├── Day_02/          # BM25 Baseline Search Engine
│   ├── corpus/      # Synthetic corpus
│   └── baseline_bm25.py  # BM25 implementation
├── Day_03/          # Hierarchical Chunking Architecture
│   ├── corpus/      # Multi-domain corpus
│   └── pipeline.py  # Chunking & retrieval pipeline
```

## Day 01: Retrieval Contracts & Golden Dataset

### Golden Dataset Format (questions.jsonl)

```python
import json

# Read golden dataset
def load_golden_dataset(path="Day_01/questions.jsonl"):
    """Load questions with expected document IDs"""
    questions = []
    with open(path, 'r') as f:
        for line in f:
            questions.append(json.loads(line))
    return questions

# Example question format
example_question = {
    "question_id": "q001",
    "query": "What are the benefits of cloud computing?",
    "expected_docs": ["doc_cloud_101", "doc_aws_intro"]
}

# Create golden dataset
def create_golden_dataset(questions, output_path="questions.jsonl"):
    """Write questions to JSONL format"""
    with open(output_path, 'w') as f:
        for q in questions:
            f.write(json.dumps(q) + '\n')
```

### Retrieval Evaluation Script

```python
# Day_01/evaluate.py
import json

def evaluate_retrieval(results, golden_dataset, k=10):
    """
    Evaluate retrieval using Recall@K metric
    
    Args:
        results: Dict mapping question_id to list of retrieved doc_ids
        golden_dataset: List of dicts with question_id and expected_docs
        k: Number of top results to consider
    
    Returns:
        recall_at_k: Average recall across all queries
    """
    recalls = []
    
    for question in golden_dataset:
        qid = question['question_id']
        expected = set(question['expected_docs'])
        retrieved = set(results.get(qid, [])[:k])
        
        if len(expected) == 0:
            continue
            
        recall = len(expected & retrieved) / len(expected)
        recalls.append(recall)
    
    return sum(recalls) / len(recalls) if recalls else 0.0

# Usage example
golden_data = load_golden_dataset("questions.jsonl")
retrieval_results = {
    "q001": ["doc_cloud_101", "doc_aws_intro", "doc_azure"],
    "q002": ["doc_security_01", "doc_encryption"]
}

recall = evaluate_retrieval(retrieval_results, golden_data, k=10)
print(f"Recall@10: {recall:.3f}")
```

## Day 02: BM25 Search Engine

### BM25 Implementation

```python
# Day_02/baseline_bm25.py
import math
from collections import defaultdict, Counter

class BM25SearchEngine:
    """
    Okapi BM25 ranking algorithm implementation
    
    Parameters:
        k1: Term frequency saturation parameter (default: 1.5)
        b: Length normalization parameter (default: 0.75)
    """
    
    def __init__(self, k1=1.5, b=0.75):
        self.k1 = k1
        self.b = b
        self.corpus = []
        self.doc_ids = []
        self.inverted_index = defaultdict(list)
        self.doc_lengths = {}
        self.avg_doc_length = 0
        self.N = 0  # Total number of documents
        
    def tokenize(self, text):
        """Simple whitespace tokenization with lowercasing"""
        return text.lower().split()
    
    def index_corpus(self, documents):
        """
        Build inverted index from corpus
        
        Args:
            documents: List of dicts with 'doc_id' and 'text' keys
        """
        self.corpus = documents
        self.N = len(documents)
        
        total_length = 0
        
        for doc in documents:
            doc_id = doc['doc_id']
            text = doc['text']
            tokens = self.tokenize(text)
            
            self.doc_ids.append(doc_id)
            self.doc_lengths[doc_id] = len(tokens)
            total_length += len(tokens)
            
            # Build inverted index: term -> [(doc_id, term_freq), ...]
            term_freq = Counter(tokens)
            for term, freq in term_freq.items():
                self.inverted_index[term].append((doc_id, freq))
        
        self.avg_doc_length = total_length / self.N if self.N > 0 else 0
    
    def idf(self, term):
        """Calculate inverse document frequency"""
        df = len(self.inverted_index.get(term, []))
        return math.log((self.N - df + 0.5) / (df + 0.5) + 1.0)
    
    def bm25_score(self, query_tokens, doc_id):
        """Calculate BM25 score for a document given query tokens"""
        score = 0.0
        doc_len = self.doc_lengths.get(doc_id, 0)
        
        for term in query_tokens:
            if term not in self.inverted_index:
                continue
            
            # Get term frequency in this document
            tf = 0
            for did, freq in self.inverted_index[term]:
                if did == doc_id:
                    tf = freq
                    break
            
            if tf == 0:
                continue
            
            idf_score = self.idf(term)
            
            # BM25 formula
            numerator = tf * (self.k1 + 1)
            denominator = tf + self.k1 * (1 - self.b + self.b * (doc_len / self.avg_doc_length))
            
            score += idf_score * (numerator / denominator)
        
        return score
    
    def search(self, query, top_k=10):
        """
        Search corpus and return top-k documents
        
        Args:
            query: Search query string
            top_k: Number of results to return
        
        Returns:
            List of (doc_id, score) tuples, sorted by score descending
        """
        query_tokens = self.tokenize(query)
        
        # Get candidate documents (documents containing at least one query term)
        candidates = set()
        for term in query_tokens:
            for doc_id, _ in self.inverted_index.get(term, []):
                candidates.add(doc_id)
        
        # Score all candidates
        scores = []
        for doc_id in candidates:
            score = self.bm25_score(query_tokens, doc_id)
            scores.append((doc_id, score))
        
        # Sort by score descending
        scores.sort(key=lambda x: x[1], reverse=True)
        
        return scores[:top_k]

# Usage Example
if __name__ == "__main__":
    # Sample corpus
    corpus = [
        {"doc_id": "doc1", "text": "Cloud computing provides scalable resources"},
        {"doc_id": "doc2", "text": "AWS offers cloud infrastructure services"},
        {"doc_id": "doc3", "text": "Data engineering involves building pipelines"},
        {"doc_id": "doc4", "text": "Machine learning models require training data"},
    ]
    
    # Initialize and index
    engine = BM25SearchEngine(k1=1.5, b=0.75)
    engine.index_corpus(corpus)
    
    # Search
    query = "cloud computing services"
    results = engine.search(query, top_k=3)
    
    print(f"Query: {query}\n")
    for doc_id, score in results:
        print(f"{doc_id}: {score:.4f}")
```

### BM25 with Recall Evaluation

```python
# Combine BM25 with evaluation
def evaluate_bm25_retrieval(corpus, queries_file, k=10):
    """
    End-to-end BM25 evaluation pipeline
    
    Args:
        corpus: List of document dicts
        queries_file: Path to questions.jsonl
        k: Recall@K parameter
    """
    # Index corpus
    engine = BM25SearchEngine()
    engine.index_corpus(corpus)
    
    # Load queries
    with open(queries_file, 'r') as f:
        queries = [json.loads(line) for line in f]
    
    # Retrieve for each query
    results = {}
    for q in queries:
        qid = q['question_id']
        query_text = q['query']
        retrieved = engine.search(query_text, top_k=k)
        results[qid] = [doc_id for doc_id, _ in retrieved]
    
    # Evaluate
    recall = evaluate_retrieval(results, queries, k=k)
    
    print(f"BM25 Recall@{k}: {recall:.3f}")
    return recall, results
```

## Day 03: Hierarchical Chunking

### Chunk ID Contract

```python
# Day_03/pipeline.py
import hashlib

def generate_chunk_id(doc_id, level, index):
    """
    Generate deterministic chunk ID
    
    Args:
        doc_id: Parent document identifier
        level: Chunk granularity (doc, section, paragraph)
        index: Sequential index within level
    
    Returns:
        Deterministic chunk ID string
    """
    id_string = f"{doc_id}#{level}#{index}"
    hash_suffix = hashlib.md5(id_string.encode()).hexdigest()[:8]
    return f"{doc_id}_{level}_{index}_{hash_suffix}"

# Example usage
chunk_id = generate_chunk_id("doc_001", "paragraph", 5)
# Output: doc_001_paragraph_5_a3f2b1c4
```

### Hierarchical Chunking Implementation

```python
# Day_03/pipeline.py
import re

class HierarchicalChunker:
    """
    Multi-level document chunking (document -> section -> paragraph)
    with parent-child metadata linkage
    """
    
    def __init__(self):
        self.chunks = []
        
    def chunk_document(self, doc_id, text):
        """
        Chunk document at multiple granularities
        
        Args:
            doc_id: Document identifier
            text: Full document text
        
        Returns:
            List of chunk dicts with metadata
        """
        chunks = []
        
        # Level 1: Whole document
        doc_chunk = {
            'chunk_id': generate_chunk_id(doc_id, 'doc', 0),
            'doc_id': doc_id,
            'level': 'doc',
            'text': text,
            'parent_id': None,
            'index': 0
        }
        chunks.append(doc_chunk)
        
        # Level 2: Sections (split by double newline or headers)
        sections = re.split(r'\n\n+|(?=^#{1,3}\s)', text, flags=re.MULTILINE)
        sections = [s.strip() for s in sections if s.strip()]
        
        for sec_idx, section_text in enumerate(sections):
            section_chunk = {
                'chunk_id': generate_chunk_id(doc_id, 'section', sec_idx),
                'doc_id': doc_id,
                'level': 'section',
                'text': section_text,
                'parent_id': doc_chunk['chunk_id'],
                'index': sec_idx
            }
            chunks.append(section_chunk)
            
            # Level 3: Paragraphs (split by single newline)
            paragraphs = section_text.split('\n')
            paragraphs = [p.strip() for p in paragraphs if p.strip()]
            
            for para_idx, para_text in enumerate(paragraphs):
                para_chunk = {
                    'chunk_id': generate_chunk_id(doc_id, 'paragraph', 
                                                  sec_idx * 100 + para_idx),
                    'doc_id': doc_id,
                    'level': 'paragraph',
                    'text': para_text,
                    'parent_id': section_chunk['chunk_id'],
                    'index': para_idx
                }
                chunks.append(para_chunk)
        
        return chunks
    
    def chunk_corpus(self, documents):
        """
        Chunk entire corpus at all granularity levels
        
        Args:
            documents: List of dicts with 'doc_id' and 'text'
        
        Returns:
            List of all chunks with metadata
        """
        all_chunks = []
        
        for doc in documents:
            doc_chunks = self.chunk_document(doc['doc_id'], doc['text'])
            all_chunks.extend(doc_chunks)
        
        return all_chunks

# Usage Example
if __name__ == "__main__":
    # Sample document
    documents = [{
        "doc_id": "doc_cloud_001",
        "text": """# Cloud Computing Overview
Cloud computing provides on-demand access to computing resources.

## Benefits
Scalability and cost efficiency are key advantages.
Pay only for what you use.

## Deployment Models
Public, private, and hybrid clouds serve different needs."""
    }]
    
    # Chunk at multiple levels
    chunker = HierarchicalChunker()
    chunks = chunker.chunk_corpus(documents)
    
    # Display chunks
    for chunk in chunks:
        print(f"ID: {chunk['chunk_id']}")
        print(f"Level: {chunk['level']}, Parent: {chunk['parent_id']}")
        print(f"Text: {chunk['text'][:80]}...\n")
```

### Chunk Granularity Evaluation

```python
# Day_03/pipeline.py
def evaluate_chunk_granularity(corpus, queries_file, levels=['doc', 'section', 'paragraph']):
    """
    Compare retrieval performance across chunk granularities
    
    Args:
        corpus: List of documents
        queries_file: Path to golden dataset
        levels: Chunk levels to evaluate
    
    Returns:
        Dict mapping level to recall score
    """
    # Generate chunks
    chunker = HierarchicalChunker()
    all_chunks = chunker.chunk_corpus(corpus)
    
    # Load queries
    with open(queries_file, 'r') as f:
        queries = [json.loads(line) for line in f]
    
    results = {}
    
    for level in levels:
        # Filter chunks by level
        level_chunks = [c for c in all_chunks if c['level'] == level]
        
        # Index chunks as documents
        chunk_docs = [
            {'doc_id': c['chunk_id'], 'text': c['text']} 
            for c in level_chunks
        ]
        
        engine = BM25SearchEngine()
        engine.index_corpus(chunk_docs)
        
        # Retrieve
        retrieval_results = {}
        for q in queries:
            qid = q['question_id']
            retrieved = engine.search(q['query'], top_k=10)
            
            # Map chunk IDs back to doc IDs for evaluation
            doc_ids = [c['doc_id'] for c in level_chunks 
                      if c['chunk_id'] in [r[0] for r in retrieved]]
            retrieval_results[qid] = doc_ids
        
        # Evaluate
        recall = evaluate_retrieval(retrieval_results, queries, k=10)
        results[level] = recall
        
        print(f"Level: {level:12s} | Recall@10: {recall:.3f}")
    
    return results
```

## Common Patterns

### Pattern 1: Building a Complete RAG Pipeline

```python
import os

def build_rag_pipeline(corpus_dir, queries_file, chunk_level='paragraph'):
    """Complete RAG pipeline from corpus to evaluation"""
    
    # 1. Load corpus
    documents = []
    for filename in os.listdir(corpus_dir):
        if filename.endswith('.txt'):
            with open(os.path.join(corpus_dir, filename), 'r') as f:
                doc_id = filename.replace('.txt', '')
                documents.append({'doc_id': doc_id, 'text': f.read()})
    
    # 2. Hierarchical chunking
    chunker = HierarchicalChunker()
    chunks = chunker.chunk_corpus(documents)
    
    # 3. Filter to desired granularity
    level_chunks = [c for c in chunks if c['level'] == chunk_level]
    chunk_docs = [
        {'doc_id': c['chunk_id'], 'text': c['text']} 
        for c in level_chunks
    ]
    
    # 4. Index with BM25
    engine = BM25SearchEngine(k1=1.5, b=0.75)
    engine.index_corpus(chunk_docs)
    
    # 5. Load and evaluate
    with open(queries_file, 'r') as f:
        queries = [json.loads(line) for line in f]
    
    results = {}
    for q in queries:
        retrieved = engine.search(q['query'], top_k=10)
        results[q['question_id']] = [r[0] for r in retrieved]
    
    recall = evaluate_retrieval(results, queries, k=10)
    
    return engine, recall, results

# Usage
engine, recall, results = build_rag_pipeline(
    corpus_dir="Day_03/corpus",
    queries_file="Day_01/questions.jsonl",
    chunk_level='paragraph'
)
```

### Pattern 2: Interactive Retrieval Simulation

```python
def interactive_search(corpus_dir, chunk_level='paragraph'):
    """Interactive RAG search interface"""
    
    # Build pipeline
    documents = []
    for filename in os.listdir(corpus_dir):
        if filename.endswith('.txt'):
            with open(os.path.join(corpus_dir, filename), 'r') as f:
                doc_id = filename.replace('.txt', '')
                documents.append({'doc_id': doc_id, 'text': f.read()})
    
    chunker = HierarchicalChunker()
    chunks = chunker.chunk_corpus(documents)
    level_chunks = [c for c in chunks if c['level'] == chunk_level]
    
    chunk_docs = [
        {'doc_id': c['chunk_id'], 'text': c['text']} 
        for c in level_chunks
    ]
    
    engine = BM25SearchEngine()
    engine.index_corpus(chunk_docs)
    
    # Interactive loop
    print(f"RAG Search Engine (chunk level: {chunk_level})")
    print("Enter query (or 'quit' to exit):\n")
    
    while True:
        query = input("> ").strip()
        if query.lower() in ['quit', 'exit']:
            break
        
        results = engine.search(query, top_k=5)
        
        print(f"\nTop 5 results for: '{query}'\n")
        for i, (chunk_id, score) in enumerate(results, 1):
            # Find chunk text
            chunk_text = next(
                (c['text'] for c in level_chunks if c['chunk_id'] == chunk_id),
                "N/A"
            )
            print(f"{i}. [{score:.4f}] {chunk_id}")
            print(f"   {chunk_text[:150]}...\n")

# Run interactive mode
# interactive_search("Day_03/corpus", chunk_level='paragraph')
```

## Troubleshooting

### Issue: Low Recall Scores

**Solution**: Try different chunk granularities or BM25 parameters

```python
# Tune BM25 parameters
for k1 in [1.2, 1.5, 2.0]:
    for b in [0.5, 0.75, 1.0]:
        engine = BM25SearchEngine(k1=k1, b=b)
        engine.index_corpus(chunk_docs)
        # Evaluate and compare
```

### Issue: Memory Issues with Large Corpus

**Solution**: Implement batch processing

```python
def index_corpus_batched(engine, documents, batch_size=1000):
    """Index corpus in batches to reduce memory usage"""
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i+batch_size]
        engine.index_corpus(batch)
```

### Issue: Chunk ID Collisions

**Solution**: Use full hash or add timestamp

```python
import time

def generate_chunk_id_safe(doc_id, level, index):
    """Generate chunk ID with timestamp to prevent collisions"""
    timestamp = int(time.time() * 1000)
    id_string = f"{doc_id}#{level}#{index}#{timestamp}"
    hash_full = hashlib.md5(id_string.encode()).hexdigest()
    return f"{doc_id}_{level}_{index}_{hash_full}"
```

### Issue: Non-English Text Tokenization

**Solution**: Use advanced tokenizers

```python
from nltk.tokenize import word_tokenize

class BM25SearchEngineAdvanced(BM25SearchEngine):
    def tokenize(self, text):
        """Advanced tokenization with NLTK"""
        import nltk
        # Download required data: nltk.download('punkt')
        return [token.lower() for token in word_tokenize(text)]
```

## Configuration

### Environment Setup

```python
# config.py
import os

# Corpus configuration
CORPUS_DIR = os.getenv('RAG_CORPUS_DIR', 'Day_03/corpus')
QUERIES_FILE = os.getenv('RAG_QUERIES_FILE', 'Day_01/questions.jsonl')

# BM25 parameters
BM25_K1 = float(os.getenv('BM25_K1', '1.5'))
BM25_B = float(os.getenv('BM25_B', '0.75'))

# Chunking configuration
DEFAULT_CHUNK_LEVEL = os.getenv('CHUNK_LEVEL', 'paragraph')

# Evaluation parameters
RECALL_K = int(os.getenv('RECALL_K', '10'))
```

### Usage with Configuration

```python
from config import *

engine = BM25SearchEngine(k1=BM25_K1, b=BM25_B)
pipeline = build_rag_pipeline(
    corpus_dir=CORPUS_DIR,
    queries_file=QUERIES_FILE,
    chunk_level=DEFAULT_CHUNK_LEVEL
)
```
