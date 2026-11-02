# Agent Platform — AWS Profile

**Status:** draft · **Extends:** agent platform specification · **Builds on:** platform event bus AWS profile

This profile shows that every agent-platform role can be filled on AWS, with at least one option each. It is not a design. Bus roles (router, queues, sync gateway, task dispatcher, secrets, telemetry) are mapped in the platform event bus AWS profile and are not repeated here.

## 1. Role mapping

| Role | AWS option(s) | Notes |
| --- | --- | --- |
| Agent handler | Lambda (queue-triggered function); Step Functions (durable workflow); ECS task on Fargate, or a Kubernetes Job on EKS as a design-time option (sandboxed task) | Class set by the handler definition (agent platform specification, section 5) |
| Inference provider | Amazon Bedrock; an external model provider's API (for example OpenAI or Anthropic); an AI gateway (for example LiteLLM, OpenRouter, or Bedrock intelligent prompt routing) | Chosen per handler. Bedrock is called with IAM; external providers and gateways with a credential from Secrets Manager |
| Question connectors | Any bus execution class (event bus AWS profile) | |
| Interview app | Static site in S3 behind CloudFront | Talks only to the sync gateway |
| Interview-read handler, interview-answer handler | Lambda behind API Gateway; ECS service behind VPC Link | API Gateway JWT authorizer validates the OIDC token |
| Interview handler | Any bus execution class | |
| Interview store | Relational database: Amazon RDS or Aurora; or Neon | Shared with the activity store or separate |
| Skills | Git repository | |
| Control plane MCP server | Control plane (event bus AWS profile) | Must be a request handler (sync path) |
| Agent session log store | S3, linked from logs in the telemetry backends | |

## 2. Constraints to carry into design

- Lambda's maximum run time bounds agent handlers in the queue-triggered function class; longer work uses Step Functions or a Fargate task.
- Fargate does not support privileged containers, so an agent handler on Fargate cannot run Docker in Docker. Builds that need a container runtime run on the CI/CD system, or on ECS on EC2 or EKS with a design that allows it.
- Bedrock model availability and quotas vary by region.
- Inference rate limits apply per provider and account; concurrency caps must respect them.

## 3. Design-time choices

See the agent platform specification's design-time choices (section 15) and the event bus specification's (section 22). The AWS options for them are in section 1 of this profile and of the event bus AWS profile.
