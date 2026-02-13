# Data Protection for LLM Applications

Comprehensive reference covering PII detection with Presidio, RAG multi-tenancy, vector store access control, audit logging, data retention, and compliance considerations.

---

## PII Detection with Presidio

### Setup

```bash
pip install presidio-analyzer presidio-anonymizer
python -m spacy download en_core_web_lg
```

### LangChain Integration

```python
from presidio_analyzer import AnalyzerEngine, RecognizerRegistry
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

def create_pii_redactor(
    entities: list[str] | None = None,
    language: str = "en",
):
    """Create a configurable PII redaction function."""
    target_entities = entities or [
        "PHONE_NUMBER", "EMAIL_ADDRESS", "CREDIT_CARD",
        "US_SSN", "IBAN_CODE", "PERSON", "LOCATION",
    ]

    def redact(text: str) -> str:
        results = analyzer.analyze(
            text=text,
            language=language,
            entities=target_entities,
            score_threshold=0.6,
        )

        if not results:
            return text

        anonymized = anonymizer.anonymize(
            text=text,
            analyzer_results=results,
            operators={
                "PERSON": OperatorConfig("replace", {"new_value": "<PERSON>"}),
                "PHONE_NUMBER": OperatorConfig("replace", {"new_value": "<PHONE>"}),
                "EMAIL_ADDRESS": OperatorConfig("replace", {"new_value": "<EMAIL>"}),
                "CREDIT_CARD": OperatorConfig("replace", {"new_value": "<CREDIT_CARD>"}),
                "US_SSN": OperatorConfig("replace", {"new_value": "<SSN>"}),
                "DEFAULT": OperatorConfig("replace", {"new_value": "<REDACTED>"}),
            },
        )
        return anonymized.text

    return redact

# Usage in a chain
redact_pii = create_pii_redactor()
chain = RunnableLambda(lambda x: {**x, "query": redact_pii(x["query"])}) | rag_chain
```

### Custom Entity Recognizers

```python
from presidio_analyzer import PatternRecognizer, Pattern

# Custom recognizer for internal employee IDs
employee_id_recognizer = PatternRecognizer(
    supported_entity="EMPLOYEE_ID",
    name="Employee ID Recognizer",
    patterns=[
        Pattern(name="emp_id", regex=r"EMP-\d{6}", score=0.9),
    ],
)

# Custom recognizer for internal project codes
project_code_recognizer = PatternRecognizer(
    supported_entity="PROJECT_CODE",
    name="Project Code Recognizer",
    patterns=[
        Pattern(name="project", regex=r"PRJ-[A-Z]{3}-\d{4}", score=0.9),
    ],
)

# Register with the analyzer
registry = RecognizerRegistry()
registry.load_predefined_recognizers()
registry.add_recognizer(employee_id_recognizer)
registry.add_recognizer(project_code_recognizer)

analyzer = AnalyzerEngine(registry=registry)
```

---

## RAG Multi-Tenancy

### Document Metadata Tagging

Always tag documents with ownership and access control metadata at ingestion:

```python
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter

def ingest_document(
    content: str,
    source: str,
    org_id: str,
    access_level: str = "internal",
    department: str | None = None,
) -> list[Document]:
    splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
    chunks = splitter.split_text(content)

    return [
        Document(
            page_content=chunk,
            metadata={
                "source": source,
                "org_id": org_id,
                "access_level": access_level,
                "department": department,
                "ingested_at": datetime.utcnow().isoformat(),
            },
        )
        for chunk in chunks
    ]
```

### Query-Time Access Enforcement

```python
class SecureRetriever:
    """Retriever that enforces access control based on user context."""

    def __init__(self, vectorstore, default_k: int = 5):
        self._vectorstore = vectorstore
        self._default_k = default_k

    def get_retriever(self, user) -> VectorStoreRetriever:
        filter_dict = {"org_id": user.org_id}

        # Apply access level filtering
        accessible_levels = self._get_accessible_levels(user)
        filter_dict["access_level"] = {"$in": accessible_levels}

        # Apply department filtering if user is department-scoped
        if user.department and not user.has_role("executive"):
            filter_dict["department"] = {"$in": [user.department, None]}

        return self._vectorstore.as_retriever(
            search_kwargs={"k": self._default_k, "filter": filter_dict}
        )

    def _get_accessible_levels(self, user) -> list[str]:
        levels = ["public"]
        if user.is_authenticated:
            levels.append("internal")
        if user.has_role("manager") or user.has_role("executive"):
            levels.append("confidential")
        if user.has_role("executive"):
            levels.append("restricted")
        return levels
```

---

## Vector Store Access Control Patterns

### Pinecone — Namespace Isolation

```python
from langchain_pinecone import PineconeVectorStore

# Each tenant gets its own namespace
def get_vectorstore(org_id: str) -> PineconeVectorStore:
    return PineconeVectorStore(
        index_name="shared-index",
        embedding=embedding_model,
        namespace=f"org_{org_id}",
    )
```

### Qdrant — Collection-Level Isolation

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient

client = QdrantClient(url="http://localhost:6333", api_key=QDRANT_KEY)

# Create isolated collection per tenant
def create_tenant_collection(org_id: str):
    client.create_collection(
        collection_name=f"docs_{org_id}",
        vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
    )

def get_vectorstore(org_id: str) -> QdrantVectorStore:
    return QdrantVectorStore(
        client=client,
        collection_name=f"docs_{org_id}",
        embedding=embedding_model,
    )
```

### Chroma — Metadata Filtering

```python
from langchain_chroma import Chroma

vectorstore = Chroma(
    collection_name="shared_docs",
    embedding_function=embedding_model,
    persist_directory="./chroma_db",
)

# Filter at query time
retriever = vectorstore.as_retriever(
    search_kwargs={
        "k": 5,
        "filter": {"org_id": user.org_id},
    }
)
```

---

## Audit Logging

### Conversation Audit Logger

```python
import json
import logging
from datetime import datetime, timezone
from langchain_core.callbacks import BaseCallbackHandler

audit_logger = logging.getLogger("llm_audit")
audit_logger.setLevel(logging.INFO)

class AuditCallback(BaseCallbackHandler):
    def __init__(self, user_id: str, session_id: str):
        self.user_id = user_id
        self.session_id = session_id

    def on_chain_start(self, serialized, inputs, **kwargs):
        audit_logger.info(json.dumps({
            "event": "chain_start",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "user_id": self.user_id,
            "session_id": self.session_id,
            "chain": serialized.get("name", "unknown"),
            "input_length": len(str(inputs)),
        }))

    def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {}) if response.llm_output else {}
        audit_logger.info(json.dumps({
            "event": "llm_completion",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "user_id": self.user_id,
            "session_id": self.session_id,
            "tokens": usage,
        }))

    def on_tool_start(self, serialized, input_str, **kwargs):
        audit_logger.info(json.dumps({
            "event": "tool_invocation",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "user_id": self.user_id,
            "session_id": self.session_id,
            "tool": serialized.get("name", "unknown"),
        }))
```

### Usage

```python
callback = AuditCallback(user_id="u_123", session_id="sess_abc")
result = await chain.ainvoke(
    {"user_input": query},
    config={"callbacks": [callback]},
)
```

---

## Data Retention and Deletion

### Conversation History TTL

```python
from langchain_community.chat_message_histories import RedisChatMessageHistory

def get_message_history(session_id: str) -> RedisChatMessageHistory:
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379/0",
        ttl=7 * 24 * 60 * 60,  # Auto-delete after 7 days
    )
```

### User Data Deletion (GDPR Right to Erasure)

```python
async def delete_user_data(user_id: str):
    """Delete all LLM-related data for a user."""

    # 1. Delete conversation history
    sessions = await get_user_sessions(user_id)
    for session_id in sessions:
        history = get_message_history(session_id)
        history.clear()

    # 2. Delete user documents from vector store
    vectorstore.delete(filter={"uploaded_by": user_id})

    # 3. Delete from LangSmith traces (if applicable)
    # Use LangSmith API to delete runs by metadata

    # 4. Log the deletion for compliance
    audit_logger.info(json.dumps({
        "event": "user_data_deletion",
        "user_id": user_id,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }))
```

---

## Compliance Considerations

### GDPR

- **Data minimization**: Only include necessary context in prompts. Never send full user records to the LLM when only a name is needed.
- **Right to erasure**: Implement data deletion workflows covering vector stores, conversation history, and trace logs.
- **Purpose limitation**: Log which prompts use personal data and why.
- **Data processing agreements**: Ensure your LLM provider (OpenAI, Anthropic) has appropriate DPAs in place.

### HIPAA

- **PHI handling**: Never send Protected Health Information to LLM APIs without a BAA (Business Associate Agreement) with the provider.
- **Redaction**: Use Presidio to redact PHI before sending to external LLMs.
- **Audit trail**: Log all LLM interactions that may involve health data.
- **On-premise models**: Consider self-hosted models for PHI processing.

### Cost and Rate Limiting

```python
from langchain_core.callbacks import BaseCallbackHandler

class UsageTracker(BaseCallbackHandler):
    """Track per-user token usage for billing and rate limiting."""

    def __init__(self, user_id: str, redis_client):
        self.user_id = user_id
        self.redis = redis_client

    async def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {}) if response.llm_output else {}
        total_tokens = usage.get("total_tokens", 0)

        key = f"usage:{self.user_id}:{datetime.utcnow().strftime('%Y-%m-%d')}"
        await self.redis.incrby(key, total_tokens)
        await self.redis.expire(key, 30 * 24 * 60 * 60)  # 30 day retention

    async def check_limit(self, daily_limit: int = 100_000) -> bool:
        key = f"usage:{self.user_id}:{datetime.utcnow().strftime('%Y-%m-%d')}"
        current = int(await self.redis.get(key) or 0)
        return current < daily_limit
```
