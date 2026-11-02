# Platform Event Bus — Slice 1: Button Demo

**Status:** draft · **Extends:** platform event bus specification · **Scope:** provider-neutral

## 1. Summary and scope

The button demo is a synthetic slice that exercises the event bus end to end without depending on any real external system. A standalone demo app plays the role of an external SaaS system:

- it sends a signed webhook when a user presses a button,
- the bus sends an acknowledgment back through the demo app's API,
- the demo app checks an event's status through a synchronous route.

In this slice every async handler is an unmanaged consumer, started by hand. This slice also creates the handler-creation skill used to build every handler.

**Read with:** the platform event bus specification. This slice depends on no other layer. The specification defines the patterns this slice implements, and they must be followed: the async path and ingress logic, the sync path and its timeouts, handlers and handler definitions, the handler contract and shared handler library, the connector pattern, events, the activity store, observability, secrets, repositories, and deployment. Its key decisions and rejected alternatives also apply. This document covers only what is specific to the slice.

The slice is provider-neutral. It names roles and execution classes only; which platform option fills each one is chosen at design time from the platform profiles.

**Builds on:** nothing; this is the first slice. Later slices reuse what is built here.

### Not implemented in this slice

- **Managed execution classes for async handlers.** Async workers and connectors are unmanaged consumers, started and configured by hand. Queue-triggered functions come in slice 2; durable workflows and sandboxed tasks in slice 4.
- **Other sync routes.** The only request handler is the event-status handler.
- **Platform telemetry bridge.** Platform-native logs and metrics (for example logs from the ingress endpoint and request authorizer when they run as platform functions) stay in the platform's own logging until a later slice.
- **AI agents, interviews, control plane.**

### Goals

- Prove the inbound and outbound connector pattern that every real integration will reuse.
- Prove the sync path with a request handler that reads directly from the activity store.
- Show one continuous trace for each button press, with logs from every handler linked to it.
- Make every failure visible in the demo app and in the bus's traces, logs, metrics, and activity store.
- Create the handler-creation skill and build this slice's handlers with it.
- Develop and test the plumbing in isolation from production systems.

### In and out of scope

| In scope | Out of scope |
| --- | --- |
| Ingress endpoint with a webhook route for the demo app | Queue-triggered function (slice 2), durable workflow and sandboxed task (slice 4) handlers |
| Event router, routing rules, and per-handler queues with dead-letter queues, generated from handler definitions | Additional sync routes (slice 6) |
| Request handler, connection handler, activity recorder (unmanaged consumers, run by hand) | Platform telemetry bridge (slice 6) |
| Activity store | Control plane, agents, interviews |
| Sync gateway route with a request authorizer, to the event-status request handler | Real external systems (git hosting, CI/CD, issue tracking, notification methods) |
| Handler-creation skill: unmanaged-consumer worker, unmanaged-consumer connector, request handler; examples in the implementation's chosen languages | |
| Standalone demo app with four buttons, an implementation dropdown (one option in this slice), and a status check | |
| Traces, metrics, and logs sent directly by bus code; an activity dashboard | |

**Telemetry in this slice** uses only direct integration with the telemetry backends: bus code sends its own telemetry over OTLP, and nothing is bridged from the platform. Integrating platform-native telemetry comes in slice 6. This slice sends:

- traces, metrics, and logs from the handlers and the request handler,
- traces and custom metrics from the ingress endpoint and request authorizer.

## 2. At a glance

1. The signed webhook enters through the ingress endpoint, which starts the trace and publishes `demo.button.pressed`.
2. The event router delivers the pressed event to the request handler, which publishes `demo.button.acked`.
3. The router delivers the acked event to the connection handler, which calls the demo app's ack API.
4. The activity recorder receives every event.

Status checks take the sync path described in section 7.

## 3. Components

| Component | Side | Role (spec) / execution class | Does |
| --- | --- | --- | --- |
| Ingress endpoint | Bus | Ingress endpoint | Verifies the HMAC signature and timestamp, de-duplicates, starts the root span, publishes CloudEvents |
| Request authorizer | Bus | Request authorizer | Checks the demo app's sync bearer token on the sync route |
| Event router and rules | Bus | Event router | Routes each event type to the right handler queues |
| Handler queues and dead-letter queues | Bus | Handler queue | One queue and dead-letter queue per handler |
| Request handler | Bus | Worker · unmanaged consumer | Consumes pressed events; publishes acked events |
| Demo app connection handler | Bus | Connector · unmanaged consumer | Consumes acked events; calls the demo app's ack API; publishes delivered or failed |
| Activity recorder | Bus | Activity recorder · unmanaged consumer | Writes every event to the activity store |
| Activity store | Bus | Activity store | One row per event, with outcome and trace ID |
| Event-status handler | Bus | Request handler | Returns an event's history, read directly from the activity store |
| Sync gateway route | Bus | Sync gateway | Public route for the demo app's status checks |
| Handler-creation skill | Bus | Bus implementation repository | AI skill used to build this slice's handlers |
| Telemetry pipeline and backends | Bus | Telemetry pipeline, telemetry backends | Traces, logs, metrics, dashboards |
| Demo app | Demo app | External system | UI with four buttons, the implementation dropdown, the event list, and status checks; ack API |
| Demo app store | Demo app | External system | Events, ack status, attempts, trace IDs |

## 4. Logical connections

| From | To | Via | What flows | Auth |
| --- | --- | --- | --- | --- |
| User's browser | Demo app | HTTPS | UI, button presses, status checks | OIDC sign-in |
| Demo app | Ingress endpoint | HTTPS, webhook route | Button-pressed webhook, with the selected implementation | HMAC signature with timestamp |
| Ingress endpoint | Request handler | Event router → request-handler queue | `demo.button.pressed` | Workload identity |
| Ingress endpoint | Activity recorder | Event router → recorder queue | `demo.button.pressed`, `platform.webhook.rejected` | Workload identity |
| Request handler | Connection handler | Event router → connection-handler queue | `demo.button.acked` | Workload identity |
| Request handler | Activity recorder | Event router → recorder queue | `demo.button.acked` | Workload identity |
| Connection handler | Demo app | HTTPS, ack API | Ack callback with `traceparent` header | Bearer token |
| Connection handler | Activity recorder | Event router → recorder queue | `demo.ack.delivered`, `demo.ack.failed` | Workload identity |
| Activity recorder | Activity store | Database write | Event rows | Database credentials |
| Demo app | Event-status handler | Sync gateway route (authorized) | Status request with `traceparent`; event history | Sync bearer token, request authorizer |
| Event-status handler | Activity store | Direct read, read-only | Event history (not recorded as activity) | Read-only credentials from the secrets store |
| Handlers, event-status handler | Telemetry pipeline | OTLP | Traces, metrics, logs | Telemetry pipeline's own auth |
| Ingress endpoint, request authorizer | Telemetry pipeline | OTLP | Traces and custom metrics | Telemetry pipeline's own auth |
| Dashboards | Activity store, telemetry backends | Data sources | Dashboard, traces, logs | Backend's own sign-in |
| Demo app UI | Telemetry backends | Browser link | Trace for each event | Backend's own sign-in |

## 5. Queues and routing rules

Queues follow the specification: one queue and dead-letter queue per handler, filled by routing rules generated from handler definitions.

| Queue | Consumer | Fed by rules matching | Dead-letter after |
| --- | --- | --- | --- |
| request-handler | Request handler | `demo.button.pressed` | Maximum attempts reached |
| connection-handler | Connection handler | `demo.button.acked` | Maximum attempts reached |
| activity-recorder | Activity recorder | Every event on the bus | Maximum attempts reached |

## 6. Normal flow

1. **Press.** A signed-in user presses the normal button. The demo app stores the event as pending with a new event ID and sends a signed webhook to the ingress endpoint.
2. **Ingress.** The ingress endpoint:
   - verifies the HMAC signature and timestamp,
   - de-duplicates by delivery ID,
   - starts the root span,
   - publishes `demo.button.pressed` to the event router, with `traceparent` inside the event,
   - returns 202 to the demo app.
3. **Route.** Routing rules send the event to the request-handler queue and the recorder queue.
4. **Request handler.** Pulls from its queue, continues the trace, records a wait span for the routing and queue time, and publishes `demo.button.acked`.
5. **Route.** Rules send the acked event to the connection-handler queue and the recorder queue.
6. **Connection handler.** Pulls from its queue, continues the trace, and calls the demo app's ack API with its bearer token and a `traceparent` header. On success it publishes `demo.ack.delivered`.
7. **Demo app.** Validates the token, marks the event acked, and stores the trace ID.
8. **Recorder.** Writes a row for each event (pressed, acked, delivered) to the activity store.
9. **See it.** The demo app shows the event as acked with its trace ID, linked to the trace in the telemetry backends. The dashboard shows the event rows, the full trace, and both handlers' logs.

## 7. Sync status checks

Each event row in the demo app has a **Check status** action. It returns the event's history from the activity store: pressed, acked, delivered or failed, with timestamps and the event's trace ID.

1. The demo app sends the request with its sync bearer token and a `traceparent` header, so the status check's trace starts in the demo app.
2. The request authorizer validates the token.
3. The sync gateway routes the request to the event-status handler.
4. The event-status handler reads the history directly from the activity store with read-only credentials. The read is not recorded as activity, but it produces a span, metrics, and log lines.
5. The demo app shows the history inline, with links to both the status check's trace and the original event's trace.

Status checks are read-only and publish no events. The event-status handler must answer well within the demo app's timeout and the sync gateway's maximum.

## 8. Failure scenarios: the four buttons

Every failure must be visible in the demo app and in the bus's traces, logs, metrics, and activity store.

| Button | What happens | Demo app shows | Bus shows |
| --- | --- | --- | --- |
| Normal | Pressed, acked, delivered | Event pending, then acked, with trace ID and trace link | Full trace; logs from both handlers; activity rows pressed, acked, delivered; success metrics |
| Bad webhook auth | Demo app signs the webhook with a wrong secret; the ingress endpoint rejects it with 401 | Event marked webhook rejected, with the response it received | Error span from the ingress endpoint; rejected-webhook metric; `platform.webhook.rejected` activity row (metadata only, no payload) |
| Bad callback token | Webhook is accepted; a flag in the payload tells the connection handler to use a wrong token | Event stays unacked; each rejected ack attempt is recorded and shown as "ack rejected: bad token" | Error span per attempt; logs; ack-failure metric; after the maximum attempts the message moves to the dead-letter queue (alert) and `demo.ack.failed` is recorded |
| Bad sync token | Demo app runs a status check with a wrong sync bearer token | Status check rejected (the 401 it received) | Error span from the request authorizer, continuing the demo app's trace; rejected-sync-request metric and alert |

- **Fault-injection guard:** the connection handler honors the bad-token flag only when a configuration switch enables it. The switch is on only in the environment running this demo.
- **Retries** follow the handler contract; for this slice the final-attempt failure event is `demo.ack.failed`.

## 9. Event contracts

Envelope, naming, and de-duplication follow the specification. The CloudEvent subject is the demo event ID.

| Type | Published by | Carries |
| --- | --- | --- |
| `demo.button.pressed` | Ingress endpoint | Demo event ID, button kind, selected implementation, fault flag |
| `demo.button.acked` | Request handler | Demo event ID, fault flag |
| `demo.ack.delivered` | Connection handler | Demo event ID, attempt count |
| `demo.ack.failed` | Connection handler | Demo event ID, attempt count, last error |
| `platform.webhook.rejected` | Ingress endpoint | Source, reason, request ID; no payload |

## 10. Bus components

Each component follows the specification's async path, sync path, handlers, and connector pattern sections. Slice-specific behavior:

| Component | Consumes | Does | Publishes |
| --- | --- | --- | --- |
| Ingress endpoint | Demo webhook route | Verifies the demo app's HMAC signature and timestamp | `demo.button.pressed` (with the selected implementation), or `platform.webhook.rejected` |
| Request handler (worker) | request-handler queue | Acknowledges the press, carrying the fault flag forward | `demo.button.acked` |
| Demo app connection handler (connector) | connection-handler queue | Calls the demo app's ack API with its bearer token; uses a wrong token when the fault flag is set and the fault-injection switch is on | `demo.ack.delivered`, or `demo.ack.failed` on the final allowed attempt |
| Activity recorder | activity-recorder queue | Writes one row per event: time, type, demo event ID, outcome, attempt count, trace ID | None |
| Request authorizer | The sync route | Validates the demo app's sync bearer token; records an error span and metric on rejection | None |
| Event-status handler (request handler) | Sync requests | Reads an event's history directly from the activity store, with a span, metrics, and logs for the read | None |

- The three async handlers have handler definitions in the bus configuration with execution class `unmanaged`. Their queues and routing rules are generated from those definitions. The handlers themselves are started and configured by hand.
- The event-status handler has a handler definition with execution class `request-handler` and is deployed with the bus.
- All four are built with the handler-creation skill.

## 11. Handler-creation skill

This slice creates the single AI skill used to build every handler on the bus, and builds this slice's handlers with it. Later slices extend the same skill with more execution classes.

**Covered in this slice:** unmanaged-consumer worker, unmanaged-consumer connector, and request handler.

**Structure (aggressive progressive disclosure).** A small core skill file asks what is being built and loads only the matching reference files:

- handler type (worker, connector, request handler),
- execution class,
- platform,
- language.

Each choice has its own reference material, and each language its own example. Implementing an unmanaged-consumer worker never loads connector, request-handler, or other platform material.

**Language neutral.** Instructions describe behavior and contracts, not one language. The implementation chooses its languages (handler languages, section 17), and the skill is refactored to add examples in them.

**What the skill mandates for every handler:**

- A handler definition in the bus configuration (filters, queue settings, execution class).
- Use of the shared handler library for the platform and language: pulling and acknowledgment, CloudEvents, de-duplication, trace continuation, and wait spans.
- Traces, metrics, and logs for every call the handler makes, including direct reads outside the bus.
- Mutations of external systems only through connectors; direct calls only for reads, with read-only credentials.
- Secrets from the secrets store, never in code or configuration files.
- For connectors: an outcome event (delivered or failed) for every outbound call, and idempotency on event ID where the external API allows.

## 12. Demo app

The demo app is a standalone external system with its own infrastructure. It connects to the bus only through the public webhook route, the sync route, and its own ack API.

- **UI sign-in:** OIDC. The ack API does not use OIDC; it checks the bearer token instead.
- **Compute:** any host that serves the UI and the ack API.
- **Store:** its own store for events, status, ack attempts and their results, and trace IDs.
- **UI:**
  - four buttons: normal, bad webhook auth, bad callback token, bad sync token,
  - an implementation dropdown that chooses the worker and connector implementation as a pair; it has one option in this slice and is carried in the event,
  - a Check status action on each event row,
  - a list of active events showing status (pending, webhook rejected, acked, ack rejected), attempt history, and the trace ID with a link to the trace in the telemetry backends.
- **Ack API:** idempotent on event ID; rejects a missing or wrong token with 401 and records the rejected attempt.
- **Secrets:** its own secrets store entries for the webhook signing secret, the expected ack token, and the sync bearer token.
- **Deployment:** defined as code, separately from the bus, so it can be torn down or rebuilt without touching bus resources.

## 13. Security and secrets

| Secret or control | Held by | Used for |
| --- | --- | --- |
| Webhook signing secret | Demo app (its secrets store); bus copy in the bus secrets store for the ingress endpoint | HMAC over body and timestamp |
| Ack bearer token | Demo app (its secrets store); bus copy delivered to the connection handler (configured by hand where it runs) | Authenticating ack callbacks |
| Sync bearer token | Demo app (its secrets store); bus copy in the bus secrets store for the request authorizer | Authenticating status checks |
| Activity store read-only credentials | Bus secrets store, delivered to the event-status handler | Direct reads of event history (the first direct read on the bus) |
| UI sign-in | The demo app's OIDC provider | Demo app users |
| Router and queue access | Workload identity per component (scoped API tokens for unmanaged consumers where the platform requires them) | Publish and consume only what each needs |
| Fault-injection switch | Connection handler configuration | Enables the bad-token flag only in the demo environment |

- Both shared secrets can be rotated; rotation is a useful exercise for this slice.
- The demo app has no access to any bus component other than the public webhook and sync routes.

## 14. Observability

### Traces

Tracing follows the specification.

- **Normal press:** ingress endpoint → wait span → request handler → wait span → connection handler → ack call to the demo app.
- The demo app stores the trace ID from the `traceparent` header on the ack call and shows it with a link to the trace.
- **Status checks:** the trace starts in the demo app and continues through the request authorizer and the event-status handler, including a span for its direct read of the activity store.

### Logs

Logging follows the specification; every line carries `trace_id` and the demo event ID.

- Handler and event-status handler logs go directly to the telemetry backends.
- Platform-native logs (for example from the ingress endpoint and request authorizer when they run as platform functions) stay in the platform's own logging until slice 6 adds the telemetry bridge.

### Metrics

- **From handlers:** events processed by type and outcome, ack attempts and failures, end-to-end latency from press to ack delivered.
- **From the event-status handler:** status checks by outcome, latency, and activity store read latency.
- **From the ingress endpoint and request authorizer, sent directly:** rejected webhooks and rejected sync requests.

### Dashboard

| Panel | Source |
| --- | --- |
| Recent events with outcome and trace links | Activity store, traces |
| Presses, acks delivered, ack failures, rejected webhooks | Metrics |
| End-to-end latency, press to ack | Metrics |
| Status checks by outcome; rejected sync requests; latency | Metrics |
| Errors by component (handlers and event-status handler) | Logs |

**Alerts** fire on any dead-lettered message and on rejected webhooks.

## 15. Deployment

| What | Managed by |
| --- | --- |
| Bus: ingress endpoint and webhook route, sync gateway route, request authorizer, event router, generated routing rules, handler queues and dead-letter queues, access policies, secrets store entries, event-status handler | Infrastructure as code |
| Bus: request handler, connection handler, activity recorder | Started and configured by hand for the demo |
| Handler definitions | Bus configuration |
| Handler code | Handler code repository |
| Handler-creation skill | Bus implementation repository |
| Activity store | Infrastructure as code |
| Demo app: compute, store, OIDC sign-in, its secrets | Infrastructure as code, separate from the bus |
| Dashboard and alerts | Telemetry backends' own provisioning |

## 16. Acceptance criteria

- Pressing the normal button shows the event acked in the demo app, with a trace ID that links to the trace.
- That trace is continuous: ingress endpoint, routing and queue wait, request handler, wait, connection handler, ack call.
- Logs from both handlers for that event are in the telemetry backends and link to the trace.
- The activity store shows pressed, acked, and delivered rows for the event.
- The bad webhook auth button shows a rejection in the demo app, and an error span, metric, and activity row on the bus.
- The bad callback token button shows rejected ack attempts in the demo app; the bus shows error spans and logs per attempt, a dead-letter alert, and a failed activity row.
- Check status returns the event's history through the event-status handler, with a trace that starts in the demo app and includes a span for the activity store read.
- The bad sync token button shows a rejection in the demo app, and a request authorizer error span and metric on the bus.
- Redelivered webhooks and messages cause no duplicate rows or acks.
- With the fault-injection switch off, the bad-token flag has no effect.
- The request handler, connection handler, and event-status handler were created with the handler-creation skill, and implementing one of them loaded only its own reference material.
- The skill's examples, in each chosen language, run as a working unmanaged-consumer worker.

## 17. Design-time choices for this slice

This slice settles these items from the specification's design-time choices (section 22). The demo is judged by what its traces, metrics, and logs show, so the observability items come first.

- **Observability:** telemetry backends, telemetry pipeline, telemetry credentials, trace links, activity store access from dashboards, wait span source, sampling (every press traced), telemetry conventions, dashboards and alerts.
- **Platform and build:** platform options, activity store database, OIDC providers (demo app sign-in), handler definition format, infrastructure as code, repository layout, handler languages, unmanaged consumer hosting, queue settings.

Not in the specification's list, specific to this slice:

- **Demo app host and store**

Not settled in this slice: platform telemetry bridge, task dispatcher, scaling, networking.

## 18. Open items

- Skill file layout and naming
- Optional slow-path option to demonstrate a sync timeout
