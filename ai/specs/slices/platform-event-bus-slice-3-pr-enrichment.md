# Platform Event Bus — Slice 3: PR Enrichment

**Status:** draft · **Extends:** platform event bus specification · **Scope:** provider-neutral

## 1. Summary and scope

When a pull or merge request is opened, the bus adds reviewers based on the repository and the directories it changes. This is the first real integration: a git host webhook in, and a git host connector out.

- **Read with:** the platform event bus specification and slice 1. This document covers only what this slice adds.
- **Builds on:** slices 1 and 2.

### Not implemented in this slice

- **CI/CD integration**, inbound async events from systems, long external operations reported back by events, cancelling superseded work, and per-entity ordering. These are in slice 6.
- **Platform telemetry bridge** (slice 6).

### Goals

- Prove the inbound and outbound connector pattern against a real external system.
- Prove a direct read from an external system inside a handler.
- Show one trace from the webhook to the reviewers appearing on the pull or merge request.

## 2. Components added

| Component | Role (spec) / execution class | Does |
| --- | --- | --- |
| Git host webhook route | Ingress endpoint | Verifies the git host's webhook signature, de-duplicates by delivery ID, publishes pull or merge request events |
| Reviewer-assignment handler | Worker · unmanaged consumer | Reads the changed files and the reviewer rules directly; publishes the reviewers to add, or a skipped event |
| Git host connector | Connector · unmanaged consumer | Adds the reviewers through the git host's API; publishes delivered or failed |
| Reviewer rules | Configuration | Map repository and directory to reviewers |

## 3. Flow

1. A pull or merge request is opened; the git host sends a signed webhook.
2. The ingress endpoint verifies it and publishes `git.pullrequest.opened`.
3. The reviewer-assignment handler reads the changed files and the reviewer rules with read-only credentials, and publishes `git.reviewers.requested` (or `git.reviewers.skipped` when no rule matches).
4. The git host connector adds the reviewers and publishes `git.reviewers.added`, or `git.reviewers.failed` on the final attempt.
5. The activity recorder records every event.

- The handler subscribes only to opened events, so updates the connector causes do not trigger it again.
- Adding a reviewer who is already present has no effect.

## 4. Event contracts

| Type | Published by | Carries |
| --- | --- | --- |
| `git.pullrequest.opened` | Ingress endpoint | Repository, pull or merge request ID, author |
| `git.reviewers.requested` | Reviewer-assignment handler | Repository, pull or merge request ID, reviewers, matched rules |
| `git.reviewers.skipped` | Reviewer-assignment handler | Repository, pull or merge request ID, reason |
| `git.reviewers.added` | Git host connector | Repository, pull or merge request ID, reviewers, attempt count |
| `git.reviewers.failed` | Git host connector | Repository, pull or merge request ID, attempt count, last error |
| `platform.webhook.rejected` | Ingress endpoint | Source, reason, request ID; no payload |

## 5. Failure scenarios

| Scenario | Outcome |
| --- | --- |
| Bad webhook signature | 401, error span, rejected-webhook metric, `platform.webhook.rejected` row |
| Git host API unavailable or rejects the call | Retries; `git.reviewers.failed` on the final attempt; message in the dead-letter queue (alert) |
| No matching rule | `git.reviewers.skipped` recorded |

## 6. Secrets

| Secret | Held by |
| --- | --- |
| Webhook signing secret | Ingress endpoint |
| Read-only git host token | Reviewer-assignment handler |
| Git host token scoped to adding reviewers | Git host connector only |

## 7. Observability

- **Trace:** ingress endpoint → wait span → reviewer-assignment handler (including spans for its direct reads) → wait span → git host connector → git host API call.
- **Metrics:** pull or merge requests enriched, skipped, and failed; reviewers added; time from webhook to reviewers added.
- **Dashboard:** recent pull or merge requests with outcome and trace links (activity store); the metrics above.
- **Alerts:** dead-lettered messages, rejected webhooks.

## 8. Acceptance criteria

- Opening a pull or merge request that matches a rule adds the expected reviewers, with one continuous trace.
- One that matches no rule records a skipped event and adds no reviewers.
- A bad signature is rejected and recorded.
- A failing git host call ends in a failed event and a dead-letter alert.
- Redelivered webhooks and messages add no duplicate reviewers or rows.
- Both handlers were created with the handler-creation skill.

## 9. Design-time choices for this slice

From the specification's design-time choices (section 22):

- **Webhook signature schemes:** the chosen git host's scheme.
- **Unmanaged consumer hosting**, **queue settings**.

Specific to this slice:

- **Git host and pilot repositories**
- **Reviewer rules:** format and location
