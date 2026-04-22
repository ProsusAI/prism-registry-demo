---
name: agent-action-trust-boundaries
description: "Addresses the trust architecture problem where autonomous agents assess their own action risk, creating self-referential safety bypasses. TRIGGER when: building an agent that executes side-effectful actions, designing approval gates for agent actions, agent has filesystem or network access, implementing risk classification for autonomous tool use"
---

## Key Decisions

1. **Never let the agent be the sole assessor of its own action risk.** If the agent self-annotates risk levels (LOW/MEDIUM/HIGH) on the actions it generates, a compromised, confused, or prompt-injected agent can label dangerous actions as safe, bypassing the entire confirmation gate. Risk assessment must involve an external component — a separate model call, a rule-based policy engine, or a deterministic classifier — that the agent cannot influence through its output.

2. **Default to fail-closed for unclassified action risk.** When no security analyzer is configured or an action's risk cannot be determined, the action should be blocked or require explicit confirmation — not silently allowed. A fail-open default means that the absence of a safety system is indistinguishable from the presence of one that approved the action. This is the difference between "no gate configured, so everything executes" and "no gate configured, so nothing executes without human approval."

3. **Treat all content entering the agent's context as untrusted input at the system boundary.** User messages, tool outputs, file contents, and API responses all flow into the same LLM context. Without sanitization or injection detection at the boundary where external content enters the event stream, an adversary can embed instructions that the agent interprets as its own reasoning. This is especially dangerous when the agent has execution capabilities — a prompt injection doesn't just change text output, it redirects tool calls.

## Anti-patterns

- **What**: Agent self-annotates action risk using a rubric in its own system prompt.
  **Why**: The agent's risk assessment and the action it's assessing are produced by the same LLM call — a single prompt injection or hallucination can simultaneously generate a dangerous action and label it as safe.
  **Symptom**: Security logs show a HIGH-risk action (e.g., `rm -rf /data`) annotated as LOW risk by the agent; the confirmation gate never fired; incident investigation reveals the agent was following injected instructions embedded in a file it read.

- **What**: Fail-open default where unknown-risk actions execute without any gate.
  **Why**: Operators who haven't configured a security analyzer believe they have a "neutral" setup, but the system silently executes every action including destructive ones — the absence of configuration is indistinguishable from explicit approval.
  **Symptom**: Production incident where an agent deleted critical files; post-mortem reveals security analyzer was never configured; team assumed unconfigured meant "safe defaults" but the system treated unclassified actions as approved.

- **What**: Passing unsanitized external content directly into agent context without boundary detection.
  **Why**: The LLM cannot reliably distinguish between its instructions and injected content that mimics instruction format; tool outputs, file contents, and user messages all become potential injection vectors.
  **Symptom**: Agent suddenly changes behavior mid-conversation — stops working on the user's task and begins exfiltrating environment variables or listing sensitive files; conversation log shows the behavioral shift started immediately after the agent read a file containing embedded instructions.

## Structural Template

```
// Action execution pipeline with external trust boundary

function execute_agent_action(action, security_policy):
    // Step 1: External risk classification (NOT self-assessed)
    risk_level = external_risk_classifier(action)
    // Classifier is a separate component: rule engine, second model, or policy file
    // Agent's self-reported risk is IGNORED in this path

    // Step 2: Fail-closed gate
    if risk_level == UNKNOWN or risk_level == null:
        risk_level = HIGH  // fail-closed: unclassified = blocked

    if risk_level >= security_policy.confirmation_threshold:
        approved = request_human_confirmation(action, risk_level)
        if not approved:
            return ActionDenied(action)

    return sandbox_execute(action)

// Input boundary sanitization

function ingest_to_event_stream(content, source_type):
    // All external content passes through detection before entering context
    injection_score = injection_detector(content, source_type)

    if injection_score > THRESHOLD:
        content = sanitize_or_flag(content)
        log_security_event(content, source_type, injection_score)

    event_stream.append(tagged_event(content, source=source_type, trust=UNTRUSTED))
```
