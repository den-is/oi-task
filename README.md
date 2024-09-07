# Building on-premises platform for hosting micro-services

## Table of Contents
- [Preface](#preface)
- [Networking](#networking)
- [Storage](#storage)
- [Nodes](#nodes)
- [Configurations management](#configurations-management)
- [Kubernetes](#kubernetes)
- [Application lifecycle management](#application-lifecycle-management)
- [Security](#security)
- [Observability](#observability)
  - [Central Logs collection](#central-logs-collection)
  - [Metrics scraping](#metrics-scraping)
  - [Alerting](#alerting)
  - [Traces collection](#traces-collection)

## Preface
Based on the information retrieved during the initial call.
Skipping paid solution for the most cases or suggesting as a secondary alternative.

## Networking
- Networking should be designed by professional Network Architects. Network should be flexible, secure, allow isolated multi-tenant environments setups.

## Storage

## Nodes

## Configurations management
- Any manual operation is strictly forbidden
  - Except labs, and personal RnD setups
  - You develop script, ci/cd pipeline, hardware definition and push it to git before releasing it to any shared environment.
- GitOps all the way for everything (network, os, kubernetes)
- Strict common formatting and linting across all contributors
  - whitespaces management
  - indendations
  - common formatting rules per PL and/or DSL
- Systems live patching should be forbidden for the most cases
  - Use updated OS images to rollout updates

## Kubernetes


## Application lifecycle management
- Splitting CI/CD into CI and CD where each part is managed by dedicated product.
- CI - building, testing, release artifacts production
  - Gitlab CI
- CD - delivery, deploy
  - Argo Projects ecosystem

## Security
- Dedicated SecOps personel.
- Images Scanning
- AAA - Authentication, Authorization, Audit
- SSO
- Kubernetes SSO access

## Observability

### Central Logs collection
- Loki
- ELK
- Agents:
  - Vector
  - Promtail
  - Fluentbit
  - Fluentd

### Metrics scraping
- [Prometheus](https://prometheus.io/) stack consists of multiple pluggble components
- Thanos
- VictoriaMetrics as alternative to prometheus

### Alerting
- [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) - for alerting based on metrics. Part of Prometheus stack
- PagerDuty or similar incident management product for efficient and shift-based communications with responsible operators.

### Traces collection
- [OpenTelemetry](https://opentelemetry.io) [Collector](https://opentelemetry.io/docs/collector/)
  - The OpenTelemetry Collector offers a vendor-agnostic implementation of how to receive, process and export telemetry data.
  - It support receiving observability data from most sources, process, and send data to almost all know Observability Products.
- Traces help to discover issues and events within application which are usually difficult to catch using low level metrics or
