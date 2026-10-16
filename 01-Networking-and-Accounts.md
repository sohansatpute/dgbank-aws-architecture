# DGBank AWS Architecture Documentation

## 1.0 Network & Account Foundation

### 1.1 AWS Region & Availability Zones (AZs)
*   **Region:** The entire architecture is hosted in the **AWS Tokyo Region (`ap-northeast-1`)**. This is critical for Japanese banking data residency and low latency for local users.
*   **Availability Zones (AZs):** The diagram shows resources deployed in `ap-northeast-1a` and `ap-northeast-1c`.
    *   *Why this matters:* If one data center fails, the other AZ keeps the bank running.
    *   **⚠️ TODO for your doc:** Note the pending action item: **"Migrate to 3 AZs (`1a`, `1c`, `1d`) and configure Aurora as a Multi-AZ cluster for maximum High Availability (HA)."**

### 1.2 The Multi-Account Strategy (AWS Organizations & IAM)
The architecture is deliberately split into **three separate AWS environments (Accounts/VPCs)**. This is a best practice called "Blast Radius Reduction." If one system gets hacked, the others remain safe.
*   **User System (利用者システム):** For external customers (End Users).
*   **Business System (業務システム):** For internal DGBank staff (Operations, Sales, Risk).
*   **Future Tech System (DGフューチャーテック):** The isolated environment for Future Tech to monitor and operate the bank.
*   **Governance:** At the bottom left, you see **AWS Organizations** and **IAM**. This controls who can log in and what they can do across all three accounts.

### 1.3 VPC (Virtual Private Cloud) & Subnets
Each system has its own VPC. Inside each VPC, we divide the network into **Subnets**:
*   **Public Subnets:** 
    *   *What lives here:* NAT Gateways, Application Load Balancers (ALBs), and WAF.
    *   *Why:* These need to be exposed to the internet to receive traffic.
*   **Private Subnets:** 
    *   *What lives here:* ECS Fargate (Backend API), Aurora Serverless (Database), ElastiCache (Cache), Lambda functions.
    *   *Why:* **Security.** You never expose your database or application logic directly to the internet. They only talk to the Public Subnet load balancers.

### 1.4 How Everything Connects (The Network Hub)
*   **Transit Gateway:** This is the central router. It connects the User VPC, Business VPC, and Future Tech VPC together. It allows Future Tech to monitor the bank without being inside the bank's main network.
*   **Internet Gateway (IGW):** Allows the Public Subnets to connect to the internet.
*   **NAT Gateway:** Allows Private Subnets (like ECS Fargate) to reach out to the internet for software updates or external APIs (like HENNGE for email), *without* allowing the internet to reach them.
*   **VPC Endpoints (S3):** A private, secure tunnel that allows your Private Subnets to talk to Amazon S3 without going over the public internet. This saves cost and increases security.

### 1.5 Monitoring the Network (VPC Flow Logs)
*   The diagram shows **VPC Flow Logs** enabled. 
*   *Comment:* This records all IP traffic going to and from network interfaces in your VPC. 
*   **⚠️ TODO Note:** *"Set up VPC Flow Logs to monitor external API communication (AnserBizSol, HENNGE, Accrete, formrun, DG AI-OCR)."*

### 1.6 The Big TODO: Egress Security (Outbound Traffic)
*   **The Problem:** How do you securely control what your private servers are allowed to talk to on the internet?
*   **Current Plan:** Deploy **AWS Network Firewall (NWFW)**. 
*   **Alternative Considered:** **Security Groups + DNS Firewall (DNSFW)**.
*   *Comment:* "Currently evaluating NWFW vs. Security Group + DNSFW. NWFW is more robust but complex to design. DNSFW is easier but may have limitations. Decision pending."