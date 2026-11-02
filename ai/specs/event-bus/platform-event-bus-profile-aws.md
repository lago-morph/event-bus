# Platform Event Bus — AWS Profile

**Status:** draft · **Extends:** platform event bus specification

This profile shows that every role and execution class in the specification can be filled on AWS, with at least one option each. It is not a design. Choosing among options, sizing, and configuration belong to the design phase.

## 1. Role mapping

| Role | AWS option(s) | Notes |
| --- | --- | --- |
| Ingress endpoint | API Gateway route → Lambda | Lambda verifies signatures over the raw body or validates system tokens |
| Event router | EventBridge bus with rules | Content-based matching and fan-out to SQS targets |
| Event archive | EventBridge archive and replay | Native |
| Handler queue | SQS queue with a dead-letter queue | Visibility timeout, receive count, redrive policy |
| Task dispatcher | Lambda (added plumbing, section 4) | Holds the SQS message until the task's completion event |
| Sync gateway | API Gateway | Routes to Lambda, to a private service via VPC Link, or to Step Functions Express |
| Request authorizer | API Gateway JWT authorizer (OIDC); Lambda authorizer for system tokens | |
| Activity recorder | Any execution class consuming an all-events queue | |
| Activity store | Postgres: Amazon RDS or Aurora; or Neon | Any queryable database fits |
| Object store | S3 | |
| Control plane | ECS service or Lambda, with a web front end | Reached through OIDC sign-in, on an authenticated public endpoint or privately (optional private networking: VPC, VPC Link) |
| OIDC provider | Keycloak (self-hosted); any OIDC provider | |
| Workload identity | IAM roles (task roles, execution roles) | |
| Secrets store | AWS Secrets Manager | |
| Scheduler | EventBridge Scheduler | Publishes to the bus |
| Telemetry pipeline | OpenTelemetry collector (for example AWS Distro for OpenTelemetry) | Platform bridge: CloudWatch Logs and metrics exported to the backends; with CloudWatch as the backend, the bridge is mostly unnecessary |
| Telemetry backends | AWS-native: CloudWatch Logs, metrics, and dashboards; X-Ray or CloudWatch Application Signals; Amazon Managed Service for Prometheus; Amazon Managed Grafana. Or a Grafana stack: Grafana with Prometheus or Mimir, Loki, Tempo (self-hosted or Grafana Cloud) | |
| Infrastructure as code | Terraform; AWS CDK or CloudFormation | Handler definitions compile to rules, queues, policies, and bindings |

## 2. Execution class mapping

| Class | Option(s) | Notes |
| --- | --- | --- |
| Queue-triggered function | Lambda with an SQS event source mapping | Per-message failure reporting |
| Durable workflow | Step Functions (Standard), started from the queue | For example via EventBridge Pipes or a Lambda |
| Request handler | Lambda; a service behind VPC Link (ECS or EKS); Step Functions Express (short only) | |
| Sandboxed task | ECS task on Fargate, started by the task dispatcher; Kubernetes Job on EKS (design-time option) | One task per message |
| Unmanaged consumer | Any process using the SQS ReceiveMessage and DeleteMessage APIs | No event source mapping or other managed consumer |

## 3. Capability coverage

| Required capability | How AWS meets it |
| --- | --- |
| Public ingress with raw body access | API Gateway → Lambda |
| Content-based routing and fan-out | EventBridge rules |
| Archive and replay | EventBridge archive |
| At-least-once, acknowledgment, visibility timeout, attempt count, DLQ | SQS |
| Per-entity ordering (optional) | SQS FIFO with message group IDs |
| Hold a message for a task's duration | SQS visibility timeout, extendable while a message is in flight |
| Sync authorization before handler | API Gateway authorizers |
| Scheduled events | EventBridge Scheduler |
| OTLP from bus code | OpenTelemetry SDKs → collector |
| Platform telemetry into backends | CloudWatch Logs and metrics, bridged to the backends |
| Workload identity | IAM roles |

## 4. Added plumbing

| Gap | Plumbing |
| --- | --- |
| Native queue-to-container triggers acknowledge on task launch, not completion | **Task dispatcher Lambda:** receives the SQS message without an automatic delete, starts the Fargate task, extends visibility while it runs, consumes `task.completed` or `task.failed`, then deletes the message or releases it for retry |

## 5. Constraints to carry into design

- API Gateway's maximum integration timeout bounds every sync request handler.
- Event and message size limits apply to EventBridge and SQS; large payloads use S3 references.
- SQS visibility timeouts have a maximum; long sandboxed tasks must extend visibility while running.
- Lambda logs and platform crash information appear only in CloudWatch Logs, so the telemetry bridge is required when the backends are not CloudWatch.

## 6. Design-time choices

See the specification's design-time choices (section 22). The AWS options for them are in sections 1 and 2.
