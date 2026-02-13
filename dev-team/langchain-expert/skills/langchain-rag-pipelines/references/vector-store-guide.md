# Vector Store Guide

Comprehensive reference for setting up and using vector stores with LangChain. Covers Chroma, FAISS, Pinecone, Weaviate, and Qdrant with setup instructions, indexing patterns, querying strategies, metadata filtering, hybrid search, and embedding model selection.

---

## Chroma

### Setup and Installation

```bash
pip install langchain-chroma chromadb
```

### Creating a Collection

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Persistent storage
vectorstore = Chroma(
    collection_name="knowledge_base",
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)

# From documents
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="knowledge_base",
    persist_directory="./chroma_db",
)
```

### Querying

```python
# Similarity search
docs = vectorstore.similarity_search("What is LCEL?", k=5)

# With scores
docs_with_scores = vectorstore.similarity_search_with_score("What is LCEL?", k=5)

# MMR (Maximal Marginal Relevance) for diversity
docs = vectorstore.max_marginal_relevance_search("What is LCEL?", k=5, fetch_k=20, lambda_mult=0.7)
```

### Metadata Filtering

```python
docs = vectorstore.similarity_search(
    "deployment steps",
    k=5,
    filter={"department": "engineering"},
)

# Complex filters with Chroma's where clause
docs = vectorstore.similarity_search(
    "deployment steps",
    k=5,
    filter={
        "$and": [
            {"department": {"$eq": "engineering"}},
            {"version": {"$gte": 2}},
        ]
    },
)
```

### Best Use Cases

Chroma is ideal for development, prototyping, and small-to-medium production deployments. It runs embedded (no server needed) and persists to disk. For large-scale production, consider a managed solution.

---

## FAISS

### Setup and Installation

```bash
pip install faiss-cpu  # or faiss-gpu for GPU acceleration
pip install langchain-community
```

### Creating an Index

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# From documents
vectorstore = FAISS.from_documents(documents=chunks, embedding=embeddings)

# Save and load
vectorstore.save_local("./faiss_index")
loaded_vectorstore = FAISS.load_local(
    "./faiss_index",
    embeddings,
    allow_dangerous_deserialization=True,
)
```

### Querying

```python
# Basic similarity search
docs = vectorstore.similarity_search("query", k=5)

# With relevance scores (normalized 0-1)
docs_with_scores = vectorstore.similarity_search_with_relevance_scores("query", k=5)

# MMR search
docs = vectorstore.max_marginal_relevance_search("query", k=5, fetch_k=20)
```

### Metadata Filtering

```python
# FAISS supports basic metadata filtering via a filter function
docs = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"source": "internal_docs"},
)
```

### Merging Indexes

```python
# Merge two FAISS indexes
vectorstore1.merge_from(vectorstore2)
```

### Best Use Cases

FAISS excels at high-speed similarity search over large datasets. It is purely in-memory (with disk serialization) and does not require a server. Best for batch processing, offline analysis, and applications where sub-millisecond search latency matters.

---

## Pinecone

### Setup and Installation

```bash
pip install langchain-pinecone pinecone-client
```

### Creating an Index

```python
from pinecone import Pinecone, ServerlessSpec
from langchain_pinecone import PineconeVectorStore
from langchain_openai import OpenAIEmbeddings

pc = Pinecone(api_key="your-api-key")

# Create index (one-time setup)
pc.create_index(
    name="knowledge-base",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = PineconeVectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    index_name="knowledge-base",
    namespace="production",
)
```

### Querying

```python
docs = vectorstore.similarity_search("query", k=5)
docs_with_scores = vectorstore.similarity_search_with_score("query", k=5)
```

### Metadata Filtering

Pinecone supports rich metadata filtering:

```python
docs = vectorstore.similarity_search(
    "deployment process",
    k=5,
    filter={
        "department": {"$eq": "engineering"},
        "version": {"$gte": 2},
        "status": {"$in": ["published", "reviewed"]},
    },
)
```

### Namespaces

Use namespaces to partition data within a single index:

```python
# Write to namespace
vectorstore = PineconeVectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    index_name="knowledge-base",
    namespace="team-a",
)

# Query specific namespace
retriever = vectorstore.as_retriever(
    search_kwargs={"k": 5, "namespace": "team-a"}
)
```

### Best Use Cases

Pinecone is a fully managed cloud vector database. Best for production deployments requiring high availability, automatic scaling, and minimal operational overhead. Supports billions of vectors with consistent low-latency queries.

---

## Weaviate

### Setup and Installation

```bash
pip install langchain-weaviate weaviate-client
```

### Creating a Collection

```python
import weaviate
from langchain_weaviate import WeaviateVectorStore
from langchain_openai import OpenAIEmbeddings

client = weaviate.connect_to_local()  # or connect_to_weaviate_cloud()

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = WeaviateVectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    client=client,
    index_name="KnowledgeBase",
)
```

### Querying

```python
docs = vectorstore.similarity_search("query", k=5)

# With score
docs_with_scores = vectorstore.similarity_search_with_score("query", k=5)
```

### Metadata Filtering

```python
docs = vectorstore.similarity_search(
    "deployment",
    k=5,
    filters=weaviate.classes.query.Filter.by_property("department").equal("engineering"),
)
```

### Hybrid Search

Weaviate natively supports hybrid search combining BM25 keyword search with vector similarity:

```python
# Hybrid search via Weaviate's built-in support
docs = vectorstore.similarity_search(
    "deployment steps",
    k=5,
    alpha=0.75,  # 0 = pure keyword, 1 = pure vector
)
```

### Best Use Cases

Weaviate excels when hybrid search (vector + keyword) is needed out of the box. It supports GraphQL queries, multi-tenancy, and built-in vectorization modules. Good for applications requiring both semantic and keyword search without assembling a custom ensemble.

---

## Qdrant

### Setup and Installation

```bash
pip install langchain-qdrant qdrant-client
```

### Creating a Collection

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient
from langchain_openai import OpenAIEmbeddings

# Local in-memory
client = QdrantClient(":memory:")

# Or connect to server
client = QdrantClient(url="http://localhost:6333")

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = QdrantVectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="knowledge_base",
    url="http://localhost:6333",
)
```

### Querying

```python
docs = vectorstore.similarity_search("query", k=5)
docs_with_scores = vectorstore.similarity_search_with_score("query", k=5)

# MMR
docs = vectorstore.max_marginal_relevance_search("query", k=5, fetch_k=20)
```

### Metadata Filtering

Qdrant provides powerful payload filtering:

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

docs = vectorstore.similarity_search(
    "query",
    k=5,
    filter=Filter(
        must=[
            FieldCondition(key="department", match=MatchValue(value="engineering")),
            FieldCondition(key="version", range=Range(gte=2)),
        ]
    ),
)
```

### Best Use Cases

Qdrant offers advanced filtering, multi-vector support, and strong performance. It runs well both self-hosted and as a managed cloud service. Particularly strong for applications requiring complex metadata filtering with high performance.

---

## Embedding Model Selection

### Comparison Matrix

| Model | Dimensions | Speed | Quality | Cost |
|-------|-----------|-------|---------|------|
| text-embedding-3-small (OpenAI) | 1536 | Fast | Good | $0.02/1M tokens |
| text-embedding-3-large (OpenAI) | 3072 | Medium | Excellent | $0.13/1M tokens |
| all-MiniLM-L6-v2 (HuggingFace) | 384 | Very Fast | Good | Free (local) |
| bge-large-en-v1.5 (BAAI) | 1024 | Medium | Excellent | Free (local) |
| Cohere embed-v3 | 1024 | Fast | Excellent | $0.10/1M tokens |

### Selection Guidelines

- **Prototyping:** Use `all-MiniLM-L6-v2` for zero-cost local development.
- **Production (English):** Use `text-embedding-3-small` for the best cost/quality tradeoff.
- **Maximum quality:** Use `text-embedding-3-large` or `bge-large-en-v1.5`.
- **Multilingual:** Use Cohere `embed-v3` or `multilingual-e5-large`.
- **Air-gapped / privacy:** Use local HuggingFace models.

### Dimension Reduction

OpenAI's v3 embeddings support dimension reduction via the `dimensions` parameter:

```python
# Reduce dimensions for faster search and smaller storage
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=1024,  # reduced from 3072
)
```

---

## Hybrid Search Implementation

For vector stores without native hybrid search, build an ensemble:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# Keyword-based retrieval
bm25_retriever = BM25Retriever.from_documents(chunks, k=10)

# Vector-based retrieval
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# Ensemble with configurable weights
hybrid_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.3, 0.7],  # favor semantic similarity
)
```

---

## Indexing Best Practices

1. **Batch inserts** -- add documents in batches of 100-500 to balance throughput and memory.
2. **Metadata normalization** -- standardize metadata keys and value types before indexing.
3. **Deduplication** -- use document hashes or IDs to prevent duplicate entries.
4. **Index updates** -- use LangChain's `index()` function with a record manager for incremental updates.
5. **Backup strategy** -- regularly export/backup vector store data, especially for self-hosted solutions.
6. **Monitor index size** -- track vector count and storage usage to plan scaling.

```python
from langchain.indexes import SQLRecordManager, index

record_manager = SQLRecordManager(
    namespace="my_collection",
    db_url="sqlite:///record_manager.db",
)
record_manager.create_schema()

# Incremental indexing with cleanup
result = index(
    docs_to_index,
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
print(f"Added: {result['num_added']}, Updated: {result['num_updated']}, "
      f"Skipped: {result['num_skipped']}, Deleted: {result['num_deleted']}")
```
