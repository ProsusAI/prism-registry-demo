---
name: memory-client-per-request-construction
description: "Memory or AI clients that perform network validation on construction must be instantiated once and reused — per-request construction adds a round-trip per request and causes concurrent validation storms under load. TRIGGER when: initializing a memory or AI client inside a request handler or tool call, benchmarking a service that constructs clients at call time, designing dependency injection for a service with a remote memory or AI backend."
---

# Memory Client Per-Request Construction
*Clients that validate credentials on construction must be constructed once — per-request instantiation fires N simultaneous validation calls under concurrent load.*

## Key decisions

1. Construct clients that perform network I/O at application startup, not inside request handlers. Without this, every incoming request pays a network round-trip for credential validation before any application logic begins — under concurrent load, this adds a full ping latency to every request's critical path.

2. Validate client health at startup, not inside the request path. Without this, startup health checks pass (no client was constructed during startup), but the first batch of real requests triggers N simultaneous construction and validation calls — which can hit rate limits or overload the validation endpoint.

3. Share a single client instance per process (or per worker), not per coroutine or per call. Without this, 100 concurrent requests each constructing a client generates 100 simultaneous credential pings — even if the client library itself handles the HTTP efficiently.

## Anti-patterns

- **What**: Creating a new `MemoryClient(api_key=...)` inside each request handler, task, or tool call
- **Why**: The client constructor fires a remote validation call synchronously; under concurrent load, N requests in-flight = N validation calls simultaneously
- **Symptom**: Under load testing, p99 latency is 300–500ms higher than expected; the spike correlates with request concurrency, not memory operation latency; profiling shows client construction as the majority of each request's wall time

## Structural template

```python
# Application startup — construct once, validate once
def create_app():
    memory_client = MemoryClient(api_key=os.environ["MEMORY_API_KEY"])
    # Validation fires here — not per-request

    @app.on_event("startup")
    async def startup():
        app.state.memory = memory_client   # shared across all requests

# Request handler — reuse, never reconstruct
@app.post("/memory")
async def store_memory(request: Request, payload: MemoryPayload):
    client = request.app.state.memory    # reuse
    return await client.add(payload.messages, user_id=payload.user_id)

# Dependency injection pattern (FastAPI / similar)
def get_memory_client(request: Request) -> MemoryClient:
    return request.app.state.memory      # no construction here

@app.get("/recall")
async def recall(query: str, client: MemoryClient = Depends(get_memory_client)):
    return await client.search(query)
```
