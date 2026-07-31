---
name: ai-data-engineering-rag-pipeline
description: Production-grade local RAG pipeline with BM25 retrieval, hierarchical chunking, and evaluation frameworks for building AI-powered search systems.
triggers:
  - how do I build a RAG pipeline with BM25
  - implement hierarchical chunking for document retrieval
  - create a local RAG system with evaluation metrics
  - set up retrieval augmented generation pipeline
  - build BM25 search engine with recall evaluation
  - implement chunking strategies for RAG
  - evaluate RAG retrieval performance
  - create golden dataset for RAG testing
---

# AI Data Engineering RAG Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a production-grade local RAG (Retrieval Augmented Generation) pipeline that includes BM25-based retrieval, hierarchical document chunking, evaluation frameworks, and retrieval contracts. The project provides hands-on implementations for building searchable knowledge bases with proper evaluation metrics.

## What This Project Does

The `ai-data-engineering-roadmap` provides:

- **Local RAG Pipeline**: Build retrieval systems without external APIs
- **BM25 Baseline Search**: Okapi BM25 ranking algorithm with inverted index
- **Hierarchical Chunking**: Multi-level document chunking (document, section, paragraph)
- **Retrieval Contracts**: Formal evaluation interfaces and golden datasets
- **Recall@K Evaluation**: Measure retrieval quality with industry-standard metrics
- **Corpus Management**: Governed document collections with metadata

## Installation

```bash
# Clone the repository
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap

# Install dependencies (Python 3.8+)
pip install nltk numpy

# Download NLTK data (required for tokenization)
python -c "import nltk; nltk.download('punkt')"
```

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/          # RAG pipeline foundations, retrieval contracts
├── Day_02/          # BM25 baseline search engine
└── Day_03/          # Hierarchical chunking architecture
```

## Day 01: Retrieval Contracts & Golden Datasets

### Creating a Golden Dataset

```python
import json

# Golden dataset format: questions.jsonl
questions = [
    {
        "question_id": "Q001",
        "query": "What are the key principles of RAG?",
        "expected_doc_ids": ["doc_001", "doc_003"],
        "context": "technical_overview"
    },
    {
        "question_id": "Q002",
        "query": "How does BM25 ranking work?",
        "expected_doc_ids": ["doc_005"],
        "context": "algorithms"
    }
]

# Save as JSONL
with open('questions.jsonl', 'w') as f:
    for q in questions:
        f.write(json.dumps(q) + '\n')
```

### Loading and Validating Questions

```python
def load_golden_dataset(filepath):
    """Load and validate golden dataset."""
    questions = []
    with open(filepath, 'r') as f:
        for line in f:
            q = json.loads(line.strip())
            assert 'question_id' in q
            assert 'query' in q
            assert 'expected_doc_ids' in q
            questions.append(q)
    return questions

# Usage
golden_questions = load_golden_dataset('questions.jsonl')
print(f"Loaded {len(golden_questions)} evaluation questions")
```

### Retrieval Contract Interface

```python
class RetrievalContract:
    """Formal interface for retrieval evaluation."""
    
    def retrieve(self, query: str, k: int = 10) -> list:
        """
        Retrieve top-k documents for a query.
        
        Returns:
            List of (doc_id, score) tuples
        """
        raise NotImplementedError
    
    def evaluate(self, golden_dataset: list) -> dict:
        """
        Evaluate retrieval quality.
        
        Returns:
            Dict with metrics: recall@k, precision@k, mrr
        """
        results = []
        for question in golden_dataset:
            retrieved = self.retrieve(question['query'], k=10)
            retrieved_ids = [doc_id for doc_id, _ in retrieved]
            
            # Calculate recall@10
            expected = set(question['expected_doc_ids'])
            found = set(retrieved_ids[:10]) & expected
            recall = len(found) / len(expected) if expected else 0
            
            results.append({
                'question_id': question['question_id'],
                'recall@10': recall
            })
        
        avg_recall = sum(r['recall@10'] for r in results) / len(results)
        return {'avg_recall@10': avg_recall, 'results': results}
```

## Day 02: BM25 Baseline Search Engine

### Building an Inverted Index

```python
import math
from collections import defaultdict
from nltk.tokenize import word_tokenize

class BM25SearchEngine:
    """Okapi BM25 ranking implementation."""
    
    def __init__(self, k1=1.5, b=0.75):
        self.k1 = k1  # Term frequency saturation
        self.b = b    # Length normalization
        self.inverted_index = defaultdict(list)
        self.doc_lengths = {}
        self.avg_doc_length = 0
        self.num_docs = 0
        self.docs = {}
    
    def tokenize(self, text):
        """Tokenize and normalize text."""
        tokens = word_tokenize(text.lower())
        return [t for t in tokens if t.isalnum()]
    
    def index_documents(self, documents):
        """
        Build inverted index from documents.
        
        Args:
            documents: List of dicts with 'doc_id' and 'text'
        """
        self.docs = {d['doc_id']: d for d in documents}
        self.num_docs = len(documents)
        
        # Build index and calculate lengths
        total_length = 0
        for doc in documents:
            tokens = self.tokenize(doc['text'])
            self.doc_lengths[doc['doc_id']] = len(tokens)
            total_length += len(tokens)
            
            # Add to inverted index
            for position, token in enumerate(tokens):
                self.inverted_index[token].append({
                    'doc_id': doc['doc_id'],
                    'position': position
                })
        
        self.avg_doc_length = total_length / self.num_docs if self.num_docs else 0
    
    def calculate_idf(self, term):
        """Calculate inverse document frequency."""
        doc_freq = len(set(posting['doc_id'] for posting in self.inverted_index.get(term, [])))
        return math.log((self.num_docs - doc_freq + 0.5) / (doc_freq + 0.5) + 1.0)
    
    def score_document(self, query_terms, doc_id):
        """Calculate BM25 score for a document."""
        score = 0.0
        doc_length = self.doc_lengths[doc_id]
        
        for term in query_terms:
            if term not in self.inverted_index:
                continue
            
            # Term frequency in document
            tf = sum(1 for p in self.inverted_index[term] if p['doc_id'] == doc_id)
            
            # IDF component
            idf = self.calculate_idf(term)
            
            # BM25 formula
            numerator = tf * (self.k1 + 1)
            denominator = tf + self.k1 * (1 - self.b + self.b * (doc_length / self.avg_doc_length))
            score += idf * (numerator / denominator)
        
        return score
    
    def search(self, query, k=10):
        """
        Search for top-k documents.
        
        Returns:
            List of (doc_id, score) tuples
        """
        query_terms = self.tokenize(query)
        
        # Get candidate documents
        candidates = set()
        for term in query_terms:
            candidates.update(p['doc_id'] for p in self.inverted_index.get(term, []))
        
        # Score all candidates
        scores = [(doc_id, self.score_document(query_terms, doc_id)) for doc_id in candidates]
        scores.sort(key=lambda x: x[1], reverse=True)
        
        return scores[:k]
```

### Using BM25 Search Engine

```python
# Sample corpus
documents = [
    {
        "doc_id": "doc_001",
        "text": "RAG combines retrieval and generation for better AI responses",
        "metadata": {"domain": "ai"}
    },
    {
        "doc_id": "doc_002",
        "text": "BM25 is a probabilistic ranking function for information retrieval",
        "metadata": {"domain": "algorithms"}
    },
    {
        "doc_id": "doc_003",
        "text": "Vector databases enable semantic search using embeddings",
        "metadata": {"domain": "databases"}
    }
]

# Initialize and index
engine = BM25SearchEngine(k1=1.5, b=0.75)
engine.index_documents(documents)

# Search
results = engine.search("information retrieval ranking", k=5)
for doc_id, score in results:
    print(f"{doc_id}: {score:.4f} - {engine.docs[doc_id]['text'][:50]}")
```

### Recall@K Evaluation

```python
def evaluate_recall_at_k(engine, golden_dataset, k=10):
    """Evaluate retrieval quality using Recall@K."""
    recalls = []
    
    for question in golden_dataset:
        # Retrieve documents
        results = engine.search(question['query'], k=k)
        retrieved_ids = [doc_id for doc_id, _ in results]
        
        # Calculate recall
        expected = set(question['expected_doc_ids'])
        retrieved = set(retrieved_ids)
        
        recall = len(expected & retrieved) / len(expected) if expected else 0
        recalls.append({
            'question_id': question['question_id'],
            'recall': recall,
            'expected': list(expected),
            'retrieved': retrieved_ids[:k]
        })
    
    avg_recall = sum(r['recall'] for r in recalls) / len(recalls)
    return {
        'avg_recall@{}'.format(k): avg_recall,
        'details': recalls
    }

# Usage
evaluation = evaluate_recall_at_k(engine, golden_questions, k=10)
print(f"Average Recall@10: {evaluation['avg_recall@10']:.2%}")
```

## Day 03: Hierarchical Chunking Architecture

### Chunk ID Contract

```python
def generate_chunk_id(doc_id: str, level: str, index: int) -> str:
    """
    Generate deterministic chunk ID.
    
    Args:
        doc_id: Parent document ID
        level: 'document', 'section', or 'paragraph'
        index: Position within parent
    
    Returns:
        Unique chunk ID (e.g., 'doc_001_sec_02_para_05')
    """
    level_prefix = {
        'document': 'doc',
        'section': 'sec',
        'paragraph': 'para'
    }
    return f"{doc_id}_{level_prefix[level]}_{index:02d}"
```

### Hierarchical Chunker

```python
import re

class HierarchicalChunker:
    """Multi-level document chunking with parent-child relationships."""
    
    def __init__(self):
        self.chunks = []
    
    def chunk_document(self, doc_id: str, text: str) -> list:
        """
        Create hierarchical chunks from document.
        
        Returns:
            List of chunk dicts with metadata and parent linkage
        """
        chunks = []
        
        # Level 1: Document-level chunk
        doc_chunk = {
            'chunk_id': generate_chunk_id(doc_id, 'document', 0),
            'doc_id': doc_id,
            'level': 'document',
            'text': text,
            'parent_id': None,
            'char_count': len(text)
        }
        chunks.append(doc_chunk)
        
        # Level 2: Section-level chunks (split on double newlines or headers)
        sections = re.split(r'\n\n+|(?=^#{1,3} )', text, flags=re.MULTILINE)
        sections = [s.strip() for s in sections if s.strip()]
        
        for sec_idx, section in enumerate(sections):
            section_chunk = {
                'chunk_id': generate_chunk_id(doc_id, 'section', sec_idx),
                'doc_id': doc_id,
                'level': 'section',
                'text': section,
                'parent_id': doc_chunk['chunk_id'],
                'section_index': sec_idx,
                'char_count': len(section)
            }
            chunks.append(section_chunk)
            
            # Level 3: Paragraph-level chunks
            paragraphs = section.split('\n')
            paragraphs = [p.strip() for p in paragraphs if p.strip() and len(p.strip()) > 20]
            
            for para_idx, paragraph in enumerate(paragraphs):
                para_chunk = {
                    'chunk_id': generate_chunk_id(doc_id, 'paragraph', para_idx),
                    'doc_id': doc_id,
                    'level': 'paragraph',
                    'text': paragraph,
                    'parent_id': section_chunk['chunk_id'],
                    'section_index': sec_idx,
                    'paragraph_index': para_idx,
                    'char_count': len(paragraph)
                }
                chunks.append(para_chunk)
        
        return chunks
    
    def get_context_window(self, chunk_id: str, chunks: list, window_size=1) -> str:
        """Retrieve chunk with surrounding context."""
        # Find target chunk
        target = next((c for c in chunks if c['chunk_id'] == chunk_id), None)
        if not target:
            return ""
        
        # Get parent for context
        if target['parent_id']:
            parent = next((c for c in chunks if c['chunk_id'] == target['parent_id']), None)
            if parent:
                return parent['text']
        
        return target['text']
```

### Using Hierarchical Chunking with BM25

```python
# Sample document
doc = {
    'doc_id': 'doc_ai_001',
    'text': '''# Introduction to RAG

Retrieval Augmented Generation combines information retrieval with language generation.

## Key Components

RAG systems consist of three main parts: retriever, knowledge base, and generator.

The retriever finds relevant documents using semantic or keyword search.

## Benefits

RAG reduces hallucination by grounding responses in real data.
It enables knowledge updates without retraining the model.'''
}

# Create hierarchical chunks
chunker = HierarchicalChunker()
chunks = chunker.chunk_document(doc['doc_id'], doc['text'])

print(f"Created {len(chunks)} chunks:")
for chunk in chunks:
    print(f"  {chunk['chunk_id']} ({chunk['level']}): {chunk['char_count']} chars")

# Index paragraph-level chunks
paragraph_chunks = [c for c in chunks if c['level'] == 'paragraph']
bm25_engine = BM25SearchEngine()
bm25_engine.index_documents(paragraph_chunks)

# Search with context retrieval
results = bm25_engine.search("What are the benefits of RAG?", k=3)
for chunk_id, score in results:
    context = chunker.get_context_window(chunk_id, chunks)
    print(f"\nScore: {score:.4f}")
    print(f"Context: {context[:100]}...")
```

## Configuration & Environment

```python
# config.py - Common configuration patterns

import os

# BM25 hyperparameters
BM25_CONFIG = {
    'k1': float(os.getenv('BM25_K1', '1.5')),    # Term frequency saturation
    'b': float(os.getenv('BM25_B', '0.75'))      # Length normalization
}

# Chunking strategy
CHUNKING_CONFIG = {
    'min_paragraph_length': int(os.getenv('MIN_PARA_LEN', '20')),
    'max_chunk_size': int(os.getenv('MAX_CHUNK_SIZE', '512')),
    'overlap_tokens': int(os.getenv('CHUNK_OVERLAP', '50'))
}

# Evaluation settings
EVAL_CONFIG = {
    'recall_k': int(os.getenv('RECALL_K', '10')),
    'golden_dataset_path': os.getenv('GOLDEN_DATASET', 'questions.jsonl')
}
```

## Common Patterns

### Pattern 1: End-to-End RAG Pipeline

```python
def build_rag_pipeline(corpus_path, golden_dataset_path):
    """Complete RAG pipeline with evaluation."""
    
    # 1. Load corpus
    with open(corpus_path, 'r') as f:
        documents = json.load(f)
    
    # 2. Hierarchical chunking
    chunker = HierarchicalChunker()
    all_chunks = []
    for doc in documents:
        chunks = chunker.chunk_document(doc['doc_id'], doc['text'])
        all_chunks.extend(chunks)
    
    # 3. Index with BM25
    paragraph_chunks = [c for c in all_chunks if c['level'] == 'paragraph']
    engine = BM25SearchEngine()
    engine.index_documents(paragraph_chunks)
    
    # 4. Evaluate
    golden_questions = load_golden_dataset(golden_dataset_path)
    results = evaluate_recall_at_k(engine, golden_questions, k=10)
    
    return engine, results

# Usage
engine, eval_results = build_rag_pipeline('corpus.json', 'questions.jsonl')
print(f"Pipeline Recall@10: {eval_results['avg_recall@10']:.2%}")
```

### Pattern 2: Iterative Retrieval with Re-ranking

```python
def iterative_retrieval(query, engine, chunks, initial_k=20, final_k=5):
    """Two-stage retrieval: BM25 then context-based re-ranking."""
    
    # Stage 1: BM25 retrieval
    initial_results = engine.search(query, k=initial_k)
    
    # Stage 2: Re-rank with parent context
    chunker = HierarchicalChunker()
    re_ranked = []
    
    for chunk_id, score in initial_results:
        chunk = next(c for c in chunks if c['chunk_id'] == chunk_id)
        
        # Boost score if parent contains query terms
        context = chunker.get_context_window(chunk_id, chunks)
        query_terms = set(engine.tokenize(query))
        context_terms = set(engine.tokenize(context))
        overlap = len(query_terms & context_terms)
        
        boosted_score = score * (1 + 0.1 * overlap)
        re_ranked.append((chunk_id, boosted_score, context))
    
    re_ranked.sort(key=lambda x: x[1], reverse=True)
    return re_ranked[:final_k]
```

## Troubleshooting

### Issue: Low Recall@K Scores

```python
# Diagnostic: Analyze retrieval failures
def diagnose_retrieval(engine, question, expected_doc_ids):
    """Debug why expected documents weren't retrieved."""
    
    results = engine.search(question['query'], k=20)
    retrieved_ids = [doc_id for doc_id, _ in results]
    
    missing = set(expected_doc_ids) - set(retrieved_ids)
    
    for doc_id in missing:
        doc = engine.docs.get(doc_id)
        if not doc:
            print(f"❌ {doc_id}: Not in index")
            continue
        
        # Check term overlap
        query_terms = set(engine.tokenize(question['query']))
        doc_terms = set(engine.tokenize(doc['text']))
        overlap = query_terms & doc_terms
        
        print(f"📄 {doc_id}:")
        print(f"   Query terms: {query_terms}")
        print(f"   Overlap: {overlap}")
        print(f"   Doc preview: {doc['text'][:100]}")
```

### Issue: Chunking Produces Too Many Small Chunks

```python
# Solution: Merge small chunks
def merge_small_chunks(chunks, min_size=50):
    """Combine adjacent chunks below minimum size."""
    merged = []
    buffer = None
    
    for chunk in chunks:
        if chunk['level'] != 'paragraph':
            merged.append(chunk)
            continue
        
        if buffer and chunk['char_count'] < min_size:
            buffer['text'] += ' ' + chunk['text']
            buffer['char_count'] += chunk['char_count']
        else:
            if buffer:
                merged.append(buffer)
            buffer = chunk.copy()
    
    if buffer:
        merged.append(buffer)
    
    return merged
```

### Issue: BM25 Favors Short Documents

```python
# Solution: Adjust length normalization parameter
# Lower b value (0.0 - 1.0) reduces length penalty

engine_short_docs = BM25SearchEngine(k1=1.5, b=0.3)  # Less length normalization
engine_long_docs = BM25SearchEngine(k1=1.5, b=0.9)   # More length normalization
```

## Advanced Usage

### Custom Tokenization

```python
import re

class CustomBM25(BM25SearchEngine):
    """BM25 with domain-specific tokenization."""
    
    def tokenize(self, text):
        """Preserve technical terms and acronyms."""
        # Keep acronyms together (e.g., RAG, BM25)
        text = text.lower()
        tokens = re.findall(r'\b[a-z]+\d+\b|\b[a-z]{2,}\b|\b\d+\b', text)
        return tokens
```

### Metadata Filtering

```python
def filtered_search(engine, query, k=10, metadata_filter=None):
    """Search with metadata constraints."""
    all_results = engine.search(query, k=k*3)  # Over-retrieve
    
    if metadata_filter:
        filtered = [
            (doc_id, score) for doc_id, score in all_results
            if all(engine.docs[doc_id]['metadata'].get(key) == value 
                   for key, value in metadata_filter.items())
        ]
        return filtered[:k]
    
    return all_results[:k]

# Usage
results = filtered_search(
    engine, 
    "machine learning algorithms",
    metadata_filter={'domain': 'ai', 'difficulty': 'advanced'}
)
```
