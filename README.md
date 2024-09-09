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
Multi-tenant HPC Platform is built on-premises and exists as a secure and private alternative to public cloud providers.  
Core management framework for applications running on top of Platform is kubernetes.  
Kubernetes provides extremely flexible applications lifecycle management as well as provides tooling for building reliable and highly available modern-age platforms.

## Networking
Physical Networking should be designed by professional Network Architects.  
Network should be flexible, secure, auditable, allow isolated multi-tenant environments setups.
For Kubernetes clusters networking I would recommend using [Cilium](https://cilium.io/) [CNI](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

Cilium provides robust and flexible networking "driver" for kubernetes clusters.
- Provides great security and observability options by utilizing modern Linux Kernel technology [eBPF](https://ebpf.io/)
- Enabled Kubernetes Network Policies engine, or is able to replace it with much more advanced Cilium Network Policies
- Ability to interconnect distinct kubernetes clusters in a single ClusterMesh making applications running on their clusters to be accessible by all Cilium ClusterMesh members.

## Storage
Storage could be the biggest pain for on-premise setups if not addressed accordingly.  
Stateful applications running in Kubernetes require block storage ([Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) mounting) available via CSI drivers.  
Backups (especially for DataBases) require reliable storage.  
Also object storage (AWS S3 and similar alternatives) is required for many applications.  

I would recommend purchasing well known Storage Systems vendor, especially for beginning.  
Building Storage Platform on your own is risky and requires dedicated maintenances/engineering team.  
Otherwise [Ceph](https://ceph.io/en/) and [MinIO](https://min.io/) are two great OSS options to consider.

## Nodes Provisioning and Configuration Management
- As operating systems for servers I would recommend [Debian](https://www.debian.org/) Linux OS distribution
- Physical servers can be bootstrapped using PXE
  - Physical servers lifecycle managedment can be achieved with tools such as [MaaS](https://maas.io/) and [Foreman](https://theforeman.org/)
  - Bootstrapping can also be achieved using disks clonning from "Golden Images"
- Initial operating system configuration can be managed using:
  - [cloud-init](https://cloud-init.io/)
  - [ansbile-pull](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html)
- Any manual operation is strictly forbidden
  - Except labs and RnD setups
- GitOps all the way for everything (network, os, kubernetes)
- Strict common formatting and linting across all contributors
  - whitespaces management
  - indendations
  - common formatting rules per PL and/or DSL
- Systems live patching should be avoided for the most cases
  - Use updated OS images to rollout updates

## Kubernetes
- Kubernetes setup can be achieved using multiple available setup methods, ready installers, or distributions
- Most important any such system should provide automatic node registration and node certificates management.
- If it is possible better to use Kubernetes OS distritubions such as Talos, or bottlerocket from AWS.
  - Such systems provide very minimal attack surface. Systems consists just of few core components absolutely crucial for running a Kubernetes cluster. Most of the times even SSH is not available on such systems and all administrative communications happen over specialized APIs.
  - Most of the times such OSs have Nvidia built-in drivers only
- At the same time, based on my experience with big known corporations many of them use simplistic methods of provisioning their kubernetes clusters using minimal bash scripts and [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## Security
Security on all levels should be managed and guided by a dedicated SecOps team.
- Access to all applications and clusters should controlled
- Audit Logs for kubernetes clusters should be enabled
- Access to kubernetes clusters should be limited and configurable. Using for example [Dex](https://dexidp.io/) and 
- Central SSO should be in front of all tools. [Keycloak](https://www.keycloak.org/) and other OpenID proxies can be helpfull here.
- User's data in network should be protected in transit and at rest
  - This can be achieved using storage encryption
  - And service meashes for traffic encryption in transit using mTLS

## Application lifecycle management
- Splitting CI/CD into CI and CD where each part is managed by a dedicated service
- Central Version Control System - [Gitlab](https://about.gitlab.com/)
- CI - building, testing, release artifacts production
  - [Gitlab CI](https://docs.gitlab.com/ee/ci/)
- CD - delivery, deploy
  - [Argo Projects](https://argoproj.github.io/) ecosystem, specifically [ArgoCD](https://argoproj.github.io/cd/)
- Autoscalling
  - Autoscalling withing kubernetes cluster is achived easily and is well established practice
  - It is achieved using [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (Horizontal Pod Autoscaling) based on various performance metrics
  - At the same time, on-premises, it is impossible to fully automate Kubernetes worker nodes automatic provisioning
  - Increasing existing Kubernetes clusters capacity should be planned beforehand

### Multi-stage multi-environment deployments
- Multi-environment deployments achieved using GitOps frameworks such as ArgoCD.
- Production servers usually should be locked and receive well tested application's versions from the lower environments.
- Multi-environment deployments are triggered based on applications branches, tags, and human-operator decisions.
- Multi-environment deployments are hosted on different clusters which dependeing on environment resemble production environment.

### PostgreSQL databases
- Initially since Kubernetes has started I was against running DataBases workloads on kubernetes clusters
- But for today I have big successfull experience with hosting PostgreSQL databases on kubernetes clusters. Especially happy with [CloudNativePG](https://cloudnative-pg.io/) operator.
- Deploying such critical and stateful service in kubernetes clusters depends on reliable [storage](#storage) setup.

## Observability

### Central Logs collection
There are several options for central logs management:
- Loki - stores logs in S3 compatible object storage.
  - I choose this option most of the times, because it well integrates in Prometheus + Grafana stack.
  - Loki requires less maintenance compared to ELK, but huge logs volumes required precise performance tweaking.
- ELK - Elasticsearch, Logstash, Kibana - classic central logging management system using ElasticSearch as a data storage.
  - Requires ElasticSeach clusters setup and maintenance expertise
  - Logstash can be replaced by any other log processing and aggregation service. E.g. Fluend, in that case ELK becomes EFK
- Logs discovery and delivery agents
  - Promtail - Loki native logs delivey agent
  - Vector - performant
  - Fluentbit
  - Fluentd
- [OpenTelemetry collector](#traces-collection) can be used as a logs receiver and transporter too

### Metrics scraping
- [Prometheus](https://prometheus.io/) stack consists of multiple pluggble components
- Thanos - using pure Prometheus for HPC environments is often not effective, especially when massive volume of metrics should be stored for longer periods of time. Thanos for help - thanos works on top of Prometheus and allows storing metrics data in Object Stores (S3 and compatible alternatives)
  - Also thanos greatly helps with metrics aggregation from multiple kubernetes clusters, makes it easy to compare data in single interface such as Grafana.
- [VictoriaMetrics](https://victoriametrics.com/) as alternative to Prometheus
  - Lots of positive reviews for its performance especially in huge environments
  - Personally I had no chance to use it, and usually I'm trying to stay compatible with community best-practices.

### Alerting
- [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) - for alerting based on metrics. Part of Prometheus stack
- PagerDuty or similar incident management product for efficient and shift-based communications with responsible operators.

### Traces collection
- [OpenTelemetry](https://opentelemetry.io) [Collector](https://opentelemetry.io/docs/collector/)
  - The OpenTelemetry Collector offers a vendor-agnostic implementation of how to receive, process and export telemetry data.
  - It support receiving observability data from most sources, process, and send data to almost all know Observability Products.
- Traces help to discover issues and events within application which are usually difficult to catch using low level metrics or

### Observability UI/UX and interactions
- [Grafana](https://grafana.com/) - is the main and basically only candidate in this category
  - Grafana is able to present data for multitude of sources from variaous categories such as Metrics, Logs, Traces and Profiling.
- Various specific tools might use their own specific interfaces. For example if ELK stack is chosen for central logs management - Kibana is used to browse logs.

### Events response and actions
- Runbooks - runbooks should be created for services with list of actions based on specific event
