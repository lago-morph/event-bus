# Platform Event Bus — Slice 5: Enhancement Issue Impact Analysis

**Status:** draft · **Extends:** platform event bus specification and agent platform specification · **Scope:** provider-neutral

## 1. Summary and scope

An agent posts a rough impact analysis on enhancement issues: which parts of the application's code would have to change, with attached evidence. It asks clarifying questions in the same comment and reacts to later comments.

- **Triggers:** an issue is created with the enhancement label, or a comment is added to an issue with the enhancement label.
- **Each run:**
  1. Reads the whole issue, including comments.
  2. Checks whether it has already posted an analysis. If it has, it decides whether the new comments or changes call for a new one.
  3. If no analysis exists, or a new one is needed, it determines the parts of the code that would change (rough, not thorough) and posts a comment with attached evidence.

This is the first agent handler, the first issue tracker connector, and the first real use of the sandboxed task class.

- **Read with:** the platform event bus specification, the agent platform specification, and slices 1 and 4. This document covers only what this slice adds.
- **Builds on:** slices 1 to 4.

### Not implemented in this slice

- **Interview app and bus notifications** (slice 6). Questions go only to the issue.
- **Platform telemetry bridge** (slice 6).

### Goals

- Prove an agent handler on the bus, running as a sandboxed task.
- Prove stateless handling: all state is read from the issue each time.
- Prove the question-and-reply loop through issue comments.
- Prove large payloads by reference (attached evidence).

## 2. Components added

| Component | Role (spec) / execution class | Does |
| --- | --- | --- |
| Issue tracker webhook route | Ingress endpoint | Verifies the issue tracker's webhook signature; publishes issue-created and comment-created events with the issue's labels |
| Impact analysis agent | Agent handler · sandboxed task | Reads the issue and its comments; decides whether to analyze; checks out the repository; determines affected code; requests a comment with evidence |
| Issue tracker connector | Connector · any async class | The only writer to the issue tracker: posts comments and attachments; publishes delivered or failed |
| Session log store | Agent session log store | Holds the agent's session logs, linked to the trace |

The task dispatcher from slice 4 starts the agent's sandboxed tasks.

## 3. Flow

1. The issue tracker sends a signed webhook for an issue created or a comment added.
2. The ingress endpoint verifies it and publishes `issues.issue.created` or `issues.comment.created`. Routing rules deliver only events for issues with the enhancement label.
3. The task dispatcher starts the impact analysis agent as a sandboxed task.
4. The agent reads the whole issue directly. It finds its own previous analysis, if any, and decides:
   - no previous analysis → analyze,
   - previous analysis and the new comments or changes call for one → analyze again,
   - otherwise → publish `impact.analysis.skipped`.
5. To analyze, the agent checks out the repository, determines the affected code, writes the evidence to the object store, and publishes `issues.comment.requested` with the comment text, any clarifying questions, and a reference to the evidence.
6. The issue tracker connector posts the comment and attachment and publishes `issues.comment.delivered`, or `issues.comment.failed` on the final attempt.
7. The task publishes `task.completed`; the dispatcher acknowledges the original message.

- **Loop prevention:** comments by the bus's own identity are ignored.
- **Questions and replies:** a person's reply is a new comment, so it starts a new run that reads the whole issue again.

## 4. Event contracts

| Type | Published by | Carries |
| --- | --- | --- |
| `issues.issue.created` | Ingress endpoint | Issue ID, labels, author |
| `issues.comment.created` | Ingress endpoint | Issue ID, labels, comment ID, author |
| `impact.analysis.skipped` | Impact analysis agent | Issue ID, reason |
| `impact.analysis.failed` | Task dispatcher (final attempt) | Issue ID, attempt count, last error |
| `issues.comment.requested` | Impact analysis agent | Issue ID, comment text, evidence reference |
| `issues.comment.delivered` | Issue tracker connector | Issue ID, comment ID, attempt count |
| `issues.comment.failed` | Issue tracker connector | Issue ID, attempt count, last error |

## 5. Failure scenarios

| Scenario | Outcome |
| --- | --- |
| Bad webhook signature | 401, error span, rejected-webhook metric, `platform.webhook.rejected` row |
| Inference provider unavailable | Task fails and is retried; `impact.analysis.failed` on the final attempt; dead-letter alert |
| Task exceeds its time limit | Treated as failed by the dispatcher |
| Issue tracker rejects the comment | `issues.comment.failed` on the final attempt; dead-letter alert |
| The agent's own comment arrives as a webhook | Ignored (loop prevention) |

## 6. Secrets

| Secret | Held by |
| --- | --- |
| Webhook signing secret | Ingress endpoint |
| Read-only issue tracker token | Impact analysis agent |
| Read-only repository credentials | Impact analysis agent |
| Inference credential | Impact analysis agent |
| Issue tracker write token | Issue tracker connector only |

## 7. Observability

- **Trace:** ingress endpoint → wait span → dispatcher → task → agent spans (inference, direct reads, checkout) → wait span → connector → issue tracker API call.
- **Metrics:** analyses run, skipped, and failed; inference calls, tokens, and cost per analysis; time from trigger to comment.
- **Session logs:** stored in the session log store and linked to the trace, for the inference provider and agent SDK this slice uses.
- **Dashboard:** recent issues with outcome, cost, and trace links (activity store); the metrics above.
- **Alerts:** dead-lettered messages, rejected webhooks, cost threshold.

## 8. Acceptance criteria

- Creating an enhancement issue produces an analysis comment with attached evidence, and one continuous trace.
- A comment on an analyzed issue that changes its meaning produces a new analysis; one that doesn't produces a skipped event.
- Issues without the enhancement label are ignored.
- The agent's own comments do not trigger new runs.
- A reply to a clarifying question is read on the next run.
- An inference outage ends in a failed event and a dead-letter alert, with no partial comment.
- Redelivered webhooks and messages cause no duplicate comments.
- Session logs for each run are stored and linked to its trace.
- The agent handler was created with the handler-creation skill.

## 9. Design-time choices for this slice

From the event bus specification's design-time choices (section 22): **webhook signature schemes** (the chosen issue tracker's scheme), **queue settings**.

From the agent platform specification's design-time choices (section 15): **concurrency caps**, **agent session logs**.

Specific to this slice:

- **Issue tracker and pilot projects**
- **Repository for each project**
- **Inference provider, agent SDK or headless coding agent** for this handler
- **Cost alert threshold**
