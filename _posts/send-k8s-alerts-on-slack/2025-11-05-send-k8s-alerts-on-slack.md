---
title: Send Alerts to Slack/Discord When a Kubernetes Pod Restarts
date: 2025-11-05
tags: [infra, debugging, kubernetes, alerts ]
description: How to Send Alerts to Slack/Discord When a Kubernetes Pod Restarts Using Grafana including Kubernetes Events and Recent Pod Logs
---

## Main Takeaway

Monitor Kubernetes pod restarts using Grafana and Prometheus, then enrich your Slack/Discord alerts with real-time Kubernetes events and recent pod logs by leveraging custom webhook payloads and integration with Loki. This comprehensive guide provides production-ready implementation steps.



## Introduction

Effective Kubernetes incident response requires not just detecting pod restarts, but understanding *why* they occurred. Combining pod restart metrics with Kubernetes cluster events and application logs provides operators with complete context for rapid troubleshooting. This technical deep-dive covers:

- Monitoring pod restarts via Prometheus and kube-state-metrics
- Capturing Kubernetes events using event exporters
- Aggregating pod logs with Grafana Loki
- Crafting rich Slack/Discord webhook payloads that include events and logs
- Building a complete alerting workflow from detection to notification



## Part 1: Foundational Monitoring Setup

### Step 1: Deploy kube-state-metrics and Prometheus

Deploy kube-state-metrics to expose pod restart counts:

```bash
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/latest/download/kube-state-metrics.yaml
```

Verify that Prometheus scrapes the metric:

```
kube_pod_container_status_restarts_total{namespace="default",pod="nginx-deployment-66b6c48dd5-abc123",container="nginx"}
```



## Part 2: Collecting Kubernetes Events

Kubernetes events provide critical context about pod state changes. They're ephemeral (default TTL: 1 hour) and stored in etcd, so exporting them is essential for long-term analysis.

### Understanding Kubernetes Events

Kubernetes events capture state transitions:

- **Pod Created**: When a pod is scheduled
- **Pod Failed**: When a container exits with non-zero status
- **BackOff**: When restarts exceed retry limits
- **Killing**: When a pod is terminated
- **Liveness Probe Failed**: When health checks fail

Each event contains:

- **Reason**: Event type (e.g., `Backoff`, `Failed`)
- **Message**: Human-readable description
- **Count**: How many times the event occurred
- **Source**: Which component reported (e.g., `kubelet`, `kube-controller-manager`)
- **Timestamp**: When the event occurred

### Deploy Event Exporter

Use **kubernetes-event-exporter** (open-source) or **kube-events** to bridge Kubernetes events to Prometheus:

```bash
# Add Helm repository
helm repo add resmoio https://resmoio.github.io/helm-charts
helm repo update

# Install event exporter
helm install event-exporter resmoio/kubernetes-event-exporter \
  --namespace monitoring \
  --set logLevel=info
```

The event exporter exposes these metrics:

```
kube_event_count{
  involved_object_kind="Pod",
  involved_object_name="my-pod",
  involved_object_namespace="default",
  reason="Backoff",
  type="Warning"
}

kube_event_unique_events_total{...}
```

### Query Recent Events for a Pod

In Prometheus, query pod-specific events:

```promql
kube_event_count{involved_object_name=~"my-pod.*",involved_object_kind="Pod"}
```



## Part 3: Aggregating Pod Logs with Grafana Loki

Loki indexes logs by label, not by content, making it ideal for Kubernetes log aggregation.

### Deploy Loki and Alloy (Log Collector)

```bash
# Add Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki Stack (includes Loki and Alloy)
helm install loki grafana/loki-stack \
  --namespace loki \
  --create-namespace \
  --set loki.persistence.enabled=true \
  --set promtail.enabled=true
```

### Configure Loki Datasource in Grafana

1. Navigate to **Configuration → Data Sources**
2. Add **Loki** as a datasource
3. Set URL to `http://loki:3100` (adjust for your setup)

### Query Pod Logs in LogQL

Query recent logs for a restarted pod:

```logql
{namespace="default",pod="nginx-deployment-66b6c48dd5-abc123"} | json
```

Extract error-level logs:

```logql
{namespace="default",pod="nginx-deployment-66b6c48dd5-abc123"} | json level="error"
```

Limit to last 100 lines:

```logql
{namespace="default",pod="nginx-deployment-66b6c48dd5-abc123"} | line_format "{{ .message }}" | limit 100
```



## Part 4: Creating Rich Alert Rules in Grafana

### Create Alert Rule with Context

In Grafana Alerting, create a rule that triggers on pod restarts:

**Rule Configuration:**

- **Query A (Prometheus):**
  ```promql
  increase(kube_pod_container_status_restarts_total[5m]) > 0
  ```
  Triggers when any pod restarts within 5 minutes.

- **Alert Condition:** Status is firing

- **Evaluation Interval:** 1 minute

**Annotations (Alert Metadata):**

```yaml
summary: "Pod {{ $labels.pod }} restarted"
description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted {{ $values.A.Value | humanize }} times in the last 5 minutes."
pod_name: "{{ $labels.pod }}"
namespace: "{{ $labels.namespace }}"
restart_count: "{{ $values.A.Value }}"
```

**Labels (Routing):**

```yaml
severity: "critical"
component: "pod-restart-alert"
```



## Part 5: Webhook Endpoint to Enrich Alerts with Events and Logs

Grafana sends webhook payloads to your custom endpoint. This endpoint fetches additional context (events and logs) before forwarding to Slack/Discord.

### Python Webhook Receiver

Create a Flask application to handle Grafana webhooks:

```python
from flask import Flask, request, jsonify
import json
import requests
from kubernetes import client, config, watch
from datetime import datetime, timedelta
import subprocess
import os

app = Flask(__name__)

# Load Kubernetes config
config.load_incluster_config()  # For in-cluster pods
# OR: config.load_kube_config()  # For local development

v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

def get_pod_recent_events(namespace, pod_name):
    """Fetch recent Kubernetes events for a pod"""
    try:
        events = v1.list_namespaced_event(namespace=namespace)
        pod_events = [
            e for e in events.items
            if e.involved_object.name == pod_name
            and e.involved_object.kind == "Pod"
        ]
        # Sort by timestamp (most recent first)
        pod_events.sort(
            key=lambda e: e.last_timestamp or e.first_timestamp,
            reverse=True
        )
        return pod_events[:5]  # Return last 5 events
    except Exception as e:
        print(f"Error fetching events: {e}")
        return []

def get_pod_logs(namespace, pod_name, container_name=None, tail_lines=50):
    """Fetch recent pod logs using kubectl"""
    try:
        cmd = [
            "kubectl", "logs",
            f"--namespace={namespace}",
            pod_name,
            f"--tail={tail_lines}"
        ]
        if container_name:
            cmd.extend(["-c", container_name])
        
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
        return result.stdout if result.returncode == 0 else f"Error: {result.stderr}"
    except Exception as e:
        return f"Failed to retrieve logs: {str(e)}"

def get_pod_previous_logs(namespace, pod_name, container_name=None, tail_lines=50):
    """Fetch logs from previous container instance (for crashes)"""
    try:
        cmd = [
            "kubectl", "logs",
            f"--namespace={namespace}",
            pod_name,
            "--previous",
            f"--tail={tail_lines}"
        ]
        if container_name:
            cmd.extend(["-c", container_name])
        
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
        return result.stdout if result.returncode == 0 else None
    except Exception as e:
        return None

def format_events_for_slack(events):
    """Format Kubernetes events as Slack message blocks"""
    blocks = []
    
    for event in events:
        blocks.append({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Reason:* `{event.reason}`\n*Message:* {event.message}\n*Count:* {event.count}\n*Time:* {event.last_timestamp or event.first_timestamp}"
            }
        })
    
    return blocks

def send_to_slack(webhook_url, message_payload):
    """Send message to Slack webhook"""
    try:
        response = requests.post(webhook_url, json=message_payload, timeout=10)
        return response.status_code == 200
    except Exception as e:
        print(f"Slack send error: {e}")
        return False

def send_to_discord(webhook_url, message_payload):
    """Send message to Discord webhook"""
    try:
        response = requests.post(webhook_url, json=message_payload, timeout=10)
        return response.status_code == 204
    except Exception as e:
        print(f"Discord send error: {e}")
        return False

@app.route('/alert', methods=['POST'])
def handle_alert():
    """Handle Grafana webhook alert"""
    try:
        payload = request.json
        
        # Extract pod information from alert
        alerts = payload.get('alerts', [])
        if not alerts:
            return jsonify({"error": "No alerts found"}), 400
        
        alert = alerts[0]
        labels = alert.get('labels', {})
        
        pod_name = labels.get('pod', 'unknown')
        namespace = labels.get('namespace', 'default')
        restart_count = alert.get('values', {}).get('A', {}).get('Value', 'unknown')
        
        # Fetch enrichment data
        pod_events = get_pod_recent_events(namespace, pod_name)
        current_logs = get_pod_logs(namespace, pod_name, tail_lines=30)
        previous_logs = get_pod_previous_logs(namespace, pod_name, tail_lines=30)
        
        # Determine target platform from query parameter
        target = request.args.get('target', 'slack')  # slack or discord
        webhook_url = os.getenv(f'{target.upper()}_WEBHOOK_URL')
        
        if not webhook_url:
            return jsonify({"error": f"{target} webhook URL not configured"}), 500
        
        # Build message payload
        if target.lower() == 'slack':
            message = build_slack_message(
                pod_name, namespace, restart_count, pod_events, current_logs, previous_logs
            )
            success = send_to_slack(webhook_url, message)
        else:  # discord
            message = build_discord_message(
                pod_name, namespace, restart_count, pod_events, current_logs, previous_logs
            )
            success = send_to_discord(webhook_url, message)
        
        return jsonify({"success": success}), 200 if success else 500
    
    except Exception as e:
        print(f"Error handling alert: {e}")
        return jsonify({"error": str(e)}), 500

def build_slack_message(pod_name, namespace, restart_count, events, logs, previous_logs):
    """Build Slack message with events and logs"""
    
    # Format logs (truncate to avoid exceeding Slack limits)
    logs_text = logs[:1500] if logs else "No logs available"
    if len(logs_text) > 1000:
        logs_text = logs_text[:1000] + "\n... (truncated)"
    
    previous_logs_text = previous_logs[:1500] if previous_logs else "No previous logs"
    if previous_logs_text and len(previous_logs_text) > 800:
        previous_logs_text = previous_logs_text[:800] + "\n... (truncated)"
    
    # Format events
    events_text = ""
    for event in events:
        events_text += f"• *{event.reason}*: {event.message} (Count: {event.count})\n"
    
    events_text = events_text or "No recent events"
    
    return {
        "text": f"🚨 Pod Restart Alert: {pod_name}",
        "blocks": [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": f"Pod Restart: {pod_name}",
                    "emoji": True
                }
            },
            {
                "type": "section",
                "fields": [
                    {
                        "type": "mrkdwn",
                        "text": f"*Pod:*\n{pod_name}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Namespace:*\n{namespace}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Restarts (5m):*\n{restart_count}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Time:*\n{datetime.utcnow().isoformat()}"
                    }
                ]
            },
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Recent Events:*\n{events_text}"
                }
            },
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Current Logs (Last 30 lines):*\n```{logs_text}```"
                }
            }
        ]
    }
    
    # Add previous logs section if available
    if previous_logs:
        return {
            **{
                "text": f"🚨 Pod Restart Alert: {pod_name}",
                "blocks": [
                    {
                        "type": "header",
                        "text": {
                            "type": "plain_text",
                            "text": f"Pod Restart: {pod_name}",
                            "emoji": True
                        }
                    },
                    {
                        "type": "section",
                        "fields": [
                            {
                                "type": "mrkdwn",
                                "text": f"*Pod:*\n{pod_name}"
                            },
                            {
                                "type": "mrkdwn",
                                "text": f"*Namespace:*\n{namespace}"
                            },
                            {
                                "type": "mrkdwn",
                                "text": f"*Restarts (5m):*\n{restart_count}"
                            },
                            {
                                "type": "mrkdwn",
                                "text": f"*Time:*\n{datetime.utcnow().isoformat()}"
                            }
                        ]
                    },
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"*Recent Events:*\n{events_text}"
                        }
                    },
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"*Current Logs:*\n```{logs_text}```"
                        }
                    },
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"*Previous Logs (before restart):*\n```{previous_logs_text}```"
                        }
                    }
                ]
            }
        }

def build_discord_message(pod_name, namespace, restart_count, events, logs, previous_logs):
    """Build Discord embed message with events and logs"""
    
    # Format logs
    logs_text = logs[:1000] if logs else "No logs available"
    previous_logs_text = previous_logs[:800] if previous_logs else "No previous logs"
    
    # Format events
    events_text = ""
    for event in events:
        events_text += f"• **{event.reason}**: {event.message} (x{event.count})\n"
    
    events_text = events_text or "No recent events"
    
    embed = {
        "title": f"Pod Restart: {pod_name}",
        "description": f"Pod restarted {restart_count} times in the last 5 minutes",
        "color": 15158332,  # Red
        "fields": [
            {
                "name": "Pod",
                "value": pod_name,
                "inline": True
            },
            {
                "name": "Namespace",
                "value": namespace,
                "inline": True
            },
            {
                "name": "Restart Count (5m)",
                "value": str(restart_count),
                "inline": True
            },
            {
                "name": "Recent Events",
                "value": f"```{events_text}```",
                "inline": False
            },
            {
                "name": "Current Logs (Last 30 lines)",
                "value": f"```{logs_text}```",
                "inline": False
            }
        ],
        "timestamp": datetime.utcnow().isoformat()
    }
    
    if previous_logs:
        embed["fields"].append({
            "name": "Previous Logs (Before Restart)",
            "value": f"```{previous_logs_text}```",
            "inline": False
        })
    
    return {
        "embeds": [embed]
    }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Deploy Webhook Receiver as Kubernetes Service

Create a deployment manifest for the webhook receiver:

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: alert-webhook
  namespace: monitoring


apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: alert-webhook
rules:
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]


apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alert-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: alert-webhook
subjects:
- kind: ServiceAccount
  name: alert-webhook
  namespace: monitoring


apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-webhook
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: alert-webhook
  template:
    metadata:
      labels:
        app: alert-webhook
    spec:
      serviceAccountName: alert-webhook
      containers:
      - name: webhook
        image: python:3.11-slim
        command: ["sh", "-c"]
        args:
          - |
            pip install flask requests kubernetes &&
            python /app/webhook.py
        volumeMounts:
        - name: webhook-code
          mountPath: /app
        ports:
        - containerPort: 5000
        env:
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: alert-webhooks
              key: slack-url
        - name: DISCORD_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: alert-webhooks
              key: discord-url
      volumes:
      - name: webhook-code
        configMap:
          name: alert-webhook-code


apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-webhook-code
  namespace: monitoring
data:
  webhook.py: |
    # [Paste the Flask code from above]


apiVersion: v1
kind: Service
metadata:
  name: alert-webhook
  namespace: monitoring
spec:
  selector:
    app: alert-webhook
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP


apiVersion: v1
kind: Secret
metadata:
  name: alert-webhooks
  namespace: monitoring
type: Opaque
stringData:
  slack-url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
  discord-url: "https://discord.com/api/webhooks/YOUR/DISCORD/WEBHOOK"
```

Deploy:

```bash
kubectl apply -f alert-webhook-deployment.yaml
```



## Part 6: Configure Grafana Webhook Contact Point

1. **Login to Grafana**
2. Navigate to **Alerting → Contact Points → New Contact Point**
3. Select **Webhook**
4. Configure:

   - **Name**: `pod-restart-webhook`
   - **URL**: `http://alert-webhook.monitoring.svc.cluster.local/alert?target=slack`
   - **HTTP Method**: POST

5. Test and save



## Part 7: Advanced Webhook Customization

### Custom Payload Template

Use Grafana's custom payload feature for fine-grained control:

```go
{{ define "custom_alert" -}}
{
  "alert_name": "{{ .GroupLabels.alertname }}",
  "status": "{{ .Status }}",
  "pod": "{{ .CommonLabels.pod }}",
  "namespace": "{{ .CommonLabels.namespace }}",
  "grafana_url": "{{ .ExternalURL }}",
  "timestamp": "{{ now.Format \"2006-01-02T15:04:05Z07:00\" }}"
}
{{- end }}
```

### Integration with Loki Logs in Alerts

Embed log queries directly in alert annotations:

```yaml
annotations:
  logs_link: "https://grafana.example.com/explore?left={\"datasource\":\"Loki\",\"queries\":[{\"refId\":\"A\",\"expr\":\"{namespace=\\\"{{ $labels.namespace }}\\\",pod=\\\"{{ $labels.pod }}\\\"}\"}]}"
```



## Part 9: Production Best Practices

### 1. Log Retention

Configure Loki retention policies to balance cost and compliance:

```yaml
ingester:
  chunk_retain_period: 1m
  max_chunk_age: 2h

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 720h  # 30 days
```

### 2. Alert Routing and Grouping

Define notification policies to avoid alert fatigue:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-alert-notification-policy
  namespace: monitoring
data:
  notification-policy.yaml: |
    receiver: 'slack-critical'
    group_by: ['namespace', 'pod']
    group_wait: 10s
    group_interval: 1m
    repeat_interval: 4h
    routes:
    - receiver: 'slack-prod'
      match:
        environment: 'production'
      group_wait: 5s
      repeat_interval: 2h
    - receiver: 'slack-staging'
      match:
        environment: 'staging'
```

### 3. Rate Limiting

Prevent webhook endpoint overload:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(app, key_func=get_remote_address)

@app.route('/alert', methods=['POST'])
@limiter.limit("100 per minute")  # Max 100 requests/minute
def handle_alert():
    # ...
```

### 4. Monitoring the Webhook Receiver

Add Prometheus metrics to track webhook performance:

```python
from prometheus_client import Counter, Histogram

webhook_requests = Counter(
    'webhook_requests_total',
    'Total webhook requests',
    ['status']
)

webhook_duration = Histogram(
    'webhook_duration_seconds',
    'Webhook processing duration'
)

@app.route('/alert', methods=['POST'])
def handle_alert():
    with webhook_duration.time():
        # Handle alert
        webhook_requests.labels(status='success').inc()
```



## Part 10: Other Scenarios

### Scenario 1: CrashLoopBackOff Detection

Combine pod restart metrics with event data to detect crash loops:

```promql
# Alert when pod restarts exceed 5 in 10 minutes
rate(kube_pod_container_status_restarts_total[10m]) > 0.5
```

Query related events:

```promql
kube_event_count{
  involved_object_kind="Pod",
  reason=~"BackOff|Failed"
}
```

### Scenario 2: Multi-Container Pod Restarts

Identify which container in a multi-container pod is restarting:

```promql
kube_pod_container_status_restarts_total{container!=""}
```

Update webhook to fetch logs per-container:

```python
# Get all containers in the pod
pod = v1.read_namespaced_pod(pod_name, namespace)
containers = [c.name for c in pod.spec.containers]

# Fetch logs for each
for container_name in containers:
    logs = get_pod_logs(namespace, pod_name, container_name)
```

### Scenario 3: Cross-Namespace Pod Restart Correlation

Alert on spikes across multiple namespaces:

```promql
sum by (namespace) (
  rate(kube_pod_container_status_restarts_total[5m])
) > 0.5
```



## Conclusion

By combining Prometheus metrics, Kubernetes events, and application logs within Grafana alerts, you create a powerful incident response system that surfaces context-rich notifications. This setup enables ops teams to move from reactive firefighting to proactive, informed incident triage.

**Key Takeaways:**

- Use **event-exporter** to bridge ephemeral Kubernetes events to Prometheus
- Deploy **Loki** to centralize and query pod logs at scale
- Build custom webhook receivers to enrich alerts with real-time cluster state
- Implement rate limiting and monitoring on webhook endpoints for production reliability
- Use structured logging and label-based routing to reduce alert fatigue
