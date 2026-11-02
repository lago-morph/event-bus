# Platform Event Bus — Architecture Specification

**Status:** draft · **Layer:** 1 (event bus) · **Scope:** provider-neutral

This specification describes the platform event bus in terms of roles, contracts, and required capabilities. It names no cloud provider. Platform profiles (AWS, Cloudflare) show that every role here can be filled on a given platform; integration profiles show which external services fit the connector pattern. Neither kind of profile is a design document. Design is a separate phase.

**Generic is an objective, not a strict rule.** The architecture is the same on every platform at the conceptual level: the same roles, contracts, and semantics. Handler code and configuration are not portable between platforms, and the bus provides no generic adapters (for example database adapters); those are other abstractions and out of scope.

## 1. Purpose and scope

The platform event bus is general-purpose plumbing for connecting external systems, internal services, AI agents, and people through events. It provides:

- authenticated inbound webhooks and system calls,
- routing and fan-out of events to handlers,
- one queue per handler,
- synchronous request paths,
- dispatch of sandboxed tasks,
- notifications to people,
- a record of everything that happened,
- a control plane,
- end-to-end observability.

Projects build on the bus by adding handlers; they do not build their own plumbing.

## 2. Document map

| Document | Purpose |
| --- | --- |
| Platform event bus specification (this document) | Roles, contracts, required capabilities |
| AWS profile | Maps every role and execution class to at least one AWS option |
| Cloudflare profile | Maps every role and execution class to at least one Cloudflare option |
| Integration profiles | Examples of external services compatible with the connector pattern |
| Agent platform specification (layer 2) | AI agents on the bus |
| Agent platform AWS profile | Maps every agent-platform role to at least one AWS option |
| Agent platform Cloudflare profile | Maps every agent-platform role to at least one Cloudflare option |
| Slice 1: Button demo | Unmanaged consumers, request handler, handler-creation skill |
| Slice 2: Button demo with queue-triggered functions | Queue-triggered function class |
| Slice 3: PR enrichment | First real integration: git host webhook and connector |
| Slice 4: Button demo with durable workflows and sandboxed tasks | Durable workflow and sandboxed task classes; task dispatcher |
| Slice 5: Enhancement issue impact analysis | First agent handler; issue tracker connector |
| Slice 6: Remaining components (provisional) | Every component not built in slices 1 to 5 |

## 3. Principles

- **Choreography, not orchestration.** The event router delivers events to handlers; each handler does its work and publishes what happened. There is no central coordinator or workflow engine across handlers. Process state lives in the systems of record (for example an issue tracker or a git host), not in the bus.
- **Async for state changes, sync for reads.** State changes travel as events through the router and handler queues. Synchronous requests are for reads and short operations.
- **Mutations through the bus; reads may be direct.** Every change to an external system is an event handled by that system's connector. A handler may read from an external system or internal store directly with read-only credentials; such reads are not recorded as activity but are always traced, metered, and logged.
- **One queue per handler.** Every handler has its own queue and dead-letter queue, generated from its handler definition.
- **Identical semantics on every platform.** Delivery, acknowledgment, retry, and failure semantics are the same on every platform. Where a platform lacks a native capability, the bus adds plumbing rather than weakening the semantics.
- **Exposure is a deployment choice.** Every component can run on the public internet behind authentication; where the platform supports private networking (for example AWS), components can instead run privately, leaving only the ingress and sync endpoints public.
- **Everything is recorded and observable.** Every event lands in the activity store, every hop is traced, and every log line links to its trace.
- **Declared, not hand-wired.** Queues, routing rules, and consumer bindings are generated from handler definitions; all infrastructure is defined as code.

## 4. Architecture roles

| Role | Responsibility | Required capabilities |
| --- | --- | --- |
| Ingress endpoint | Receives inbound async events from external systems | Public HTTPS endpoint; access to the raw request body; can run code to verify source signatures or system tokens and publish events |
| Event router | Delivers each event to every handler that subscribes to it | Content-based matching on event attributes; fan-out to many handler queues; accepts events from ingress, handlers, and other publishers |
| Event archive | Keeps every routed event for replay | Durable storage of events; replay of a time range or filter back through the router |
| Handler queue | Holds events for one handler | At-least-once delivery; acknowledgment by the consumer; redelivery after a visibility timeout; a visible attempt count; dead-letter queue after a maximum attempt count |
| Handler | Consumes its queue and does one job | Runs in one of the execution classes (section 5) |
| Task dispatcher | Starts a sandboxed task per queued message and acknowledges the message only when the task completes (section 10) | Can hold a message unacknowledged for the task's duration; can start sandboxed tasks; consumes completion events |
| Sync gateway | Accepts synchronous requests and routes them to request handlers | Public or private HTTPS; authorization before the handler runs; passes trace context through; enforces a request timeout |
| Request authorizer | Authenticates sync requests | Validates OIDC tokens (people) and bearer or signed tokens (systems) |
| Activity recorder | Writes every event to the activity store | A handler subscribed to all events |
| Activity store | Queryable record of every event | A database supporting queries by entity and time, readable by the observability stack |
| Object store | Holds large payloads and archives | Durable object storage; referenced from events (claim check) |
| Control plane | Human and tool interface to the bus | Web app, API, and MCP server over the activity store; publishes manual actions as events |
| OIDC provider | Identity for people | Standard OIDC; usable by the sync gateway, control plane, and other apps |
| Workload identity | Identity for bus components | Each component authenticates to platform services without long-lived shared secrets where the platform allows |
| Secrets store | Credentials for external systems and internal stores | Scoped access per component; rotation |
| Scheduler | Publishes timed events | Cron-style and one-off schedules that publish events to the router |
| Telemetry pipeline | Collects traces, metrics, and logs | Accepts OTLP from bus code; bridges platform-native logs and metrics into the backends |
| Telemetry backends | Store and display telemetry | Traces, metrics, logs, and dashboards, with links between them (for example a Grafana-based stack) |
| Infrastructure as code | Defines and deploys everything | Declarative definitions of all resources, including those generated from handler definitions |

## 5. Execution classes

Handlers run in execution classes defined by behavior. A platform profile shows at least one option per class.

| Class | Behavior | Typical use |
| --- | --- | --- |
| Queue-triggered function | Invoked with messages from its queue; short-lived; acknowledges on success | Most workers and connectors |
| Durable workflow | Multi-step, durably checkpointed execution started from its queue; steps retried independently | Multi-step async work with retries |
| Request handler | Answers sync requests within the request timeout | Sync reads and short operations |
| Sandboxed task | An isolated Linux environment per message, started by the task dispatcher | Code tasks, builds, tools that need a full environment |
| Unmanaged consumer | No managed consumer: the bus creates the queue and its routing, and the consumer pulls and acknowledges through the platform's own queue API (for example SQS receive and delete, or Cloudflare Queues HTTP pull). The bus provides no API around it | Development and special cases |

All classes share the same handler contract (section 9). Scaling is not an execution class; how a handler scales is a design decision.

## 6. Logical connections

| From | To | Via | What flows |
| --- | --- | --- | --- |
| External system | Handlers | Webhook or system call → ingress endpoint → event router → handler queue | Inbound events |
| Any publisher (ingress, handlers, control plane, scheduler) | Handlers | Event router → handler queue | Events and commands |
| Every publisher | Activity recorder | Event router (match all) → recorder queue | Every event |
| Activity recorder | Activity store | Database write | Event rows |
| Event router | Event archive | Router writes every event | Archived events |
| External caller | Request handler | Sync gateway (authorized) | Request and response |
| Handler or task dispatcher | Sandboxed task | Dispatcher starts task | Task input |
| Sandboxed task | Task dispatcher | Event router → dispatcher queue | Task completed or failed |
| Connector | External system | Outbound API call with trace context | Mutations |
| Any handler | External system or internal store | Direct read with read-only credentials | Reads (traced, not recorded) |
| Person | Control plane | HTTPS, OIDC sign-in | UI, reads, manual actions |
| MCP client | Control plane MCP server | MCP over HTTPS, OIDC | Queries and actions |
| All bus code | Telemetry pipeline | OTLP | Traces, metrics, logs |
| Platform | Telemetry pipeline | Platform telemetry bridge | Platform logs and metrics |

## 7. Async path

Webhook or system call → ingress endpoint → event router → handler queues → handlers.

**Ingress endpoint.** Every inbound route runs the same ingress logic, the uniform front door for all inbound connectors:

1. Authenticate the sender: verify the source's signature (for example an HMAC over body and timestamp, or the scheme the source supports) and reject stale timestamps, or validate a system token (the same identity model as external sync callers, section 16).
2. De-duplicate by the source's delivery ID.
3. Start the root span, or continue the caller's `traceparent` when a system caller sends one. Webhook senders cannot send trace context, so their traces start here.
4. Wrap the payload as a CloudEvent with `traceparent` in the distributed-tracing extension; store large payloads in the object store and reference them.
5. Publish to the event router and return success.
6. On rejection: return 401, record an error span, a log line, and a metric, and publish `platform.webhook.rejected` with metadata only. Rejected payloads never reach the router.

**Event router.** Matches each event against routing rules generated from handler definitions and delivers a copy to each matching handler's queue. Every routed event is also written to the event archive.

## 8. Sync path

Synchronous request and response, for reads and short operations where the caller needs the answer now.

- Requests pass through the sync gateway, which authorizes them before the request handler runs.
- The caller decides how long it waits, and the sync gateway enforces a maximum. A handler that runs longer looks like a hang to the caller. Anything that can run long goes on the async path; the caller gets an acknowledgment and learns the outcome from an event or a later read.
- Sync operations are idempotent where possible, so a retry after a timeout causes no duplicate effect.
- A request handler that changes state also publishes an event.
- Callers the bus controls send `traceparent`; the handler continues that trace, or starts one when none arrives.
- Sync requests never reach connectors. Sync targets are request handlers only.

## 9. Handlers

| Handler type | Does |
| --- | --- |
| Worker | Processes an event and publishes results as new events |
| Connector | Talks to one external system with that system's credential; publishes the outcome (delivered or failed) of every outbound call |
| Activity recorder | Writes every event to the activity store |

### Handler definitions

Every handler is declared by a handler definition in the bus configuration (section 19). The definition is the only place its subscriptions, queue, and execution class are configured. Platform tooling compiles definitions into routing rules, queues, dead-letter queues, access policies, and consumer bindings.

A handler definition contains:

- name and owner,
- event filters, one per subscription,
- queue settings: visibility timeout, maximum attempts, optional per-entity ordering key,
- execution class,
- optional platform-specific settings.

The format of handler definitions on each platform is a design decision. The YAML below is illustrative only.

```yaml
name: ticket-review
owner: platform-team
filters:                       # event-matching rules, one per subscription
  - source: [issues]
    type: [issues.issue.labeled]
    data: { label: [review] }
queue:
  visibilityTimeoutSeconds: 300
  maxAttempts: 5
  serializeBy: null            # optional entity key for per-entity ordering
execution:
  class: queue-function        # queue-function | workflow | request-handler |
                               # sandboxed-task | unmanaged
profile: {}                    # optional platform-specific settings
```

### Handler contract (all execution classes)

- **At-least-once delivery.** Handlers are idempotent on the CloudEvent `id`.
- **Acknowledge only after success.** A message is acknowledged only when the handler's work is complete. Failure leaves it for redelivery.
- **Bounded retries.** At the maximum attempt count the message moves to the dead-letter queue. On the final allowed attempt the handler publishes a failure event (`<domain>.<entity>.failed`) so the outcome is recorded.
- **Trace continuation.** The handler continues the trace from the event's `traceparent` and records a wait span (see section 18).
- **Graceful shutdown** for long-running processes: stop pulling, finish or release in-flight messages.

### Shared handler library

One library per platform and language, covering: pulling or receiving, acknowledgment, CloudEvent publishing, de-duplication, trace continuation, wait spans, structured logging with `trace_id` and entity IDs. Each handler then contains only its own logic.

## 10. Sandboxed task dispatch

Sandboxed tasks keep the same semantics as every other handler: the message is acknowledged only when the work is done.

1. The task dispatcher takes a message from the handler's queue and keeps it unacknowledged.
2. It records the dispatch (task ID, message receipt or lease, attempt) in the activity store and starts a sandboxed task with the event as input.
3. The task does its work and publishes `task.completed` or `task.failed` with its task ID.
4. The dispatcher consumes that event and acknowledges the original message (completed), or releases it for retry (failed). On the final allowed attempt it publishes the failure event, and the message moves to the dead-letter queue.
5. If no completion arrives within the task's time limit, the dispatcher treats it as failed. Message visibility must outlast the task, either by a long enough visibility timeout or by extending it while the task runs.

## 11. Choreography

- Routing rules decide which handlers receive an event.
- Each handler does its own work and publishes what happened; other handlers react.
- Process state lives in the system of record that owns it. Handlers read it when they need it.
- Long multi-step machine work runs inside one handler (for example a durable workflow or a sandboxed task), not across a chain of coordinated handlers.
- Timed behavior is a scheduled event handled like any other.

Handler definitions are the readable map of who reacts to what; traces and the activity store show what actually happened.

## 12. Connector pattern

| Direction | Path | Auth |
| --- | --- | --- |
| Inbound, async | Webhook or system call → ingress endpoint → event router → handler queues | Source's webhook signature or system token |
| Inbound, sync | Sync gateway → request handler | OIDC token or system token |
| Outbound | Event → the system's connector → external API | The system's credential, held only by its connector |

- Outbound calls carry `traceparent`.
- Outbound calls are at-least-once; they are idempotent on event ID where the external API allows.
- Each connector publishes the outcome of every outbound call.
- Reads may bypass connectors (section 3).
- Fault-injection flags exist only for synthetic test connectors, honored only when a configuration switch enables them.

### Notifications

The bus can notify people through notification methods (for example email, Slack, or Discord). Which methods are supported is implementation specific (section 22).

- A handler publishes a notification event; the connector for the notification method sends it.
- Notifications carry links to where the person acts; people do not act inside the notification method.
- Non-urgent notifications can be batched into a digest.

## 13. Events

- **Envelope:** CloudEvents. Core attributes `id`, `source`, `type`, `subject` (the main entity ID), `time`; the distributed-tracing extension for `traceparent`; extensions for entity IDs that matter across the bus.
- **Naming:** `<domain>.<entity>.<action>`. `platform.*` is reserved for events the bus itself emits.
- **Events and commands:** an event says something happened; a command asks a handler to do something. Both travel the same way.
- **Size:** events stay small. Large payloads go to the object store and the event references them.
- **Idempotency:** the CloudEvent `id` is the de-duplication key.
- **Loop prevention:** handlers ignore events caused by the bus's own identities where reacting would create a loop.
- **Rejected input** is never published with its payload.
- **Triggers from people's actions:** a handler can react to something a person does in an external system, for example setting a label on an issue, so a person can start automated work deliberately.

## 14. Activity recording and the activity store

The activity recorder subscribes to every event and writes one row per event: time, type, source, subject, entity IDs, outcome, attempt count, trace ID. It is idempotent on event ID.

The activity store is a database. It answers "what happened" for any entity and is the data source for activity dashboards and the control plane. Entity-level data (commit, ticket, pipeline run) lives here, not in metric labels.

## 15. Control plane

The control plane is the bus's human and tool interface. It does not orchestrate.

- **Web app:** activity by entity, recent events and failures, dead-letter contents, links to traces and logs.
- **API:** synchronous reads over the activity store and other internal data.
- **MCP server:** the same reads and actions as tools for MCP clients.
- **Manual actions:** rerun, reroute, redrive a dead-lettered message; each is published as an event.
- **Access:** OIDC sign-in; not directly exposed without authentication. Each action is recorded with the person who took it.

## 16. Identity and authentication

| Who | Authenticates how |
| --- | --- |
| Webhook senders | Source signature, verified at ingress |
| External systems calling ingress or the sync gateway | Bearer or signed system token, checked at ingress or by the request authorizer |
| People | An OIDC provider (identity federation is out of scope; see below). Self-registration, where used, must not grant broader network access than intended |
| Bus components | Workload identity; service tokens for internal HTTP where needed |
| Connectors to external systems | That system's credential |

Every action is attributed to an identity: the person, the calling system, or the handler's identity.

**Identity federation is out of scope.** The bus does not federate identities across handlers and connectors. If the same person uses multiple identities (for example in an OIDC provider, as notification targets, and in logins to cloud services), identity mapping is done outside the platform.

## 17. Secrets

- Credentials come from the secrets store, scoped per component, never from code or configuration files.
- Each external system's credential lives only in that system's connector; read-only credentials for direct reads are scoped to the handlers that read.
- Model inputs, logs, and traces never contain secrets.
- Shared secrets (webhook signing secrets, sync tokens) are rotatable.

## 18. Observability

**Traces**

- Inbound async traces start at the ingress endpoint; sync traces continue the caller's `traceparent` when present.
- `traceparent` travels inside each CloudEvent and as a header on every outbound call.
- Managed queues and routers usually emit no spans. Each consumer creates a **wait span** from the event's time and the message's enqueue time to its receive time, so traces are continuous.
- Direct reads produce spans like any other call.
- Spans are not rebuilt from logs after the fact.

**Logs:** every line carries `trace_id` and the relevant entity IDs, so logs link to traces.

**Metrics:** handler outcomes, failures, and latency from bus code; queue depth, dead-letter counts, and platform service metrics through the platform telemetry bridge. Entity-level data stays out of metric labels.

**Alerts:** any dead-lettered message, rejected webhooks, rejected sync requests.

## 19. Configuration, repository, and deployment

- **Repositories.** There are three kinds of content:
  - the bus implementation,
  - the bus configuration (handler definitions),
  - handler code (one or more repositories).

  They may be separate repositories, one monorepo, or a combination. Where each lives is configured per platform and is a design decision.
- **Infrastructure as code** defines every resource; generated resources come from handler definitions.
- **Handler-creation skill.** The bus implementation holds one AI skill for creating handlers: workers, connectors, and request handlers, for every execution class. Instructions are language neutral. The implementation chooses its languages (section 22), and the skill is refactored to add examples in them. It uses aggressive progressive disclosure (role → execution class → platform → language), so implementing one choice loads only that choice's material. It mandates handler definitions, the shared handler library, the handler contract, and telemetry for every call including direct reads. The skill knows a task may need several handlers of different types and execution classes, chooses them from the work each part does, and creates a handler definition and implementation for each. Later layers add handler types (for example the agent platform's agent handler).

## 20. Key decisions and rejected alternatives

| Decision | Rejected alternative | Why |
| --- | --- | --- |
| Choreography; state in systems of record | A workflow engine or homegrown state machines coordinating handlers | Every flow so far is event-driven; a coordinator adds a second source of state |
| Uniform ingress logic for every webhook | Per-source listeners | One place for verification, de-duplication, and trace start |
| Routing generated from handler definitions | Hand-written rules or routing in code | Declared, reviewable configuration |
| One queue and dead-letter queue per handler | Shared queues | Isolation, independent retries, clear ownership |
| Same delivery semantics on every platform, adding plumbing where needed | Accepting platform-specific semantics | The handler contract and the skill mean the same thing everywhere |
| Sandboxed tasks acknowledged on completion via a dispatcher | Acknowledging on task launch | Launch success says nothing about task success |
| Notifications carry links | Acting inside the notification method | Keeps actions in one authenticated place |
| Live spans plus synthetic wait spans | Rebuilding spans from logs | Reconstruction is late, incomplete, and needs extra machinery |
| Entity-level data in the activity store | Metric labels | Cardinality |
| OIDC for people | Platform-specific identity | Portable across platforms and providers |

## 21. Glossary

| Term | Meaning |
| --- | --- |
| Event | A record that something happened, as a CloudEvent |
| Command | An event asking a handler to do something |
| Handler | Code that consumes its own queue: worker, connector, or activity recorder |
| Handler definition | The declaration of a handler's filters, queue settings, and execution class |
| Execution class | How a handler runs: queue-function, workflow, request-handler, sandboxed-task, unmanaged |
| Unmanaged consumer | A handler queue with no managed consumer, consumed through the platform's own queue API |
| Connector | The handler that talks to one external system with that system's credential |
| Notification method | A way of notifying people, for example email, Slack, or Discord |
| Task dispatcher | The component that runs sandboxed tasks and acknowledges their messages on completion |
| Wait span | A synthetic span covering time spent in routing and queues, created by the consumer |
| Activity store | The database recording every event |
| Platform profile | A document mapping every role and execution class to at least one option on one platform |
| Integration profile | Examples of external services compatible with the connector pattern |

## 22. Design-time choices

This is the single list of choices left to the design phase. Other documents refer to these items by name; the platform profiles show the options available for them.

### Observability

| Choice | What to decide |
| --- | --- |
| Telemetry backends | Where traces, metrics, and logs are stored and viewed (for example a hosted Grafana account, a self-hosted Grafana stack, or the platform's native tools) |
| Telemetry pipeline | Whether bus code sends OTLP directly to the backends or through a collector, and where a collector runs |
| Telemetry credentials | How OTLP credentials are held and scoped per component |
| Platform telemetry bridge | How platform-native logs and metrics reach the backends: exported, or queried in place |
| Trace links | How the demo app, dashboards, and the control plane build a link to a trace |
| Activity store access from dashboards | How the backends reach the activity store as a data source |
| Wait span source | Which message timestamps each platform exposes for enqueue and receive times |
| Sampling | Sampling rate per environment |
| Telemetry conventions | Metric names, label budget, and required log fields |
| Dashboards and alerts | How they are provisioned as code, and where alerts are delivered |

### Platform and build

| Choice | What to decide |
| --- | --- |
| Platform options | Which profile option fills each role and execution class |
| Task dispatcher | How it is invoked and where dispatch records are kept |
| Activity store database | Which database, per platform |
| OIDC providers | For people using the bus and for each app that signs people in |
| Handler definition format | The format per platform and how definitions compile into resources |
| Infrastructure as code | The tool per platform |
| Repository layout | Where the bus implementation, bus configuration, and handler code live |
| Handler languages | Which languages handlers are written in, and the handler-creation skill's example languages |
| Unmanaged consumer hosting | Where unmanaged consumers run |
| Scaling | How each handler scales |
| Networking | Public or private networking per component |
| Queue settings | Maximum attempts and visibility timeout per handler |
| Webhook signature schemes | The scheme per webhook source |
| Notification methods | Which notification methods are supported |
