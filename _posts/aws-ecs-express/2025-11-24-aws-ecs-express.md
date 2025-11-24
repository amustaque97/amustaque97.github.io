---
title: AWS Releases ECS Express
date: 2025-11-24
tags: [aws, containers, devops]
description: a simpler way to run containerized web services
---

AWS released AWS ECS Express recently, a couple of days ago, a new capability from Amazon Elastic Container Service (Amazon ECS) that helps you launch highly available, scalable containerized applications with a single command. ECS Express Mode automates infrastructure setup including domains, networking, load balancing, and auto scaling through simplified APIs. This means you can focus on building applications while deploying with confidence using Amazon Web Services (AWS) best practices. Furthermore, when your applications evolve and require advanced features, you can seamlessly configure and access the full capabilities of the resources, including Amazon ECS.

This new offering is aimed at teams who want to run containerized applications without wiring up the full set of ECS primitives themselves. Express handles provisioning of Application Load Balancers, target groups, security groups, task definitions, and autoscaling policies for you, and performs rolling (zero-downtime) updates when service configuration changes.

## Why this matters

- Faster onboarding: developers can describe a single `primary_container` and minimal configuration, and the service provisions the rest. No hand-rolling ALBs and target groups for each app.
- Zero-downtime updates: built-in rolling deployments mean safer configuration changes and smoother releases.
- Built-in best-practices: useful defaults for health checks, logging, and autoscaling remove a lot of boilerplate and potential misconfiguration.

## Key features

- Declarative service provisioning that creates an ECS service, task definition, ALB (with ingress), target groups, and autoscaling.
- Support for container logging (CloudWatch), environment variables, secrets from Secrets Manager / SSM, and custom health check paths.
- Network configuration for `awsvpc` tasks with subnets and security groups.
- Autoscaling based on CPU or memory with configurable min/max task counts.
- `wait_for_steady_state` and configurable timeouts for create/update/delete operations.

## Quick Terraform example

Here is a minimal example showing how to use the resource (note the `time_sleep` caveat below):

```terraform
resource "time_sleep" "wait_for_iam" {
  depends_on      = [aws_iam_role_policy_attachment.infrastructure]
  create_duration = "7s"
}

resource "aws_ecs_express_gateway_service" "example" {
  execution_role_arn       = aws_iam_role.execution.arn
  infrastructure_role_arn  = aws_iam_role.infrastructure.arn

  primary_container {
    image          = "nginx:stable"
    container_port = 80
  }

  depends_on = [time_sleep.wait_for_iam]
}
```

## IAM Role Timing (important)

If you create IAM roles and then immediately reference them in the same Terraform apply, AWS eventual consistency can cause failures (the new role may not be fully propagated). To reduce failures:

- Add a small `time_sleep` resource (example above) and make your Express service depend on it; or
- Split your changes into two `apply` steps (create roles first, then services); or
- Implement an explicit polling/wait mechanism for role propagation in your automation.

A short 5–10 second wait typically resolves propagation issues in CI systems.

## When to use Express vs full ECS control

Use Express when:
- You want a fast, low-friction way to run a single container web service with sensible defaults.
- You prefer managing high-level service configuration and letting AWS handle the infra details.

Prefer full ECS (manual task definitions, service, ALB, target groups) when:
- You need complex multi-container task definitions, sidecars, or very custom networking/security requirements.
- You need fine-grained control over every part of the deployment stack.

Blog link: https://aws.amazon.com/blogs/aws/build-production-ready-applications-without-infrastructure-complexity-using-amazon-ecs-express-mode/