# Building on-premises platform for hosting micro-services

## Table of Contents
- [Preface](#preface)
- [Networking](#networking)
- [Storage](#storage)
- [Nodes Provisioning and Configuration Management](#nodes-provisioning-and-configuration-management)
- [Kubernetes](#kubernetes)
- [Security](#security)
- [Application lifecycle management](#application-lifecycle-management)
  - [Multi-stage multi-environment deployments](#multi-stage-multi-environment-deployments)
  - [PostgreSQL databases](#postgresql-databases)
- [Observability](#observability)
  - [Central Logs collection](#central-logs-collection)
  - [Metrics scraping](#metrics-scraping)
  - [Alerting](#alerting)
  - [Traces collection](#traces-collection)
  - [Observability UI/UX and interactions](#observability-uiux-and-interactions)
  - [Events response and actions](#events-response-and-actions)

## Preface
Based on the information retrieved during the initial call.  
The multi-tenant HPC Platform is built on-premises and exists as a secure, private alternative to public cloud providers.  
The core management framework for applications running on top of the platform is Kubernetes.  
Kubernetes provides flexible application lifecycle management and tooling for building reliable, highly available modern platforms.

## Networking
Physical networking should be designed by professional Network Architects.  
The network must be flexible, secure, auditable, and support isolated multi-tenant environments.  
For Kubernetes cluster networking, I recommend using [Cilium](https://cilium.io/) [CNI](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/).

Cilium provides a robust and flexible networking "driver" for Kubernetes clusters.
- Offers advanced security and observability options using modern Linux Kernel technology [eBPF](https://ebpf.io/).
- Enables Kubernetes Network Policies or replaces them with more advanced Cilium Network Policies.
- Interconnects distinct Kubernetes clusters into a single ClusterMesh, allowing applications to communicate across all ClusterMesh members.

## Storage
Storage is often the biggest challenge for on-prem setups if not addressed early.  
Stateful applications in Kubernetes require block storage ([Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)) via CSI drivers.  
Backups, especially for databases, need reliable storage.  
Additionally, object storage (e.g., AWS S3 or alternatives) is required.

I recommend procuring storage from a reputable vendor, especially in the beginning.  
Building a storage platform in-house is risky and requires a dedicated maintenance/engineering team.  
Otherwise, [Ceph](https://ceph.io/en/) and [MinIO](https://min.io/) are excellent OSS alternatives to consider.

## Nodes Provisioning and Configuration Management
- For servers, I recommend using the [Debian](https://www.debian.org/) Linux OS distribution.
- Physical servers can be bootstrapped via PXE.
  - Lifecycle management of physical servers can be done with tools like [MaaS](https://maas.io/) and [Foreman](https://theforeman.org/).
  - Bootstrapping can also be achieved through disk cloning from "Golden Images."
- Initial OS configuration can be managed using:
  - [cloud-init](https://cloud-init.io/)
  - [ansible-pull](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html)
- Manual operations are strictly forbidden, except in labs and R&D setups.
- GitOps should be enforced across everything (network, OS, in-cluster Kubernetes configurations (both ops and business)).
- Strict common formatting and linting across all contributors:
  - Whitespace management
  - Indentation
  - Common formatting rules per PL/DSL
- Avoid live patching systems in most cases.
  - Use updated OS images for rolling out updates.

## Kubernetes
- Kubernetes can be set up using several methods, ready installers, or distributions.
- The most important aspect is automatic node registration and node certificate management.
- When possible, use Kubernetes OS distributions such as Talos or Bottlerocket from AWS.
  - These systems offer minimal attack surfaces and only include essential components for running a Kubernetes cluster. Most often, even SSH is unavailable, with administrative communications happening through specialized APIs.
  - These OSs often come with Nvidia drivers support only.
- Despite this, many large enterprises opt for simpler methods to provision Kubernetes clusters, often using minimal bash scripts and [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).
- Other option [RKE2](https://www.rancher.com/products/secure-kubernetes-distribution) from [Rancher](https://www.rancher.com/), but from my experience, at the moment of writing it is not possible to completely automate every aspect of RKE environment setup. Still is quite popular option for on-prem setup within enterprises.

## Security
Security at all levels should be managed and guided by a dedicated SecOps team.
- Access to all applications and clusters must be controlled.
- Kubernetes audit logs must be enabled.
- Access to Kubernetes clusters should be restricted and configurable using tools like [Dex](https://dexidp.io/).
- A central SSO should secure all tools. [Keycloak](https://www.keycloak.org/) or other OpenID proxies are helpful.
- User data in transit and at rest must be protected.
  - This can be achieved through storage encryption and service meshes with mTLS for network encryption.

## Application lifecycle management
- Separate CI from CD, with each managed by dedicated services.
- Central Version Control System - [GitLab](https://about.gitlab.com/).
- CI handles building, testing, and release artifact production.
  - [GitLab CI](https://docs.gitlab.com/ee/ci/).
- CD handles delivery and deployment.
  - [Argo Projects](https://argoproj.github.io/) ecosystem, specifically [ArgoCD](https://argoproj.github.io/cd/).
- [Jenkins](https://www.jenkins.io/) can be used to build automation control-panels for ad-hoc tasks.
- Autoscaling:
  - Kubernetes cluster autoscaling is well-established and achieved through [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) based on various performance metrics.
  - On-prem automation of Kubernetes worker node provisioning is impractical.
  - Increasing Kubernetes cluster capacity must be planned in advance.

### Multi-stage multi-environment deployments
- Achieved using GitOps frameworks like ArgoCD.
- Production servers should be locked and only receive well-tested application versions from lower environments.
- Multi-environment deployments are triggered by application branches, tags, and human operator decisions.
- Deployments occur on different clusters, with environments resembling production conditions.

### PostgreSQL databases
- Initially, I was against running databases on Kubernetes clusters.
- However, I now have extensive experience successfully hosting PostgreSQL databases on Kubernetes, especially with the [CloudNativePG](https://cloudnative-pg.io/) operator.
- Deploying stateful services like PostgreSQL depends on reliable [storage](#storage).

## Observability

### Central Logs collection
Several options exist for centralized log management:
- Loki - stores logs in S3-compatible object storage.
  - I often choose Loki because it integrates well with the Prometheus + Grafana stack.
  - Loki requires less maintenance than ELK but needs precise tuning for large volumes of logs.
- ELK (Elasticsearch, Logstash, Kibana) - a classic log management solution using Elasticsearch as a data store.
  - Requires Elasticsearch expertise for cluster setup and maintenance.
  - Logstash can be replaced by other log processors like Fluentd, turning ELK into EFK.
- Log discovery and delivery agents:
  - Promtail - Loki’s native log delivery agent.
  - Vector - highly performant.
  - Fluentbit / Fluentd.
- [OpenTelemetry collector](#traces-collection) can also act as a log receiver and transporter.

### Metrics scraping
- [Prometheus](https://prometheus.io/) stack includes multiple pluggable components.
- Thanos - Using pure Prometheus in HPC environments is often inefficient, especially for long-term metric storage. Thanos addresses this by enabling object storage (S3 and alternatives).
  - Thanos also helps aggregate metrics across Kubernetes clusters, simplifying data comparison in Grafana.
- [VictoriaMetrics](https://victoriametrics.com/) - an alternative to Prometheus.
  - It's well-reviewed for performance in large environments.
  - Personally, I haven't used it, but I prefer sticking to community best practices.

### Alerting
- [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) - part of the Prometheus stack for metric-based alerting.
- PagerDuty or similar incident management tools should be used for operator communication.

### Traces collection
- [OpenTelemetry](https://opentelemetry.io) [Collector](https://opentelemetry.io/docs/collector/) provides a vendor-neutral implementation for receiving, processing, and exporting telemetry data.
  - It supports receiving observability data from most sources and can export to almost all observability platforms.
- Tracing helps uncover application issues that can’t be caught with low-level metrics.

### Observability UI/UX and interactions
- [Grafana](https://grafana.com/) is the primary choice for visualizing observability data.
  - Grafana supports data from various sources, including metrics, logs, traces, and profiling.
- Specific tools may use their own interfaces (e.g., Kibana for logs if ELK is used).

### Events response and actions
- Runbooks should be created for services, listing actions for specific events.
- Events can trigger actions using various methods such as ChatOps methodology or dedicated events processing frameworks such as [Argo Events](https://argoproj.github.io/events/)
