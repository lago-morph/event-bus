# Platform Event Bus — Slice 4: Button Demo with Durable Workflows and Sandboxed Tasks

**Status:** draft · **Extends:** platform event bus specification · **Scope:** provider-neutral

## 1. Summary and scope

This slice adds two more implementations of the button demo's async handlers, and the task dispatcher that sandboxed tasks need:

- a worker and a connector in the **durable workflow** class,
- a worker and a connector in the **sandboxed task** class, started by the task dispatcher.

The demo app's implementation dropdown gains **durable workflow** and **sandboxed task**.

- **Read with:** the platform event bus specification (especially sandboxed task dispatch) and slices 1 and 2. This document covers only what this slice adds.
- **Builds on:** slices 1 to 3.

### Not implemented in this slice

- **Platform telemetry bridge** (slice 6).
- **New sync routes.** The request handler class was exercised in slice 1.

### Goals

- Prove the durable workflow class, with steps retried independently.
- Build the task dispatcher and prove that a sandboxed task's message is acknowledged only when the task completes.
- Extend the handler-creation skill to both classes.

## 2. Components added

| Component | Role (spec) / execution class | Does |
| --- | --- | --- |
| Request handler (workflow) | Worker · durable workflow | Acknowledges the press as a workflow step; publishes acked events |
| Connection handler (workflow) | Connector · durable workflow | Calls the demo app's ack API as a retried step; publishes delivered or failed |
| Task dispatcher | Task dispatcher | Starts a sandboxed task per message, holds the message until the task completes, then acknowledges or releases it |
| Request handler (task) | Worker · sandboxed task | Acknowledges the press inside a sandboxed task; publishes acked events and `task.completed` |
| Connection handler (task) | Connector · sandboxed task | Calls the demo app's ack API inside a sandboxed task; publishes delivered and `task.completed`, or `task.failed` |
| Handler-creation skill | Bus implementation repository | Gains durable workflow and sandboxed task material |

## 3. Queues and routing rules

| Queue | Consumer | Fed by rules matching |
| --- | --- | --- |
| request-handler-workflow | Request handler (workflow) | `demo.button.pressed` with implementation `workflow` |
| connection-handler-workflow | Connection handler (workflow) | `demo.button.acked` with implementation `workflow` |
| request-handler-task | Task dispatcher | `demo.button.pressed` with implementation `sandboxed-task` |
| connection-handler-task | Task dispatcher | `demo.button.acked` with implementation `sandboxed-task` |
| task-dispatcher | Task dispatcher | `task.completed`, `task.failed` |

Each queue has a dead-letter queue after maximum attempts.

## 4. Flow and failure scenarios

- **Durable workflow:** the same flow as slice 1. The ack call is a workflow step retried on its own; when retries are exhausted, the workflow publishes `demo.ack.failed` and the message moves to the dead-letter queue.
- **Sandboxed task:** sandboxed task dispatch as the specification describes:
  1. The dispatcher records the dispatch in the activity store and starts a task with the event as input.
  2. The task does its work and publishes `task.completed` or `task.failed`.
  3. The dispatcher acknowledges or releases the original message. On the final attempt it publishes `demo.ack.failed` for the connection handler, and the message moves to the dead-letter queue.
  4. A task with no completion within its time limit is treated as failed.
- **Four buttons:** the same outcomes as slice 1 on both new paths.

## 5. Event contracts added

| Type | Published by | Carries |
| --- | --- | --- |
| `task.completed` | Sandboxed task | Task ID, demo event ID |
| `task.failed` | Sandboxed task | Task ID, demo event ID, error |

## 6. Observability

- **Workflow trace:** wait span → workflow → one span per step, including retries.
- **Task trace:** wait span → dispatcher → task start → task spans → completion event → dispatcher acknowledgment.
- **Metrics:** task starts, completions, failures, timeouts, and duration; workflow step retries.
- **Dashboard:** slice 1's panels split by implementation, plus task and workflow panels.

## 7. Acceptance criteria

- Selecting **durable workflow** and pressing normal shows the event acked, with one continuous trace including step spans.
- Selecting **sandboxed task** and pressing normal shows the event acked, with one continuous trace through the dispatcher and the task.
- The original message for a sandboxed task is acknowledged only after `task.completed` arrives.
- The bad callback token button on both new paths ends in rejected attempts, a failed activity row, and a dead-letter alert.
- Earlier implementations still work unchanged.
- The new handlers were created with the handler-creation skill, and each loaded only its own class's material.

## 8. Design-time choices for this slice

From the specification's design-time choices (section 22):

- **Platform options:** durable workflow, sandboxed task.
- **Task dispatcher:** how it is invoked and where dispatch records are kept.
- **Queue settings:** visibility must outlast the task.

## 9. Open items

- Optional fault to demonstrate a task timeout
