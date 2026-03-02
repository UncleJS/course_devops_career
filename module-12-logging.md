# Module 12: Logging & Log Management

> **Course**: DevOps Career Path  
> **Audience**: Beginner → Intermediate  
> **Prerequisites**: Module 06 (Kubernetes), Module 11 (Monitoring & Observability)

---

## Table of Contents

1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Logging Fundamentals](#logging-fundamentals)
4. [Structured Logging](#structured-logging)
5. [The ELK Stack](#the-elk-stack)
   - [Elasticsearch](#elasticsearch)
   - [Logstash](#logstash)
   - [Kibana](#kibana)
   - [Filebeat](#filebeat)
   - [Full ELK Deployment Example](#full-elk-deployment-example)
6. [Grafana Loki Stack](#grafana-loki-stack)
   - [Loki Architecture](#loki-architecture)
   - [Promtail](#promtail)
   - [LogQL](#logql)
   - [Loki Deployment](#loki-deployment)
7. [Fluentd & Fluent Bit](#fluentd--fluent-bit)
8. [Kubernetes Logging Patterns](#kubernetes-logging-patterns)
9. [Log Retention, Rotation & Compliance](#log-retention-rotation--compliance)
10. [Centralized Logging Architecture Patterns](#centralized-logging-architecture-patterns)
11. [Comparing Logging Stacks](#comparing-logging-stacks)
12. [Tools & Commands Reference](#tools--commands-reference)
13. [Hands-On Labs](#hands-on-labs)
14. [Further Reading](#further-reading)

---

## Overview

Logs are the raw narrative of your system — every event, error, transaction, and state change produces log data. This module covers the full lifecycle: structured log emission, collection, shipping, indexing, querying, and retention. You will work with both the **ELK Stack** (Elasticsearch, Logstash, Kibana) and the **Loki Stack** (Loki, Promtail, Grafana) — the two dominant open-source centralized logging solutions.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module, you will be able to:

- Explain the lifecycle of a log event from emission to query
- Implement structured logging in applications (JSON format)
- Deploy and configure the full ELK Stack
- Write Elasticsearch queries and build Kibana dashboards
- Deploy Loki + Promtail and write LogQL queries
- Ship logs with Filebeat, Fluent Bit, and Promtail
- Implement Kubernetes-native logging patterns
- Design log retention, rotation, and archival policies
- Choose the right logging stack for a given use case

[↑ Back to TOC](#table-of-contents)

---

## Logging Fundamentals

### Log levels

| Level | Numeric | Meaning | When to use |
|-------|---------|---------|-------------|
| **TRACE** | 5 | Finest detail | Deep debugging only — never in production |
| **DEBUG** | 4 | Diagnostic detail | Development and troubleshooting |
| **INFO** | 3 | Normal operation | Service startup, user actions, milestones |
| **WARN** | 2 | Unexpected but recoverable | Deprecated API usage, slow response |
| **ERROR** | 1 | Failure, operation failed | Exception caught, request failed |
| **FATAL** | 0 | Unrecoverable, process exit | Out of memory, config missing |

### Log lifecycle

```
Application
    │
    │ emits log event
    ▼
Log Shipper (Filebeat / Promtail / Fluent Bit)
    │
    │ forwards (with buffering)
    ▼
Log Aggregator (Logstash / Fluentd / Loki)
    │
    │ parses, enriches, routes
    ▼
Log Store (Elasticsearch / Loki / S3)
    │
    │ indexed, queryable
    ▼
Visualization (Kibana / Grafana)
    │
    │ search, dashboard, alert
    ▼
Engineer / On-call
```

### The syslog standard

```
# Syslog format: RFC 3164
<priority>timestamp hostname app[pid]: message

# Example:
<34>Mar  2 14:22:01 web-01 nginx[1234]: 2026/03/02 14:22:01 [error] 1234#1234: *1 connect() failed

# Syslog facilities (0-23)
0  = kern     8  = uucp
1  = user     9  = cron
3  = daemon   16-23 = local0-local7
```

[↑ Back to TOC](#table-of-contents)

---

## Structured Logging

Unstructured logs are human-readable but machine-hostile. Structured logs (JSON) are both.

### Unstructured (avoid)

```
2026-03-02 14:22:01 ERROR Failed to connect to database after 3 retries host=db-01 latency=2340ms
```

### Structured (prefer)

```json
{
  "timestamp": "2026-03-02T14:22:01.453Z",
  "level": "error",
  "service": "api-gateway",
  "host": "web-01",
  "trace_id": "abc123def456",
  "user_id": "u-9981",
  "event": "db_connection_failed",
  "database_host": "db-01",
  "retry_count": 3,
  "latency_ms": 2340,
  "message": "Failed to connect to database after 3 retries"
}
```

### Structured logging in Python

```python
import logging
import json
import sys
from datetime import datetime, timezone


class JSONFormatter(logging.Formatter):
    """Emit log records as JSON lines."""

    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname.lower(),
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        # Merge any extra fields passed via extra={}
        for key, value in record.__dict__.items():
            if key not in logging.LogRecord.__dict__ and not key.startswith("_"):
                if key not in log_entry:
                    log_entry[key] = value
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry)


def get_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    logger.addHandler(handler)
    return logger


# Usage
log = get_logger("api.orders")

log.info("Order created", extra={"order_id": "ord-123", "user_id": "u-456", "amount": 99.99})
log.error("Payment failed", extra={"order_id": "ord-123", "error_code": "CARD_DECLINED"})
```

### Structured logging in Node.js (pino)

```javascript
import pino from 'pino';

const log = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: 'order-service',
    env: process.env.NODE_ENV,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level(label) {
      return { level: label };
    },
  },
});

// Usage
log.info({ orderId: 'ord-123', userId: 'u-456' }, 'Order created');
log.error({ orderId: 'ord-123', err: new Error('Card declined') }, 'Payment failed');
```

### Key fields to always include

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO 8601 UTC | When the event occurred |
| `level` | string | Log level |
| `service` | string | Application/service name |
| `trace_id` | string | Distributed trace ID (pass via headers) |
| `request_id` | string | Per-request unique ID |
| `user_id` | string | Authenticated user (omit if no auth context) |
| `host` | string | Emitting host/pod |
| `message` | string | Human-readable summary |

[↑ Back to TOC](#table-of-contents)

---

## The ELK Stack

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana. In modern deployments, Logstash is often replaced by the lighter **Filebeat** (direct to Elasticsearch) or **Fluent Bit**.

```
┌──────────────────────────────────────────────────────────────────┐
│                         ELK STACK                                │
│                                                                  │
│  ┌─────────────┐    ┌───────────┐    ┌─────────────────────────┐ │
│  │  Filebeat   │───►│ Logstash  │───►│    Elasticsearch        │ │
│  │  (on hosts) │    │ (parse /  │    │    (index & store)      │ │
│  │             │    │  enrich)  │    │                         │ │
│  └─────────────┘    └───────────┘    └──────────┬──────────────┘ │
│                                                 │                │
│                                                 ▼                │
│                                      ┌──────────────────────┐   │
│                                      │        Kibana        │   │
│                                      │  (search + dashboards│   │
│                                      │   + alerts)          │   │
│                                      └──────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Elasticsearch

Elasticsearch is a distributed, document-oriented search and analytics engine built on Apache Lucene.

#### Core concepts

| Concept | Description |
|---------|-------------|
| **Index** | Collection of documents (like a database table) |
| **Document** | A JSON object stored in an index |
| **Shard** | Horizontal partition of an index (Lucene index) |
| **Replica** | Copy of a shard for HA and read scaling |
| **Mapping** | Schema definition for document fields |
| **ILM** | Index Lifecycle Management — automate index aging |

#### Elasticsearch REST API

```bash
# Check cluster health
curl -s http://localhost:9200/_cluster/health | jq .

# List all indices
curl -s http://localhost:9200/_cat/indices?v

# Create an index with explicit mapping
curl -X PUT http://localhost:9200/app-logs-2026.03 \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "timestamp":   { "type": "date" },
        "level":       { "type": "keyword" },
        "service":     { "type": "keyword" },
        "host":        { "type": "keyword" },
        "trace_id":    { "type": "keyword" },
        "message":     { "type": "text", "analyzer": "standard" },
        "latency_ms":  { "type": "long" },
        "status_code": { "type": "integer" }
      }
    }
  }'

# Index a document
curl -X POST http://localhost:9200/app-logs-2026.03/_doc \
  -H "Content-Type: application/json" \
  -d '{"timestamp":"2026-03-02T14:22:01Z","level":"error","service":"api","message":"timeout"}'

# Search — match all errors
curl -s -X GET http://localhost:9200/app-logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "filter": [
          { "term": { "level": "error" } },
          { "range": { "timestamp": { "gte": "now-1h" } } }
        ]
      }
    },
    "sort": [{ "timestamp": "desc" }],
    "size": 20
  }'

# Full-text search in message field
curl -s -X GET http://localhost:9200/app-logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": {
        "message": "connection refused"
      }
    }
  }'

# Aggregation — count errors per service
curl -s -X GET http://localhost:9200/app-logs-*/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "term": { "level": "error" } },
    "aggs": {
      "errors_by_service": {
        "terms": { "field": "service", "size": 10 }
      }
    },
    "size": 0
  }'
```

#### Index Lifecycle Management (ILM)

```json
// PUT _ilm/policy/app-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "10gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

[↑ Back to TOC](#table-of-contents)

---

### Logstash

Logstash is a server-side data processing pipeline: input → filter → output.

#### `logstash.conf` — full pipeline example

```ruby
input {
  # Receive from Filebeat
  beats {
    port => 5044
    ssl  => false
  }

  # Also accept syslog
  syslog {
    port => 5140
    type => "syslog"
  }

  # Accept from Kafka (for high-volume)
  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["app-logs"]
    codec => json
  }
}

filter {
  # Parse JSON logs
  if [type] == "application" {
    json {
      source => "message"
      target => "parsed"
      remove_field => ["message"]
    }
    mutate {
      rename => { "[parsed][timestamp]" => "@timestamp" }
      rename => { "[parsed][level]"     => "level"      }
      rename => { "[parsed][service]"   => "service"    }
      rename => { "[parsed][message]"   => "message"    }
    }
  }

  # Parse Nginx access logs (unstructured)
  if [type] == "nginx" {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code:int} %{NUMBER:bytes:int} "%{DATA:referrer}" "%{DATA:user_agent}"'
      }
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
      remove_field => ["timestamp"]
    }
    mutate {
      convert => { "status_code" => "integer" }
      convert => { "bytes"       => "integer" }
    }
    # Tag 5xx errors
    if [status_code] >= 500 {
      mutate { add_tag => ["error", "5xx"] }
    }
  }

  # GeoIP enrichment
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }

  # Drop health check noise
  if [request] =~ "/health" {
    drop {}
  }

  # Add environment tag
  mutate {
    add_field => { "environment" => "${ENV:production}" }
  }
}

output {
  # Send to Elasticsearch with daily indices
  elasticsearch {
    hosts    => ["http://elasticsearch:9200"]
    index    => "%{[type]}-%{+YYYY.MM.dd}"
    template_name    => "app-logs"
    template_overwrite => false
  }

  # Also write errors to separate index
  if "error" in [tags] {
    elasticsearch {
      hosts => ["http://elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }

  # Debug — print to stdout
  # stdout { codec => rubydebug }
}
```

[↑ Back to TOC](#table-of-contents)

---

### Kibana

Kibana provides a web UI for searching, visualizing, and alerting on Elasticsearch data.

#### Key features

| Feature | Description |
|---------|-------------|
| **Discover** | Full-text search and log browsing with time filters |
| **Dashboard** | Compose visualizations into monitoring boards |
| **Visualize** | Bar charts, line charts, maps, metric panels |
| **Alerts** | Rule-based alerting on Elasticsearch data |
| **APM** | Application Performance Monitoring |
| **Lens** | Drag-and-drop chart builder |
| **Canvas** | Custom reporting and presentations |

#### KQL (Kibana Query Language) examples

```kql
# Find all errors
level: "error"

# Errors from a specific service in last 15 min (use time filter)
level: "error" AND service: "api-gateway"

# Full-text search
message: "connection refused"

# Status codes 500–599
status_code >= 500 AND status_code < 600

# Complex filter
service: ("api-gateway" OR "order-service") AND level: ("error" OR "warn")

# Wildcard
request: /api/users/*

# Negation
NOT service: "health-checker"
```

[↑ Back to TOC](#table-of-contents)

---

### Filebeat

Filebeat is a lightweight log shipper (Go binary, ~40MB) that tails log files and ships to Logstash or Elasticsearch.

#### `filebeat.yml`

```yaml
filebeat.inputs:
  # Application JSON logs
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      type: application
      environment: production
    fields_under_root: true
    multiline.pattern: '^{'
    multiline.negate: true
    multiline.match: after

  # Nginx access logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      type: nginx

  # Nginx error logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      type: nginx-error

  # System logs
  - type: log
    enabled: true
    paths:
      - /var/log/messages
      - /var/log/syslog
    fields:
      type: syslog

# Processors — enrich events before shipping
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_docker_metadata: ~
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
        - logs_path:
            logs_path: "/var/log/containers/"

# Output — send to Logstash
output.logstash:
  hosts: ["logstash:5044"]
  loadbalance: true

# Alternatively — send directly to Elasticsearch
# output.elasticsearch:
#   hosts: ["http://elasticsearch:9200"]
#   index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

# Internal metrics
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["http://elasticsearch:9200"]

# Logging
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

[↑ Back to TOC](#table-of-contents)

---

### Full ELK Deployment Example

```yaml
# docker-compose.yml — ELK Stack
version: '3.8'

volumes:
  esdata01: {}
  kibanadata: {}

networks:
  elk:

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      - node.name=es01
      - cluster.name=elk-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false          # Enable in production!
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk
    healthcheck:
      test: curl -fs http://localhost:9200/_cluster/health || exit 1
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5044:5044"      # Beats input
      - "5140:5140/udp"  # Syslog
    environment:
      - "LS_JAVA_OPTS=-Xmx512m -Xms512m"
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    volumes:
      - kibanadata:/usr/share/kibana/data
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.13.0
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - LOGSTASH_HOST=logstash:5044
    networks:
      - elk
    depends_on:
      - logstash
```

[↑ Back to TOC](#table-of-contents)

---

## Grafana Loki Stack

Loki is a horizontally scalable, highly available log aggregation system designed to be cost-effective and easy to operate. Unlike Elasticsearch, **Loki only indexes metadata (labels) — not log content**, making it much cheaper to store and operate.

### Loki Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          LOKI STACK                              │
│                                                                  │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐  │
│  │  Promtail   │───►│     Loki     │◄───│      Grafana       │  │
│  │  (on hosts  │    │  (log store) │    │  (query + display) │  │
│  │  / pods)    │    │              │    │  LogQL queries     │  │
│  └─────────────┘    │  Chunks:S3   │    └────────────────────┘  │
│                     │  Index:BoltDB│                             │
│  ┌─────────────┐    │  /DynamoDB   │                             │
│  │ Fluent Bit  │───►│              │                             │
│  │  (optional) │    └──────────────┘                             │
│  └─────────────┘                                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Loki Deployment

```yaml
# docker-compose.yml — Loki + Promtail + Grafana
version: '3.8'

volumes:
  loki_data: {}
  grafana_data: {}

networks:
  loki:

services:
  loki:
    image: grafana/loki:2.9.4
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    networks:
      - loki

  promtail:
    image: grafana/promtail:2.9.4
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./promtail-config.yml:/etc/promtail/config.yml:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - loki

  grafana:
    image: grafana/grafana:10.4.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/loki.yml:ro
    networks:
      - loki
```

#### `loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 90d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_entries_limit_per_query: 5000

ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /loki/rules-temp
  alertmanager_url: http://alertmanager:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
```

[↑ Back to TOC](#table-of-contents)

---

### Promtail

Promtail is the log collection agent for Loki — it tails files, attaches labels, and pushes to Loki.

#### `promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # System logs
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          host: web-01
          __path__: /var/log/syslog

  # All files in /var/log/app
  - job_name: application
    static_configs:
      - targets:
          - localhost
        labels:
          job: app
          env: production
          __path__: /var/log/app/*.log
    pipeline_stages:
      # Parse JSON log lines
      - json:
          expressions:
            level:   level
            service: service
            trace_id: trace_id
      # Extract level as a label (for filtering)
      - labels:
          level:
          service:
      # Set timestamp from log field
      - timestamp:
          source: timestamp
          format: RFC3339Nano

  # Docker container logs
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'

  # Kubernetes pod logs
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - docker: {}
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_container_name]
        target_label: container
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

[↑ Back to TOC](#table-of-contents)

---

### LogQL

LogQL is Loki's query language. It borrows from PromQL — a log stream selector followed by optional filter and metric expressions.

#### Log stream selectors

```logql
# Select all logs from the nginx job
{job="nginx"}

# Select by multiple labels
{job="app", env="production", level="error"}

# Regex label match
{job=~"app.*", env!="dev"}
```

#### Log filters (pipeline)

```logql
# Line filter — include lines containing the string
{job="nginx"} |= "error"

# Line filter — exclude lines
{job="nginx"} != "healthz"

# Regex filter
{job="nginx"} |~ "5[0-9]{2}"

# JSON parser + field filter
{job="app"} | json | level="error"

# Logfmt parser (key=value format)
{job="app"} | logfmt | duration > 500ms

# Pattern parser (extract named fields)
{job="nginx"} | pattern `<ip> - <_> [<_>] "<method> <path> <_>" <status> <size>`

# Filter on extracted field
{job="nginx"} | pattern `<ip> - <_> [<_>] "<method> <path> <_>" <status> <size>` | status >= "500"
```

#### Metric queries (LogQL → Prometheus-like metrics)

```logql
# Rate of log lines per second
rate({job="nginx"}[5m])

# Rate of error logs per second
rate({job="app"} | json | level="error" [5m])

# Count entries over time window
count_over_time({job="app"}[1h])

# Bytes received per second
bytes_rate({job="nginx"}[5m])

# Top 5 IPs by request count (last 1 hour)
topk(5,
  sum by (ip) (
    count_over_time({job="nginx"} | pattern `<ip> - <_>` [1h])
  )
)

# 95th percentile latency from JSON logs
quantile_over_time(0.95,
  {job="api"} | json | unwrap latency_ms [5m]
) by (service)
```

[↑ Back to TOC](#table-of-contents)

---

## Fluentd & Fluent Bit

| Feature | Fluentd | Fluent Bit |
|---------|---------|-----------|
| Language | Ruby + C | C only |
| Memory | ~40MB | ~650KB |
| Plugins | 500+ | 70+ |
| Best for | Aggregation tier | Edge/node log shipping |
| Kubernetes | DaemonSet aggregator | DaemonSet collector |

#### Fluent Bit configuration

```ini
# /etc/fluent-bit/fluent-bit.conf

[SERVICE]
    Flush        5
    Daemon       Off
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name         tail
    Path         /var/log/containers/*.log
    Parser       docker
    Tag          kube.*
    Refresh_Interval 5
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On

[FILTER]
    Name         kubernetes
    Match        kube.*
    Kube_URL     https://kubernetes.default.svc:443
    Kube_CA_File /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log    On
    K8S-Logging.Parser On
    K8S-Logging.Exclude Off

[FILTER]
    Name         grep
    Match        kube.*
    Exclude      log /healthz

[OUTPUT]
    Name         es
    Match        kube.*
    Host         elasticsearch
    Port         9200
    Index        fluent-bit-kube
    Type         _doc
    Retry_Limit  3

[OUTPUT]
    Name         loki
    Match        kube.*
    Host         loki
    Port         3100
    Labels       job=fluent-bit, env=production
    Label_Keys   $kubernetes['namespace_name'],$kubernetes['pod_name']
```

[↑ Back to TOC](#table-of-contents)

---

## Kubernetes Logging Patterns

### Pattern 1: Node-level DaemonSet (most common)

```yaml
# fluent-bit DaemonSet (simplified)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

### Pattern 2: Sidecar logging container

```yaml
# Pod with log-shipping sidecar
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar-logging
spec:
  containers:
    # Main application — writes logs to shared volume
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app

    # Sidecar — ships logs to Loki
    - name: promtail
      image: grafana/promtail:2.9.4
      args:
        - -config.file=/etc/promtail/config.yml
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: promtail-config
          mountPath: /etc/promtail

  volumes:
    - name: app-logs
      emptyDir: {}
    - name: promtail-config
      configMap:
        name: promtail-sidecar-config
```

### Kubernetes log best practices

```
✅ Always log to stdout/stderr — Kubernetes captures these automatically
✅ Use structured (JSON) logging
✅ Include pod name, namespace, and container in labels (auto-added by collectors)
✅ Don't write logs to files inside containers — ephemeral storage
✅ Set resource limits on logging DaemonSets (memory, CPU)
✅ Use log-level environment variables (LOG_LEVEL=info)
✅ Rotate and limit log file sizes via Docker/container runtime config
```

[↑ Back to TOC](#table-of-contents)

---

## Log Retention, Rotation & Compliance

### Log rotation — logrotate

```ini
# /etc/logrotate.d/app-logs
/var/log/app/*.log {
    daily
    rotate 30          # Keep 30 days
    compress
    delaycompress      # Don't compress the most recent rotated file
    missingok
    notifempty
    sharedscripts
    postrotate
        # Signal app to reopen log files
        systemctl kill -s HUP app.service
    endscript
}
```

### Docker logging configuration

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5",
    "compress": "true",
    "labels": "environment,service"
  }
}
```

### Kubernetes container log limits

```yaml
# Node-level kubelet config
# /var/lib/kubelet/config.yaml
containerLogMaxSize: "100Mi"
containerLogMaxFiles: 5
```

### Retention policy guidelines

| Log type | Hot (searchable) | Warm (compressed) | Cold (archived) | Delete |
|----------|-----------------|-------------------|-----------------|--------|
| Application logs | 30 days | 60 days | 1 year | After 1 year |
| Security/audit logs | 90 days | 1 year | 5 years (compliance) | Per policy |
| Access logs | 14 days | 30 days | 90 days | After 90 days |
| Debug logs | 7 days | — | — | After 7 days |
| System logs | 30 days | 90 days | — | After 90 days |

> **Compliance note**: PCI-DSS requires 1 year log retention (3 months immediately accessible). HIPAA: 6 years. SOC2: defined by your organization's policy.

### Archiving to S3-compatible storage

```bash
# Ship old Elasticsearch indices to S3 (snapshot)
# Register S3 repository
curl -X PUT "http://elasticsearch:9200/_snapshot/s3_backup" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "es-log-archive",
      "region": "us-east-1",
      "base_path": "snapshots"
    }
  }'

# Create snapshot
curl -X PUT "http://elasticsearch:9200/_snapshot/s3_backup/snapshot-2026-01" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "app-logs-2026.01.*",
    "include_global_state": false
  }'
```

[↑ Back to TOC](#table-of-contents)

---

## Centralized Logging Architecture Patterns

### Pattern 1: Small team (< 100 nodes)

```
Hosts → Promtail → Loki → Grafana
Cost: Near-zero   Ops: Minimal   Query: LogQL
```

### Pattern 2: Medium enterprise (100–1000 nodes)

```
Hosts → Filebeat → Logstash → Elasticsearch → Kibana
Cost: Moderate   Ops: Moderate   Query: KQL/EQL
ILM: hot→warm→cold→delete (90 days)
```

### Pattern 3: High-volume (> 1000 nodes / 1TB+ per day)

```
Hosts → Fluent Bit (edge)
              │
              ▼
          Kafka (buffer + fan-out)
         /          \
        ▼             ▼
   Logstash       Logstash
   (filter)       (filter)
        \           /
         ▼         ▼
      Elasticsearch cluster
      (coordinating → data → master nodes)
              │
              ▼
           Kibana
```

### Pattern 4: Multi-cloud / Kubernetes

```
Kubernetes DaemonSet (Fluent Bit / Promtail)
          │
          ├──► Loki (recent, fast, cheap)  ──► Grafana
          │
          └──► S3/GCS/Azure Blob (long-term archive)
```

[↑ Back to TOC](#table-of-contents)

---

## Comparing Logging Stacks

| Feature | ELK Stack | Loki Stack | Splunk | CloudWatch |
|---------|-----------|------------|--------|------------|
| **Full-text index** | ✅ Yes | ❌ No (labels only) | ✅ Yes | Partial |
| **Storage cost** | Higher | Lower (3-10x) | Very high | Pay per GB |
| **Query language** | KQL / EQL | LogQL | SPL | Insights QL |
| **Schema on write** | Yes (mapping) | No (raw) | Yes | No |
| **Kubernetes native** | Good | Excellent | Plugin | AWS only |
| **Alerting** | Built-in | Via Grafana | Built-in | Built-in |
| **Self-hosted** | Yes | Yes | Yes / SaaS | SaaS only |
| **Best for** | Full-text search, compliance | Cloud-native, cost-sensitive | Enterprise, SOC | AWS workloads |

> **Rule of thumb**: If you need to search *inside* log content frequently (full-text), ELK is better. If you primarily filter by labels/metadata and correlate with metrics, Loki + Grafana is simpler and cheaper.

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

### Elasticsearch

```bash
# Cluster status
curl -s http://localhost:9200/_cluster/health?pretty

# Node info
curl -s http://localhost:9200/_nodes/stats?pretty | jq '.nodes | to_entries[] | {name: .value.name, heap_used: .value.jvm.mem.heap_used_percent}'

# Index stats
curl -s http://localhost:9200/_cat/indices?v&h=index,docs.count,store.size&s=store.size:desc

# Delete old index
curl -X DELETE http://localhost:9200/app-logs-2025.12.*

# ILM status
curl -s http://localhost:9200/_ilm/status
curl -s http://localhost:9200/.ds-app-logs*/_ilm/explain?pretty
```

### Logstash

```bash
# Test pipeline config
/usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/

# Check pipeline stats
curl -s http://localhost:9600/_node/stats/pipelines | jq .

# Reload config hot
curl -X POST http://localhost:9600/_node/reload
```

### Filebeat

```bash
# Test config
filebeat test config -e

# Test output connectivity
filebeat test output

# Run once and exit (debug)
filebeat -e -d "*" --once

# Check registry (tracking which files were read)
cat /var/lib/filebeat/registry/filebeat/data.json | jq .
```

### Loki / Promtail

```bash
# Query Loki via HTTP API
curl -s 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="nginx"} |= "error"' \
  --data-urlencode 'start=1700000000000000000' \
  --data-urlencode 'end=1700003600000000000' \
  --data-urlencode 'limit=50' | jq .

# Loki labels
curl -s http://localhost:3100/loki/api/v1/labels | jq .

# Promtail status
curl -s http://localhost:9080/metrics | grep promtail_targets

# logcli (Loki CLI tool)
logcli query '{job="nginx"}' --limit=100 --since=1h
logcli labels
logcli series '{job="app"}' --since=1h
```

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 1 — Deploy the Loki Stack Locally (Beginner)

**Goal**: Run Loki + Promtail + Grafana, ship local logs, and write LogQL queries.

```bash
# Create project structure
mkdir loki-lab && cd loki-lab

# Create the docker-compose.yml (use the example from this module)
# Create loki-config.yml and promtail-config.yml

# Configure Promtail to tail /var/log/syslog or /var/log/messages
# Start the stack
docker compose up -d    # or: podman-compose up -d

# Open Grafana: http://localhost:3000 (admin/admin123)
# Add Loki datasource: http://loki:3100
# Go to Explore → select Loki → query: {job="syslog"}
```

**Exercises**:
1. Filter for ERROR lines: `{job="syslog"} |= "error"`
2. Count log rate: `rate({job="syslog"}[5m])`
3. Count total entries last hour: `count_over_time({job="syslog"}[1h])`

---

### Lab 2 — Ship Application Logs to Loki with JSON Parsing (Intermediate)

**Goal**: Write a Python app that emits structured JSON logs and ship them to Loki.

```python
# app.py — generate sample JSON log traffic
import time
import random
import logging
from json_logger import get_logger   # Use the JSONFormatter from this module

log = get_logger("lab-app")

services = ["auth", "orders", "payments", "inventory"]
levels = ["info"] * 7 + ["warn"] * 2 + ["error"]

while True:
    level = random.choice(levels)
    svc = random.choice(services)
    latency = random.randint(5, 3000)
    getattr(log, level)(
        f"Request processed",
        extra={"service": svc, "latency_ms": latency, "status": "ok" if level == "info" else "error"}
    )
    time.sleep(0.5)
```

Configure Promtail to:
1. Tail `app.log`
2. Parse JSON and extract `level` and `service` as labels
3. In Grafana, build a panel showing error rate by service:
   ```logql
   sum by(service) (rate({job="app", level="error"}[5m]))
   ```

---

### Lab 3 — ELK Stack with Nginx Log Parsing (Intermediate)

**Goal**: Deploy ELK, ship Nginx logs, parse with Logstash grok, visualize in Kibana.

1. Deploy ELK stack with `docker compose up -d`
2. Generate Nginx traffic: `ab -n 1000 -c 10 http://localhost/`
3. Filebeat ships `/var/log/nginx/access.log` → Logstash
4. Logstash grok parses the access log format
5. In Kibana Discover: search `status_code: 404` and `method: POST`
6. Create a dashboard with:
   - Line chart: requests per minute over time
   - Pie chart: requests by status code
   - Data table: top 10 requested URLs

---

### Lab 4 — Loki Alerting Rule (Intermediate)

**Goal**: Define a Loki alerting rule that fires when error rate exceeds threshold.

```yaml
# /loki/rules/alerts.yml
groups:
  - name: app_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="app", level="error"}[5m])) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High application error rate"
          description: "Error rate is {{ $value | humanize }} per second"
```

1. Add the rule to `loki-config.yml` ruler section
2. Restart Loki
3. Generate errors in your test app
4. Observe alert firing in Grafana → Alerting

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash Filter Plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)
- [Kibana KQL Reference](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Filebeat Documentation](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/query/)
- [Promtail Documentation](https://grafana.com/docs/loki/latest/send-data/promtail/)
- [Fluent Bit Documentation](https://docs.fluentbit.io/manual/)
- [12-Factor App — Logs](https://12factor.net/logs)
- [Google SRE Book — Logging](https://sre.google/sre-book/practical-alerting/)

[↑ Back to TOC](#table-of-contents)

---

*© UncleJS — Licensed under [Creative Commons BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution.*
