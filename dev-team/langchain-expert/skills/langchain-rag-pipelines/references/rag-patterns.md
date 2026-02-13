# Advanced RAG Patterns

This reference covers advanced Retrieval-Augmented Generation patterns beyond naive RAG, with LCEL-based implementations for each approach.

---

## Naive RAG

The baseline pattern: retrieve top-k documents, stuff into prompt, generate answer.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda
from langchain_openai import ChatOpenAI

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer based on context:\n{context}"),
    ("human", "{question}"),
])

naive_rag = (
    RunnableParallel(
        context=retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough(),
    )
    | prompt
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)
```

**Limitations:** Single query perspective, no relevance filtering, no query transformation, retrieval quality depends entirely on embedding similarity.

---

## Advanced RAG: Multi-Query Retrieval

Generate multiple reformulations of the user query to improve recall. Different phrasings capture different relevant documents.

```python
from langchain.retrievers import MultiQueryRetriever
from langchain_openai import ChatOpenAI

multi_query_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0.7),
)

# Generates 3 query variations by default, retrieves for each, deduplicates
multi_query_rag = (
    RunnableParallel(
        context=multi_query_retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough(),
    )
    | prompt
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)
```

### Custom Multi-Query Generation

To control how query variations are generated:

```python
from langchain_core.prompts import ChatPromptTemplate

query_gen_prompt = ChatPromptTemplate.from_template(
    "Generate 3 different search queries to find information that answers "
    "this question from different angles. Return one query per line.\n\n"
    "Question: {question}\n\nQueries:"
)

query_gen_chain = query_gen_prompt | llm | StrOutputParser()

def generate_and_retrieve(question: str):
    queries_text = query_gen_chain.invoke({"question": question})
    queries = [q.strip() for q in queries_text.strip().split("\n") if q.strip()]
    queries.append(question)  # always include original

    all_docs = []
    seen_ids = set()
    for q in queries:
        docs = retriever.invoke(q)
        for doc in docs:
            doc_id = doc.metadata.get("id", doc.page_content[:100])
            if doc_id not in seen_ids:
                seen_ids.add(doc_id)
                all_docs.append(doc)
    return all_docs
```

---

## Advanced RAG: HyDE (Hypothetical Document Embedding)

Generate a hypothetical answer first, then use its embedding to find similar real documents. This bridges the gap between question embeddings and document embeddings.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

hyde_prompt = ChatPromptTemplate.from_template(
    "Write a detailed passage that would answer this question. "
    "Do not indicate uncertainty -- write as if you know the answer.\n\n"
    "Question: {question}\n\nPassage:"
)

hyde_chain = hyde_prompt | ChatOpenAI(model="gpt-4o-mini") | StrOutputParser()

def hyde_retrieve(question: str):
    hypothetical_doc = hyde_chain.invoke({"question": question})
    # Use the hypothetical doc as the search query
    docs = vectorstore.similarity_search(hypothetical_doc, k=5)
    return docs

hyde_retriever = RunnableLambda(hyde_retrieve)

hyde_rag = (
    RunnableParallel(
        context=hyde_retriever | RunnableLambda(format_docs),
        question=RunnablePassthrough(),
    )
    | prompt
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)
```

---

## Advanced RAG: Step-Back Prompting

Abstract the question to a higher-level concept, retrieve for both the original and abstracted question, then generate from the combined context.

```python
step_back_prompt = ChatPromptTemplate.from_template(
    "Given this question, generate a more general 'step-back' question that "
    "captures the broader concept needed to answer it.\n\n"
    "Original: {question}\n\nStep-back question:"
)

step_back_chain = step_back_prompt | llm | StrOutputParser()

def step_back_retrieve(question: str):
    step_back_question = step_back_chain.invoke({"question": question})
    original_docs = retriever.invoke(question)
    step_back_docs = retriever.invoke(step_back_question)

    # Combine and deduplicate
    seen = set()
    combined = []
    for doc in original_docs + step_back_docs:
        key = doc.page_content[:100]
        if key not in seen:
            seen.add(key)
            combined.append(doc)
    return combined

step_back_rag = (
    RunnableParallel(
        context=RunnableLambda(step_back_retrieve) | RunnableLambda(format_docs),
        question=RunnablePassthrough(),
    )
    | prompt
    | llm
    | StrOutputParser()
)
```

---

## Parent-Child Retrieval

Index small chunks (children) for precise retrieval, but return the larger parent documents for richer context.

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain.storage import InMemoryStore

parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=50)

docstore = InMemoryStore()

parent_retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# Index documents -- stores parents in docstore, children in vectorstore
parent_retriever.add_documents(documents)

# Retrieval: searches children, returns parents
parent_docs = parent_retriever.invoke("What is the deployment process?")
```

---

## Contextual Compression

Retrieve broadly, then compress each document to extract only the relevant portions.

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import (
    LLMChainExtractor,
    LLMChainFilter,
    EmbeddingsFilter,
    DocumentCompressorPipeline,
)
from langchain_text_splitters import CharacterTextSplitter

# LLM-based extraction: extracts relevant sentences
llm_extractor = LLMChainExtractor.from_llm(ChatOpenAI(model="gpt-4o-mini"))

# Embedding-based filter: removes documents below similarity threshold
embedding_filter = EmbeddingsFilter(
    embeddings=embeddings,
    similarity_threshold=0.75,
)

# Pipeline: split -> filter by embedding -> extract with LLM
compressor_pipeline = DocumentCompressorPipeline(
    transformers=[
        CharacterTextSplitter(chunk_size=500, chunk_overlap=0),
        embedding_filter,
        llm_extractor,
    ]
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor_pipeline,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

---

## Self-RAG (Self-Reflective RAG)

The model decides whether to retrieve, evaluates retrieved documents for relevance, and assesses its own answer for groundedness.

```python
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field
from typing import Literal

class RetrievalDecision(BaseModel):
    needs_retrieval: bool = Field(description="Whether retrieval is needed")
    reasoning: str = Field(description="Why retrieval is or isn't needed")

class RelevanceGrade(BaseModel):
    is_relevant: Literal["yes", "no"]
    reasoning: str

class GroundednessCheck(BaseModel):
    is_grounded: Literal["yes", "no"]
    reasoning: str

decide_llm = ChatOpenAI(model="gpt-4o-mini").with_structured_output(RetrievalDecision)
grade_llm = ChatOpenAI(model="gpt-4o-mini").with_structured_output(RelevanceGrade)
ground_llm = ChatOpenAI(model="gpt-4o-mini").with_structured_output(GroundednessCheck)

def self_rag(question: str) -> str:
    # Step 1: Decide if retrieval is needed
    decision = decide_llm.invoke(f"Does this question need document retrieval? {question}")

    if decision.needs_retrieval:
        docs = retriever.invoke(question)

        # Step 2: Grade each document for relevance
        relevant_docs = []
        for doc in docs:
            grade = grade_llm.invoke(
                f"Is this document relevant to '{question}'?\n\n{doc.page_content}"
            )
            if grade.is_relevant == "yes":
                relevant_docs.append(doc)

        context = format_docs(relevant_docs) if relevant_docs else "No relevant documents found."
    else:
        context = "No retrieval performed."

    # Step 3: Generate answer
    answer = rag_chain.invoke({"context": context, "question": question})

    # Step 4: Check groundedness
    check = ground_llm.invoke(
        f"Is this answer grounded in the context?\n\nContext: {context}\n\nAnswer: {answer}"
    )
    if check.is_grounded == "no":
        answer += "\n\n[Warning: Answer may not be fully supported by retrieved documents]"

    return answer
```

---

## CRAG (Corrective RAG)

Evaluate retrieval quality and fall back to web search if documents are not relevant.

```python
def corrective_rag(question: str) -> str:
    docs = retriever.invoke(question)

    # Grade retrieval quality
    relevant = []
    for doc in docs:
        grade = grade_llm.invoke(
            f"Is this relevant to '{question}'?\n\n{doc.page_content}"
        )
        if grade.is_relevant == "yes":
            relevant.append(doc)

    # If insufficient relevant docs, supplement with web search
    if len(relevant) < 2:
        web_results = web_search_tool.invoke(question)
        from langchain_core.documents import Document
        web_docs = [Document(page_content=r, metadata={"source": "web"}) for r in web_results]
        relevant.extend(web_docs)

    context = format_docs(relevant)
    return generation_chain.invoke({"context": context, "question": question})
```

---

## Reranking

Retrieve a broad set of candidates and rerank with a cross-encoder for higher precision.

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_community.document_compressors import CohereRerank

# Cohere reranker
reranker = CohereRerank(
    model="rerank-english-v3.0",
    top_n=5,
)

reranking_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)

# Or implement custom reranking with a cross-encoder
from sentence_transformers import CrossEncoder

cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank_docs(question: str, docs, top_k: int = 5):
    pairs = [(question, doc.page_content) for doc in docs]
    scores = cross_encoder.predict(pairs)
    scored_docs = sorted(zip(scores, docs), key=lambda x: x[0], reverse=True)
    return [doc for _, doc in scored_docs[:top_k]]
```

---

## Fusion RAG (Reciprocal Rank Fusion)

Combine results from multiple retrieval strategies using reciprocal rank fusion scoring.

```python
def reciprocal_rank_fusion(results_list: list[list], k: int = 60) -> list:
    """Fuse multiple ranked lists using RRF scoring."""
    fused_scores = {}
    doc_map = {}

    for results in results_list:
        for rank, doc in enumerate(results):
            doc_id = doc.page_content[:100]
            if doc_id not in fused_scores:
                fused_scores[doc_id] = 0.0
                doc_map[doc_id] = doc
            fused_scores[doc_id] += 1.0 / (rank + k)

    sorted_ids = sorted(fused_scores, key=fused_scores.get, reverse=True)
    return [doc_map[doc_id] for doc_id in sorted_ids]

def fusion_retrieve(question: str):
    queries = [question] + generate_query_variations(question)
    all_results = [retriever.invoke(q) for q in queries]
    return reciprocal_rank_fusion(all_results)
```

---

## Pattern Selection Guide

| Pattern | When to Use | Complexity | Cost |
|---------|------------|------------|------|
| Naive RAG | Prototyping, simple Q&A | Low | Low |
| Multi-Query | Low recall, ambiguous queries | Low | Medium |
| HyDE | Short queries, vocabulary mismatch | Medium | Medium |
| Step-Back | Questions requiring background knowledge | Medium | Medium |
| Parent-Child | Need both precision and context | Medium | Low |
| Contextual Compression | Long documents, noisy retrieval | Medium | High |
| Self-RAG | High-stakes, need groundedness | High | High |
| CRAG | Knowledge base may be incomplete | High | High |
| Reranking | High precision needed, budget available | Medium | Medium |
| Fusion RAG | Multiple retrieval strategies available | Medium | Medium |

For small-scale prototypes, start with Naive RAG and add complexity only when evaluation metrics show gaps. For production systems handling diverse queries, combine Multi-Query retrieval with Reranking as a strong baseline.
