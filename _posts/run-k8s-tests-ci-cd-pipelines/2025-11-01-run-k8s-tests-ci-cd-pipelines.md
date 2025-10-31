---
title: Running Kubernetes Tests in CI/CD Pipelines
date: 2025-11-01
tags: [infra, debugging, docker, pipelines, ]
description: Want to know at what time text got added or remoted in your entire git history? 
---

We recently tackled an interesting networking challenge while building our GitHub Actions CI/CD pipeline for testing Kubernetes adapters. I thought I'd share the problem, the root cause, and the elegant solution we implemented.

The Scenario

Our setup involved:

1. A PHP test container running on a Docker Compose orchestration network

2. A KinD (Kubernetes in Docker) cluster on its own isolated kind network

3. Tests that needed to communicate with the Kubernetes API server

Simple enough, right? Wrong. 😅

**The Problem**

Our K8s adapter tests were timing out when trying to reach the Kubernetes API server. The container could successfully run on the orchestration network with other services, but kubectl couldn't authenticate with the cluster. The kubeconfig was correctly mounted, kubectl was installed, but the connection refused.

After digging through logs, we realized: the networks were completely isolated. Docker's bridge networking creates firewall-like boundaries between networks. Services on the orchestration network have zero visibility into the kind network, and vice versa.

**Why This Matters**

This is actually a security feature! Docker's network isolation prevents:

- Accidental cross-service communication

- Services interfering with each other

- Sprawling network dependencies

But in our case, we needed cross-network communication for our tests to work.

**The Solution**

Here's where Docker's flexibility shines. A single container can be connected to multiple networks simultaneously. This is exactly what we needed.

In our GitHub Actions workflow, right after starting the container with docker compose up -d, we added:

```bash
# Get the test container ID
`CONTAINER_ID=$(docker compose ps -q tests)`

# Connect it to the KinD network
`docker network connect kind "$CONTAINER_ID"`
```

Now our test container had network access to both:

The orchestration network—for inter-service communication with other containers

The kind network—for accessing the Kubernetes API server

The Implementation

Our updated workflow step:

```yaml
- name: Start test container
  run: |
    docker compose up -d && sleep 15
    
    # Connect container to KinD network for K8s access
    CONTAINER_ID=$(docker compose ps -q tests)
    docker network connect kind "$CONTAINER_ID"

- name: Setup Docker network for KinD access
  run: |
    # Make kubeconfig accessible to containers
    kind get kubeconfig --internal > /tmp/kind-kubeconfig

- name: Verify connectivity
  run: |
    docker compose exec -T tests kubectl cluster-info

- name: Run K8s tests
  run: |
    docker compose exec -T tests vendor/bin/phpunit tests/K8sCLITest.php
```

**Key Insights**

Docker networks aren't monolithic—containers can be part of multiple networks

Isolation is the default—you have to explicitly connect networks

Multi-network architecture enables flexible CI/CD patterns

Simple shell commands in your workflow can solve complex networking challenges


**Results**

- ✅ Tests successfully connect to KinD cluster
- ✅ No network timeouts or connection refused errors
- ✅ Clean separation of concerns
- ✅ Maintainable, debuggable CI/CD pipeline

If you're building CI/CD pipelines with Docker, Kubernetes, and GitHub Actions, I hope this helps! Drop a comment if you've encountered similar challenges.