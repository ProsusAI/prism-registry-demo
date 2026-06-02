---
name: file-backed-store-implicit-path
description: "File-backed embedded stores must require an explicit path in config — auto-injected temp paths create disk-backed behavior invisible to callers and lost on container restart. TRIGGER when: configuring a file-backed vector store or embedded database, deploying a service that uses an embedded store to a containerized environment, designing default configuration for a store that can run in-memory or file-backed modes."
---

# File-Backed Store Implicit Path
*Auto-injecting a temp path for a file-backed store hides disk writes from the caller and silently loses data on container restart.*

## Key decisions

1. Require `path` to be set explicitly in config for any file-backed embedded store. Without this, a zero-argument config silently opens a disk-backed store at a temp directory — callers who expect in-memory behavior or a config error instead get invisible disk writes.

2. Log the effective storage path at startup, regardless of how the path was determined. Without this, developers in containerized environments discover data loss only after their first container restart; there is no trace of where the store was writing.

3. Separate in-memory and file-backed modes at the config level rather than inferring from path absence. Without this, callers cannot express "I want ephemeral" and "I want durable" as distinct, explicit intentions — omitting the path always means one or the other by convention.

## Anti-patterns

- **What**: Setting `path = f"/tmp/{provider}"` in config validation when the caller did not specify a path
- **Why**: `/tmp` is ephemeral in most container runtimes (tmpfs or overlayfs); callers who omitted `path` expected either in-memory behavior or a validation error, not silent writes to a path that survives the process but not the container
- **Symptom**: After container restart, all stored records are gone; the system reports success on writes and empty results on reads; the root cause (writing to a temp path) requires a config audit to find

## Structural template

```python
class EmbeddedStoreConfig(BaseModel):
    mode: Literal["memory", "file"]    # explicit — no inference from path
    path: Optional[str] = None

    @model_validator(mode="after")
    def validate_path(self):
        if self.mode == "file" and not self.path:
            raise ValueError(
                "path is required for file-backed mode. "
                "Set path='/data/store' for persistence, "
                "or mode='memory' for ephemeral storage."
            )
        if self.mode == "memory" and self.path:
            raise ValueError("path must not be set for memory mode.")
        return self

# At startup: always log the effective storage location
def open_store(config: EmbeddedStoreConfig):
    effective_path = config.path if config.mode == "file" else ":memory:"
    log.info("store_opened", mode=config.mode, path=effective_path)
    return Store(config)

# Correct usage
file_store   = EmbeddedStoreConfig(mode="file",   path="/data/store")
memory_store = EmbeddedStoreConfig(mode="memory")

# Incorrect — raises at config time, not at first container restart
bad_config = EmbeddedStoreConfig(mode="file")    # ValueError: path required
```
