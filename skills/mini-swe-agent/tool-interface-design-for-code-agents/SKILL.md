---
name: tool-interface-design-for-code-agents
description: "Architectural decisions for designing the tool layer of agents that execute code — tool granularity, action parsing compatibility, and termination mechanisms. TRIGGER when: designing tools for a code-execution agent, choosing between one generic tool and many specialized tools, supporting models without native tool-call APIs, implementing task completion detection across execution environments"
---

## Key Decisions

1. A single generic execution tool (e.g., "run command") trades tool selection complexity for command generation complexity. The LLM never faces tool routing errors or invalid tool name hallucinations, because there is only one tool to call. The cost is losing per-tool parameter validation and structured guardrails — you cannot enforce path constraints on a file-edit if "file-edit" is just a shell command.

2. When choosing one generic tool, the LLM's effective discretion lives inside the command string, not in tool selection. This means tool selection accuracy metrics become meaningless — you must instead evaluate command correctness, which requires parsing the generated commands. Multi-tool systems get tool selection metrics for free but pay with a larger error surface: wrong tool name, wrong parameter schema, tool-not-found hallucinations.

3. Support both native tool-call parsing and text-based regex parsing as separate model adapter classes. Models with tool-call API support get structured, validated arguments; models without it get regex extraction from fenced code blocks. The text-based mode should enforce exactly one action per response (the regex cannot reliably distinguish multiple actions), while the tool-call mode can handle multiple actions natively. Without dual parsing, you lock out either older/local models or lose structured validation.

4. Detect task completion via a sentinel string in command output rather than a dedicated termination tool or API endpoint. The sentinel approach (agent echoes a known string, environment detects it and raises a completion signal) works identically across all execution backends — local shell, container, remote VM — without requiring each backend to implement a separate termination API. A dedicated "submit" tool is cleaner for multi-tool systems but couples termination to the tool-call layer, which breaks text-based parsing modes.

## Anti-patterns

- **What**: Defining many narrow tools (file_read, file_write, search, test_run) for a code-execution agent without measuring tool selection accuracy first.
  **Why**: LLMs hallucinate tool names and confuse similar tools (file_read vs. file_view), and each tool's parameter schema is another surface for format errors — errors that consume a turn and cost money to recover from.
  **Symptom**: Agent logs show repeated format errors cycling between wrong tool names and malformed parameters, with 20-30% of turns wasted on tool selection mistakes rather than task progress.

- **What**: Supporting only native tool-call parsing and assuming all target models implement it.
  **Why**: Local models, older API versions, and some providers return plain text without structured tool_calls, causing the parser to find zero actions and enter an error loop.
  **Symptom**: Agent works perfectly with one provider's models but produces infinite "no tool calls found" error loops when switched to another model, with no useful diagnostic pointing to the parsing layer.

## Structural Template

```
// Tool definition layer — one generic tool
TOOL_SCHEMA = {
  name: "execute",
  parameters: { command: string }
}

// Action parsing — two adapters behind a common interface
interface ActionParser {
  parse(model_response) -> list[Action]
  format_observation(action, output) -> Message
}

ToolCallParser implements ActionParser {
  // Extracts from structured tool_calls in API response
  // Validates tool name matches TOOL_SCHEMA
  // Supports multiple actions per response
}

TextRegexParser implements ActionParser {
  // Extracts from fenced code blocks via regex
  // Enforces exactly one action per response
  // Returns action without tool_call_id metadata
}

// Termination detection — in the execution layer, not the tool layer
ExecutionEnvironment.execute(action) -> Output {
  output = run_command(action.command)
  if output starts with SENTINEL_STRING:
    raise TaskCompleted(submission=remaining_output)
  return output
}

// Model adapter selects parser based on capability
ModelAdapter(parser: ActionParser) {
  query(messages) -> Message {
    response = llm_api.complete(messages, tools=[TOOL_SCHEMA] if parser is ToolCallParser else none)
    actions = parser.parse(response)
    return message_with_actions(response, actions)
  }
}
```
