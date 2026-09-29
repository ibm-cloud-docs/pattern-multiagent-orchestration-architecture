---

copyright:
  years: 2026
lastupdated: "2026-09-29"

keywords: multi-agent systems, watsonx orchestrate, agent orchestration, mcp, a2a, watsonx governance, agentic ai, enterprise ai, reference architecture

subcollection: pattern-multiagent-orchestration-architecture
content-type: white-paper

---

{{site.data.keyword.attribute-definition-list}}

# Multi-agent orchestration reference architecture
{: #white-paper}

This paper outlines design patterns, implementation approaches, and governance controls for building enterprise-grade multi-agent systems on IBM Cloud. It explains how watsonx Orchestrate, open protocols such as MCP and A2A, and watsonx.governance can be combined to support interoperable, auditable, and production-ready agentic systems. The guidance is aimed at organizations that need strong governance, hybrid deployment flexibility, and operational control.
{: shortdesc}

## Pattern overview
{: #pattern-overview}

This reference architecture defines the design, deployment, and operational patterns for building production-grade multi-agent systems on IBM Cloud using watsonx Orchestrate as the orchestration plane, MCP for tool and data integration, and A2A for agent-to-agent collaboration. It covers supervisor/sub-agent hierarchies, persistent agent memory, MCP/A2A topology, governance wiring, and observability.

## Executive summary
{: #executive-summary}

Enterprise AI is moving from single-agent use cases to coordinated systems of specialized agents. This paper focuses on the design patterns, implementation approaches, and governance controls needed to build production-grade multi-agent systems on IBM Cloud. It explains when to use single-agent versus multi-agent approaches, compares common multi-agent orchestration patterns, and highlights the trade-offs between determinism, latency, token cost, and operational risk.

The paper positions watsonx Orchestrate as the multi-agent control plane, with MCP for agent-to-tool connectivity and A2A for agent-to-agent interoperability. It also reviews implementation options across open frameworks and protocols, including LangGraph, Crew AI, ACP, Google ADK, and the Microsoft Agent Framework, showing how these can participate in a broader enterprise orchestration model rather than replace the control plane.

A core theme of the paper is that enterprise success depends not only on orchestration, but on governance. The document therefore emphasizes technical safeguards, human-in-the-loop checkpoints, auditability, monitoring, confidentiality controls, and deployment governance. IBM's differentiated value is reflected in the strength of its governance and hybrid deployment story — particularly watsonx.governance, OpenPages, IBM Sovereign Core, and Red Hat OpenShift — for organizations that need traceability, policy enforcement, and regulatory readiness.

## What is a single-agent design pattern
{: #single-agent-design}

The purpose of a single agent is to perform an autonomous task within a given context and boundary. The agent operates within a defined scope: it receives a goal, breaks it into steps, and executes them independently without requiring human input at each step. Its behavior is governed by a system prompt that sets the role, constraints, and boundaries of what the agent is allowed to do. Within a session, the agent maintains memory of prior steps, allowing it to chain actions and make contextual decisions until the goal is complete.

A single agent is mainly used for querying and performing multiple steps to accomplish a task.

The single-agent design pattern uses the following components:

Prompts
:   The system prompt defines the agent's role, tone, task boundaries, and constraints. It is the primary mechanism for controlling agent behavior.

Knowledge base
:   An external data source (vector database, document store, or structured data) that the agent queries to retrieve relevant context that it was not trained on.

Behavior
:   The internal reasoning loop that determines how the agent interprets inputs, decides what action to take next, and when to stop.

Tools
:   External capabilities that the agent can invoke at runtime, such as APIs, web search, code execution, or database queries. Tool connectivity is standardized through the Model Context Protocol (MCP), an open protocol that defines how agents discover, connect to, and call external tools and data sources in a consistent way across frameworks.

### System design
{: #single-agent-system-design}

A system design takes you through all aspects of a single-agent solution.

![Single-agent design pattern](image/single-agent.png "Diagram showing single-agent components including system prompt, reasoning loop, memory store, and tool registry"){: caption="Single-agent design pattern" caption-side="bottom"}

In high-level system context with tools and services, a single-agent architecture is composed of four core layers:

Orchestration layer
:   Receives the user goal, manages the agent loop, and coordinates between components.

Tool registry
:   A catalog of available tools that the agent can invoke, along with their schemas and access controls. In MCP-based architectures, tools are exposed as MCP servers — lightweight services that advertise their capabilities to the agent and handle execution. The agent connects to these servers through the MCP client built into the orchestration layer.

Memory store
:   Maintains short-term (in-session) context and optionally long-term memory across sessions using a vector or key-value store.

Output handler
:   Validates, formats, and delivers the agent's final response to the consuming application or user.

Single agents are categorized into three main patterns depending on task complexity, flexibility, transparency, and requirements:

Default
:   Ideal for quick tasks, dynamic and prompt-driven. The agent responds directly based on its system prompt and input context without an explicit reasoning chain. Best suited for well-defined, low-ambiguity tasks such as summarization, classification, or single-step data retrieval.

ReAct
:   Ideal for exploratory tasks like research, adaptive and highly transparent. The agent alternates between reasoning and acting: it thinks through the problem, takes an action (such as a search or tool call), observes the result, then reasons again. This loop continues until a satisfactory answer is reached, making it effective for tasks that require gathering and synthesizing information from multiple sources.

Plan-Act
:   Ideal for multi-step workflows, structured and highly transparent. The agent first generates a full plan of steps before executing any action. This separation of planning and execution makes it predictable and auditable, and suited for complex workflows where order of operations and error recovery matter.

### Frameworks
{: #single-agent-frameworks}

- watsonx Orchestrate — low code and no code
- LangChain
- CrewAI
- CUDA
- Model Context Protocol (MCP) — open standard for agent-to-tool connectivity, enabling agents to communicate with any MCP-compliant server regardless of the underlying framework.

## What is a multi-agent design pattern
{: #multi-agent-design-pattern}

In multi-agent patterns, specialized agents are combined to solve complex problems through either deterministic workflows between agents or dynamic, coordinated, and orchestrated workflows. Unlike a single agent, multi-agent systems distribute work across agents with distinct roles, tools, and knowledge boundaries — enabling parallelism, specialization, and fault isolation, at the cost of coordination overhead and compounding error probability.

The key design decision is whether agents follow a fixed topology (deterministic — preferred for enterprise, auditable workloads) or dynamically negotiate tasks based on context (orchestrated / LLM-driven — higher flexibility, lower predictability).

### Reliability math: why topology choice matters
{: #reliability-math}

Every additional agent-to-agent hop introduces an independent probability of failure (misrouted task, dropped context, hallucinated intermediate output). For n sequential or hierarchical hops each with independent accuracy p, the compounded end-to-end reliability approximates pⁿ. A 3-step chain at 90% per-step accuracy yields 0.9³ ≈ 73% — not 90%. This is the primary argument for: (a) keeping topologies as shallow as the task allows, (b) placing evaluator/critic checkpoints after high-risk steps rather than only at the end, and (c) reserving open-ended orchestration (such as Magentic, Group Chat) for problems that genuinely cannot be decomposed up front.

### Pattern comparison matrix
{: #pattern-comparison-matrix}

| Pattern | Determinism | Latency profile | Token cost | Primary failure mode |
| - | - | - | - | - |
| Coordinator / Dispatcher | High | Low (1 routing hop) | Low | Misrouting to wrong specialist |
| Hierarchical | High | Medium (multi-tier) | Medium–High | Error compounding across tiers |
| Sequential Pipeline | Very High | Medium (sum of steps) | Medium | Hard stop if one stage fails |
| Concurrent (Fan-Out/Gather) | High | Low (parallel wall-clock) | High (N× calls) | Conflicting outputs at merge |
| Loop (Evaluator-Optimizer) | Medium | High (N iterations) | High (N× calls) | Non-convergence / infinite loop |
| Group Chat / Collaborative Synthesis | Low–Medium | Medium–High | High | Groupthink / unresolved conflict |
| Handoff (Peer-to-Peer) | Medium | Variable | Medium | Dropped context on transfer |
| Magentic (Dynamic Manager) | Low | Unbounded / hard to predict | Very High | Runaway planning loops, cost overrun |
| Custom Logic / Conditional Routing | High | Variable (rule-driven) | Low–Medium | Rule drift / untested branches |
| Human-in-the-Loop Gate | Very High (at gate) | Adds wait time | Low (gate itself) | Approval bottleneck / alert fatigue |
{: caption="Multi-agent pattern comparison" caption-side="bottom"}

---

## Multi-agent deployment pattern descriptions
{: #multi-agent-deployment-patterns}

### 1. Coordinator or Dispatcher pattern
{: #coordinator-dispatcher-pattern}

A central agent receives the goal, decomposes it into sub-tasks, and delegates each to a specialized agent. It aggregates results and resolves conflicts before returning a final output. This is the multi-agent analogue of an API gateway: one entry point, many backend specialists.

![Coordinator or Dispatcher pattern](image/coordinator.png "Diagram showing coordinator agent decomposing user goal and delegating sub-tasks to specialist agents"){: caption="Coordinator or Dispatcher pattern" caption-side="bottom"}

#### Engineering considerations
{: #coordinator-engineering}

Latency
:   Single routing hop — low time-to-first-token impact if the coordinator uses a small or fast model.

Cost
:   Cheapest multi-agent topology; only the coordinator classification call plus one specialist call are billed per request.

Failure mode
:   Misclassification. Mitigate with a confidence threshold on the routing decision and a fallback branch to clarify with user rather than a forced guess.

---

### 2. Hierarchical orchestration pattern
{: #hierarchical-orchestration-pattern}

Agents are organized in layers where higher-level agents plan and lower-level agents execute. Enables complex goal decomposition across multiple tiers of specialization. A central agent routes tasks to domain experts, who might themselves further decompose work to sub-specialists.

![Hierarchical orchestration pattern](image/Hierarchical.png "Diagram showing multi-tier agent hierarchy with supervisor agent delegating to domain agents and sub-agents"){: caption="Hierarchical orchestration pattern" caption-side="bottom"}

#### Engineering considerations
{: #hierarchical-engineering}

Best for
:   Complex, multi-domain problems requiring both strategic oversight and tactical execution.

Cost and latency
:   Moderate token efficiency — some redundancy exists between tiers as context is re-summarized upward; latency is the sum of the slowest branch per tier.

Compounding error
:   Apply the pⁿ reliability model per tier, not per agent — a 3-tier hierarchy with 92% per-tier fidelity degrades to ~78% end-to-end.

HITL gate
:   Place approval checkpoints at Tier 1 (final aggregation) for irreversible actions; Tier 2 and Tier 3 remain autonomous for read-only analysis.

---

### 3. Sequential pipeline pattern
{: #sequential-pipeline-pattern}

Agents operate in a fixed pipeline where the output of one agent becomes the input of the next. Best for tasks with clear, ordered dependencies. This is the most deterministic and lowest-latency-variance multi-agent pattern because a predefined workflow agent — not an LLM — governs the transition between steps.

![Sequential pipeline pattern](image/Sequential.png "Diagram showing sequential agent pipeline where each agent processes output from previous step"){: caption="Sequential pipeline pattern" caption-side="bottom"}

#### Engineering considerations
{: #sequential-engineering}

Advantage
:   Reduced latency and operational cost relative to LLM-orchestrated routing, since no model call is needed to decide what runs next.

Trade-off
:   The rigid, predefined structure makes it difficult to adapt to dynamic conditions or skip unnecessary steps — an unneeded slow step still executes, which can inflate cumulative latency.

Failure mode
:   A hard stop. If the reviewer agent crashes, the editor never runs — pipelines need per-stage retries and dead-letter handling, not silent skips.

---

### 4. Concurrent (Parallel fan-out / gather) pattern
{: #concurrent-pattern}

Multiple agents work on independent sub-tasks simultaneously and results are merged. Reduces latency for tasks that can be parallelized, such as a primary agent spawning parallel reviewers to independently check an infrastructure deployment request for security, performance, and cost before a gather step consolidates findings.

![Concurrent (Parallel fan-out / gather) pattern](image/Concurrent.png "Diagram showing parallel execution across multiple agents with a subsequent gather step"){: caption="Concurrent (Parallel fan-out / gather) pattern" caption-side="bottom"}

#### Engineering considerations
{: #concurrent-engineering}

Advantage
:   Reduces overall wall-clock latency compared to sequential execution by gathering diverse information from multiple sources at the same time.

Trade-off
:   Running multiple agents in parallel increases immediate resource utilization and token consumption (N× concurrent API calls), raising operational cost and rate-limit pressure; the gather step requires non-trivial logic to reconcile conflicting outputs.

Failure mode
:   Partial results — unlike a sequential pipeline, some branches complete while others fail; design the gather step to degrade gracefully (report partial confidence) rather than block on 100% branch completion.

---

### 5. Loop (Evaluator-optimizer) pattern
{: #loop-evaluator-optimizer-pattern}

A producer agent generates output; a critic/evaluator agent scores it against defined criteria; feedback is passed back and the producer revises. The loop repeats until a quality threshold or maximum iteration count is reached.

![Loop (Evaluator-optimizer) pattern](image/Loop.png "Diagram showing iterative loop between producer agent and critic evaluator agent with threshold check"){: caption="Loop (Evaluator-optimizer) pattern" caption-side="bottom"}

#### Engineering considerations
{: #loop-engineering}

Risk
:   Non-convergence. Always cap `max_iterations` and define an explicit, machine-checkable exit condition (score ≥ threshold OR iteration count reached) rather than relying on the critic's free-text judgment alone.

Cost
:   Highest per-task token cost of any pattern that terminates in bounded time — each iteration re-sends growing context to both agents.

---

### 6. Group chat / Collaborative synthesis pattern
{: #group-chat-collaborative-pattern}

Multiple agents with different perspectives or roles participate in a shared conversation thread, observed and optionally steered by a moderator (which may be a human, an agent, or both), converging on a consensus response.

![Group chat / Collaborative synthesis pattern](image/Collaborative.png "Diagram showing multi-agent collaborative chat with moderator and participant agents in a shared thread"){: caption="Group chat / Collaborative synthesis pattern" caption-side="bottom"}

#### Engineering considerations
{: #group-chat-engineering}

Best for
:   Open-ended analysis, brainstorming, or scenarios requiring multiple expert perspectives where structure emerges dynamically.

Trade-off
:   Lowest determinism of the deterministic-leaning patterns; risk of groupthink or unresolved conflicts if no moderator enforces a termination rule. Token cost is high — every turn is broadcast to all participants.

HITL
:   Group chat is a natural place for a human observer role — read access to the thread without blocking agent turns, escalating only on unresolved disagreement.

---

### 7. Handoff (Peer-to-peer) pattern
{: #handoff-peer-to-peer-pattern}

Control passes explicitly from one specialist agent to another as context requirements change, with only one agent active at a time. Handoff orchestration is the canonical implementation, illustrated by a support scenario: triage agent → technical infrastructure agent → financial resolution agent → customer support, with each agent deciding when to redirect.

![Handoff (Peer-to-peer) pattern](image/handoff.png "Diagram showing stateful task handoff from one specialized agent to another sequentially"){: caption="Handoff (Peer-to-peer) pattern" caption-side="bottom"}

#### Engineering considerations
{: #handoff-engineering}

Advantage
:   Dynamic specialization — one agent can identify the need for a specialist and forward the task, avoiding wasted compute on an ill-suited agent.

Risk
:   Dropped context on transfer. Clear interfaces and an explicit state-transfer payload (not a free-text summary alone) are required so the receiving agent does not have to re-derive prior reasoning.

Failure mode
:   Handoff loops — two agents repeatedly transferring the same task back and forth. Guard with a transfer counter and a forced escalation to a human or default agent after N transfers.

---

### 8. Magentic (Dynamic manager) orchestration pattern
{: #magentic-dynamic-manager-pattern}

A manager agent maintains a live task-and-progress ledger, dynamically assigns and reprioritizes sub-tasks across specialist agents, and loops until the goal is evaluated as complete. Derived from Microsoft Research's MagenticOne system, this is the least deterministic pattern in either vendor's catalog, designed specifically for open-ended problems that do not have a predetermined plan of approach.

![Magentic (Dynamic manager) orchestration pattern](image/Manager.png "Diagram showing dynamic manager maintaining task ledger and allocating tasks to specialist agents iteratively"){: caption="Magentic (Dynamic manager) orchestration pattern" caption-side="bottom"}

Architectural guidance: use with caution.
{: note}

Consistent with a preference for structured, predictable workflows: Magentic orchestration should be reserved for problems that genuinely resist upfront decomposition. It is the most powerful pattern and the easiest to misuse — latency is unbounded and hard to predict, token cost is the highest of any pattern (the manager re-plans on every loop), and the primary production risk is a runaway planning loop that silently exhausts budget. If a Sequential or Hierarchical pipeline can be reasoned about and debugged, it is worth more in production than a Magentic workflow with unpredictable token burn.

#### Mandatory guardrails when Magentic is used
{: #magentic-guardrails}

- Hard ceiling on manager re-planning iterations and total tool-call budget per session.
- HITL authorization gate before any tool call that performs a write, delete, or financial action (see Governance section below).
- Real-time cost/latency dashboards with automatic circuit-breaker cutoff on budget or time overrun.

---

### 9. Custom logic / Conditional routing pattern
{: #custom-logic-conditional-routing-pattern}

Workflow routing is driven by conditional rules or business logic rather than a fixed topology or an LLM-driven decision. Allows dynamic branching based on intermediate results while remaining fully deterministic and auditable — the routing function is ordinary code, not a model call.

![Custom logic / Conditional routing pattern](image/Conditional.png "Diagram showing code-based rule evaluation routing tasks deterministically to distinct agent branches"){: caption="Custom logic / Conditional routing pattern" caption-side="bottom"}

#### Cross-reference
{: #custom-logic-cross-reference}

Google ADK
:   Custom Python routing logic layered over `sub_agents`, bypassing AutoFlow's LLM-driven transfer in favor of deterministic if/else branching on structured intermediate output.

Azure
:   Equivalent to workflow-as-code implementations (such as Durable Functions or Logic Apps fan-out) driving agent invocation based on business rules rather than model-generated routing.

#### Engineering considerations
{: #custom-logic-engineering}

- Highest auditability of any dynamic-branching pattern — every route is testable with standard unit tests, independent of LLM non-determinism.
- Risk is rule drift: as business logic evolves, untested branches accumulate. Treat the routing function itself as production code under the same CI/CD gates as the agents it invokes.

---

### 10. Human-in-the-loop (HITL) gate pattern
{: #human-in-the-loop-pattern}

A human review or approval step is embedded at defined points in the workflow. Critical for high-stakes decisions where full autonomy is not acceptable.

![Human-in-the-loop (HITL) gate pattern](image/hitl.png "Diagram showing human review checkpoint pausing autonomous execution until explicit approval is granted"){: caption="Human-in-the-loop (HITL) gate pattern" caption-side="bottom"}

#### Design decisions for every HITL gate
{: #hitl-design-decisions}

Mandatory versus optional
:   A mandatory gate makes the orchestration synchronous at that step — persist state at the checkpoint so the workflow can resume without replaying prior agent work.

Approval versus feedback
:   Decide whether the human response simply advances the workflow (approval) or loops back to the agent for revision (feedback, converging with the Loop pattern).

Scope
:   Gate specific tool invocations rather than entire agent turns, so the orchestration proceeds autonomously for low-risk actions (read-only queries) and blocks only on sensitive operations (writes, deletes, financial transactions, external communications).

#### Common implementation mistake
{: #hitl-common-mistake}

Creating unnecessary coordination complexity by using a Group Chat or Magentic pattern when a basic Sequential or Concurrent pattern with a single HITL gate would satisfy the requirement. Escalate topology complexity only after confirming the simpler pattern cannot express the required branching.

---

## Multi-agent orchestration implementation
{: #multi-agent-orchestration-implementation}

Multi-agent orchestration implementation involves the following areas:

Local agent-to-agent workflows
:   Can be implemented by using LangGraph, a graph-based orchestration framework where agents are nodes and transitions are edges. Supports stateful, cyclical workflows with fine-grained control over agent sequencing and memory.

External or partner agents
:   Communicate by using A2A, ACP, and MCP standardized protocols that allow agents built on different frameworks or hosted by different organizations to discover, authenticate, and exchange tasks with each other securely.

### Frameworks
{: #multi-agent-frameworks}

Local workflows
:   LangGraph, CrewAI.

Agent Communication Protocol (ACP)
:   Developed by IBM Research. An open protocol for agent-to-agent communication within and across enterprise environments, with support for asynchronous messaging and structured task handoff.

A2A protocol
:   Housed by the Linux Foundation and contributed by Google. Designed for cross-organization agent interoperability, enabling agents to collaborate across company and platform boundaries.

Model Context Protocol (MCP)
:   Provides the tool and resource connectivity layer; MCP servers expose capabilities that any participating agent can discover and invoke, regardless of the orchestration framework in use.

Google Agent Development Kit (ADK)
:   Open source framework providing native Sequential, Parallel, and Loop workflow agents, plus LLM-driven delegation through AutoFlow — the reference implementation for the patterns detailed previously.

Microsoft Agent Framework
:   Open source framework unifying AutoGen orchestration with Semantic Kernel enterprise foundations; provides native Sequential, Concurrent, Handoff, Group Chat, and Magentic orchestration across Python and .NET.

---

## Agentic AI governance
{: #agentic-ai-governance}

### Technical safeguards
{: #technical-safeguards}

Interruptability
:   The ability to turn an agent off as an emergency stop mechanism. A user can activate a graceful shutdown procedure for its agent at any time: both for halting a specific category of actions (revoking access to financial credentials) and for terminating the agent operation generally. In practice, every agent must expose a cancellation interface and respect a kill signal within a defined response window.

Guardrails
:   Agent behavior limits, role-based access control (RBAC), prompt filtering, and output constraints. Guardrails are enforced at two layers: input guardrails that validate and sanitize what enters the agent, and output guardrails that inspect responses before they are returned. Guardrails prevent prompt injection, data leakage, and out-of-scope actions.

Testing and monitoring
:   Hallucination detection and compliance checks (feature drift, model drift). Agents must be tested beyond functional correctness; evaluation must include adversarial prompts, boundary condition testing, and red-teaming for misuse scenarios. In production, continuous monitoring tracks behavioral drift over time.

Human-in-the-loop
:   Oversight for critical decisions. Defines the threshold at which an agent must pause and escalate to a human before proceeding. The threshold must be risk-calibrated: financial transactions above a limit, irreversible actions, or low-confidence decisions all warrant a human checkpoint.

Confidential data
:   Encryption, access control, and anonymization. Agents must enforce data minimization by accessing only the data needed for the task. Personally identifiable information (PII) and sensitive data must be anonymized before entering the agent context, and all data in transit and at rest must be encrypted.

### Process controls and structures
{: #process-controls-structures}

Risk-based actions
:   Define non-autonomous boundaries. Not every action can be fully delegated to an agent. Actions are classified by risk level: read-only operations can be fully autonomous, while write, delete, or financial actions require human approval or a secondary confirmation step.

Auditability
:   Full traceability of agent decisions. Every decision, tool call, and data access made by an agent must be logged with sufficient detail to reconstruct the reasoning chain. Audit logs must capture the input received, reasoning steps, tools invoked, outputs produced, and the identity of the agent and user.

Monitoring
:   Real-time observability across layers. Spans the orchestration layer, individual agents, and tool calls. Key signals include latency, token consumption, error rates, tool call frequency, and output confidence scores. Anomaly detection triggers alerts when behavior deviates from the baseline.

Accountability
:   Clear ownership of models, agents, and orchestration. Each agent in a system must have a designated owner responsible for its behavior, performance, and compliance. Ownership maps to the model version in use, the system prompt, the tools it can access, and the data it touches.

### Deployment governance
{: #deployment-governance}

Models
:   Versioning and performance tracking (hyperparameter optimization). Model versions must be pinned in deployment. Agents must not silently pick up a new model version. Performance baselines are established at release and tracked continuously; regression in accuracy, latency, or safety metrics triggers a rollback.

Agent orchestration
:   Guardrails on autonomy. The orchestration layer enforces scope boundaries at runtime. Agents cannot exceed the permissions defined in their deployment configuration. Tool access, data scope, and action types are all constrained by policy, not just by prompt instruction.

Security
:   Authentication, sandboxing, and rate limits. Each agent operates in an isolated execution environment (sandbox) to prevent lateral movement in case of compromise. All tool calls require authenticated service identities. Rate limits prevent runaway agents from exhausting resources or triggering downstream systems unexpectedly.

Observability
:   Dashboards, alerts, and anomaly detection. A unified observability stack covers all agents in the system, including both infrastructure metrics and agent-level behavioral telemetry. Dashboards surface decision traces, tool call patterns, and confidence distributions alongside standard site reliability engineering (SRE) signals.

Deployment governance must be part of CI/CD automation during the development of agents. This means governance checks — safety tests, guardrail validation, permission audits, and performance regression tests — are executed as automated gates in the pipeline before any agent reaches production. No agent can be deployed without passing its governance baseline.

## Deployment
{: #deploy}

You can accelerate the deployment of the services used within the Multi-agent orchestration reference architecture through the use of deployable architectures.  

### Before you begin
{: #deploy-prereqs}

You need the following items to deploy and configure this reference architecture:

* An [IBM Cloud account](https://cloud.ibm.com/registration).
* Required IAM access policies defined in each of the deployable architecture.

### Provision Architecture
{: #deploy-provision}

1. [Create and configures an instance of IBM watsonx Orchestrate](https://cloud.ibm.com/catalog/7a4d68b4-cf8b-40cd-a3d1-f49aff526eb3/architecture/deploy-arch-ibm-watsonx-orchestrate-91ae7825-6b75-46f4-a305-c675c95aa3bc-global?catalog_query=aHR0cHM6Ly9jbG91ZC5pYm0uY29tL2NhdGFsb2c%2Fc2VhcmNoPXdhdHNvbngjc2VhcmNoX3Jlc3VsdHM%3D){: external}.
2. Learn to [develop agents with no code using watsonx Orchestrate](https://developer.ibm.com/tutorials/develop-agents-no-code-watsonx-orchestrate/){: external}.
3. [Creates and configures an instance of IBM watsonx.governance](https://cloud.ibm.com/catalog/7a4d68b4-cf8b-40cd-a3d1-f49aff526eb3/architecture/deploy-arch-ibm-watsonx-governance-5d7c4272-dc45-49e6-8ef3-7f86af1e30d7-global?catalog_query=aHR0cHM6Ly9jbG91ZC5pYm0uY29tL2NhdGFsb2c%2Fc2VhcmNoPXdhdHNvbngjc2VhcmNoX3Jlc3VsdHM%3D){: external}.

## References
{: #references}

The following sources support the product capabilities, protocol specifications, and positioning described in this reference architecture. Links were verified current as of 2026; vendor pages evolve, so consult the canonical documentation for the latest details.

### IBM watsonx Orchestrate
{: #ibm-watsonx-orchestrate}

- [IBM watsonx Orchestrate — Multi-agent orchestration](https://www.ibm.com/products/watsonx-orchestrate/multi-agent-orchestration){: external} — Supervisor/router/planner model, agent styles (ReAct, Plan-Act, deterministic), and AI Gateway model selection.
- [IBM watsonx Orchestrate — AI Agent Builder](https://www.ibm.com/products/watsonx-orchestrate/ai-agent-builder){: external} — No-code, low-code, and pro-code build paths and AI Gateway provider choice (Granite, OpenAI, Anthropic, Google Gemini, Mistral, Ollama).
- [watsonx Orchestrate Agent Development Kit (ADK) — Developer documentation](https://developer.watson-orchestrate.ibm.com/){: external} — ADK reference, Developer Edition, and protocol support.

### Open protocols (MCP and A2A)
{: #open-protocols-mcp-a2a}

- [Introducing the Model Context Protocol — Anthropic](https://www.anthropic.com/news/model-context-protocol){: external} — Original MCP announcement (November 2024).
- [Model Context Protocol — Specification](https://modelcontextprotocol.io/specification/2025-11-25){: external} — Authoritative MCP protocol requirements and JSON-RPC schema.
- [Linux Foundation launches the Agent2Agent (A2A) Protocol Project](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents){: external} — Vendor-neutral governance established 23 June 2025.
- [Google Cloud donates A2A to the Linux Foundation — Google Developers Blog](https://developers.googleblog.com/en/google-cloud-donates-a2a-to-linux-foundation/){: external} — Founding members and project scope.
- [A2A Protocol — Official site](https://a2a-protocol.org/){: external} and [A2A specification (GitHub)](https://github.com/a2aproject/A2A){: external} — Agent Cards, task lifecycle states, and transport (HTTP + SSE + JSON-RPC 2.0).

### IBM Granite models
{: #ibm-granite-models}

- [IBM Granite 4.0 — Model documentation](https://www.ibm.com/granite/docs/models/granite){: external} — Hybrid Mamba-2/transformer architecture, MoE, and model tiers (H-Small, H-Tiny, H-Micro, Nano).
- [Introducing the IBM Granite 4.1 family of models — IBM Research](https://research.ibm.com/blog/granite-4-1-ai-foundation-models){: external} — Latest Granite family, instruction-following and tool-calling performance, and toggleable reasoning.

### Governance, sovereignty, and observability
{: #governance-sovereignty-observability}

- [IBM watsonx.governance](https://www.ibm.com/products/watsonx-governance){: external} — Factsheets, agentic evaluation metrics, Governance Graph, and Risk Atlas.
- [IBM OpenPages](https://www.ibm.com/products/openpages){: external} — Risk and regulatory-compliance evidence management.
- [IBM Instana Observability](https://www.ibm.com/products/instana){: external} — End-to-end distributed tracing for agent workloads.
- [OpenTelemetry](https://opentelemetry.io/){: external} — Vendor-neutral trace/metric/log instrumentation standard.
