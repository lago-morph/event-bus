# Platform Event Bus — Slice 6: Remaining Components (provisional)

**Status:** provisional · **Extends:** platform event bus specification and agent platform specification · **Scope:** provider-neutral

## 1. Summary and scope

This slice implements every component of both layers not built in slices 1 to 5. It is a component list only. The use case that exercises them is to be identified, and the list may change as slices 1 to 5 are built.

- **Read with:** the platform event bus specification and the agent platform specification.
- **Builds on:** slices 1 to 5.

## 2. Event bus components

| Component | Notes |
| --- | --- |
| CI/CD connector | Starts and aborts pipelines with trace context |
| Inbound async events from systems | A system calls the ingress endpoint with a system token (for example a pipeline step reporting its result) |
| Long external operations reported back by events | For example a pipeline run started by the connector and completed by its status event |
| Cancelling superseded work | For example aborting an older pipeline run when a newer one starts |
| Per-entity ordering | Optional; only if a handler definition asks for it |
| Event archive and replay | Replay of a time range or filter back through the router |
| Scheduler | Cron-style and one-off scheduled events |
| Notifications | At least one notification method |
| Control plane | Web app, API, and MCP server (a request handler) over the activity store |
| Manual actions as events | Rerun, reroute, dead-letter redrive |
| OIDC authorization for people on sync routes | Request authorizer validating OIDC tokens |
| Platform telemetry bridge | Platform-native logs and metrics into the telemetry backends, including queue depth and dead-letter counts |

## 3. Agent platform components

| Component | Notes |
| --- | --- |
| Interview app | OIDC sign-in with restricted registration |
| Interview-read and interview-answer handlers | Request handlers behind the sync gateway |
| Interview handler and interview store | Records questions and answers |
| Questions on pull or merge requests | Git host review comments as a question channel |
| Skills run headless | The same skill in an interactive session and in an agent handler |
| Agents using the control plane MCP server | Agent token |
| Agent session log analysis and replay tooling | Requirements need elicitation |

## 4. Open items

- Use case or use cases that exercise these components
- Whether to split this slice
