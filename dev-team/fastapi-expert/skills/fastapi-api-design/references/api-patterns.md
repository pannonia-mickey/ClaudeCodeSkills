# Advanced API Patterns for FastAPI

This reference provides implementation patterns for HATEOAS, cursor-based pagination, bulk operations, file uploads, streaming responses, and WebSocket design in FastAPI applications.

---

## HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS enriches API responses with links that guide clients to available actions, reducing the need for clients to construct URLs.

### Link Model

```python
from pydantic import BaseModel

class Link(BaseModel):
    href: str
    rel: str
    method: str = "GET"

class HATEOASMixin(BaseModel):
    links: list[Link] = []

class ProductResponse(HATEOASMixin):
    id: int
    name: str
    price: float
    category_id: int
```

### Generating Links

```python
from fastapi import Request

def build_product_links(request: Request, product_id: int) -> list[Link]:
    base = str(request.base_url).rstrip("/")
    return [
        Link(href=f"{base}/api/v1/products/{product_id}", rel="self"),
        Link(href=f"{base}/api/v1/products/{product_id}", rel="update", method="PATCH"),
        Link(href=f"{base}/api/v1/products/{product_id}", rel="delete", method="DELETE"),
        Link(
            href=f"{base}/api/v1/products/{product_id}/reviews",
            rel="reviews",
        ),
        Link(
            href=f"{base}/api/v1/categories/{product_id}",
            rel="category",
        ),
    ]

@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: int,
    request: Request,
    service: ProductService = Depends(),
):
    product = await service.get_or_404(product_id)
    product.links = build_product_links(request, product_id)
    return product
```

### Collection Links with Pagination

```python
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    links: list[Link]

def build_pagination_links(
    request: Request, page: int, page_size: int, total: int
) -> list[Link]:
    base = str(request.url).split("?")[0]
    links = [
        Link(href=f"{base}?page={page}&page_size={page_size}", rel="self"),
        Link(href=f"{base}?page=1&page_size={page_size}", rel="first"),
    ]
    total_pages = (total + page_size - 1) // page_size
    if page < total_pages:
        links.append(
            Link(href=f"{base}?page={page + 1}&page_size={page_size}", rel="next")
        )
    if page > 1:
        links.append(
            Link(href=f"{base}?page={page - 1}&page_size={page_size}", rel="prev")
        )
    links.append(
        Link(href=f"{base}?page={total_pages}&page_size={page_size}", rel="last")
    )
    return links
```

---

## Cursor-Based Pagination

Cursor pagination uses an opaque token to mark the position in the result set, avoiding the performance cliff of large offsets.

### Encoding and Decoding Cursors

```python
import base64
import json
from datetime import datetime

def encode_cursor(values: dict) -> str:
    """Encode pagination values into an opaque cursor string."""
    serializable = {}
    for key, value in values.items():
        if isinstance(value, datetime):
            serializable[key] = value.isoformat()
        else:
            serializable[key] = value
    return base64.urlsafe_b64encode(json.dumps(serializable).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    """Decode an opaque cursor string back into pagination values."""
    try:
        return json.loads(base64.urlsafe_b64decode(cursor.encode()))
    except Exception:
        raise HTTPException(400, "Invalid cursor")
```

### Cursor Pagination Repository Method

```python
from sqlalchemy import select, and_, or_

class EventRepository(BaseRepository[Event]):
    model = Event

    async def list_with_cursor(
        self,
        cursor: str | None = None,
        limit: int = 20,
        direction: str = "next",
    ) -> tuple[list[Event], str | None, str | None]:
        stmt = select(Event).order_by(Event.created_at.desc(), Event.id.desc())

        if cursor:
            values = decode_cursor(cursor)
            cursor_dt = datetime.fromisoformat(values["created_at"])
            cursor_id = values["id"]

            if direction == "next":
                stmt = stmt.where(
                    or_(
                        Event.created_at < cursor_dt,
                        and_(Event.created_at == cursor_dt, Event.id < cursor_id),
                    )
                )
            else:
                stmt = stmt.where(
                    or_(
                        Event.created_at > cursor_dt,
                        and_(Event.created_at == cursor_dt, Event.id > cursor_id),
                    )
                )

        # Fetch one extra to determine if there are more results
        stmt = stmt.limit(limit + 1)
        result = await self.session.execute(stmt)
        items = list(result.scalars().all())

        has_more = len(items) > limit
        if has_more:
            items = items[:limit]

        next_cursor = None
        if has_more and items:
            last = items[-1]
            next_cursor = encode_cursor(
                {"created_at": last.created_at, "id": last.id}
            )

        prev_cursor = None
        if cursor and items:
            first = items[0]
            prev_cursor = encode_cursor(
                {"created_at": first.created_at, "id": first.id}
            )

        return items, next_cursor, prev_cursor
```

### Endpoint Implementation

```python
class CursorPage(BaseModel, Generic[T]):
    items: list[T]
    next_cursor: str | None
    previous_cursor: str | None
    has_more: bool

@router.get("/events", response_model=CursorPage[EventResponse])
async def list_events(
    cursor: str | None = Query(None, description="Opaque pagination cursor"),
    limit: int = Query(20, ge=1, le=100, description="Number of items to return"),
    repo: EventRepository = Depends(),
):
    items, next_cursor, prev_cursor = await repo.list_with_cursor(
        cursor=cursor, limit=limit
    )
    return CursorPage(
        items=items,
        next_cursor=next_cursor,
        previous_cursor=prev_cursor,
        has_more=next_cursor is not None,
    )
```

---

## Bulk Operations

### Batch Create

```python
class BulkCreateRequest(BaseModel):
    items: list[ProductCreate] = Field(..., min_length=1, max_length=100)

class BulkCreateResponse(BaseModel):
    created: list[ProductResponse]
    errors: list[BulkError]

class BulkError(BaseModel):
    index: int
    detail: str

@router.post("/bulk", response_model=BulkCreateResponse, status_code=207)
async def bulk_create_products(
    request: BulkCreateRequest,
    service: ProductService = Depends(),
):
    created = []
    errors = []

    for index, item in enumerate(request.items):
        try:
            product = await service.create(item)
            created.append(product)
        except AppError as e:
            errors.append(BulkError(index=index, detail=e.detail))

    return BulkCreateResponse(created=created, errors=errors)
```

### Batch Update

```python
class BulkUpdateItem(BaseModel):
    id: int
    data: ProductUpdate

class BulkUpdateRequest(BaseModel):
    items: list[BulkUpdateItem] = Field(..., min_length=1, max_length=100)

@router.patch("/bulk", response_model=BulkCreateResponse, status_code=207)
async def bulk_update_products(
    request: BulkUpdateRequest,
    service: ProductService = Depends(),
):
    created = []
    errors = []

    for index, item in enumerate(request.items):
        try:
            product = await service.update(item.id, item.data)
            created.append(product)
        except AppError as e:
            errors.append(BulkError(index=index, detail=e.detail))

    return BulkCreateResponse(created=created, errors=errors)
```

### Batch Delete

```python
class BulkDeleteRequest(BaseModel):
    ids: list[int] = Field(..., min_length=1, max_length=100)

class BulkDeleteResponse(BaseModel):
    deleted_count: int
    not_found_ids: list[int]

@router.post("/bulk-delete", response_model=BulkDeleteResponse)
async def bulk_delete_products(
    request: BulkDeleteRequest,
    service: ProductService = Depends(),
):
    deleted_count = 0
    not_found = []

    for product_id in request.ids:
        if await service.delete(product_id):
            deleted_count += 1
        else:
            not_found.append(product_id)

    return BulkDeleteResponse(
        deleted_count=deleted_count, not_found_ids=not_found
    )
```

---

## File Upload Patterns

### Single File Upload with Validation

```python
from fastapi import UploadFile, File, HTTPException

ALLOWED_MIME_TYPES = {"image/jpeg", "image/png", "image/webp"}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

@router.post("/{product_id}/images", status_code=201)
async def upload_product_image(
    product_id: int,
    file: UploadFile = File(..., description="Product image (JPEG, PNG, or WebP)"),
    storage: StorageBackend = Depends(get_storage),
    service: ProductService = Depends(),
):
    # Validate MIME type
    if file.content_type not in ALLOWED_MIME_TYPES:
        raise HTTPException(
            status_code=422,
            detail=f"File type '{file.content_type}' not allowed. Use: {ALLOWED_MIME_TYPES}",
        )

    # Validate file size
    content = await file.read()
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(status_code=413, detail="File exceeds 10 MB limit")

    # Upload to storage
    url = await storage.upload(
        content=content,
        filename=f"products/{product_id}/{file.filename}",
        content_type=file.content_type,
    )

    return {"image_url": url}
```

### Multiple File Upload

```python
@router.post("/{product_id}/gallery", status_code=201)
async def upload_product_gallery(
    product_id: int,
    files: list[UploadFile] = File(..., max_length=10),
    storage: StorageBackend = Depends(get_storage),
):
    if len(files) > 10:
        raise HTTPException(400, "Maximum 10 files per upload")

    uploaded = []
    errors = []

    for index, file in enumerate(files):
        try:
            if file.content_type not in ALLOWED_MIME_TYPES:
                raise ValueError(f"Invalid file type: {file.content_type}")
            content = await file.read()
            if len(content) > MAX_FILE_SIZE:
                raise ValueError("File too large")

            url = await storage.upload(
                content=content,
                filename=f"products/{product_id}/gallery/{file.filename}",
                content_type=file.content_type,
            )
            uploaded.append({"filename": file.filename, "url": url})
        except ValueError as e:
            errors.append({"index": index, "filename": file.filename, "error": str(e)})

    return {"uploaded": uploaded, "errors": errors}
```

---

## Streaming Responses

### Server-Sent Events (SSE)

```python
from fastapi.responses import StreamingResponse
import asyncio
import json

@router.get("/events/stream")
async def stream_events(
    user: User = Depends(get_current_user),
    event_bus: EventBus = Depends(get_event_bus),
):
    async def event_generator():
        queue = await event_bus.subscribe(user.id)
        try:
            while True:
                event = await asyncio.wait_for(queue.get(), timeout=30.0)
                data = json.dumps(event)
                yield f"data: {data}\n\n"
        except asyncio.TimeoutError:
            yield f"data: {json.dumps({'type': 'keepalive'})}\n\n"
        except asyncio.CancelledError:
            return

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

### Large JSON Streaming

```python
import json
from typing import AsyncIterator

async def stream_large_collection(
    repo: ProductRepository, batch_size: int = 500
) -> AsyncIterator[bytes]:
    """Stream a large JSON array without loading everything into memory."""
    yield b'{"items": ['

    first = True
    offset = 0
    while True:
        items = await repo.get_many(skip=offset, limit=batch_size)
        if not items:
            break

        for item in items:
            if not first:
                yield b","
            first = False
            yield json.dumps(item.model_dump(), default=str).encode()

        offset += batch_size

    total = await repo.count()
    yield f'], "total": {total}}}'.encode()

@router.get("/products/export")
async def export_all_products(repo: ProductRepository = Depends()):
    return StreamingResponse(
        stream_large_collection(repo),
        media_type="application/json",
    )
```

---

## WebSocket Patterns

### Typed Message Protocol

```python
from pydantic import BaseModel
from typing import Literal

class WSMessage(BaseModel):
    type: str

class ChatMessage(WSMessage):
    type: Literal["chat"] = "chat"
    text: str
    channel: str

class PresenceMessage(WSMessage):
    type: Literal["presence"] = "presence"
    status: Literal["online", "offline", "typing"]
    user_id: str

class ErrorMessage(WSMessage):
    type: Literal["error"] = "error"
    code: str
    detail: str
```

### Room-Based WebSocket Manager

```python
from collections import defaultdict

class RoomManager:
    def __init__(self):
        self.rooms: dict[str, dict[str, WebSocket]] = defaultdict(dict)

    async def join(self, room: str, user_id: str, ws: WebSocket):
        await ws.accept()
        self.rooms[room][user_id] = ws
        await self.broadcast(room, PresenceMessage(status="online", user_id=user_id))

    async def leave(self, room: str, user_id: str):
        self.rooms[room].pop(user_id, None)
        if not self.rooms[room]:
            del self.rooms[room]
        else:
            await self.broadcast(room, PresenceMessage(status="offline", user_id=user_id))

    async def broadcast(self, room: str, message: WSMessage, exclude: str | None = None):
        for uid, ws in self.rooms.get(room, {}).items():
            if uid != exclude:
                try:
                    await ws.send_json(message.model_dump())
                except Exception:
                    await self.leave(room, uid)

    async def send_to_user(self, room: str, user_id: str, message: WSMessage):
        ws = self.rooms.get(room, {}).get(user_id)
        if ws:
            await ws.send_json(message.model_dump())

room_manager = RoomManager()

@app.websocket("/ws/{room}/{user_id}")
async def room_websocket(
    websocket: WebSocket,
    room: str,
    user_id: str,
):
    await room_manager.join(room, user_id, websocket)
    try:
        while True:
            raw = await websocket.receive_json()
            message = ChatMessage(**raw, channel=room)
            await room_manager.broadcast(room, message, exclude=user_id)
    except WebSocketDisconnect:
        await room_manager.leave(room, user_id)
```

### WebSocket with Authentication

```python
from fastapi import WebSocket, WebSocketException, status

async def authenticate_websocket(websocket: WebSocket) -> User:
    """Extract and validate auth token from WebSocket connection."""
    token = websocket.query_params.get("token")
    if not token:
        # Check subprotocol headers as alternative
        token = websocket.headers.get("sec-websocket-protocol")

    if not token:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)

    try:
        payload = decode_jwt(token)
        user = await get_user_by_id(payload["sub"])
        if not user:
            raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
        return user
    except JWTError:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)

@app.websocket("/ws/secure")
async def secure_websocket(websocket: WebSocket):
    user = await authenticate_websocket(websocket)
    await websocket.accept()
    await websocket.send_json({"type": "connected", "user_id": str(user.id)})
    try:
        while True:
            data = await websocket.receive_json()
            # Process authenticated messages
            await websocket.send_json({"type": "ack", "id": data.get("id")})
    except WebSocketDisconnect:
        pass
```

### Heartbeat and Reconnection

```python
import asyncio

@app.websocket("/ws/{client_id}")
async def websocket_with_heartbeat(websocket: WebSocket, client_id: str):
    await websocket.accept()

    async def send_heartbeat():
        while True:
            try:
                await asyncio.sleep(30)
                await websocket.send_json({"type": "ping"})
            except Exception:
                break

    heartbeat_task = asyncio.create_task(send_heartbeat())

    try:
        while True:
            data = await websocket.receive_json()
            if data.get("type") == "pong":
                continue  # Heartbeat response
            # Process regular messages
            await websocket.send_json({"type": "ack"})
    except WebSocketDisconnect:
        heartbeat_task.cancel()
```
