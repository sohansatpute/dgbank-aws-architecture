# Networking Services

Services that move traffic into, out of, and between the DGBank environments.

---

### Amazon Route 53

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | The internet's directory service. It converts a domain name like `dgbank.jp` into the numeric address a computer needs. |
| **Why DGBank uses it** | Customers and staff reach the platform by name, not IP address. Route 53 provides reliable, low-latency DNS resolution with health checks that can redirect traffic away from unhealthy endpoints. |
| **Where it sits in the flow** | First service touched in every inbound journey (login, balance, transfer, upload). See `docs/03-request-journeys.md`. |
| **Real-world analogy** | The contact list on your phone. You tap "Tanaka-san" and the phone figures out the number. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per hosted zone per month, plus a small charge per million queries. |
| **Failure impact** | If Route 53 cannot resolve the domain, no user can reach the platform at all. This is a critical service. |
| **Key configuration notes** | DGBank uses the Tokyo region. Route 53 is global by design, so it is not tied to the region. |

---

### Amazon CloudFront

| Field | Description |
|---|---|
| **Category** | Networking / Edge |
| **Plain English** | A global delivery network that stores copies of the app's static files closer to users, so pages load faster. |
| **Why DGBank uses it** | Mobile users in Japan get faster load times, and CloudFront absorbs a large amount of traffic before it reaches the origin, reducing load on the backend. It also integrates directly with AWS WAF and Shield for edge protection. |
| **Where it sits in the flow** | Second hop in every inbound journey. Sits between Route 53 and AWS WAF. |
| **Real-world analogy** | Convenience stores. Instead of driving to a central warehouse for milk, you walk to the corner shop. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per GB of data transferred and per request. |
| **Failure impact** | Users would fall back to hitting the origin directly if configured; otherwise pages would fail to load. |
| **Key configuration notes** | DGBank uses an ACM-issued TLS certificate for HTTPS. WAF rules are attached at the CloudFront distribution level. |

---

### AWS WAF (Web Application Firewall)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Plain English** | A filter that inspects every incoming web request and blocks known attacks before they reach the application. |
| **Why DGBank uses it** | As a bank, DGBank is a high-value target for attacks such as SQL injection, cross-site scripting, and credential stuffing. WAF enforces rules that block these patterns at the edge. |
| **Where it sits in the flow** | Third hop in every inbound journey — after CloudFront, before the application. |
| **Real-world analogy** | The metal detector at an airport. Everyone passes through, but dangerous items are stopped before boarding. |
| **Managed or self-hosted** | Fully managed by AWS. Rules are configured by DGBank. |
| **Cost model** | Per web ACL per month, plus per million requests. |
| **Failure impact** | If WAF fails "open," traffic still flows but unprotected. If it fails "closed," all traffic is blocked. DGBank's configuration must be documented. |
| **Key configuration notes** | Rate-limiting rules protect against brute-force attacks on the login endpoint. |

---

### AWS Shield Advanced

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Plain English** | Protection against Distributed Denial of Service (DDoS) attacks — coordinated floods of traffic designed to take a service offline. |
| **Why DGBank uses it** | A bank outage caused by DDoS is both a financial and reputational risk. Shield Advanced provides enhanced protection, 24/7 access to the AWS DDoS Response Team, and cost protection against attack-driven traffic spikes. |
| **Where it sits in the flow** | Always-on, layer 3/4/7 protection. Interacts with CloudFront, Route 53, and ALBs. |
| **Real-world analogy** | Flood defences around a city — always in place, rarely noticed until needed. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Monthly subscription plus data transfer out. |
| **Failure impact** | Not a transaction-path service; failure would mean losing enhanced DDoS mitigation during an attack. |
| **Key configuration notes** | Shield Advanced is a paid tier above the automatic (free) Shield Standard. |

---

### Amazon VPC (Virtual Private Cloud)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | A private, isolated network inside AWS where DGBank's resources live. Nothing gets in or out unless DGBank says so. |
| **Why DGBank uses it** | Isolation is a regulatory and security requirement for banking. DGBank runs three separate VPCs — User System, Business System, and Future Tech Operations — so a compromise in one cannot directly affect the others. |
| **Where it sits in the flow** | The container for everything. All compute and data services live inside a VPC. |
| **Real-world analogy** | A gated office building. You cannot walk in without passing reception and having a valid reason. |
| **Managed or self-hosted** | The VPC itself is AWS-managed. What lives inside is DGBank's responsibility. |
| **Cost model** | The VPC is free. NAT Gateways, Transit Gateway attachments, and data transfer are billed. |
| **Failure impact** | A VPC is a logical construct, not a physical device, so it does not "fail" in the traditional sense. Misconfiguration, however, can break connectivity. |
| **Key configuration notes** | Three VPCs — User, Business, and Future Tech. Connected via Transit Gateway. |

---

### Public and Private Subnets

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | Subnets are sections inside a VPC. Public subnets can talk to the internet; private subnets cannot be reached directly from outside. |
| **Why DGBank uses it** | Databases and application servers must never be directly reachable from the internet. Only load balancers and NAT Gateways sit in public subnets. |
| **Where it sits in the flow** | Public subnets handle ingress; private subnets host compute and data. |
| **Real-world analogy** | A hotel lobby (public) versus guest rooms (private). You can walk into the lobby unescorted, but you need a key card for a room. |
| **Managed or self-hosted** | Configured by DGBank. |
| **Cost model** | No direct charge. |
| **Failure impact** | Misconfigured route tables can cause services to lose connectivity. |
| **Key configuration notes** | DGBank currently uses two AZs (`1a`, `1c`). A pending item is to expand to three (`1a`, `1c`, `1d`). |

---

### AWS Transit Gateway

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | A central router that connects multiple VPCs together. |
| **Why DGBank uses it** | The User, Business, and Future Tech VPCs need to communicate selectively. Transit Gateway provides a single hub instead of a mesh of peer-to-peer connections, which becomes unmanageable as the number of VPCs grows. |
| **Where it sits in the flow** | Used for east-west traffic — between DGBank environments and the Future Tech operations VPC. |
| **Real-world analogy** | A major train station where all lines meet. You change trains there instead of building a direct line between every pair of cities. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per attachment per hour plus per GB of data processed. |
| **Failure impact** | Losing Transit Gateway connectivity would isolate Future Tech from monitoring DGBank services. |
| **Key configuration notes** | Route tables on Transit Gateway control which VPCs can reach which. This is a security boundary. |

---

### NAT Gateway

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | Allows servers in private subnets to reach the internet (for example, to call an external API) without allowing the internet to reach them. |
| **Why DGBank uses it** | ECS Fargate and Lambda need to call external services such as HENNGE (email), Accrete (SMS), and Anser BizSol. NAT Gateway enables outbound-only connectivity. |
| **Where it sits in the flow** | Egress path for all private-subnet services calling external endpoints. |
| **Real-world analogy** | A one-way door. You can leave the building, but nobody can come in through that door. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per hour plus per GB processed. NAT Gateway costs are one of the more significant line items in many AWS bills. |
| **Failure impact** | External API calls from private services would fail. Internal-only flows would continue. |
| **Key configuration notes** | DGBank uses NAT Gateways for outbound traffic. A pending item is deciding between Network Firewall and DNS Firewall for egress inspection. |

---

### Internet Gateway (IGW)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | The bridge between a VPC and the public internet. |
| **Why DGBank uses it** | Public subnets require an Internet Gateway to accept inbound traffic from end users. |
| **Where it sits in the flow** | Attached to each VPC that needs internet-facing resources. |
| **Real-world analogy** | The main entrance to the building. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | No direct charge. |
| **Failure impact** | Removing the IGW would cut off all internet ingress to the VPC. |
| **Key configuration notes** | Only public subnets have routes to the IGW. Private subnets route to NAT Gateway instead. |

---

### VPC Endpoint (S3)

| Field | Description |
|---|---|
| **Category** | Networking |
| **Plain English** | A private, direct connection between a VPC and an AWS service (in this case, Amazon S3) that does not use the public internet. |
| **Why DGBank uses it** | Sending data to S3 over the public internet is slower, costlier (NAT Gateway charges), and less secure. VPC Endpoints keep traffic inside the AWS network. |
| **Where it sits in the flow** | Used by ECS Fargate and Lambda when reading from or writing to S3. |
| **Real-world analogy** | A private tunnel between two buildings, instead of driving on public roads. |
| **Managed or self-hosted** | Configured by DGBank; managed by AWS. |
| **Cost model** | Gateway endpoints for S3 are free. Interface endpoints carry hourly and data charges. |
| **Failure impact** | S3 access from the VPC would fall back to NAT Gateway if configured, or fail entirely. |
| **Key configuration notes** | Endpoint policies can restrict which S3 buckets are reachable. |

---

### VPC Flow Logs

| Field | Description |
|---|---|
| **Category** | Networking / Monitoring |
| **Plain English** | A record of every network connection made to, from, and within a VPC. |
| **Why DGBank uses it** | Regulatory and security requirements demand a record of network activity. Flow Logs allow investigation of suspicious traffic and prove compliance. |
| **Where it sits in the flow** | Passive observer. Does not affect traffic; only records it. |
| **Real-world analogy** | CCTV in a building — it does not stop anything, but it captures what happened. |
| **Managed or self-hosted** | Configured by DGBank. Data is stored in CloudWatch Logs or S3. |
| **Cost model** | Charges apply based on the volume of log data collected and stored. |
| **Failure impact** | No impact on the platform, but loss of audit trail. |
| **Key configuration notes** | A pending item is to configure Flow Logs specifically to monitor external API calls to Anser BizSol, HENNGE, Accrete, formrun, and DG AI-OCR. |

---

### Internal Application Load Balancer (ALB)

| Field | Description |
|---|---|
| **Category** | Networking / Compute |
| **Plain English** | Distributes incoming API requests across multiple ECS Fargate containers so no single container is overwhelmed. |
| **Why DGBank uses it** | Provides high availability and even load distribution. If one ECS task is unhealthy, the ALB stops sending traffic to it. |
| **Where it sits in the flow** | Between API Gateway and ECS Fargate. |
| **Real-world analogy** | A host at a restaurant directing guests to whichever table is ready. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per hour plus per LCU (load balancer capacity unit) consumed. |
| **Failure impact** | Without ALB, ECS tasks would need to be directly addressable, breaking high availability. |
| **Key configuration notes** | DGBank uses an **internal** ALB, meaning it is not exposed to the internet — only reachable inside the VPC. |

---

### AWS Network Firewall (NWFW)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Plain English** | A managed firewall that inspects and controls outbound traffic from the VPC to the internet. |
| **Why DGBank uses it** | Banks must control and log all egress traffic. NWFW allows DGBank to permit only approved external endpoints (Anser BizSol, HENNGE, Accrete, and so on). |
| **Where it sits in the flow** | Between the private subnets and the NAT Gateway/IGW. |
| **Real-world analogy** | A customs checkpoint on the way out of the country, inspecting what is leaving. |
| **Managed or self-hosted** | Fully managed by AWS. Rules configured by DGBank. |
| **Cost model** | Per firewall endpoint per hour plus per GB processed. |
| **Failure impact** | If configured to fail-closed, all egress would stop. If fail-open, traffic flows but is uninspected. |
| **Key configuration notes** | **Pending decision:** DGBank is currently evaluating whether NWFW is required, or whether Security Groups plus DNS Firewall would meet the requirement with lower operational complexity. |

---

### DNS Firewall (DNSFW)

| Field | Description |
|---|---|
| **Category** | Networking / Security |
| **Plain English** | A control that blocks DNS lookups to known-bad or unapproved domains. |
| **Why DGBank uses it** | As an alternative or complement to Network Firewall, DNSFW can prevent services from resolving malicious or unapproved domains — a cheaper and simpler control than full packet inspection. |
| **Where it sits in the flow** | Applied to Route 53 Resolver queries originating from the VPC. |
| **Real-world analogy** | A phonebook with certain numbers blacked out. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per million DNS queries inspected. |
| **Failure impact** | If DNSFW blocks a required domain by mistake, that integration breaks. |
| **Key configuration notes** | Being evaluated alongside NWFW as part of the pending egress security decision. |

---