---
name: langchain-rag-pipelines
triggers:
  - "RAG pipeline"
  - "vector store"
  - "document loader"
  - "text splitter"
  - "embeddings"
  - "retriever"
description: |
  This skill should be used when the task involves building Retrieval-Augmented Generation pipelines with LangChain, including document loading, text splitting, embedding generation, vector store setup and querying, retriever configuration, and assembling end-to-end RAG chains. It covers both naive and advanced RAG patterns.
---

# LangChain RAG Pipelines

## RAG Architecture Overview

Retrieval-Augmented Generation combines a retrieval system with a generative model. The pipeline has two phases: an **indexing phase** (load, split, embed, store) and a **query phase** (retrieve, augment prompt, generate).

```
Indexing:  Documents -> Loader -> Splitter -> Embeddings -> Vector Store
Query:    Question -> Retriever -> Context + Question -> LLM -> Answer
```

## Document Loading

### Common Loaders

To load documents from various sources, use the appropriate loader:

```python
from langchain_community.document_loaders import (
    PyPDFLoader,
    TextLoader,
    CSVLoader,
    WebBaseLoader,
    DirectoryLoader,
    UnstructuredMarkdownLoader,
)

# Single PDF
pdf_docs = PyPDFLoader("report.pdf").load()

# Entire directory of files
dir_docs = DirectoryLoader(
    "docs/",
    glob="**/*.md",
    loader_cls=UnstructuredMarkdownLoader,
    show_progress=True,
).load()

# Web pages
web_docs = WebBaseLoader(
    web_paths=["https://example.com/page1", "https://example.com/page2"],
).load()
```

### Adding Metadata

To enrich documents with metadata for later filtering:

```python
for doc in docs:
    doc.metadata["source_type"] = "internal_kb"
    doc.metadata["department"] = "engineering"
    doc.metadata["indexed_at"] = datetime.utcnow().isoformat()
```

## Text Splitting

### Choosing the Right Splitter

To split documents while preserving semantic coherence:

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    MarkdownHeaderTextSplitter,
    TokenTextSplitter,
    HTMLSectionSplitter,
)

# General purpose -- respects paragraph and sentence boundaries
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len,
)
chunks = splitter.split_documents(docs)
```

To split Markdown by headers for structure-aware chunking:

```python
headers_to_split = [
    ("#", "h1"),
    ("##", "h2"),
    ("###", "h3"),
]

md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split)
md_chunks = md_splitter.split_text(markdown_text)
```

To split by token count for precise context window management:

```python
token_splitter = TokenTextSplitter(
    encoding_name="cl100k_base",
    chunk_size=500,
    chunk_overlap=50,
)
```

### Chunk Size Guidelines

- **Small chunks (200-500 tokens):** Higher retrieval precision, more chunks needed, risk of losing context.
- **Medium chunks (500-1000 tokens):** Good balance for most use cases.
- **Large chunks (1000-2000 tokens):** More context per chunk, lower precision, fewer chunks needed.
- **Overlap (10-20% of chunk size):** Prevents information loss at boundaries.

## Embedding Models

### Selecting an Embedding Model

To choose and configure embedding models:

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.embeddings import HuggingFaceEmbeddings

# OpenAI (high quality, API cost)
openai_embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",  # or text-embedding-3-large
    dimensions=1536,
)

# Local HuggingFace (free, no API dependency)
local_embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True},
)
```

### Caching Embeddings

To avoid recomputing embeddings for the same documents:

```python
from langchain_community.embeddings import CacheBackedEmbeddings
from langchain_community.storage import LocalFileStore

store = LocalFileStore("./embedding_cache")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=openai_embeddings,
    document_embedding_cache=store,
    namespace="text-embedding-3-small",
)
```

## Vector Store Setup

### Creating and Populating a Vector Store

To set up a vector store and index documents:

```python
from langchain_chroma import Chroma
from langchain_community.vectorstores import FAISS

# Chroma (persistent, good for development)
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=cached_embeddings,
    collection_name="knowledge_base",
    persist_directory="./chroma_db",
)

# FAISS (in-memory, fast for prototyping)
vectorstore = FAISS.from_documents(
    documents=chunks,
    embedding=cached_embeddings,
)
vectorstore.save_local("./faiss_index")
```

### Incremental Indexing

To add documents to an existing vector store without re-indexing everything:

```python
from langchain.indexes import SQLRecordManager, index

record_manager = SQLRecordManager(
    namespace="knowledge_base",
    db_url="sqlite:///record_manager.db",
)
record_manager.create_schema()

result = index(
    new_chunks,
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
# result = {"num_added": 15, "num_updated": 3, "num_skipped": 42, "num_deleted": 0}
```

## Retriever Configuration

### Basic Retriever

To create a retriever from a vector store:

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",       # or "mmr" or "similarity_score_threshold"
    search_kwargs={
        "k": 5,                     # number of documents to retrieve
        "score_threshold": 0.7,     # minimum similarity (for threshold search)
    },
)
```

### Multi-Query Retriever

To improve recall by generating multiple query perspectives:

```python
from langchain.retrievers import MultiQueryRetriever

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0.3),
)
```

### Contextual Compression Retriever

To filter and compress retrieved documents for relevance:

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(ChatOpenAI(model="gpt-4o-mini"))
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)
```

### Ensemble Retriever

To combine multiple retrieval strategies:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(chunks, k=5)
semantic = vectorstore.as_retriever(search_kwargs={"k": 5})

ensemble = EnsembleRetriever(
    retrievers=[bm25, semantic],
    weights=[0.4, 0.6],
)
```

## Basic RAG Chain

### Minimal RAG with LCEL

To assemble a complete RAG chain:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda

def format_docs(docs):
    return "\n\n".join(
        f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
        for doc in docs
    )

prompt = ChatPromptTemplate.from_messages([
    ("system", (
        "Answer the question based only on the following context. "
        "If the context does not contain the answer, say so.\n\n"
        "Context:\n{context}"
    )),
    ("human", "{question}"),
])

rag_chain = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough(),
    )
    | prompt
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)

answer = rag_chain.invoke("How does incremental indexing work?")
```

### RAG with Source Attribution

To return both the answer and the source documents:

```python
rag_with_sources = RunnableParallel(
    answer=rag_chain,
    sources=retriever | RunnableLambda(
        lambda docs: [
            {"source": d.metadata.get("source"), "page": d.metadata.get("page")}
            for d in docs
        ]
    ),
)

result = rag_with_sources.invoke("What are the deployment steps?")
# result = {"answer": "...", "sources": [{"source": "deploy.md", "page": 3}, ...]}
```

### Conversational RAG

To add conversation history to a RAG pipeline:

```python
from langchain_core.prompts import MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

contextualize_prompt = ChatPromptTemplate.from_messages([
    ("system", "Given the chat history and latest question, reformulate the question to be standalone."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

contextualize_chain = contextualize_prompt | llm | StrOutputParser()

qa_prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer based on context:\n{context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(llm, retriever, contextualize_prompt)

conversational_rag = create_retrieval_chain(history_aware_retriever, combine_docs_chain)

store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

with_history = RunnableWithMessageHistory(
    conversational_rag,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history",
    output_messages_key="answer",
)
```

## Production Considerations

1. **Chunk size tuning** -- experiment with different chunk sizes and measure retrieval quality using LangSmith evaluation datasets.
2. **Embedding model selection** -- balance quality, cost, and latency. Benchmark with your actual data.
3. **Retrieval diversity** -- use MMR (Maximal Marginal Relevance) or ensemble retrievers to avoid redundant results.
4. **Metadata filtering** -- add rich metadata at indexing time to enable filtered retrieval at query time.
5. **Re-indexing strategy** -- use incremental indexing with record managers to handle document updates efficiently.
6. **Context window management** -- track token counts and truncate context to fit within model limits.

## References

For advanced RAG patterns including multi-query, HyDE, parent-child retrieval, self-RAG, CRAG, and reranking strategies, see [rag-patterns.md](references/rag-patterns.md).

For detailed vector store setup guides covering Chroma, FAISS, Pinecone, Weaviate, and Qdrant with metadata filtering and hybrid search, see [vector-store-guide.md](references/vector-store-guide.md).
