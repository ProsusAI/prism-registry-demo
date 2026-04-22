---
name: agent-knowledge-and-tool-injection
description: "Prevents agents from silently operating without relevant domain knowledge or with a stale tool set, when knowledge and tools are loaded at startup rather than resolved per request. TRIGGER when: injecting domain knowledge or repository context into agent prompts, integrating external tool servers whose available tools may change at runtime, agent silently ignores relevant knowledge because input phrasing does not match triggers, building a system where knowledge sources or tools are added after initial deployment."
---

## Key Decisions

1. **Choose between keyword-trigger knowledge injection and semantic search based on whether your failure mode is false negatives or false positives.** Keyword triggers are deterministic and zero-latency: the knowledge is injected if and only if a trigger word matches. The failure mode is false negatives — the user's phrasing never matches the trigger, so the agent acts without the knowledge and produces a wrong result that is hard to attribute to missing context. Semantic search retrieves on similarity and catches paraphrase, but it introduces retrieval latency, an embedding model dependency, and false positives (injecting loosely related knowledge that inflates context). Choose keyword triggers when the trigger vocabulary is small and controlled; choose semantic retrieval when users' phrasing is unpredictable or when the knowledge base is large.

2. **List available tools from external servers at each session start, not once at process start.** When tools are listed once at process boot and cached, any tool added or removed from the external server is invisible to the agent until a restart. In deployments where external tool servers evolve independently of the agent process, the agent will call tools that no longer exist (producing opaque errors) or will never discover new tools (silently missing capabilities). Refreshing the tool list at session start adds one round-trip per conversation but keeps the agent's capability set synchronized with the server's actual state.

## Anti-patterns

- **What**: Registering domain knowledge files with specific trigger phrases and expecting users to phrase their requests to match.
  **Why**: Users describe their intent in natural language; they do not know which trigger words activate which knowledge modules.
  **Symptom**: Agent produces subtly wrong output for a known domain (e.g., a specific framework or API); the knowledge file exists and contains the correct information, but the user's task description used synonyms that never matched the trigger; diagnosed only when a developer manually checks which knowledge was injected for a failing session.

- **What**: Connecting to external tool servers at agent process startup and never refreshing the tool list during the process lifetime.
  **Why**: External tool servers are independently deployed and may add, rename, or remove tools on their own release cycle.
  **Symptom**: Agent calls a tool that was renamed in the last server deploy; receives an "unknown tool" error; the error message says nothing about version mismatch; the fix looks like a bug in the agent when it is actually a stale capability list.

## Structural Template

```
# Knowledge injection — two strategies, chosen at config time
class KnowledgeInjector:
    def get_relevant(self, user_message, available_knowledge):
        if config.injection_strategy == "keyword":
            # deterministic, zero-latency, fails on paraphrase
            return [k for k in available_knowledge
                    if any(trigger in user_message for trigger in k.triggers)]

        elif config.injection_strategy == "semantic":
            # handles paraphrase, adds latency + embedding dependency
            query_embedding = embed(user_message)
            return retrieve_top_k(query_embedding, available_knowledge, k=config.top_k)

# Tool listing — per session, not per process
class AgentSession:
    async def initialize(self, tool_servers):
        self.tools = []
        for server in tool_servers:
            async with server.connect() as client:
                server_tools = await client.list_tools()  # fresh on each session
            self.tools.extend(server_tools)
        # tools are now current as of session start

    # NOT at process startup:
    # PROCESS_TOOLS = connect_and_list_all_servers()  # stale after first server deploy

# Knowledge file format (triggering metadata separate from content)
knowledge_entry = {
    "triggers": ["keyword_a", "keyword_b"],   # small, controlled vocabulary
    "content": "...",                          # injected verbatim or as summary
    "scope": "repo | global | task"
}
```
