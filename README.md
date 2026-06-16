# Lab 1 – Auto Scaling Web Tier on AWS

## Architecture Overview

```
Internet
   │
   ▼
[ALB – internet-facing]  ←── Public Subnet A (AZ-1) + Public Subnet B (AZ-2)
   │
   ▼  (round-robin)
[EC2 instances – private]  ←── Private Subnet A (AZ-1) + Private Subnet B (AZ-2)
   │
   ▼ (outbound only)
[NAT Gateway]  →  Internet (for dnf/yum updates)
```

- **VPC CIDR:** 10.0.0.0/16  
- **Public subnets:** 10.0.1.0/24 (AZ-1) · 10.0.2.0/24 (AZ-2) — ALB only  
- **Private subnets:** 10.0.11.0/24 (AZ-1) · 10.0.12.0/24 (AZ-2) — EC2 only  
- **NAT Gateway:** single Regional NAT in Public Subnet A  
- **ASG:** min=1 · desired=1 · max=4  
- **Scale-out trigger:** average CPU > 30 % (TargetTrackingScaling)  
- **Scale-in cooldown:** 5 minutes (prevents thrashing)

---

## Repository Contents

| File | Purpose |
|---|---|
| `autoscaling-stack.yaml` | Single CloudFormation template – full stack |
| `README.md` | This file |

---

## Prerequisites

- AWS CLI configured (`aws configure`)
- IAM permissions to create VPC, EC2, ELB, IAM, AutoScaling, CloudFormation resources
- A GitHub account (for CloudFormation GitSync)

---

## Deployment

### Option A – AWS CLI (quickest for testing)

```bash
aws cloudformation deploy \
  --template-file autoscaling-stack.yaml \
  --stack-name lab1-autoscaling \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Get the ALB DNS after deployment:

```bash
aws cloudformation describe-stacks \
  --stack-name lab1-autoscaling \
  --query "Stacks[0].Outputs[?OutputKey=='ALBDNSName'].OutputValue" \
  --output text
```

### Option B – CloudFormation GitSync (required for full marks)

1. **Push this repo to GitHub** (public or private with a CodeStar connection).

2. In the AWS Console → CloudFormation → **Create stack** → **From Git**.

3. Connect your GitHub repo and point it at `autoscaling-stack.yaml`.

4. CloudFormation GitSync will auto-deploy on every `git push` to the configured branch — satisfying the "repeatability and auditability" requirement.

---

## Validation Checklist

### 1 – Application accessible via ALB

```bash
# Replace with your actual DNS from the Outputs tab
curl http://<ALBDNSName>
```

You should see an HTML page showing the **Instance ID**, Private IP, and AZ.

### 2 – Traffic served by multiple instances (round-robin)

Refresh the page several times (or run the loop below). The Instance ID must change, proving multiple instances are serving requests.

```bash
for i in {1..10}; do
  curl -s http://<ALBDNSName> | grep "Instance ID"
done
```

### 3 – CPU stress test → scale-out

SSH is disabled. Use **SSM Session Manager** instead:

```bash
# Open a session into one of the running instances
aws ssm start-session --target <instance-id> --region us-east-1
```

Inside the session, run the stress tool (pre-installed by UserData):

```bash
# Spike all CPUs for 5 minutes – enough to breach the 30% threshold
stress --cpu $(nproc) --timeout 300
```

### 4 – Observe scale-out in real time

Watch the ASG activity from another terminal:

```bash
watch -n 5 "aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names lab1-autoscaling-asg \
  --query 'AutoScalingGroups[0].{Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:Instances[*].{ID:InstanceId,State:LifecycleState,Health:HealthStatus}}' \
  --output table"
```

You should see `DesiredCapacity` increase beyond 1, and new instances appear with `LifecycleState: InService`.

### 5 – New instances automatically join the target group

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TargetGroupArn> \
  --query 'TargetHealthDescriptions[*].{ID:Target.Id,Health:TargetHealth.State}' \
  --output table
```

All healthy instances should show `healthy`.

---

## Scaling Policy Explanation

| Setting | Value | Rationale |
|---|---|---|
| Policy type | TargetTrackingScaling | AWS manages both scale-out and scale-in automatically |
| Metric | ASGAverageCPUUtilization | Direct measure of compute demand |
| Target | 30 % | Low threshold makes scale-out easy to trigger in a demo |
| Scale-out cooldown | 60 s | Fast response to spikes |
| Scale-in cooldown | 300 s | Prevents premature termination while load is still present |

**Why TargetTracking over StepScaling?**  
TargetTracking continuously adjusts the desired count to keep the metric at the target, so it both scales out *and* scales in automatically without writing multiple alarm/adjustment pairs.

---

## Security Notes

- EC2 instances have **no inbound SSH** and are in **private subnets** — not reachable from the internet.
- The EC2 security group only accepts traffic **from the ALB security group** on port 80.
- Shell access for debugging uses **SSM Session Manager** (no key pair, no open port 22).
- The IAM instance role grants only `AmazonSSMManagedInstanceCore` — least privilege.

---

## Cost Notes

- The NAT Gateway is **Regional (single-AZ)** — acceptable for a lab. For production, deploy one per AZ for fault tolerance.
- Instances start at **t3.micro** (free-tier eligible). Change `InstanceType` parameter to scale up.
- Delete the stack when done: `aws cloudformation delete-stack --stack-name lab1-autoscaling`
