---
name: embedded-store-client-sharing
description: "File-backed embedded stores (RocksDB, LMDB) allow only one writer per path — multiple lazy-initialized clients on the same path deadlock or crash. TRIGGER when: initializing two logical stores backed by the same embedded database, using an embedded vector store alongside a secondary index on the same storage path, designing lazy initialization for stores that may share a file path."
---

# Embedded Store Client Sharing
*RocksDB-backed embedded stores acquire an exclusive write lock; a second initialization on the same path deadlocks silently.*

## Key decisions

1. When two components need access to the same file-backed embedded store path, share the client instance rather than creating separate instances. Without this, the second initialization attempt waits indefinitely for the exclusive write lock held by the first — the process hangs with no error or timeout.

2. Detect the shared-path case at initialization time and inject the existing client before opening the store. Without this, the deadlock manifests only when both components initialize in the same process — typically production, where both are used, while unit tests mock or use distinct paths.

3. Track initialized clients per path at the factory level, not per instance. Without this, two components initializing the same path in separate code paths cannot discover each other's lock status and coordinate reuse.

## Anti-patterns

- **What**: Instantiating a second store client with the same `path` for a different logical use case (e.g., a secondary index alongside the primary store)
- **Why**: Embedded stores backed by RocksDB acquire an exclusive directory lock; a second process or object acquiring the same lock blocks indefinitely
- **Symptom**: Process hangs at startup with no error message, no timeout, no log output; the hang only appears when both components initialize in the same process; unit tests pass because they use mocked stores or distinct temporary paths

## Structural template

```python
class EmbeddedStoreFactory:
    _open_clients: dict[str, Client] = {}   # path → shared client

    def create(self, config: StoreConfig) -> Store:
        if config.is_file_backed and config.path in self._open_clients:
            # Reuse: inject existing client, skip re-opening the locked path
            config.client = self._open_clients[config.path]
            return Store(config)

        client = Client.open(config.path)   # acquires exclusive write lock
        if config.is_file_backed:
            self._open_clients[config.path] = client
        return Store(config)

# At the call site: secondary store reuses the primary's client
def init_stores(primary_config, secondary_config):
    factory = EmbeddedStoreFactory()
    primary = factory.create(primary_config)
    # If secondary_config.path == primary_config.path, factory reuses the client
    secondary = factory.create(secondary_config)
    return primary, secondary

# Manual injection pattern (no factory)
def init_secondary(primary_store, secondary_config):
    if (
        secondary_config.is_file_backed
        and secondary_config.path == primary_store.config.path
    ):
        secondary_config.client = primary_store.client   # share, don't re-open
    return Store(secondary_config)
```
