# AWS Cloud Concepts and Deployment — In-Depth Enterprise Guide

A practical AWS learning guide for Java/Spring Boot developers. It focuses on cloud fundamentals, networking, security, compute, storage, databases, containers, deployment architecture, CI/CD, infrastructure as code, observability, reliability, cost, and enterprise scenarios.

> **Important:** AWS services and console screens evolve. This guide explains durable concepts and common architectures. Confirm current regional availability, quotas, pricing, service limits, and recommended configurations in AWS documentation before implementing production systems.

## 1. Cloud computing fundamentals

### What is cloud computing?
Cloud computing provides on-demand access to compute, storage, networking, databases, and managed services without owning all underlying infrastructure.

### Cloud service models
- **IaaS:** Virtual machines, networks, and storage that you configure and manage. Example: Amazon EC2.
- **PaaS / managed platform:** AWS manages more of the platform while you focus on application deployment. Examples include managed database and application platform services.
- **SaaS:** Complete software delivered as a service. Example: hosted business applications.

### Deployment models
- Public cloud: infrastructure operated by a cloud provider.
- Private cloud: cloud-like infrastructure dedicated to an organization.
- Hybrid cloud: integration of on-premises and cloud environments.
- Multi-cloud: use of multiple cloud providers, usually for specific business or resilience reasons.

### Shared responsibility model
AWS is responsible for security **of** the cloud: physical facilities, hardware, and foundational infrastructure.
The customer is responsible for security **in** the cloud, with the exact split depending on the service:
- Identity, permissions, and credentials.
- Data classification, encryption choices, and access.
- Application code and dependencies.
- Network configuration and security rules.
- Operating system patching on customer-managed compute.
- Configuration of managed services and backups.

Using a managed service reduces operational work, but does not remove customer responsibility for access, data, and configuration.

## 2. AWS global infrastructure

<AsyncImage query="AWS global infrastructure regions availability zones edge locations architecture diagram" aspectRatio="16:9" width="100%" />

### Regions
A Region is a separate geographic area containing multiple Availability Zones. Select a Region based on:
- Customer and data residency requirements.
- Latency to users and dependent systems.
- Required service availability.
- Disaster recovery strategy.
- Cost and operational requirements.

### Availability Zones (AZs)
An AZ consists of one or more discrete data centers with independent infrastructure. Deploying across multiple AZs helps protect against an AZ-level failure.

### Edge locations
AWS edge locations support services such as content delivery and DNS. Amazon CloudFront uses edge locations to cache and deliver content closer to users.

### High availability vs disaster recovery
- High availability: keep the service operating through component/AZ failures.
- Disaster recovery: restore service after a larger regional or data-loss event.

Multi-AZ is not automatically multi-Region disaster recovery. Define RTO and RPO:
- **RTO:** Maximum acceptable time to restore service.
- **RPO:** Maximum acceptable data loss measured in time.

## 3. AWS account and organization structure

Common enterprise structure:
```text
AWS Organization
  ├── Management account
  ├── Security / Audit account
  ├── Log Archive account
  ├── Shared Services account
  ├── Development account
  ├── Test / Staging account
  └── Production account
```

Concepts:
- AWS Organizations: centrally manage multiple accounts.
- Organizational Units (OUs): group accounts by purpose or policy.
- Service Control Policies (SCPs): define maximum permissions available to accounts; they do not grant permissions.
- IAM Identity Center: workforce access and federated sign-in.
- Separate accounts: reduce blast radius and separate billing/security boundaries.

Avoid using the root user for daily work. Protect it with strong authentication and tightly controlled recovery procedures.

## 4. IAM — Identity and Access Management

IAM controls who can access which AWS resources and what actions they can perform.

### Main concepts
- User: identity for a person or workload in older/direct IAM usage; workforce access is often better managed through federation/Identity Center.
- Group: collection of IAM users.
- Role: assumable identity with temporary credentials.
- Policy: JSON document defining allowed or denied actions.
- Trust policy: defines who can assume a role.
- Permission policy: defines what the role can do.
- Permission boundary: maximum permissions an identity-based policy can grant.
- Resource-based policy: permissions attached to a resource, such as an S3 bucket policy.

### Example least-privilege policy (illustrative)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::company-product-files/imports/*"
    }
  ]
}
```

### IAM best practices
- Apply least privilege.
- Prefer roles and temporary credentials over long-lived access keys.
- Use instance profiles/task roles/pod identity mechanisms for workloads.
- Separate deployment permissions from runtime permissions.
- Restrict production access and audit privileged actions.
- Use conditions for account, Region, resource tags, source identity, and network context where appropriate.
- Review unused permissions and rotate credentials when long-lived credentials are unavoidable.
- Never put AWS access keys in source code, Docker images, or frontend applications.

### Authentication vs authorization
- Authentication: establish identity.
- Authorization: decide whether that identity may perform an action.

For a Spring Boot application, application-level user authorization and AWS IAM authorization are separate layers.

## 5. Networking — VPC fundamentals

<AsyncImage query="AWS VPC public private subnets internet gateway NAT gateway route table architecture diagram" aspectRatio="16:9" width="100%" />

A Virtual Private Cloud (VPC) is a logically isolated network in an AWS Region.

### Core components
- CIDR block: IP address range for the VPC/subnets.
- Subnet: IP range associated with one AZ.
- Route table: directs network traffic.
- Internet Gateway (IGW): enables internet connectivity for resources with suitable routes and public addressing.
- NAT Gateway: allows private-subnet resources to initiate outbound internet connections without accepting unsolicited inbound internet connections.
- Security Group: stateful virtual firewall attached to network interfaces/resources.
- Network ACL: stateless subnet-level traffic filter.
- VPC endpoints: private connectivity to supported AWS services.
- VPC peering / Transit Gateway: connect VPC networks.
- VPN / Direct Connect: connect on-premises networks to AWS.

### Public vs private subnet
A subnet is considered public when its route table has a route to an Internet Gateway. A resource generally also needs a public IP and suitable security rules to be reachable from the internet.

Private subnets do not have a direct route to an Internet Gateway. They may use NAT for outbound internet access or VPC endpoints for private access to AWS services.

### Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Scope | Resource/network interface | Subnet |
| State | Stateful | Stateless |
| Rules | Allow rules | Allow and deny rules |
| Return traffic | Automatically allowed for established flows | Must be explicitly allowed |
| Common use | Main workload firewall | Additional subnet-level control |

### Example enterprise VPC layout
```text
VPC: 10.0.0.0/16
  AZ-a                         AZ-b
  Public subnet                Public subnet
    ALB                           ALB
    NAT Gateway                   NAT Gateway

  Private application subnet    Private application subnet
    ECS / EC2 / EKS workloads     ECS / EC2 / EKS workloads

  Private database subnet      Private database subnet
    RDS primary/standby placement across AZs
```

Use multiple AZs for critical workloads. Avoid exposing application servers and databases directly to the public internet.

### DNS and routing
- Route 53: DNS and health-check/routing capabilities.
- Private hosted zones: internal DNS names for VPC-connected environments.
- Load balancer DNS: stable endpoint for application traffic.
- Route tables: control where packets are sent; security groups/NACLs control whether traffic is allowed.

## 6. Compute services

### Amazon EC2
Virtual machines where you control the operating system and installed software.

Use EC2 when:
- You need OS-level control.
- You have legacy software or custom agents.
- You need specialized instance types/configuration.
- You operate software that does not fit managed container/serverless options.

Responsibilities include OS patching, hardening, monitoring, scaling, and application deployment.

### Auto Scaling Groups (ASG)
Maintain a desired number of EC2 instances and scale based on metrics or schedules. Use health checks, launch templates, multiple AZs, and graceful instance draining.

### AWS Lambda
Runs functions in response to events without managing servers directly.

Good for:
- Event-driven tasks.
- Lightweight APIs and integrations.
- File processing triggers.
- Scheduled automation.
- Short-lived background work.

Consider runtime limits, cold starts, concurrency, retries, idempotency, payload limits, and execution duration. Long-running workloads may fit containers or batch services better.

### Elastic Beanstalk
A managed application deployment platform that provisions underlying resources. It can be useful for simpler web application deployments, though many organizations prefer ECS/EKS or platform-specific approaches for standardized container operations.

## 7. Containers — Docker, ECR, ECS, EKS

<AsyncImage query="AWS container deployment architecture ECR ECS Fargate Application Load Balancer diagram" aspectRatio="16:9" width="100%" />

### Docker concepts
- Image: immutable package containing application and runtime dependencies.
- Container: running instance of an image.
- Dockerfile: instructions to build an image.
- Registry: stores and distributes images.
- Volume: persistent or shared storage mount.
- Network: container connectivity.

### Amazon ECR
Elastic Container Registry stores container images. Common flow:
1. Build Docker image.
2. Scan and tag image.
3. Authenticate to ECR.
4. Push image.
5. Deployment service pulls the image.

Use immutable version tags or digests for production releases. Avoid relying only on `latest`.

### Amazon ECS
Elastic Container Service orchestrates containers.
- Cluster: logical grouping of capacity/services.
- Task definition: container configuration, CPU/memory, roles, ports, logging.
- Task: running instance of a task definition.
- Service: maintains desired task count and deployments.
- Launch type/capacity provider: determines where tasks run.
- Fargate: serverless compute for containers.
- ECS on EC2: you manage EC2 capacity; ECS schedules tasks onto it.

### AWS Fargate
Runs containers without managing the underlying EC2 instances. You still manage task sizing, networking, IAM roles, image, scaling, logging, and application behavior.

### Amazon EKS
Managed Kubernetes control plane. You still need to understand Kubernetes workloads, nodes/capacity, networking, IAM integration, upgrades, add-ons, observability, and cluster operations.

### ECS vs EKS
| Area | ECS | EKS |
|---|---|---|
| Orchestrator | AWS-native ECS | Kubernetes |
| Learning curve | Generally simpler for AWS-focused teams | Higher; Kubernetes expertise required |
| Portability | AWS-specific orchestration | Kubernetes ecosystem/portability |
| Operations | Less Kubernetes control-plane complexity | More Kubernetes ecosystem control |
| Best fit | AWS-native container platforms | Existing Kubernetes standards or ecosystem needs |

Choose based on team skills and platform requirements, not popularity alone.

## 8. Load balancing and application traffic

### Elastic Load Balancing types
- Application Load Balancer (ALB): Layer 7 HTTP/HTTPS routing, host/path rules, integrations with container services.
- Network Load Balancer (NLB): Layer 4 TCP/UDP/TLS use cases, high-performance network traffic.
- Gateway Load Balancer: deploy and scale network appliances.

### Typical web request flow
```text
User
  |
Route 53 DNS
  |
CloudFront / WAF (optional)
  |
Application Load Balancer
  |
ECS Service / EC2 Auto Scaling Group
  |
Spring Boot application
  |
RDS / ElastiCache / SQS / other services
```

Configure health checks, TLS certificates, target deregistration delay, timeouts, and security group rules. A load balancer health check should test readiness, not merely that a process exists.

## 9. Storage services

### Amazon S3
Object storage for files and unstructured data:
- Supplier files and exports.
- Images and documents.
- Backups and logs.
- Static website assets.
- Data lake objects.

Concepts:
- Bucket and object key.
- Versioning.
- Lifecycle rules.
- Storage classes.
- Encryption.
- Bucket policies and IAM.
- Replication.
- Multipart upload.
- Event notifications.
- Presigned URLs.

S3 is object storage, not a traditional POSIX file system or relational database.

### S3 security checklist
- Block public access unless a deliberate public use case exists.
- Use bucket policies and least-privilege roles.
- Enable encryption at rest.
- Enable versioning for important data.
- Use lifecycle policies to control cost.
- Log/audit access where required.
- Consider replication and backup requirements.
- Prevent accidental deletion for critical data.

### EBS vs EFS vs S3
| Service | Type | Typical use |
|---|---|---|
| EBS | Block storage | EC2 boot/data volumes |
| EFS | Managed shared file system | Shared Linux file access across instances |
| S3 | Object storage | Files, media, backups, data lake |

## 10. Database services

### Amazon RDS
Managed relational databases such as PostgreSQL, MySQL, MariaDB, Oracle, and SQL Server (availability varies by engine/Region).

Features may include:
- Automated backups and point-in-time recovery.
- Multi-AZ deployment options.
- Read replicas (engine/deployment dependent).
- Encryption.
- Monitoring and maintenance windows.
- Parameter and option groups.

Multi-AZ primarily supports availability/failover; it is not the same as a read-scaling read replica.

### Amazon Aurora
AWS relational database compatible with MySQL or PostgreSQL. It separates compute and distributed storage architecture and offers managed availability/scaling features. Validate engine compatibility and feature differences before migration.

### Amazon DynamoDB
Managed NoSQL key-value/document database.
- Partition key and optional sort key.
- Provisioned or on-demand capacity modes.
- Secondary indexes.
- TTL.
- Streams.
- Global tables.
- Conditional writes and transactions.

Design access patterns before table structure. A relational normalization mindset does not translate directly to DynamoDB.

### ElastiCache
Managed in-memory caching using Redis OSS or Memcached offerings (current product naming/options may evolve). Use for caching, session data, and suitable low-latency access patterns. Define TTL, invalidation, consistency, and cache-failure behavior.

### Database deployment decisions
Consider:
- Multi-AZ availability.
- Backup retention and restore tests.
- Encryption and key management.
- Connection pool limits.
- Schema migration strategy.
- Read/write workload.
- Cross-Region recovery.
- Data residency and retention.
- Database maintenance and version upgrades.

## 11. Messaging and integration services

### Amazon SQS
Managed message queue:
- Standard queues: high throughput and at-least-once delivery; duplicates and ordering variation are possible.
- FIFO queues: ordering and deduplication features within defined limits and configuration.

Concepts:
- Visibility timeout.
- Message retention.
- Long polling.
- Dead-letter queue (DLQ).
- Redrive policy.
- Message group ID (FIFO).
- Idempotent consumer.

### Amazon SNS
Pub/sub notification service. A topic can fan out messages to multiple subscribers, including queues, endpoints, and other integrations.

### Amazon EventBridge
Event bus and routing service for events from AWS services, custom applications, and supported SaaS integrations. Useful for event routing and rules-based integration.

### SQS vs SNS vs EventBridge vs Kafka
| Service | Primary model | Good fit |
|---|---|---|
| SQS | Queue | Buffer work for consumers |
| SNS | Pub/sub fan-out | Notify multiple subscribers |
| EventBridge | Event bus/routing | Route events based on event patterns |
| Kafka / Amazon MSK | Partitioned event log/stream | High-throughput streams, replay, multiple consumer groups |

Choose based on replay, ordering, throughput, retention, operational requirements, and integration ecosystem.

## 12. Security services and encryption

### AWS KMS
Key Management Service manages cryptographic keys and key policies. Integrates with services such as S3, EBS, RDS, and application encryption workflows.

### Secrets Manager and Systems Manager Parameter Store
Store application secrets/configuration centrally. Choose based on rotation, secret lifecycle, access patterns, and cost/feature requirements.

Do not bake secrets into:
- Git repositories.
- Docker images.
- AMIs.
- Frontend bundles.
- Plain-text user data or logs.

### AWS WAF and Shield
- WAF: web application firewall rules for HTTP(S) traffic.
- Shield: DDoS protection services.

### CloudTrail
Records AWS account/API activity for audit and investigation. Configure organization/account-level collection and protect log destinations.

### GuardDuty, Security Hub, Inspector
Security services that help detect threats, aggregate findings, and assess vulnerabilities. They complement—not replace—secure architecture and patching.

## 13. Monitoring and observability

### Amazon CloudWatch
Common capabilities:
- Metrics.
- Logs.
- Alarms.
- Dashboards.
- Log Insights.
- Container and service monitoring integrations.

For Spring Boot, expose Actuator metrics and integrate with a supported metrics pipeline. Avoid exposing Actuator endpoints publicly without access controls.

### AWS X-Ray and distributed tracing
Trace requests across services to understand latency and dependencies. OpenTelemetry is also widely used; choose a consistent tracing approach and propagate trace context across HTTP and messaging.

### Useful production metrics
Application:
- Request rate, error rate, p50/p95/p99 latency.
- JVM heap, GC, thread pools.
- Connection pool usage.
- Business errors and job failures.

Infrastructure:
- CPU, memory, disk, network.
- ALB target health and response time.
- ECS task restarts / desired vs running count.
- RDS connections, CPU, storage, replica lag.
- SQS queue depth and oldest message age.
- Kafka consumer lag if using Kafka/MSK.

Create alerts around user impact and actionable thresholds. Avoid alerting on every metric fluctuation.

## 14. Deploying a Spring Boot application to AWS

### Option A: EC2 deployment

```text
Internet
  |
ALB
  |
EC2 Auto Scaling Group (private subnets)
  |
Spring Boot JAR / system service
  |
RDS (private database subnets)
```

Steps:
1. Build and test the JAR.
2. Provision VPC, subnets, routes, security groups, ALB, and EC2.
3. Create an IAM instance role with least privilege.
4. Install/configure the supported Java runtime.
5. Deploy the JAR using a controlled process.
6. Configure externalized properties and secrets.
7. Run as a non-root OS user with a service manager.
8. Configure logs, metrics, health checks, and alarms.
9. Automate deployment and rollback.

Trade-off: you manage OS patching, runtime, scaling, and instance lifecycle.

### Option B: ECS Fargate deployment (common container path)

<AsyncImage query="AWS ECS Fargate deployment architecture ALB ECR private subnets RDS diagram" aspectRatio="16:9" width="100%" />

```text
Developer
   |
CI pipeline
   | build/test/scan
Docker image
   |
Amazon ECR
   |
ECS Service (Fargate tasks in private subnets)
   |
Application Load Balancer
   |
Spring Boot API
   |
RDS / S3 / SQS / Kafka
```

Steps:
1. Create a production-ready Dockerfile.
2. Build and test the image in CI.
3. Scan dependencies and image.
4. Push a versioned image to ECR.
5. Define ECS task definition: image, CPU/memory, ports, logs, environment, secrets, roles.
6. Create ECS service in private subnets.
7. Attach ALB target group and health check.
8. Configure task auto scaling.
9. Configure CloudWatch logs/metrics/alarms.
10. Roll out a new task definition and verify health.

### Example Dockerfile (multi-stage)
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
COPY src ./src
RUN mvn -B -DskipTests=false package

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 10001 appuser
COPY --from=build /workspace/target/*.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

Use approved base images, pin versions/digests where appropriate, scan images, and avoid copying build tools into the runtime image. Adapt Maven/JDK versions to the project.

### ECS task role vs task execution role
- Task execution role: used by ECS/Fargate agent for tasks such as pulling ECR images and sending logs.
- Task role: credentials available to the application container for calls to AWS services.

Do not grant application S3 access to the execution role when the application itself needs that access; use the task role.

### Option C: EKS deployment
Use EKS when Kubernetes capabilities, ecosystem, or organizational standards justify the additional operational complexity.

Typical flow:
1. Build and scan container image.
2. Push to ECR.
3. Deploy Kubernetes manifests/Helm chart.
4. Configure Deployment, Service, Ingress/Gateway, HPA, probes, resource requests/limits.
5. Integrate IAM with workloads using a supported pod identity mechanism.
6. Configure cluster logging, metrics, tracing, upgrades, and node capacity.

### Option D: Lambda
Package a function or supported container image, configure execution role, event source, concurrency, timeout, memory, logs, and retry/DLQ behavior. Use for suitable event-driven workloads, not as a default replacement for every Spring Boot service.

## 15. CI/CD pipeline for AWS

### Typical pipeline
```text
Git push / Pull Request
       |
Compile + Unit Tests
       |
Static Analysis + Dependency Scan
       |
Build Docker Image
       |
Container/Image Scan
       |
Push Image to ECR
       |
Deploy to Dev
       |
Integration / Smoke Tests
       |
Approval / Policy Gate
       |
Deploy to Production
       |
Health Verification
       |
Rollback if required
```

### AWS deployment tools
- CodePipeline: pipeline orchestration.
- CodeBuild: managed build execution.
- CodeDeploy: deployment automation for supported compute targets and strategies.
- CloudFormation: AWS infrastructure as code.
- CDK: define infrastructure using programming languages.
- Systems Manager: fleet and operational management.
- External CI tools such as Jenkins or GitHub Actions can also deploy to AWS using federated roles.

### CI/CD security
- Prefer OIDC/federated role assumption over long-lived cloud keys in CI.
- Separate build and deployment roles.
- Restrict production deployment permissions.
- Scan dependencies and images.
- Sign/verify artifacts where required.
- Store secrets in managed secret stores.
- Record deployment identity and artifact digest.

## 16. Infrastructure as Code (IaC)

IaC defines cloud resources in version-controlled, repeatable configuration.

### AWS CloudFormation
AWS-native declarative templates for infrastructure.

### AWS CDK
Define infrastructure using familiar programming languages; CDK synthesizes CloudFormation templates.

### Terraform
Multi-provider IaC tool widely used for AWS. Uses providers, resources, modules, variables, outputs, and state.

### Terraform example (illustrative)
```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "product_imports" {
  bucket = "example-company-product-imports"
}
```

This is only a minimal example. Production infrastructure needs naming strategy, encryption, public access blocking, versioning, lifecycle rules, logging, ownership, and state management.

### IaC best practices
- Keep infrastructure code in Git.
- Review plans before applying.
- Use reusable modules.
- Separate environments/accounts.
- Store Terraform state remotely with locking and encryption using a supported backend design.
- Do not store secrets in state unless protected and necessary.
- Detect drift.
- Avoid manual console changes that are not reconciled to code.
- Use policy checks and automated tests.

## 17. Deployment strategies

### Rolling deployment
Gradually replace old instances/tasks with new ones. Ensure enough capacity and backward compatibility.

### Blue/green
Run old and new environments side by side, then switch traffic. Useful for fast rollback, but costs more temporarily and requires compatible data changes.

### Canary
Send a small percentage of traffic to the new version, observe metrics, then increase gradually.

### Feature flags
Deploy code while controlling feature exposure separately. Feature flags need ownership, expiry, and cleanup.

### Database migration safety
Use expand-and-contract:
1. Add new nullable column/table/index.
2. Deploy code compatible with old and new schema.
3. Backfill data.
4. Switch reads/writes.
5. Remove old schema only after all consumers are migrated.

Avoid deploying a destructive schema change before all old application instances have stopped using it.

## 18. High availability, scaling, and resilience

### Multi-AZ design
- ALB across multiple AZs.
- ECS tasks/EC2 instances spread across AZs.
- RDS Multi-AZ where appropriate.
- NAT Gateway design that avoids a single-AZ dependency for critical workloads.
- S3 and managed services with their documented durability/availability characteristics.

### Scaling types
- Vertical scaling: increase instance/task size.
- Horizontal scaling: add instances/tasks.
- Scheduled scaling: scale around predictable demand.
- Target tracking: scale based on a metric.
- Queue-depth scaling: add workers as backlog grows.

### Stateless application design
Keep session/state outside replaceable application tasks where possible. Use a database, cache, or token-based approach. Store files in S3 rather than local container storage.

### Graceful shutdown
On deployment/termination:
- Stop accepting new work.
- Allow in-flight requests to complete within a deadline.
- Stop polling new messages.
- Finish or safely abandon work with idempotent retry.
- Close database and messaging clients.
- Ensure load balancer deregistration and task shutdown timing are aligned.

## 19. Cost management and FinOps

Major cost drivers:
- EC2/ECS/EKS compute.
- NAT Gateway processing and hourly cost.
- Data transfer, especially cross-AZ and internet egress.
- RDS instance/storage/backup.
- S3 storage and requests.
- Load balancers.
- CloudWatch logs/metrics retention and ingestion.
- Kafka/MSK or other managed messaging capacity.

Cost controls:
- Right-size compute and databases.
- Use autoscaling with safe minimum capacity.
- Set log retention.
- Apply S3 lifecycle policies.
- Review NAT and data-transfer paths.
- Use Savings Plans/Reserved Instances when workload commitments are understood.
- Use Spot for fault-tolerant workloads.
- Tag resources by owner, application, environment, and cost center.
- Use budgets, Cost Explorer, and anomaly detection.
- Remove unused resources and old snapshots/images under retention policy.

Do not optimize cost by removing redundancy from critical systems without understanding availability impact.

## 20. Enterprise case study — Retail PIM on AWS

### Requirements
- Spring Boot APIs for product, category, and pricing.
- React web application.
- Supplier file imports.
- Product images/documents.
- Kafka events for product changes.
- Search indexing and channel publishing.
- High availability and controlled deployments.

### Reference architecture
```text
Users
  |
Route 53
  |
CloudFront + WAF (optional)
  |
Application Load Balancer
  |
ECS Fargate Services (private subnets, multiple AZs)
  |          |             |
Catalog    Pricing       Import
  |          |             |
RDS       RDS/DB         S3 files
  |
Outbox publisher
  |
Kafka / Amazon MSK
  |                  |
Search consumer   Publishing consumer
  |                  |
Search store       Channel APIs

Supporting:
IAM roles | Secrets Manager | KMS | CloudWatch | X-Ray/OpenTelemetry
ECR | CI/CD | Terraform/CDK/CloudFormation
```

### Example request flow
1. React client calls Catalog API through ALB.
2. Catalog validates the request and writes product data to RDS.
3. Catalog writes an outbox event in the same DB transaction.
4. Publisher sends event to Kafka/MSK.
5. Search consumer updates the search index.
6. Publishing consumer evaluates channel eligibility and calls channel endpoints.
7. Images are stored in S3; API returns authorized object references or presigned URLs.

### Design considerations
- Put ALB in public subnets and application tasks in private subnets.
- Keep RDS in isolated/private database subnets.
- Use ECS task role for application access to S3/Secrets Manager as needed.
- Store DB credentials in Secrets Manager and rotate according to policy.
- Use KMS encryption and least-privilege bucket policies.
- Use outbox/idempotency for reliable events.
- Configure multi-AZ capacity and health checks.
- Set RTO/RPO and test backup restoration and regional recovery.
- Monitor request latency, DB connections, consumer lag, and import backlog.

## 21. Common AWS deployment failures

| Symptom | Common causes to investigate |
|---|---|
| ALB returns 502/503 | Target health, port mapping, app startup, security group, health check path |
| ECS task keeps restarting | Application crash, bad config/secret, memory limit, failed health check |
| ECS cannot pull image | ECR permissions, execution role, network path, image tag/digest |
| App cannot connect to RDS | Subnet routes, security group, DNS, credentials, DB availability, connection limits |
| App cannot access S3 | Task role permissions, bucket policy, KMS key policy, endpoint/network |
| Secrets not available | IAM permissions, secret ARN/Region, KMS permissions, task definition config |
| Lambda times out | Downstream latency, VPC networking, insufficient memory/CPU, timeout setting |
| Deployment appears healthy but API fails | Health check too shallow, missing dependencies, wrong route/config |
| High AWS bill | NAT/data transfer, overprovisioned compute, logs, snapshots, idle resources |

Debug in layers:
1. DNS and endpoint.
2. Load balancer listener and target health.
3. Security groups and routes.
4. Container/task/instance state and logs.
5. IAM role and resource policy.
6. Application configuration and dependency connectivity.
7. Database, broker, and downstream health.
8. Recent deployment and infrastructure changes.

## 22. Production readiness checklist

### Architecture
- [ ] Workload and service boundaries are clear.
- [ ] Availability Zones and failure modes are understood.
- [ ] RTO/RPO are documented.
- [ ] Network paths are private wherever possible.
- [ ] Scaling limits and quotas are reviewed.

### Security
- [ ] Least-privilege IAM roles.
- [ ] No long-lived credentials in code/images.
- [ ] TLS and encryption at rest.
- [ ] Secrets are centrally managed.
- [ ] Security groups are restrictive.
- [ ] Logs do not expose secrets or sensitive payloads.
- [ ] Audit logging and vulnerability scanning are enabled.

### Deployment
- [ ] Immutable versioned image/artifact.
- [ ] CI tests and security scans.
- [ ] Health checks and graceful shutdown.
- [ ] Backward-compatible database migrations.
- [ ] Rollback plan.
- [ ] Deployment approval/audit trail.

### Operations
- [ ] Logs, metrics, and traces.
- [ ] Actionable alarms and dashboards.
- [ ] Backup and restore tests.
- [ ] Capacity and cost monitoring.
- [ ] Incident runbooks.
- [ ] Patch and upgrade plan.

## 23. AWS interview questions

### Cloud and infrastructure
1. Explain IaaS, PaaS, and SaaS.
2. Explain AWS Region, AZ, and edge location.
3. What is the shared responsibility model?
4. What is the difference between high availability and disaster recovery?
5. Explain RTO and RPO.
6. What is an AWS account and why use multiple accounts?

### IAM and networking
7. IAM user vs role?
8. Identity policy vs resource policy?
9. What is a trust policy?
10. What is least privilege?
11. What is a VPC?
12. Public vs private subnet?
13. Internet Gateway vs NAT Gateway?
14. Security Group vs Network ACL?
15. What is a VPC endpoint?
16. How do VPC peering and Transit Gateway differ?

### Compute and containers
17. EC2 vs Lambda?
18. ECS vs EKS?
19. ECS task role vs execution role?
20. What is Fargate?
21. What is an ECR repository?
22. How does an ALB route traffic to ECS tasks?
23. How do container health checks work?

### Storage and databases
24. S3 vs EBS vs EFS?
25. S3 versioning and lifecycle policies?
26. RDS Multi-AZ vs read replica?
27. RDS vs DynamoDB?
28. What is ElastiCache?
29. How do you secure S3 access?

### Deployment and operations
30. Explain a CI/CD pipeline to ECS.
31. How do you deploy without downtime?
32. Rolling vs blue/green vs canary?
33. How do you store secrets?
34. How do you monitor Spring Boot on AWS?
35. How do you troubleshoot ALB 502 errors?
36. How do you troubleshoot ECS task startup failure?
37. How do you reduce AWS cost?
38. How do you design a multi-AZ application?
39. How do you implement disaster recovery?
40. How would you deploy a Spring Boot + React + Kafka system on AWS?

## 24. Hands-on exercises

### Foundations
1. Create an AWS account safely and configure MFA.
2. Create a VPC with public and private subnets across two AZs.
3. Configure route tables, Internet Gateway, NAT, and security groups.
4. Create an IAM role with narrowly scoped S3 read access.
5. Create an S3 bucket with encryption, versioning, and lifecycle rules.

### Compute and deployment
6. Deploy a Spring Boot JAR to EC2 behind an ALB.
7. Containerize the application.
8. Push the image to ECR.
9. Deploy to ECS Fargate in private subnets.
10. Configure ALB health checks and ECS service scaling.
11. Store DB credentials in Secrets Manager and inject them securely.
12. Configure CloudWatch logs and alarms.

### Data and integration
13. Connect ECS to private RDS.
14. Upload supplier files to S3.
15. Trigger a Lambda function from an S3 event.
16. Send background work through SQS and implement a DLQ.
17. Publish domain events to Kafka/MSK and monitor consumer lag.

### Automation and resilience
18. Build a CI/CD pipeline that deploys a versioned ECR image.
19. Provision VPC and ECS infrastructure using Terraform or CDK.
20. Perform a rolling deployment and rollback.
21. Simulate an AZ/task failure and verify service recovery.
22. Restore an RDS backup in a test environment.
23. Set up a budget and identify the top cost drivers.
24. Write an operational runbook for an application outage.

## 25. Suggested 30-day AWS learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | Cloud basics, Regions, AZs, accounts | AWS architecture notes |
| 4–7 | IAM, VPC, subnets, routing, security | Two-AZ network |
| 8–10 | EC2, ASG, ALB | Spring Boot on EC2 |
| 11–14 | Docker, ECR, ECS, Fargate | Containerized API on ECS |
| 15–17 | S3, RDS, DynamoDB, ElastiCache | API connected to managed data |
| 18–20 | SQS, SNS, EventBridge, Kafka/MSK | Async workflow |
| 21–23 | IAM security, KMS, Secrets Manager | Secure deployment |
| 24–26 | CloudWatch, tracing, alarms | Production dashboard |
| 27–28 | CI/CD and IaC | Automated deployment |
| 29–30 | HA, DR, cost and capstone | Architecture and runbook |

## 26. Capstone — Deploy an Enterprise PIM Platform on AWS

### Application components
- React frontend.
- Spring Boot Product Catalog API.
- Spring Boot Pricing API.
- Spring Boot Import Worker.
- PostgreSQL on RDS.
- S3 for supplier files and product images.
- Kafka/MSK for product events.
- Search projection service.
- ECS Fargate for container workloads.
- ALB, Route 53, and optional CloudFront/WAF.
- IAM, KMS, Secrets Manager.
- CloudWatch and distributed tracing.
- ECR and CI/CD.
- Terraform/CDK/CloudFormation.

### Acceptance criteria
- API is reachable through a secure HTTPS endpoint.
- Application tasks run in private subnets across multiple AZs.
- Database is not publicly accessible.
- Workloads use IAM roles, not embedded credentials.
- Product data and outbox event intent commit atomically.
- Consumers handle duplicate events.
- Logs, metrics, traces, and alarms are available.
- Deployment is repeatable from pipeline/IaC.
- A failed deployment can be rolled back.
- Backup restoration and failure recovery are tested.
- Cost ownership and budgets are defined.

---

## Official AWS documentation
- AWS documentation: https://docs.aws.amazon.com/
- AWS Well-Architected Framework: https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html
- Amazon VPC: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
- IAM: https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- Amazon ECS: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
- Amazon EKS: https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- Amazon S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Amazon RDS: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html
- AWS Lambda: https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
- Amazon CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- AWS CloudFormation: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
