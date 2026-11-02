# Platform Event Bus — Integration Profiles

**Status:** draft · **Extends:** platform event bus specification

Integration profiles list example external services for each integration category and confirm they fit the connector pattern. They are deliberately light: they are not designs and not recommendations.

## What "compatible" means

A service fits the bus if it offers what its category needs from this list:

- **API** for mutations (called by its connector) and reads (direct, read-only).
- **Outbound events** about changes, delivered as webhooks to the ingress endpoint, or through a platform event mechanism that a small forwarder publishes to the bus.
- **Verifiable delivery:** a webhook signature or shared secret the ingress endpoint can check.
- **Sync-friendly reads:** read calls that complete well within a request handler's timeout; long operations are started asynchronously and report back through events.
- **Standard protocols** where the category has one (for example OIDC for identity providers, the S3 API for object storage).

## Categories

| Category | Example services | API | Outbound events | Notes |
| --- | --- | --- | --- | --- |
| Git hosting | GitHub, GitLab, Bitbucket | Yes | Webhooks (push, pull or merge requests, comments) | Each offers webhook verification (signature or secret token) |
| CI/CD | GitHub Actions, GitLab CI/CD, Bitbucket Pipelines, Jenkins, CircleCI | Yes: start a pipeline or workflow on demand, independent of git changes | Run status via webhooks, or a final pipeline step that calls the bus's webhook route | Git-host events can also trigger pipelines natively; the bus still sees the runs through their status events |
| Issue tracking | Jira, GitHub Issues, GitLab Issues, Bitbucket issues | Yes | Webhooks (issue created or updated, comments, labels, sprint changes where supported) | |
| Object storage | Amazon S3, Cloudflare R2 | Yes (S3 API) | Change notifications go to the platform's messaging (for example S3 to EventBridge or SQS; R2 to Queues), not to webhooks | A forwarder publishes them to the bus; on the same platform this can feed the router directly |
| Notification methods (optional) | Email, Slack, Discord | Yes (send messages) | Inbound events where supported | Used by the bus's notifications |
| Database | Neon and other Postgres services; Amazon RDS; Amazon DynamoDB; Cloudflare D1; any database reachable over the network | Yes (SQL drivers, HTTP drivers, service APIs) | Not required | Used for handler data and direct reads; a queryable database also serves as the activity store |
| OIDC identity | Auth0, Google, Cloudflare Access, Keycloak | OIDC discovery and token endpoints | Not required | Standard OIDC; tokens validated by the request authorizer and control plane |

## Per-category connector shape

| Category | Inbound (async) | Outbound (connector) | Direct reads |
| --- | --- | --- | --- |
| Git hosting | Webhook → ingress | Comments, labels, statuses or checks | Repository contents, pull or merge request details |
| CI/CD | Status webhook → ingress | Start a pipeline or workflow with inputs and trace context | Run status and logs |
| Issue tracking | Webhook → ingress | Comments, labels, transitions | Issue fields and comments |
| Object storage | Change notification → forwarder → router | Writes | Reads |
| Database | Not applicable | Writes by the owning handler | Reads |
| OIDC identity | Not applicable | Not applicable | Token validation and discovery |
