# PROJECT 1: Highly Available, Fault-Tolerant AWS Web App Architecture

Practicing a production-style AWS infrastructure built to demonstrate core cloud engineering fundamentals: high availability, fault tolerance, auto scaling, and observability — with no single point of failure.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Why These Services](#why-these-services)
- [Prerequisites](#prerequisites)
- [Build Steps](#build-steps)
- [Verification & Testing](#verification--testing)
- [Challenges & Troubleshooting](#challenges--troubleshooting)
- [Cost Considerations](#cost-considerations)
- [Teardown](#teardown)
- [What This Project Demonstrates](#what-this-project-demonstrates)

---

## Overview

This project answers a specific engineering question: **how do you build a system that keeps serving users even when parts of it fail, without manual intervention?**

Rather than deploying a single EC2 instance (which fails the moment traffic spikes or a server crashes), this architecture spans two Availability Zones, load-balances traffic across an auto-scaling fleet of web servers, runs a self-healing Multi-AZ database, and monitors itself with alarms that actually notify on failure — all of which was tested, not just configured.

## Architecture Diagram
![Architecture Diagram](./project1-architecture-diagram.png)

**Network layout:**
- 1 VPC across 2 Availability Zones
- 2 public subnets → Application Load Balancer
- 2 private subnets → EC2 instances and RDS (no direct internet exposure)
- 1 NAT Gateway → allows private-subnet resources outbound internet access (e.g. package installs) without being reachable from it

**Security model:** traffic can only flow ALB → EC2 → RDS, enforced by chaining security groups (each one only trusts the previous one, not IP ranges):

`Internet → alb-sg (0.0.0.0/0:80) → ec2-sg (from alb-sg only:80) → rds-sg (from ec2-sg only:3306)`

## Why These Services

| Service | Alternative considered | Why this choice |
|---|---|---|
| **Application Load Balancer** | Network Load Balancer | ALB operates at the HTTP layer, enabling proper health checks and (if needed later) path/host-based routing. NLB is for extreme low-latency/non-HTTP use cases — not needed here. |
| **Auto Scaling Group** | Fixed EC2 fleet | A fixed fleet forces you to provision for peak load 24/7, wasting money during quiet periods, or risk falling over during spikes. ASG ties capacity to actual demand. |
| **RDS Multi-AZ** | Single-AZ RDS, or self-managed MySQL on EC2 | Self-managed replication/failover is significant engineering effort and risk. Single-AZ RDS has no automatic recovery if that AZ has an issue. Multi-AZ offloads failover logic to AWS, tested and reliable. |
| **Private subnets for EC2/RDS** | Everything in public subnets | Keeps the attack surface to a single, purpose-built entry point (the ALB) instead of exposing app servers and the database directly to the internet. |
| **CloudWatch** | Third-party observability (Datadog, etc.) | Native to AWS, zero integration overhead, free at this scale — the correct default before justifying a third-party tool at larger scale. |

## Prerequisites

- An AWS account (this project uses paid resources — see [Cost Considerations](#cost-considerations))
- Basic familiarity with the AWS Management Console
- An AWS budget alert configured (recommended, not required) to catch unexpected spend

## Build Steps

### 1. VPC and Networking
Created via the "VPC and more" wizard: 1 VPC (`10.0.0.0/16`), 2 AZs, 2 public + 2 private subnets, 1 Internet Gateway, 1 NAT Gateway, and route tables — all wired automatically.

### 2. Security Groups
Created in dependency order, since each references the one before it:
- `alb-sg` — inbound HTTP 80 from `0.0.0.0/0`
- `ec2-sg` — inbound HTTP 80 from `alb-sg` only
- `rds-sg` — inbound MySQL 3306 from `ec2-sg` only

### 3. RDS (Multi-AZ MySQL)
Deployed using the **Production** template (required to unlock Multi-AZ — the Free Tier template hides this option), in the private subnets, not publicly accessible, with deletion protection disabled for easy teardown.

### 4. EC2 Launch Template
Amazon Linux 2023, t3.micro, with a User Data script that installs Apache and generates a page displaying the instance's ID and Availability Zone — pulled from the EC2 instance metadata service using **IMDSv2** (token-based requests, required by default on this AMI).

```bash
#!/bin/bash
yum install -y httpd
systemctl start httpd
systemctl enable httpd

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>AWS HA Architecture Demo</title></head>
<body>
  <h1>Hello from EC2</h1>
  <p>Instance ID: $INSTANCE_ID</p>
  <p>Availability Zone: $AZ</p>
</body>
</html>
EOF
```

### 5. Target Group
HTTP:80, health check path `/`, default thresholds — no targets registered manually; the ASG handles registration.

### 6. Application Load Balancer
Internet-facing, deployed across both public subnets, listener on HTTP:80 forwarding to the target group above.

### 7. Auto Scaling Group
Min 2 / Desired 2 / Max 4, attached to the target group, ELB health checks enabled (so app-level failures are caught, not just instance-level crashes), 300-second health check grace period and instance warmup.

**Scaling policy: Target Tracking**, target value **50% average CPU utilization** — chosen deliberately over a manual two-threshold "scale out at X%, scale in at Y%" setup. Target tracking lets AWS's own algorithm manage the scale-out/scale-in thresholds and cooldowns dynamically, which avoids instance "flapping" (rapid, wasteful add/remove cycles) that a poorly-tuned manual threshold pair can cause.

### 8. S3
A private bucket with secure defaults, provisioned to demonstrate correct storage configuration.

### 9. CloudWatch
- **Dashboard** with widgets for EC2 CPU, ALB request count, target response time, and healthy/unhealthy host count
- **Alarm** on `UnhealthyHostCount >= 1`, wired to an SNS topic emailing on trigger

## Verification & Testing

Configuration alone isn't proof — each piece of this architecture was actually tested:

**Load balancing:** confirmed by refreshing the ALB's URL repeatedly and observing the displayed Instance ID alternate between the two running instances.

**Auto healing / monitoring:** intentionally removed the `ec2-sg` inbound rule to cut ALB→EC2 connectivity, then confirmed:
- Target Group flipped both instances to `unhealthy`
- The CloudWatch alarm transitioned to `In alarm`
- An SNS email notification was received
- The ASG attempted to replace the unhealthy instances (visible in its Activity log)

Then reverted the rule and confirmed full recovery (targets back to `healthy`, alarm back to `OK`).

**Database failover:** using RDS's **"Reboot with failover"** option, forced the Multi-AZ database to promote its standby in the second Availability Zone to primary. Verified via the console that the instance's Availability Zone changed before vs. after — confirming automatic failover actually works, not just that the checkbox was enabled.

## Challenges & Troubleshooting

Real issues hit during the build — documented because working through them was the most valuable part of the project:

**1. ALB unreachable — `ERR_CONNECTION_TIMED_OUT`**
Security groups, subnet placement, NACLs, and ALB state were all correctly configured. Root cause: the browser auto-prepended `https://` to the bare DNS name, and the ALB has no HTTPS listener (HTTP:80 only). Resolved by explicitly typing `http://` in the URL.

**2. Page loaded, but Instance ID / Availability Zone fields were blank**
Amazon Linux 2023 requires **IMDSv2** (token-based metadata requests) by default. The original User Data script used the older tokenless (IMDSv1-style) `curl` request, which was silently rejected — Apache still started fine, so the page loaded, just without the dynamic data. Fixed by rewriting the script to fetch a session token first, publishing a new Launch Template version, and running an **Instance Refresh** on the ASG to roll the fix out to already-running instances (editing the template alone doesn't affect existing instances).

**3. Auto Scaling Group "flapping" risk**
Initially considered a manual step-scaling setup (scale out at 70% CPU / scale in at 40%). Switched to **target tracking** at a 50% target instead — reduces configuration surface and lets AWS manage thresholds dynamically rather than risking two closely-tuned static alarms causing rapid scale in/out cycles.

## Cost Considerations

| Resource | Free tier eligible? | Notes |
|---|---|---|
| EC2 t2/t3.micro | Yes (750 hrs/mo, 12 months) | |
| RDS db.t3.micro, single-AZ | Yes | |
| **RDS Multi-AZ** | No | ~doubles single-AZ cost — used deliberately here to enable a genuine failover test |
| S3 | Yes (5GB) | |
| CloudWatch (basic) | Yes | |
| **Application Load Balancer** | No | ~$16–20/month |
| **NAT Gateway** | No | ~$32+/month + data processing |

The non-free-tier resources (ALB, NAT Gateway, Multi-AZ RDS) were run for a short, deliberate build-test-teardown window rather than left running continuously.

## Teardown

Resources were deleted in dependency order to avoid orphaned references or blocked deletions:

1. CloudWatch alarm and dashboard, SNS topic
2. Auto Scaling Group (terminates EC2 instances)
3. Load Balancer
4. Target Group
5. RDS instance
6. NAT Gateway
7. Elastic IP (unattached IPs still incur charges if left behind)
8. VPC (last, once everything inside it is removed)

## What This Project Demonstrates

- Designing multi-AZ network architecture with proper public/private subnet segmentation
- Chaining security groups by reference (not IP ranges) to enforce least-privilege traffic flow
- Configuring load balancing, health checks, and target-tracking auto scaling
- Deploying and testing a self-healing, Multi-AZ relational database
- Building observability (dashboards + alarms) and validating it by intentionally inducing failure
- Debugging real, non-obvious issues (browser protocol handling, IMDSv2 metadata requirements) using a systematic process of elimination
