---
name: async-client-sync-constructor-io
description: "Avoid synchronous network I/O inside async client constructors — it blocks the event loop during construction and stalls all concurrent requests. TRIGGER when: adding API key validation or network ping to an async client constructor, constructing a client that hits a remote endpoint on init, designing an async SDK wrapper around a synchronous HTTP library."
---

# Async Client Sync Constructor I/O
*Synchronous network calls in an async constructor block the entire event loop — observable only under concurrent production load.*

## Key decisions

1. Never call a synchronous HTTP library (`requests`, `urllib`) from inside an `async` class's `__init__`. Without this, every client construction blocks the entire event loop for the full round-trip latency — under FastAPI or aiohttp, all concurrent requests queue behind the blocking init call.

2. Defer network validation to first use via a lazy async guard, or expose an explicit async factory function (`async def create_client()`). Without this, the error surface only under concurrent production load; unit tests that construct the client outside a shared event loop never reveal the block.

3. If eager validation is required, run synchronous I/O via `asyncio.get_event_loop().run_in_executor(None, sync_validate)` to yield the event loop during the call. Without this, the loop cannot process any other coroutine for the full ping duration.

## Anti-patterns

- **What**: Calling `requests.get("/ping")` inside an `async` class constructor to validate credentials at construction time
- **Why**: Python's `requests` is synchronous; it does not yield the event loop even when called from inside a coroutine
- **Symptom**: Under 10+ concurrent users, request latency spikes correlate with client construction; all in-flight requests queue behind the blocking ping; unit tests pass because they construct the client serially, outside a shared async event loop

## Structural template

```python
class AsyncClient:
    def __init__(self, api_key: str, host: str):
        self.api_key = api_key
        self.host = host
        self._validated = False           # lazy validation sentinel

    async def _ensure_validated(self):
        if self._validated:
            return
        async with httpx.AsyncClient() as http:
            await http.get(
                f"{self.host}/ping",
                headers={"Authorization": f"Bearer {self.api_key}"},
            )
        self._validated = True

    async def add(self, *args, **kwargs):
        await self._ensure_validated()   # fires once, yields event loop
        # ... rest of method

# Alt: async factory (fail-fast style)
async def create_client(api_key: str, host: str) -> AsyncClient:
    client = AsyncClient(api_key, host)
    await client._ensure_validated()     # validation before first use
    return client

# Never: requests.get() inside __init__
# class AsyncClient:
#     def __init__(self, api_key):
#         requests.get(f"{host}/ping")   # blocks event loop
```
