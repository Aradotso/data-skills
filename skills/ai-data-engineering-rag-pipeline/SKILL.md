---
name: ai-data-engineering-rag-pipeline
description: Production-grade local RAG pipeline with BM25 search, hierarchical chunking, and retrieval evaluation frameworks
triggers:
  - build a rag pipeline with evaluation
  - implement bm25 search with recall metrics
  - create hierarchical chunking for document retrieval
  - set up local rag system with chunking strategies
  - evaluate retrieval performance with golden datasets
  - design production rag architecture
  - implement inverted index search engine
  - build document chunking with parent-child metadata
---

# AI & Data Engineering RAG Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill provides expertise in building production-grade local RAG (Retrieval-Augmented Generation) pipelines using the ai-data-engineering-roadmap project. It covers BM25 baseline search engines, hierarchical chunking architectures, retrieval contracts, and evaluation frameworks with golden datasets.

## What This Project Does

The ai-data-engineering-roadmap project implements a complete RAG pipeline from scratch with:

- **BM25 Baseline Search Engine**: Okapi BM25 ranking with inverted index
- **Hierarchical Chunking**: Document, section, and paragraph-level chunking with parent-child metadata
- **Retrieval Contracts**: Governed corpus with deterministic chunk IDs
- **Evaluation Framework**: Recall@K metrics with golden datasets (questions.jsonl)
- **Production Patterns**: Failure mode analysis and interactive retrieval simulation

## Installation

```bash
# Clone the repository
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap

# Install dependencies (Python 3.8+)
pip install numpy nltk

# Download NLTK data for tokenization
python -c "import nltk; nltk.download('punkt')"
```

## Key Components

### Day 01: RAG Pipeline & Retrieval Contracts

Located in `Day_01/`, this module establishes the foundation:

```python
# questions.jsonl format - Golden dataset structure
{
  "question": "What is RAG?",
  "expected_chunks": ["chunk_001", "chunk_002"],
  "context": "retrieval-augmented generation"
}
```

**Key Files:**
- `questions.jsonl`: Golden dataset with expected retrieval results
- `corpus/`: Governed document collection
- `evaluate.py`: Retrieval evaluation script

### Day 02: BM25 Baseline Search Engine

Located in `Day_02/`, implements Okapi BM25 ranking:

```python
# baseline_bm25.py - Core BM25 implementation
import math
from collections import defaultdict
from typing import List, Dict, Set

class BM25Retriever:
    def __init__(self, k1=1.5, b=0.75):
        """
        Initialize BM25 retriever
        
        Args:
            k1: Term frequency saturation parameter (default 1.5)
            b: Length normalization parameter (default 0.75)
        """
        self.k1 = k1
        self.b = b
        self.inverted_index = defaultdict(list)
        self.doc_lengths = {}
        self.avg_doc_length = 0
        self.num_docs = 0
        self.idf_scores = {}
        
    def build_index(self, documents: List[Dict]):
        """Build inverted index from documents"""
        self.num_docs = len(documents)
        total_length = 0
        
        # Build inverted index
        for doc in documents:
            doc_id = doc['id']
            tokens = self._tokenize(doc['text'])
            self.doc_lengths[doc_id] = len(tokens)
            total_length += len(tokens)
            
            # Add to inverted index
            unique_tokens = set(tokens)
            for token in unique_tokens:
                self.inverted_index[token].append(doc_id)
        
        # Calculate average document length
        self.avg_doc_length = total_length / self.num_docs if self.num_docs > 0 else 0
        
        # Calculate IDF scores
        for term, doc_list in self.inverted_index.items():
            df = len(doc_list)
            self.idf_scores[term] = math.log((self.num_docs - df + 0.5) / (df + 0.5) + 1)
    
    def _tokenize(self, text: str) -> List[str]:
        """Simple tokenization - lowercase and split"""
        return text.lower().split()
    
    def search(self, query: str, k: int = 10) -> List[str]:
        """
        Search using BM25 ranking
        
        Args:
            query: Search query string
            k: Number of results to return
            
        Returns:
            List of document IDs ranked by BM25 score
        """
        query_tokens = self._tokenize(query)
        scores = defaultdict(float)
        
        for term in query_tokens:
            if term not in self.inverted_index:
                continue
                
            idf = self.idf_scores[term]
            
            for doc_id in self.inverted_index[term]:
                # Calculate term frequency in document
                doc_tokens = self._tokenize(self.documents[doc_id]['text'])
                tf = doc_tokens.count(term)
                
                # BM25 formula
                doc_length = self.doc_lengths[doc_id]
                norm_factor = 1 - self.b + self.b * (doc_length / self.avg_doc_length)
                score = idf * (tf * (self.k1 + 1)) / (tf + self.k1 * norm_factor)
                scores[doc_id] += score
        
        # Sort by score and return top k
        ranked_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [doc_id for doc_id, score in ranked_docs[:k]]

# Usage example
retriever = BM25Retriever(k1=1.5, b=0.75)

# Load corpus
documents = [
    {"id": "doc1", "text": "RAG combines retrieval and generation"},
    {"id": "doc2", "text": "BM25 is a ranking function for search"},
    {"id": "doc3", "text": "Chunking strategies improve retrieval accuracy"}
]

retriever.build_index(documents)
retriever.documents = {doc['id']: doc for doc in documents}

# Search
results = retriever.search("what is RAG", k=10)
print(f"Top results: {results}")
```

**Evaluation with Recall@10:**

```python
import json

def evaluate_recall_at_k(retriever, questions_file, k=10):
    """
    Evaluate Recall@K on golden dataset
    
    Args:
        retriever: BM25Retriever instance
        questions_file: Path to questions.jsonl
        k: Number of results to consider
        
    Returns:
        Average Recall@K score
    """
    total_recall = 0
    num_questions = 0
    
    with open(questions_file, 'r') as f:
        for line in f:
            question = json.loads(line)
            query = question['question']
            expected = set(question['expected_chunks'])
            
            # Retrieve top-k results
            results = retriever.search(query, k=k)
            retrieved = set(results)
            
            # Calculate recall
            if len(expected) > 0:
                recall = len(expected & retrieved) / len(expected)
                total_recall += recall
                num_questions += 1
    
    return total_recall / num_questions if num_questions > 0 else 0

# Run evaluation
recall_score = evaluate_recall_at_k(retriever, "Day_01/questions.jsonl", k=10)
print(f"Recall@10: {recall_score:.4f}")
```

### Day 03: Hierarchical Chunking Architecture

Located in `Day_03/`, implements multi-level document chunking:

```python
# pipeline.py - Hierarchical chunking with parent-child metadata
import hashlib
import re
from typing import List, Dict, Optional

class HierarchicalChunker:
    def __init__(self):
        self.chunks = []
        
    def generate_chunk_id(self, doc_id: str, level: str, index: int) -> str:
        """
        Generate deterministic chunk ID
        
        Args:
            doc_id: Document identifier
            level: Chunk level (document/section/paragraph)
            index: Position index
            
        Returns:
            Deterministic chunk ID
        """
        raw_id = f"{doc_id}_{level}_{index}"
        return hashlib.md5(raw_id.encode()).hexdigest()[:16]
    
    def chunk_document(self, doc_id: str, text: str) -> List[Dict]:
        """
        Apply hierarchical chunking strategy
        
        Levels:
        1. Document: Entire document as single chunk
        2. Section: Split on headers (## or ###)
        3. Paragraph: Split on double newlines
        
        Args:
            doc_id: Document identifier
            text: Full document text
            
        Returns:
            List of chunk dictionaries with metadata
        """
        chunks = []
        
        # Level 1: Document-level chunk
        doc_chunk = {
            "chunk_id": self.generate_chunk_id(doc_id, "document", 0),
            "doc_id": doc_id,
            "level": "document",
            "text": text,
            "parent_id": None,
            "children": []
        }
        chunks.append(doc_chunk)
        
        # Level 2: Section-level chunks (split on headers)
        sections = re.split(r'\n#{2,3}\s+', text)
        section_chunks = []
        
        for i, section in enumerate(sections):
            if not section.strip():
                continue
                
            section_chunk = {
                "chunk_id": self.generate_chunk_id(doc_id, "section", i),
                "doc_id": doc_id,
                "level": "section",
                "text": section.strip(),
                "parent_id": doc_chunk["chunk_id"],
                "children": []
            }
            section_chunks.append(section_chunk)
            doc_chunk["children"].append(section_chunk["chunk_id"])
            chunks.append(section_chunk)
            
            # Level 3: Paragraph-level chunks
            paragraphs = section.split('\n\n')
            
            for j, para in enumerate(paragraphs):
                if not para.strip() or len(para.strip()) < 50:
                    continue
                    
                para_chunk = {
                    "chunk_id": self.generate_chunk_id(doc_id, "paragraph", i * 100 + j),
                    "doc_id": doc_id,
                    "level": "paragraph",
                    "text": para.strip(),
                    "parent_id": section_chunk["chunk_id"],
                    "children": []
                }
                section_chunk["children"].append(para_chunk["chunk_id"])
                chunks.append(para_chunk)
        
        return chunks
    
    def get_context_window(self, chunk_id: str, window_size: int = 1) -> List[Dict]:
        """
        Retrieve chunk with surrounding context (parent/children/siblings)
        
        Args:
            chunk_id: Target chunk ID
            window_size: Number of sibling chunks to include
            
        Returns:
            List of chunks with context
        """
        # Find target chunk
        target = None
        for chunk in self.chunks:
            if chunk["chunk_id"] == chunk_id:
                target = chunk
                break
        
        if not target:
            return []
        
        context_chunks = [target]
        
        # Add parent
        if target["parent_id"]:
            parent = next((c for c in self.chunks if c["chunk_id"] == target["parent_id"]), None)
            if parent:
                context_chunks.append(parent)
        
        # Add children
        for child_id in target["children"]:
            child = next((c for c in self.chunks if c["chunk_id"] == child_id), None)
            if child:
                context_chunks.append(child)
        
        return context_chunks

# Usage example
chunker = HierarchicalChunker()

sample_doc = """
# Introduction to RAG

## What is RAG?

Retrieval-Augmented Generation combines retrieval systems with language models.

It improves accuracy by grounding responses in retrieved documents.

## Chunking Strategies

Document chunking is critical for effective retrieval.

Hierarchical chunking preserves semantic structure.
"""

chunks = chunker.chunk_document("doc_001", sample_doc)
chunker.chunks = chunks

# Display chunk hierarchy
for chunk in chunks:
    indent = "  " * (0 if chunk["level"] == "document" else 1 if chunk["level"] == "section" else 2)
    print(f"{indent}[{chunk['level']}] {chunk['chunk_id']}: {chunk['text'][:50]}...")
    
# Get context window for a paragraph chunk
para_chunk_id = chunks[-1]["chunk_id"]
context = chunker.get_context_window(para_chunk_id)
print(f"\nContext for {para_chunk_id}:")
for c in context:
    print(f"  - [{c['level']}] {c['text'][:50]}...")
```

## Configuration Patterns

### BM25 Tuning

```python
# Conservative tuning (prefer exact matches)
retriever = BM25Retriever(k1=1.2, b=0.75)

# Aggressive tuning (more forgiving on document length)
retriever = BM25Retriever(k1=2.0, b=0.5)

# Default Okapi BM25
retriever = BM25Retriever(k1=1.5, b=0.75)
```

### Chunking Strategy Selection

```python
# Small documents: Use paragraph-level chunks
if avg_doc_length < 1000:
    target_level = "paragraph"
    
# Medium documents: Use section-level chunks
elif avg_doc_length < 5000:
    target_level = "section"
    
# Large documents: Use mixed strategy
else:
    target_level = "mixed"  # Retrieve sections, expand to paragraphs
```

## Common Workflows

### Building a Complete RAG Pipeline

```python
import json
from pathlib import Path

# 1. Load corpus
corpus_dir = Path("Day_01/corpus")
documents = []

for doc_file in corpus_dir.glob("*.txt"):
    with open(doc_file, 'r') as f:
        documents.append({
            "id": doc_file.stem,
            "text": f.read()
        })

# 2. Apply hierarchical chunking
chunker = HierarchicalChunker()
all_chunks = []

for doc in documents:
    chunks = chunker.chunk_document(doc["id"], doc["text"])
    all_chunks.extend(chunks)

chunker.chunks = all_chunks

# 3. Build BM25 index on paragraph-level chunks
para_chunks = [c for c in all_chunks if c["level"] == "paragraph"]
retriever = BM25Retriever(k1=1.5, b=0.75)
retriever.build_index(para_chunks)
retriever.documents = {c["chunk_id"]: c for c in para_chunks}

# 4. Evaluate on golden dataset
recall_score = evaluate_recall_at_k(retriever, "Day_01/questions.jsonl", k=10)
print(f"Paragraph-level Recall@10: {recall_score:.4f}")

# 5. Interactive retrieval with context expansion
query = "explain hierarchical chunking"
results = retriever.search(query, k=3)

print("\nRetrieved chunks with context:")
for chunk_id in results:
    context = chunker.get_context_window(chunk_id)
    print(f"\n--- {chunk_id} ---")
    for ctx in context:
        print(f"[{ctx['level']}] {ctx['text'][:100]}...")
```

### Batch Evaluation

```python
def batch_evaluate_strategies(corpus, questions_file):
    """Compare chunking strategies"""
    strategies = {
        "document": lambda c: c["level"] == "document",
        "section": lambda c: c["level"] == "section",
        "paragraph": lambda c: c["level"] == "paragraph"
    }
    
    results = {}
    
    for strategy_name, filter_fn in strategies.items():
        # Apply strategy
        filtered_chunks = [c for c in all_chunks if filter_fn(c)]
        
        # Build index
        retriever = BM25Retriever()
        retriever.build_index(filtered_chunks)
        retriever.documents = {c["chunk_id"]: c for c in filtered_chunks}
        
        # Evaluate
        recall = evaluate_recall_at_k(retriever, questions_file, k=10)
        results[strategy_name] = recall
        
    return results

# Run comparison
scores = batch_evaluate_strategies(all_chunks, "Day_01/questions.jsonl")
for strategy, score in scores.items():
    print(f"{strategy}: Recall@10 = {score:.4f}")
```

## Troubleshooting

### Low Recall Scores

**Issue**: Recall@10 below 0.5

**Solutions**:
```python
# 1. Check tokenization consistency
query_tokens = retriever._tokenize("test query")
doc_tokens = retriever._tokenize("test document")
print(f"Query tokens: {query_tokens}")
print(f"Doc tokens: {doc_tokens}")

# 2. Verify golden dataset alignment
with open("questions.jsonl") as f:
    for line in f:
        q = json.loads(line)
        print(f"Expected chunks exist: {all(chunk in retriever.documents for chunk in q['expected_chunks'])}")

# 3. Lower BM25 b parameter for shorter chunks
retriever = BM25Retriever(k1=1.5, b=0.3)  # Less length normalization
```

### Memory Issues with Large Corpora

**Issue**: Out of memory during index building

**Solutions**:
```python
# Use generator for document loading
def document_generator(corpus_dir):
    for doc_file in Path(corpus_dir).glob("*.txt"):
        with open(doc_file) as f:
            yield {"id": doc_file.stem, "text": f.read()}

# Build index in batches
batch_size = 1000
for i, batch in enumerate(chunked(document_generator("corpus"), batch_size)):
    print(f"Processing batch {i}...")
    retriever.build_index(list(batch))
```

### Chunk ID Collisions

**Issue**: Duplicate chunk IDs

**Solutions**:
```python
# Verify deterministic IDs
chunk_ids = [c["chunk_id"] for c in all_chunks]
assert len(chunk_ids) == len(set(chunk_ids)), "Duplicate chunk IDs detected"

# Add content hash for uniqueness
def generate_chunk_id(doc_id, level, index, content):
    content_hash = hashlib.md5(content.encode()).hexdigest()[:8]
    raw_id = f"{doc_id}_{level}_{index}_{content_hash}"
    return hashlib.md5(raw_id.encode()).hexdigest()[:16]
```

## Best Practices

1. **Always evaluate on golden datasets**: Use `questions.jsonl` format for reproducible metrics
2. **Chunk granularity matters**: Start with paragraph-level, adjust based on domain
3. **Preserve metadata**: Keep parent-child relationships for context expansion
4. **Tune BM25 parameters**: Domain-specific tuning can improve recall by 10-20%
5. **Version your indices**: Track chunk IDs deterministically for reproducibility

This skill enables AI agents to build, evaluate, and optimize production RAG pipelines using proven architectures from the ai-data-engineering-roadmap project.
