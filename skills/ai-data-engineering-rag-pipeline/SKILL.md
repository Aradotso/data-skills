---
name: ai-data-engineering-rag-pipeline
description: Production-grade local RAG pipeline with BM25 retrieval, hierarchical chunking, and evaluation contracts for document Q&A systems
triggers:
  - build a local RAG pipeline with BM25
  - implement hierarchical chunking for retrieval
  - create a document retrieval system with evaluation
  - set up BM25 baseline search engine
  - evaluate RAG pipeline with recall metrics
  - design chunking strategy for RAG
  - implement retrieval contracts and golden datasets
  - build production RAG with inverted index
---

# AI Data Engineering RAG Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides a production-grade local Retrieval-Augmented Generation (RAG) pipeline implementation with BM25-based retrieval, hierarchical chunking strategies, and comprehensive evaluation frameworks. It demonstrates architectural patterns for building testable, scalable document retrieval systems.

## What This Project Does

The roadmap implements three core modules:

1. **Day-01**: Establishes retrieval contracts with governed corpus, golden datasets (`questions.jsonl`), and evaluation scripts
2. **Day-02**: Implements BM25 baseline search engine with inverted index, Okapi BM25 ranking, and Recall@10 evaluation
3. **Day-03**: Builds full RAG pipeline with hierarchical chunking (document/section/paragraph levels), parent-child metadata linkage, and failure mode analysis

## Installation

```bash
# Clone the repository
git clone https://github.com/Nahid-mahmud555/ai-data-engineering-roadmap.git
cd ai-data-engineering-roadmap

# Install dependencies (project uses Python)
pip install -r requirements.txt
```

Common dependencies for RAG pipelines:
```bash
pip install rank-bm25 numpy pandas scikit-learn sentence-transformers
```

## Project Structure

```
ai-data-engineering-roadmap/
├── Day_01/  # Retrieval contracts and golden datasets
├── Day_02/  # BM25 baseline implementation
└── Day_03/  # Full RAG pipeline with hierarchical chunking
```

## Key Components

### Day-01: Retrieval Contracts & Golden Datasets

Set up evaluation infrastructure with golden question-answer pairs:

```python
import json

# Load golden dataset
def load_golden_questions(filepath="questions.jsonl"):
    """Load golden Q&A pairs for evaluation"""
    questions = []
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            questions.append(json.loads(line))
    return questions

# Golden dataset format
golden_qa = {
    "question_id": "q001",
    "question": "What are the key components of a RAG pipeline?",
    "expected_doc_ids": ["doc_123", "doc_456"],
    "context": "technical_overview"
}

# Save golden dataset
def save_golden_questions(questions, filepath="questions.jsonl"):
    with open(filepath, 'w', encoding='utf-8') as f:
        for q in questions:
            f.write(json.dumps(q, ensure_ascii=False) + '\n')
```

### Day-02: BM25 Baseline Search Engine

Implement Okapi BM25 ranking with inverted index:

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
        # Tokenize corpus
        self.tokenized_corpus = [doc.lower().split() for doc in corpus]
        # Build BM25 index
        self.bm25 = BM25Okapi(self.tokenized_corpus)
    
    def search(self, query, top_k=10):
        """
        Retrieve top-k documents for query
        
        Args:
            query: Search query string
            top_k: Number of results to return
            
        Returns:
            List of (doc_index, score) tuples
        """
        tokenized_query = query.lower().split()
        scores = self.bm25.get_scores(tokenized_query)
        
        # Get top-k indices
        top_indices = np.argsort(scores)[::-1][:top_k]
        results = [(idx, scores[idx]) for idx in top_indices]
        
        return results
    
    def evaluate_recall(self, questions, k=10):
        """
        Evaluate Recall@k on golden dataset
        
        Args:
            questions: List of golden Q&A dicts
            k: Recall cutoff
            
        Returns:
            recall_at_k: Float between 0 and 1
        """
        hits = 0
        total = 0
        
        for q in questions:
            query = q['question']
            expected_ids = set(q['expected_doc_ids'])
            
            results = self.search(query, top_k=k)
            retrieved_ids = set([str(idx) for idx, _ in results])
            
            # Check if any expected doc in top-k
            if expected_ids & retrieved_ids:
                hits += 1
            total += 1
        
        return hits / total if total > 0 else 0.0

# Usage example
corpus = [
    "RAG combines retrieval and generation for QA",
    "BM25 is a probabilistic ranking function",
    "Chunking strategies affect retrieval quality"
]

engine = BM25SearchEngine(corpus)
results = engine.search("What is RAG?", top_k=3)

for idx, score in results:
    print(f"Doc {idx} (score: {score:.2f}): {corpus[idx]}")
```

### Day-03: Hierarchical Chunking Pipeline

Implement multi-granularity chunking with parent-child relationships:

```python
import hashlib
from typing import List, Dict, Optional

class Document:
    def __init__(self, doc_id: str, content: str, metadata: Dict = None):
        self.doc_id = doc_id
        self.content = content
        self.metadata = metadata or {}

class Chunk:
    def __init__(
        self, 
        content: str, 
        doc_id: str, 
        level: str,
        parent_id: Optional[str] = None,
        metadata: Dict = None
    ):
        self.content = content
        self.doc_id = doc_id
        self.level = level  # 'document', 'section', 'paragraph'
        self.parent_id = parent_id
        self.metadata = metadata or {}
        self.chunk_id = self._generate_chunk_id()
    
    def _generate_chunk_id(self) -> str:
        """Generate deterministic chunk ID"""
        content_hash = hashlib.md5(self.content.encode()).hexdigest()[:8]
        return f"{self.doc_id}_{self.level}_{content_hash}"

class HierarchicalChunker:
    def __init__(
        self, 
        paragraph_size: int = 512,
        section_size: int = 2048
    ):
        self.paragraph_size = paragraph_size
        self.section_size = section_size
    
    def chunk_document(self, document: Document) -> List[Chunk]:
        """
        Create hierarchical chunks from document
        
        Returns:
            List of Chunk objects at different granularities
        """
        chunks = []
        
        # Level 1: Document-level chunk
        doc_chunk = Chunk(
            content=document.content,
            doc_id=document.doc_id,
            level='document',
            metadata=document.metadata
        )
        chunks.append(doc_chunk)
        
        # Level 2: Section-level chunks (split by double newline)
        sections = document.content.split('\n\n')
        for i, section in enumerate(sections):
            if len(section.strip()) == 0:
                continue
            
            section_chunk = Chunk(
                content=section,
                doc_id=document.doc_id,
                level='section',
                parent_id=doc_chunk.chunk_id,
                metadata={'section_index': i}
            )
            chunks.append(section_chunk)
            
            # Level 3: Paragraph-level chunks
            paragraphs = self._split_by_tokens(section, self.paragraph_size)
            for j, para in enumerate(paragraphs):
                para_chunk = Chunk(
                    content=para,
                    doc_id=document.doc_id,
                    level='paragraph',
                    parent_id=section_chunk.chunk_id,
                    metadata={'paragraph_index': j, 'section_index': i}
                )
                chunks.append(para_chunk)
        
        return chunks
    
    def _split_by_tokens(self, text: str, max_tokens: int) -> List[str]:
        """Split text into chunks by approximate token count"""
        words = text.split()
        chunks = []
        current_chunk = []
        current_length = 0
        
        for word in words:
            word_length = len(word.split()) + 1  # Approximate tokens
            if current_length + word_length > max_tokens:
                if current_chunk:
                    chunks.append(' '.join(current_chunk))
                current_chunk = [word]
                current_length = word_length
            else:
                current_chunk.append(word)
                current_length += word_length
        
        if current_chunk:
            chunks.append(' '.join(current_chunk))
        
        return chunks

# RAG Pipeline with Hierarchical Retrieval
class RAGPipeline:
    def __init__(self, chunker: HierarchicalChunker):
        self.chunker = chunker
        self.chunks = []
        self.search_engines = {}  # One per granularity level
    
    def index_documents(self, documents: List[Document]):
        """Index documents with hierarchical chunking"""
        all_chunks = []
        for doc in documents:
            chunks = self.chunker.chunk_document(doc)
            all_chunks.extend(chunks)
        
        self.chunks = all_chunks
        
        # Build separate BM25 index for each level
        for level in ['document', 'section', 'paragraph']:
            level_chunks = [c for c in all_chunks if c.level == level]
            if level_chunks:
                corpus = [c.content for c in level_chunks]
                self.search_engines[level] = BM25SearchEngine(corpus)
    
    def retrieve(
        self, 
        query: str, 
        level: str = 'paragraph',
        top_k: int = 5
    ) -> List[Chunk]:
        """
        Retrieve chunks at specified granularity
        
        Args:
            query: Search query
            level: 'document', 'section', or 'paragraph'
            top_k: Number of chunks to return
        """
        if level not in self.search_engines:
            raise ValueError(f"No index for level: {level}")
        
        engine = self.search_engines[level]
        results = engine.search(query, top_k=top_k)
        
        # Map results back to chunks
        level_chunks = [c for c in self.chunks if c.level == level]
        retrieved_chunks = [level_chunks[idx] for idx, _ in results]
        
        return retrieved_chunks
    
    def retrieve_with_context(
        self, 
        query: str, 
        retrieval_level: str = 'paragraph',
        context_level: str = 'section',
        top_k: int = 5
    ) -> List[Dict]:
        """
        Retrieve at one level but return parent context
        
        Args:
            query: Search query
            retrieval_level: Level to search at
            context_level: Level to return as context
            top_k: Number of results
            
        Returns:
            List of dicts with chunk and context
        """
        retrieved = self.retrieve(query, level=retrieval_level, top_k=top_k)
        
        results = []
        for chunk in retrieved:
            # Find parent chunk
            parent = next(
                (c for c in self.chunks if c.chunk_id == chunk.parent_id),
                None
            )
            
            results.append({
                'chunk': chunk,
                'context': parent if parent and parent.level == context_level else chunk,
                'metadata': chunk.metadata
            })
        
        return results

# Usage example
documents = [
    Document(
        doc_id="doc001",
        content="""Introduction to RAG

Retrieval-Augmented Generation combines information retrieval with language generation.

Architecture Components

The system consists of a retriever and a generator. The retriever finds relevant documents.""",
        metadata={'source': 'tutorial.md'}
    )
]

chunker = HierarchicalChunker(paragraph_size=128, section_size=512)
pipeline = RAGPipeline(chunker)
pipeline.index_documents(documents)

# Retrieve paragraphs with section context
results = pipeline.retrieve_with_context(
    query="What are the components of RAG?",
    retrieval_level='paragraph',
    context_level='section',
    top_k=3
)

for result in results:
    print(f"Chunk ID: {result['chunk'].chunk_id}")
    print(f"Content: {result['chunk'].content[:100]}...")
    print(f"Context: {result['context'].content[:100]}...")
    print(f"Metadata: {result['metadata']}\n")
```

## Evaluation Patterns

### Recall@K Evaluation

```python
def evaluate_retrieval_pipeline(pipeline, golden_questions, k=10):
    """Comprehensive evaluation across chunk levels"""
    results = {}
    
    for level in ['document', 'section', 'paragraph']:
        recall_scores = []
        
        for q in golden_questions:
            query = q['question']
            expected_ids = set(q['expected_doc_ids'])
            
            try:
                retrieved = pipeline.retrieve(query, level=level, top_k=k)
                retrieved_ids = set([c.doc_id for c in retrieved])
                
                # Calculate recall
                hits = len(expected_ids & retrieved_ids)
                recall = hits / len(expected_ids) if expected_ids else 0
                recall_scores.append(recall)
            except Exception as e:
                print(f"Error for query '{query}': {e}")
                recall_scores.append(0)
        
        avg_recall = sum(recall_scores) / len(recall_scores)
        results[level] = {
            'recall_at_k': avg_recall,
            'k': k,
            'num_queries': len(golden_questions)
        }
    
    return results

# Run evaluation
eval_results = evaluate_retrieval_pipeline(pipeline, golden_questions, k=10)
for level, metrics in eval_results.items():
    print(f"{level.upper()}: Recall@{metrics['k']} = {metrics['recall_at_k']:.3f}")
```

## Configuration

### Environment Variables

```bash
# Set corpus location
export RAG_CORPUS_PATH=/path/to/documents

# Set golden dataset path
export RAG_GOLDEN_DATASET=/path/to/questions.jsonl

# BM25 parameters
export BM25_K1=1.5
export BM25_B=0.75

# Chunking parameters
export CHUNK_PARAGRAPH_SIZE=512
export CHUNK_SECTION_SIZE=2048
```

### Loading Configuration in Code

```python
import os

class RAGConfig:
    CORPUS_PATH = os.getenv('RAG_CORPUS_PATH', './corpus')
    GOLDEN_DATASET = os.getenv('RAG_GOLDEN_DATASET', './questions.jsonl')
    BM25_K1 = float(os.getenv('BM25_K1', '1.5'))
    BM25_B = float(os.getenv('BM25_B', '0.75'))
    PARAGRAPH_SIZE = int(os.getenv('CHUNK_PARAGRAPH_SIZE', '512'))
    SECTION_SIZE = int(os.getenv('CHUNK_SECTION_SIZE', '2048'))
```

## Common Patterns

### Pattern 1: Corpus Ingestion

```python
import glob

def ingest_corpus(corpus_dir: str) -> List[Document]:
    """Ingest all markdown files from directory"""
    documents = []
    
    for filepath in glob.glob(f"{corpus_dir}/**/*.md", recursive=True):
        with open(filepath, 'r', encoding='utf-8') as f:
            content = f.read()
            doc_id = filepath.replace(corpus_dir, '').strip('/').replace('/', '_')
            
            documents.append(Document(
                doc_id=doc_id,
                content=content,
                metadata={'filepath': filepath}
            ))
    
    return documents
```

### Pattern 2: Failure Mode Analysis

```python
def analyze_failures(pipeline, golden_questions, threshold=0.5):
    """Identify queries with low retrieval performance"""
    failures = []
    
    for q in golden_questions:
        results = pipeline.retrieve(q['question'], top_k=10)
        retrieved_ids = set([c.doc_id for c in results])
        expected_ids = set(q['expected_doc_ids'])
        
        recall = len(expected_ids & retrieved_ids) / len(expected_ids)
        
        if recall < threshold:
            failures.append({
                'question': q['question'],
                'recall': recall,
                'expected': list(expected_ids),
                'retrieved': list(retrieved_ids)
            })
    
    return failures
```

### Pattern 3: Deterministic Chunk IDs

```python
def generate_deterministic_id(doc_id: str, content: str, level: str) -> str:
    """Create reproducible chunk identifier"""
    content_normalized = content.strip().lower()
    content_hash = hashlib.sha256(content_normalized.encode()).hexdigest()[:12]
    return f"{doc_id}::{level}::{content_hash}"
```

## Troubleshooting

### Issue: Low Recall Scores

**Problem**: Recall@10 < 0.3

**Solutions**:
- Reduce chunk size for finer granularity
- Try different chunking levels (paragraph vs section)
- Expand query with synonyms
- Check if golden dataset doc IDs match indexed doc IDs

```python
# Debug retrieval
def debug_retrieval(pipeline, query, expected_doc_id):
    for level in ['paragraph', 'section', 'document']:
        results = pipeline.retrieve(query, level=level, top_k=5)
        print(f"\n{level.upper()} LEVEL:")
        for i, chunk in enumerate(results):
            match = "✓" if chunk.doc_id == expected_doc_id else "✗"
            print(f"  {match} {i+1}. {chunk.doc_id}: {chunk.content[:80]}...")
```

### Issue: Memory Usage with Large Corpus

**Problem**: Out of memory with 10k+ documents

**Solutions**:
- Process documents in batches
- Use sparse matrix representation for BM25
- Index only specific chunk levels

```python
def index_in_batches(pipeline, documents, batch_size=100):
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i+batch_size]
        pipeline.index_documents(batch)
```

### Issue: Inconsistent Chunk IDs

**Problem**: Chunk IDs change between runs

**Solution**: Ensure deterministic ID generation

```python
# Verify chunk ID stability
def verify_chunk_stability(doc, chunker, runs=3):
    chunk_ids_per_run = []
    
    for _ in range(runs):
        chunks = chunker.chunk_document(doc)
        chunk_ids = [c.chunk_id for c in chunks]
        chunk_ids_per_run.append(chunk_ids)
    
    # All runs should produce identical IDs
    assert all(ids == chunk_ids_per_run[0] for ids in chunk_ids_per_run)
```

This skill provides comprehensive patterns for building, evaluating, and troubleshooting production RAG pipelines using the architectures demonstrated in this roadmap.
