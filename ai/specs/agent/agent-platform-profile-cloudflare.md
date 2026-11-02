# Agent Platform — Cloudflare Profile

**Status:** draft · **Extends:** agent platform specification · **Builds on:** platform event bus Cloudflare profile

This profile shows that every agent-platform role can be filled on Cloudflare, with at least one option each. It is not a design. Bus roles (router, queues, sync gateway, task dispatcher, secrets, telemetry) are mapped in the platform event bus Cloudflare profile and are not repeated here.

## 1. Role mapping

| Role | Cloudflare option(s) | Notes |
| --- | --- | --- |
| Agent handler | Worker as a queue push consumer (queue-triggered function); Workflows (durable workflow); container per task, started by the task dispatcher (sandboxed task) | Class set by the handler definition (agent platform specification, section 5) |
| Inference provider | An external model provider's API (for example OpenAI or Anthropic); Workers AI; an AI gateway (for example LiteLLM, OpenRouter, or Cloudflare AI Gateway) | Chosen per handler. Workers AI is called through a binding; external providers and gateways with a credential from Workers secrets or Secrets Store |
| Question connectors | Any bus execution class (event bus Cloudflare profile) | |
| Interview app | Worker with static assets | Talks only to the sync gateway |
| Interview-read handler, interview-answer handler | Worker; a container reached through a Worker | JWT validation in the Worker, or Cloudflare Access in front |
| Interview handler | Any bus execution class | |
| Interview store | Relational database: D1; or a database through Hyperdrive (for example Neon) | Shared with the activity store or separate |
| Skills | Git repository | |
| Control plane MCP server | Control plane (event bus Cloudflare profile) | Must be a request handler (sync path) |
| Agent session log store | R2, linked from logs in the telemetry backends | |

## 2. Constraints to carry into design

- Worker CPU and duration limits bound agent handlers in the queue-triggered function class; longer work uses Workflows or a container.
- Container instances can be stopped by platform events such as host restarts; agent handlers in containers must tolerate restarts (for example by pushing work in progress to a working branch).
- Containers support Docker in Docker, so builds that need a container runtime can run inside the sandboxed task.
- Inference rate limits apply per provider and account; concurrency caps must respect them.

## 3. Design-time choices

See the agent platform specification's design-time choices (section 15) and the event bus specification's (section 22). The Cloudflare options for them are in section 1 of this profile and of the event bus Cloudflare profile.
