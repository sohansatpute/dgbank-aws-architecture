# Networking Services

Services that move traffic into, out of, and between the DGBank environments.

---

### Amazon Route 53

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A highly available and scalable Domain Name System (DNS) web service that translates domain names into IP addresses and routes end-user requests to the appropriate endpoint. |
| **Purpose in DGBank** | Provides reliable DNS resolution for all DGBank endpoints and enables health-check-based routing to maintain service availability. |
| **Position in architecture** | Entry point of every inbound request. Operates at the edge layer, before CloudFront. |

---

### Amazon CloudFront

| Field | Description |
|---|---|
| **Category** | Networking / Edge |
| **Service overview** | A global content delivery network (CDN) that caches content at edge locations to reduce latency and offload traffic from origin servers. |
| **Purpose in DGBank** | Accelerates content delivery for end users in Japan, reduces load on backend services, and serves as the integration point for AWS WAF and Shield at the edge. |
| **Position in architecture** | Second layer of inbound request processing. Sits between Route 53 and AWS WAF. |

---

### AWS WAF (Web Application Firewall)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Service overview** | A web application firewall that monitors and filters HTTP(S) requests based on configurable rules to protect against common web exploits. |
| **Purpose in DGBank** | Protects all public-facing endpoints from SQL injection, cross-site scripting, credential stuffing, and other OWASP Top 10 threats before they reach the application. |
| **Position in architecture** | Third layer of inbound request processing, immediately after CloudFront. |

---

### AWS Shield Advanced

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Service overview** | A managed Distributed Denial of Service (DDoS) protection service that provides enhanced mitigation, real-time visibility, and access to the AWS DDoS Response Team (DRT). |
| **Purpose in DGBank** | Protects against volumetric and application-layer DDoS attacks, with cost protection for attack-driven traffic spikes. |
| **Position in architecture** | Always-on protection layer across CloudFront, Route 53, and Application Load Balancers. |

---

### Amazon VPC (Virtual Private Cloud)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A logically isolated virtual network within AWS where resources are provisioned with full control over IP addressing, subnets, route tables, and gateways. |
| **Purpose in DGBank** | Provides network isolation between the three environments (User, Business, Future Tech) to limit blast radius and enforce regulatory separation. |
| **Position in architecture** | Foundational layer. All compute and data services reside within a VPC. |

---

### Public and Private Subnets

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | Subnet segments within a VPC that control whether resources are reachable from the internet (public) or isolated from it (private). |
| **Purpose in DGBank** | Ensures that only internet-facing components such as load balancers and NAT Gateways are exposed, while application servers and databases remain inaccessible from the internet. |
| **Position in architecture** | Public subnets handle ingress; private subnets host ECS Fargate, Aurora, ElastiCache, and Lambda. |

---

### AWS Transit Gateway

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A central hub that connects multiple VPCs and on-premises networks through a single gateway, simplifying network topology. |
| **Purpose in DGBank** | Enables controlled communication between the User, Business, and Future Tech VPCs without requiring complex VPC peering. |
| **Position in architecture** | Backbone for east-west traffic between DGBank environments and Future Tech operations. |

---

### NAT Gateway

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A managed service that enables resources in private subnets to initiate outbound connections to the internet while preventing inbound connections from the internet. |
| **Purpose in DGBank** | Allows ECS Fargate and Lambda functions to call external APIs such as Anser BizSol, HENNGE, and Accrete without exposing them to inbound internet traffic. |
| **Position in architecture** | Egress path for private-subnet services requiring external connectivity. |

---

### Internet Gateway (IGW)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A horizontally scaled, redundant, and highly available VPC component that allows communication between a VPC and the internet. |
| **Purpose in DGBank** | Enables public-facing resources to accept inbound traffic from end users. |
| **Position in architecture** | Attached to each VPC that hosts internet-facing resources. |

---

### VPC Endpoint (S3)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Service overview** | A private connection between a VPC and an AWS service that keeps traffic within the AWS network instead of traversing the public internet. |
| **Purpose in DGBank** | Enables secure, cost-effective access to Amazon S3 from ECS and Lambda without NAT Gateway charges or public exposure. |
| **Position in architecture** | Used for all S3 read and write operations from private subnets. |

---

### VPC Flow Logs

| Field | Description |
|---|---|
| **Category** | Networking / Monitoring |
| **Service overview** | A feature that captures IP traffic information to and from network interfaces within a VPC, delivered to CloudWatch Logs or S3. |
| **Purpose in DGBank** | Provides a network-level audit trail for security investigations and regulatory compliance. Used to monitor external API communication. |
| **Position in architecture** | Passive monitoring layer across all VPCs. |

---

### Internal Application Load Balancer (ALB)

| Field | Description |
|---|---|
| **Category** | Networking / Compute |
| **Service overview** | A load balancer that distributes incoming application traffic across multiple targets, such as ECS tasks, within a VPC. |
| **Purpose in DGBank** | Provides high availability and even traffic distribution to ECS Fargate tasks. Health checks remove unhealthy targets automatically. |
| **Position in architecture** | Between API Gateway and ECS Fargate. Internal-facing only. |

---

### AWS Network Firewall (NWFW)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Service overview** | A managed network firewall and intrusion detection and prevention service for VPCs, with fine-grained control over inbound and outbound traffic. |
| **Purpose in DGBank** | Provides egress traffic inspection and control to ensure only approved external endpoints are reachable. |
| **Position in architecture** | Between private subnets and NAT Gateway/IGW. Currently under evaluation. |

---

### DNS Firewall (DNSFW)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Service overview** | A Route 53 Resolver feature that filters outbound DNS queries based on domain lists, blocking resolution of malicious or unapproved domains. |
| **Purpose in DGBank** | Alternative or complement to Network Firewall for egress control, with lower operational complexity. |
| **Position in architecture** | Applied to Route 53 Resolver queries originating from the VPC. Currently under evaluation. |

---