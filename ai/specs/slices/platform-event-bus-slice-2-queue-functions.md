# Platform Event Bus — Slice 2: Button Demo with Queue-Triggered Functions

**Status:** draft · **Extends:** platform event bus specification · **Scope:** provider-neutral

## 1. Summary and scope

This slice adds a second implementation of the button demo's async handlers: a worker and a connector in the queue-triggered function class. The demo app's implementation dropdown gains a **queue-triggered function** option, and each press runs through the implementation it selects.

- **Read with:** the platform event bus specification and slice 1. This document covers only what this slice adds.
- **Builds on:** slice 1, including its unmanaged consumers, event-status request handler, and handler-creation skill.

### Not implemented in this slice

- **Other execution classes.** Durable workflow and sandboxed task come in slice 4.
- **Platform telemetry bridge.** Platform-native logs and metrics stay in the platform's own logging until slice 6.
- **New sync routes.** The request handler class was exercised in slice 1.

### Goals

- Prove the queue-triggered function class for a worker and a connector.
- Keep slice 1's unmanaged-consumer path working unchanged alongside the new one.
- Extend the handler-creation skill to queue-triggered functions without loading material for other classes.

## 2. Components added

| Component | Role (spec) / execution class | Does |
| --- | --- | --- |
| Request handler (function) | Worker · queue-triggered function | Consumes pressed events selected for this implementation; publishes acked events |
| Demo app connection handler (function) | Connector · queue-triggered function | Consumes acked events selected for this implementation; calls the demo app's ack API; publishes delivered or failed |
| Handler-creation skill | Bus implementation repository | Gains queue-triggered worker and connector material |

The activity recorder, ingress endpoint, request authorizer, event-status handler, and demo app are unchanged from slice 1.

## 3. Queues and routing rules

| Queue | Consumer | Fed by rules matching | Dead-letter after |
| --- | --- | --- | --- |
| request-handler-function | Request handler (function) | `demo.button.pressed` with implementation `queue-function` | Maximum attempts reached |
| connection-handler-function | Connection handler (function) | `demo.button.acked` with implementation `queue-function` | Maximum attempts reached |

- Slice 1's request-handler and connection-handler rules gain the condition "implementation is `unmanaged`".
- `demo.button.acked` carries the selected implementation forward so the connection handler for the same implementation receives it.

## 4. Flow and failure scenarios

- **Normal flow:** the same as slice 1, through the function handlers.
- **Four buttons:** the same as slice 1. The bad callback token scenario ends the same way: rejected attempts, `demo.ack.failed` on the final attempt, and the message in the dead-letter queue.
- **Per-message failure:** a failed message is retried on its own; other messages in the same delivery are acknowledged.

## 5. Handler-creation skill

- **Added:** queue-triggered function worker and connector.
- **Progressive disclosure:** implementing a queue-triggered function loads only that class's material for the chosen platform and language.

## 6. Observability

- Follows slice 1. Handler telemetry is sent directly over OTLP.
- **Dashboard:** slice 1's panels, split by implementation.

## 7. Acceptance criteria

- Selecting **queue-triggered function** and pressing normal shows the event acked, with one continuous trace through the function handlers.
- The unmanaged-consumer implementation still works unchanged.
- The bad callback token button on the function path produces rejected attempts, a dead-letter alert, and a failed activity row.
- Redelivered messages cause no duplicate rows or acks.
- The function handlers were created with the handler-creation skill, and implementing them loaded only queue-triggered function material.

## 8. Design-time choices for this slice

From the specification's design-time choices (section 22):

- **Platform options:** queue-triggered function.
- **Queue settings:** visibility timeout above the function's timeout.
