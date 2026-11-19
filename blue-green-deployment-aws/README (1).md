<h1 align="center">Blue-Green Deployment on AWS</h1>

<p align="center">
  Zero-downtime releases with instant rollback using <b>EC2</b>, <b>Application Load Balancer</b> and <b>Target Groups</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonaws&logoColor=white" alt="EC2">
  <img src="https://img.shields.io/badge/AWS-ALB-8C4FFF?logo=amazonaws&logoColor=white" alt="ALB">
  <img src="https://img.shields.io/badge/Apache-httpd-D22128?logo=apache&logoColor=white" alt="Apache">
  <img src="https://img.shields.io/badge/OS-Amazon%20Linux%202023-232F3E?logo=amazonaws&logoColor=white" alt="AL2023">
  <img src="https://img.shields.io/badge/Region-us--east--1-blue" alt="Region">
  <img src="https://img.shields.io/badge/Status-Completed-brightgreen" alt="Status">
</p>

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Deployment Flow](#deployment-flow)
4. [Traffic Switch Sequence](#traffic-switch-sequence)
5. [Tech Stack](#tech-stack)
6. [Security Design](#security-design)
7. [Implementation Steps](#implementation-steps)
8. [Results](#results)
9. [Key Learnings](#key-learnings)
10. [Future Improvements](#future-improvements)
11. [Cleanup](#cleanup)
12. [Author](#author)

---

## Overview

**Blue-Green deployment** is a release strategy that runs two identical production environments side by side:

| Environment | Version | Role |
|---|---|---|
| 🔵 **Blue** | App v1.0 | Current live version |
| 🟢 **Green** | App v2.0 | New version, tested before going live |

Traffic is moved from Blue to Green by changing **one ALB listener rule**. If anything goes wrong, pointing the listener back to Blue is the rollback. No downtime and no redeploy.

**Why this matters**

- Zero downtime during releases
- Instant rollback (seconds, not minutes)
- New version can be verified in a production-like environment before the switch
- Old version stays untouched until you are confident

---

## Architecture

```mermaid
flowchart TB
    User([Internet User]) -->|HTTP 80| ALB

    subgraph VPC["Default VPC - us-east-1"]
        direction TB
        subgraph PUB["Public subnets (us-east-1a, us-east-1b)"]
            ALB["Application Load Balancer<br/>BlueGreen-ALB<br/>Security Group: ALB-SG"]
            LST{"Listener HTTP:80<br/>forwards to ONE target group"}
            ALB --> LST
            LST -->|Active| BTG["Blue-TG"]
            LST -.->|Standby| GTG["Green-TG"]
            BTG --> BLUE["EC2: Blue-Server<br/>App v1.0 - BLUE<br/>Security Group: EC2-SG"]
            GTG --> GREEN["EC2: Green-Server<br/>App v2.0 - GREEN<br/>Security Group: EC2-SG"]
        end
    end

    style BLUE fill:#1e40af,color:#fff
    style GREEN fill:#15803d,color:#fff
    style BTG fill:#3b82f6,color:#fff
    style GTG fill:#22c55e,color:#fff
    style ALB fill:#8C4FFF,color:#fff
```

---

## Deployment Flow

```mermaid
flowchart TD
    A([Start]) --> B["Deploy Blue environment<br/>App v1.0 on EC2"]
    B --> C["Create Blue-TG and register Blue-Server"]
    C --> D["Create ALB<br/>Listener forwards to Blue-TG"]
    D --> E{"Blue page live<br/>and healthy?"}
    E -- No --> F["Check SG rules, httpd,<br/>health check path"]
    F --> E
    E -- Yes --> G["Deploy Green environment<br/>App v2.0 on EC2"]
    G --> H["Create Green-TG and register Green-Server"]
    H --> I{"Green-TG<br/>Healthy?"}
    I -- No --> J["Fix instance, User Data<br/>or Security Group"]
    J --> I
    I -- Yes --> K["Switch listener<br/>Blue-TG to Green-TG"]
    K --> L{"Green working<br/>as expected?"}
    L -- Yes --> M(["Green is the new production<br/>Keep Blue as backup"])
    L -- No --> N["Rollback<br/>Listener back to Blue-TG"]
    N --> O(["Blue serves traffic again"])

    style A fill:#334155,color:#fff
    style M fill:#15803d,color:#fff
    style O fill:#1e40af,color:#fff
    style N fill:#b91c1c,color:#fff
```

---

## Traffic Switch Sequence

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant L as ALB Listener (HTTP:80)
    participant B as Blue-TG / Blue-Server
    participant G as Green-TG / Green-Server

    Note over L: Phase 1 - forward to Blue-TG
    U->>L: GET /
    L->>B: forward
    B-->>U: App v1.0 - BLUE

    Note over L: Phase 2 - listener edited to Green-TG
    U->>L: GET /
    L->>G: forward
    G-->>U: App v2.0 - GREEN

    Note over L: Phase 3 - rollback to Blue-TG
    U->>L: GET /
    L->>B: forward
    B-->>U: App v1.0 - BLUE
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud platform | AWS (AWS Academy Learner Lab) |
| Region | `us-east-1` (N. Virginia) |
| Compute | EC2, Amazon Linux 2023, t2/t3.micro |
| Web server | Apache `httpd`, installed via EC2 User Data |
| Load balancing | Application Load Balancer (Layer 7), 2 Target Groups |
| Networking | Default VPC, 2 Availability Zones |
| Security | 2 Security Groups with least-privilege rules |

---

## Security Design

| Security Group | Inbound rule | Source |
|---|---|---|
| `ALB-SG` | HTTP 80 | `0.0.0.0/0` (public) |
| `EC2-SG` | HTTP 80 | `ALB-SG` only |

EC2 instances accept traffic **only from the load balancer**, so they cannot be reached directly from the internet on port 80.

---

## Implementation Steps

<details>
<summary><b>1. Create Security Groups</b></summary>

- `ALB-SG`: inbound HTTP (80) from `0.0.0.0/0`
- `EC2-SG`: inbound HTTP (80) with source `ALB-SG`
</details>

<details>
<summary><b>2. Launch EC2 instances</b></summary>

Amazon Linux 2023, `EC2-SG`, public IP enabled. User Data installs and starts `httpd` and writes the page.

- `Blue-Server` serves `App v1.0 - BLUE Environment`
- `Green-Server` serves `App v2.0 - GREEN Environment`

**Blue User Data**

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
cat > /var/www/html/index.html <<'HTML'
<html>
  <body style="background-color:blue; color:white; font-family:serif;">
    <h1>App v1.0 - BLUE Environment</h1>
  </body>
</html>
HTML
```

For Green, change `blue` to `green` and the heading to `App v2.0 - GREEN Environment`.
</details>

<details>
<summary><b>3. Create Target Groups</b></summary>

- `Blue-TG` with Blue-Server
- `Green-TG` with Green-Server
- Protocol HTTP:80, health check path `/`, default VPC
</details>

<details>
<summary><b>4. Create the Application Load Balancer</b></summary>

- Name `BlueGreen-ALB`, internet-facing, IPv4
- Subnets in `us-east-1a` and `us-east-1b`
- Security group `ALB-SG`
- Listener `HTTP:80` forwarding to `Blue-TG`
</details>

<details>
<summary><b>5. Switch to Green</b></summary>

Load Balancers, `BlueGreen-ALB`, Listeners and rules, `HTTP:80`, Manage listener, Edit listener, set default action to `Green-TG`, Save changes. Hard refresh the browser (`Ctrl + Shift + R`).
</details>

<details>
<summary><b>6. Rollback to Blue</b></summary>

Edit the same listener again and set the default action back to `Blue-TG`. Save and hard refresh.
</details>

---

## Results

The **same ALB URL** served different versions depending only on the listener target.

| Stage | Result | Screenshot |
|---|---|---|
| 1. Initial state | Blue live, v1.0 | ![Blue](screenshots/01-blue.png) |
| 2. Listener switched | Forwarding to Green-TG | ![Listener](screenshots/02-listener-green.png) |
| 3. After switch | Green live, v2.0 | ![Green](screenshots/03-green.png) |
| 4. After rollback | Blue live again, v1.0 | ![Rollback](screenshots/04-rollback-blue.png) |
| 5. Health | Blue-TG target Healthy | ![Healthy](screenshots/05-target-groups-healthy.png) |

---

## Key Learnings

- Traffic switching happens at the load balancer, so the old version stays untouched and ready for rollback.
- A target shows `Unused` until its target group is attached to a listener, then moves to `Healthy` once health checks pass.
- Restricting `EC2-SG` to `ALB-SG` keeps instances off the public internet.
- Browsers cache pages, so use a hard refresh or Incognito to verify a switch.
- Always check target health **before** switching the listener, otherwise users get 503 errors.

---

## Future Improvements

- [ ] Weighted routing for canary releases (for example 90/10)
- [ ] Auto Scaling Groups behind each target group
- [ ] HTTPS with AWS Certificate Manager
- [ ] Custom domain with Route 53
- [ ] CloudWatch alarms for automatic rollback
- [ ] Infrastructure as Code (Terraform or CloudFormation)
- [ ] CI/CD pipeline for automated deployments

---

## Cleanup

Delete in this order to avoid charges (the ALB is billed hourly):

1. Application Load Balancer
2. Target Groups
3. EC2 instances (terminate)
4. Security Groups (`EC2-SG` first, then `ALB-SG`)
5. End the Learner Lab session

---

## Author

**Amisha Sinha**

[![GitHub](https://img.shields.io/badge/GitHub-amishasinha18-181717?logo=github)](https://github.com/amishasinha18)
