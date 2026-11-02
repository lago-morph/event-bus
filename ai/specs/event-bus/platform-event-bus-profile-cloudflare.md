# Platform Event Bus — Cloudflare Profile

**Status:** draft · **Extends:** platform event bus specification

This profile shows that every role and execution class in the specification can be filled on Cloudflare, with at least one option each. It is not a design. Beta and preview services are acceptable on this platform. Every component runs on the public internet behind authentication; Cloudflare private networking is not used.

## 1. Role mapping

| Role | Cloudflare option(s) | Notes |
| --- | --- | --- |
| Ingress endpoint | Worker on an HTTP route | Verifies signatures over the raw body or validates system tokens |
| Event router | Router Worker (added plumbing, section 4) consuming a bus queue | Content-based matching compiled from handler definitions; fans out to handler queues |
| Event archive | Router writes every event to R2; a replay tool republishes (added plumbing) | |
| Handler queue | Cloudflare Queues with a dead-letter queue | Push consumers (Workers) or HTTP pull consumers (anywhere); retries, attempt count, visibility timeout for pull |
| Task dispatcher | Worker using pull-consumer leases (added plumbing) | Holds the lease until the task's completion event |
| Sync gateway | Worker on an HTTP route | Routes to request handlers; containers reached through a Worker |
| Request authorizer | JWT validation in the Worker; Cloudflare Access in front | |
| Activity recorder | Any execution class consuming an all-events queue | |
| Activity store | Postgres via Hyperdrive (for example Neon); D1 | Any queryable database fits |
| Object store | R2 | |
| Control plane | Worker with static assets, behind Cloudflare Access | |
| OIDC provider | Cloudflare Access with an OIDC identity provider (for example Auth0, Google, or Keycloak); any OIDC provider validated directly | |
| Workload identity | Worker bindings to platform resources; scoped API tokens for anything outside Workers (for example pull consumers in containers) | API tokens are kept in the secrets store |
| Secrets store | Workers secrets; Secrets Store | |
| Scheduler | Cron Triggers publishing to the bus; Workflows sleeps for one-off delays | |
| Telemetry pipeline | OTLP sent directly from bus code; platform export of Workers telemetry | Platform bridge: Workers logs and analytics (including queue metrics) exported to the backends |
| Telemetry backends | Grafana with Prometheus or Mimir, Loki, Tempo (self-hosted or Grafana Cloud) | Must be reachable from Cloudflare |
| Infrastructure as code | Terraform (Cloudflare provider); Wrangler | Handler definitions compile to queues, consumers, router rules, and bindings |

## 2. Execution class mapping

| Class | Option(s) | Notes |
| --- | --- | --- |
| Queue-triggered function | Worker as a queue push consumer | Explicit per-message acknowledgment and retry |
| Durable workflow | Workflows, started by a Worker consumer | Steps retried independently |
| Request handler | Worker; a container reached through a Worker | |
| Sandboxed task | Container per task, started by the task dispatcher | Image and instance size chosen at start; filesystem snapshots available (beta) |
| Unmanaged consumer | Any process using the Queues HTTP pull API (pull, acknowledge, retry) | No push consumer configured |

## 3. Capability coverage

| Required capability | How Cloudflare meets it |
| --- | --- |
| Public ingress with raw body access | Worker |
| Content-based routing and fan-out | Router Worker (plumbing) |
| Archive and replay | R2 plus replay tool (plumbing) |
| At-least-once, acknowledgment, visibility timeout, attempt count, DLQ | Queues |
| Per-entity ordering (optional) | Not native; a serializer per entity (for example a Durable Object) if ever needed (plumbing) |
| Hold a message for a task's duration | Pull-consumer lease with a long visibility timeout |
| Sync authorization before handler | Worker JWT validation; Cloudflare Access |
| Scheduled events | Cron Triggers |
| OTLP from bus code | Direct OTLP over HTTP from Workers and containers |
| Platform telemetry into backends | Workers logs and analytics export |
| Workload identity | Bindings; scoped API tokens |

## 4. Added plumbing

| Gap | Plumbing |
| --- | --- |
| No content-based router with fan-out | **Router Worker:** consumes a bus queue, matches compiled rules, sends copies to handler queues |
| No native archive and replay | Router writes events to R2; a replay tool republishes a range |
| Sandboxed tasks must be acknowledged on completion | **Task dispatcher Worker:** pulls with a lease, starts the container, consumes `task.completed` or `task.failed`, then acknowledges or retries the lease |
| No native per-entity ordering (optional) | Per-entity serializer, only if a handler definition asks for it |

## 5. Constraints to carry into design

- Worker CPU and duration limits bound queue-triggered functions and request handlers.
- Queue message size limits apply; large payloads use R2 references.
- Pull-consumer visibility timeouts have a maximum; tasks that may run longer need a design answer.
- Container instances can be stopped by platform events such as host restarts; sandboxed tasks and container request handlers must tolerate restarts.

## 6. Design-time choices

See the specification's design-time choices (section 22). The Cloudflare options for them are in sections 1 and 2.

Decided in this profile:

- **Handler languages:** TypeScript for the handler-creation skill's examples.
