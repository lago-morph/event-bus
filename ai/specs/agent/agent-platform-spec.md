# Agent Platform — Architecture Specification

**Status:** draft · **Layer:** 2 (agent platform) · **Scope:** provider-neutral · **Builds on:** platform event bus specification

This specification describes the agent platform in terms of roles, contracts, and required capabilities. Like the platform event bus specification, it names no cloud provider. Platform profiles show at least one option per role; they are not design documents.

## 1. Purpose and scope

The agent platform runs AI agents on the platform event bus and lets agents ask people questions. It adds no plumbing of its own:

- agents are handlers,
- questions and answers are events,
- everything is recorded and traced as the event bus specification describes.

Project-specific agent behavior belongs to the project's own specification.

## 2. Document map

| Document | Purpose |
| --- | --- |
| Platform event bus specification | Layer 1: roles, contracts, execution classes, design-time choices |
| Agent platform specification (this document) | Layer 2: agents, questions, and interviews on the bus |
| Agent platform AWS profile | Maps every agent-platform role to at least one AWS option |
| Agent platform Cloudflare profile | Maps every agent-platform role to at least one Cloudflare option |
| Slice 5: Enhancement issue impact analysis | First agent handler; questions in issue comments |
| Slice 6: Remaining components (provisional) | Interview app and the remaining agent-platform components |

## 3. Principles

- **Agents are handlers.** Each agent is an agent handler with a handler definition, a queue, and an execution class, like any other handler.
- **Inference is chosen per handler.** Each agent handler chooses its inference provider.
- **Agents propose; people decide.** Agents never write to protected branches. Code changes arrive as branches and pull or merge requests; comments go through connectors.
- **Same skill for people and agents.** A workflow encoded as a skill behaves the same whether a developer runs it in an interactive coding agent session or an agent runs it headless.

## 4. Roles

| Role | Responsibility | Required capabilities |
| --- | --- | --- |
| Agent handler | A handler whose logic calls a model: analysis, review, question generation, code changes | Runs in the bus execution class its work needs (section 5); access to an inference provider |
| Inference provider | Model calls for agent handlers | A model provider's API or an AI gateway (section 8) |
| Question connectors | Post questions where people already work | Bus connectors for issue tracking and git hosting (integration profiles) |
| Interview app | Lets people who don't work in the systems of record see and answer questions addressed to them | Web app with OIDC sign-in; talks only to the sync gateway |
| Interview-read handler | Returns the caller's pending questions and their context | Request handler; reads the interview store directly |
| Interview-answer handler | Accepts an answer from the interview app and publishes it as an event | Request handler that publishes an event; checks the caller against the addressee list |
| Interview handler | Records questions addressed to interview-app users and consumes their answers | Handler in any bus execution class |
| Interview store | Questions, addressees, answers, status | A database |
| Skills | Encoded workflows used by interactive sessions and agent handlers | Kept in a git repository |
| Control plane MCP server | Tools for MCP clients, including agents | Provided by the event bus control plane |
| Agent session log store | Full records of agent sessions | Object store, linked to traces (section 13) |

## 5. Execution classes

Agent handlers use the bus's execution classes; this layer defines none of its own. Each agent handler's definition sets its class from the work it does:

| Work | Execution class |
| --- | --- |
| Short reasoning: analysis, review, writing a question | Queue-triggered function |
| Multi-step work with independently retried steps | Durable workflow |
| Work that needs a full environment: repository checkout, a headless coding agent, builds | Sandboxed task, started by the task dispatcher |

- One task may span several handlers with different classes. A handler hands work on by publishing an event, for example a queue-triggered function that hands a code change to a sandboxed-task agent handler.
- Interview-read and interview-answer handlers are request handlers; the interview handler and question connectors use any async class.
- **Concurrency** of agent handlers is capped to bound external load and inference spend.

### Agent handlers that change code

An agent handler that changes code runs as a sandboxed task and additionally has:

- a headless coding agent with the same skills and MCP tools developers use, running the whole multi-step sequence in one task (check out, change, build and test, push a working branch, iterate),
- a scoped working-branch push token,
- builds and tests either inside the sandboxed task or requested from the CI/CD system through the bus, chosen per handler.

Completion follows the bus's sandboxed task dispatch: the task publishes `task.completed` or `task.failed`, and the dispatcher acknowledges the message only then.

## 6. Logical connections

| From | To | Via | What flows | Auth |
| --- | --- | --- | --- | --- |
| Bus publishers | Agent handlers | Event router → handler queue | Tasks for the agent | Workload identity |
| Agent handlers | Bus | Event router | Results, questions, task completed or failed | Workload identity |
| Agent handlers | Inference provider | HTTPS | Inference | Inference credential from the secrets store |
| Agent handlers | Systems of record | Direct read-only APIs; git over HTTPS | Context; working-branch pushes | Read-only tokens; a scoped token for working-branch pushes |
| Agent handlers | Control plane MCP server | MCP over HTTPS | The same tools developers use | Agent token |
| Question connectors | Issue tracker, git host | Their APIs | Questions as comments | Each system's credential |
| Interview handler | People | Bus notifications | Notification of a question, with a link to the interview app | Per notification method |
| Issue tracker, git host | Agent handlers | Comment webhook → ingress endpoint → event router → handler queue | Replies to questions | Webhook signature |
| Person (browser) | Interview app | HTTPS | UI | OIDC sign-in |
| Interview app | Interview-read handler | Sync gateway (authorized) | My pending questions | OIDC token; handler filters by the caller's identity |
| Interview app | Interview-answer handler | Sync gateway (authorized) | Answers, published as events | OIDC token; caller identity from the request authorizer, checked against the addressee list |
| Interview handler | Interview store | Database write | Questions and answers | Database credentials |
| Agent handlers | Agent session log store | Object store write | Session logs | Workload identity |

## 7. Agent runtime

- Agent handlers use an agent SDK (for example the OpenAI Agents SDK or the Claude Agent SDK) or, for code changes, a headless coding agent (for example Codex or Claude Code). The choice is made per handler.
- The agent runtime is part of the handler's implementation; the platform does not prescribe one.

## 8. Inference

- Each agent handler calls its inference provider using credentials from the secrets store.
- A provider can be a model provider's API directly, or an AI gateway in front of one or more providers (for example LiteLLM, OpenRouter, or Amazon Bedrock intelligent prompt routing).
- Model inputs contain only the content needed for the task and never credentials.
- Agent tracing is exported through OpenTelemetry to the bus's telemetry backends.

## 9. Questions and channels

Agents ask people questions where those people already work. Each question is addressed to one or more people.

| Situation | Addressee | Channel | Notification | Answer returns via |
| --- | --- | --- | --- | --- |
| Work tracked in a ticket, before code exists | Developer | Issue tracker comment | Issue tracker mention | Comment webhook → bus |
| Work in a pull or merge request | Developer | Git host review comment | Git host mention | Comment webhook → bus |
| Domain or business questions | Subject-matter experts, product owners, or anyone who doesn't work in those tools | Interview app | Bus notification | Interview app → interview-answer handler → bus |

- **Multiple identities - not in scope.** Federating one person's identities across applications or environments is done outside the platform (event bus specification, section 16).
- **Stateless handling.** Each reply is its own event. The handler reads the current context from the system of record and responds; it keeps no conversation state of its own.
- **Answers can be configured to be optional or required.**
- **Structured replies.** Questions offer lettered options where possible; agents interpret free text and ask a follow-up only when a reply is unclear.
- **Rerouting is manual,** done in the control plane.
- **Question content** carries only what is needed to answer: plain-language behavior, examples, ticket context. No secrets or code dumps. The agent that writes a question redacts it.

## 10. Interview app

- **Sign-in:** OIDC. Registration is restricted (for example by an allowlist) and grants access to the interview app only.
- **Reads (sync):** sync gateway → interview-read handler, which returns only questions addressed to the caller.
- **Answers:** sync gateway → interview-answer handler, which publishes the answer as an event for the interview handler and returns an acknowledgment. The caller's identity is taken from the request authorizer, never from the request body, and checked against the question's addressee list.
- **Notifications:** the platform uses the bus's notifications (event bus specification, section 12) to tell people who don't work in the systems of record that a question is waiting, with a link to it in the interview app.

## 11. Skills and MCP

- **Skills** encode workflows (for example "update this change and confirm behavior changes with the developer"). The same skill runs in a developer's interactive coding agent session and in a headless agent handler.
- **Handler-creation skill** (event bus specification, section 19): agent handler is one of its handler types.
- **MCP:** the event bus control plane's MCP server provides tools to MCP clients, including developers' sessions and agents.
- **Skill-driven questions** follow the channel rules in section 9: a skill that needs a developer's confirmation asks in the ticket before a pull or merge request exists, or in the pull or merge request after.

## 12. Identity and secrets

Follows the event bus specification, plus:

| Credential | Held by |
| --- | --- |
| Inference credential | Agent handlers, scoped per handler where the provider allows |
| Read-only tokens for systems of record | Agent handlers that read them |
| Working-branch push token | Agent handlers that change code; cannot write to protected branches |
| Agent token for the MCP server | Agent handlers that use MCP |

## 13. Observability

Follows the event bus specification, plus:

- **Agent spans** exported through OpenTelemetry, continuing the triggering event's trace.
- **Metrics:** inference calls, tokens, and cost per agent handler and per task; questions asked, answered, and unanswered; time to answer.
- **Activity store rows** for every question and answer, keyed by the entity they concern.
- **Agent session logs** are kept in the object store, linked to the triggering trace, and indexed in the telemetry backends' logs. They should later be readable by analysis or replay tooling (for example LiteLLM). Requirements need elicitation before implementation.
- **Supported providers and SDKs.** Capturing session logs depends on the inference provider and agent SDK, so session logs are supported for a chosen set of them (section 15). The session log library can be extended to support more later.

## 14. Key decisions and rejected alternatives

| Decision | Rejected alternative | Why |
| --- | --- | --- |
| Agents as bus handlers | A separate agent framework with its own UI, API, and tool registry (for example kagent or Ark) | It would compete with the bus and control plane |
| One agent handler type, with the execution class set by its handler definition | Separate role types for reasoning agents and code-task agents | The bus's execution classes already express the difference |
| Multi-step code changes in one sandboxed task | A workflow engine coordinating steps | Not needed at current complexity |
| Stateless question handling; context read from systems of record | Handlers keeping their own conversation state | State lives in the systems of record; each reply is handled independently |
| Interview app separate from the control plane | Interviews inside the control plane | Interview users need access to their own questions only; the two can be secured differently at deployment (for example the control plane reachable only over VPN, the interview app on the public internet) |
| Answers through a request handler that publishes an event | Answers sent to the ingress endpoint | The interview app authenticates people with OIDC, which is the sync gateway's job |

## 15. Design-time choices

The event bus specification's design-time choices (its section 22) apply. Items most affected by this layer: platform options, OIDC providers, activity store database, telemetry conventions.

Agent-platform choices:

| Choice | What to decide |
| --- | --- |
| Concurrency caps | How agent handler concurrency is capped |
| Interview store | Shared with the activity store or separate |
| Notification methods | Which to use; at least one is required by the agent platform (optional for the bus) |
| Agent session logs | Requirements, after elicitation; which inference providers and agent SDKs the session log library supports first |

Inference provider, agent SDK, headless coding agent, and where builds and tests run are chosen per handler, not for the platform, except for session log support.

## 16. Glossary

| Term | Meaning |
| --- | --- |
| Agent handler | A handler whose logic calls a model, in whichever execution class its work needs |
| Question | An event asking one or more people for input |
| Interview app | The web app where people outside the systems of record answer questions |
| Skill | An encoded workflow usable by people and agents alike |
| Agent session log | The full record of one agent session |
