# Secure 3-Tier Web Application on AWS

A production-style 3-tier architecture deployed on AWS Free Tier — custom VPC networking, IAM least-privilege access, an Auto Scaling Group behind an Application Load Balancer, a private RDS database, S3 with lifecycle policies, CloudWatch monitoring with SNS alerting, and a GitHub Actions CI/CD pipeline.

## Architecture

```
Internet
   │
   ▼
[Application Load Balancer]  ── public-a, public-b (alb-sg: HTTP 80 from anywhere)
   │
   ▼
[EC2 Auto Scaling Group]     ── public-a, public-b (ec2-sg: HTTP 80 from ALB only)
   │
   ▼
[RDS MySQL — Single-AZ]      ── private-db-a, private-db-b (rds-sg: MySQL 3306 from EC2 only, Public access: No)
```

**Design note:** EC2 instances run in public subnets rather than fully private ones. This was a deliberate trade-off — a NAT Gateway (required for private instances to reach AWS Systems Manager without extra cost) isn't Free Tier eligible. Security is still enforced entirely through security groups: `ec2-sg` only accepts traffic from `alb-sg`, with no SSH exposed to the internet. RDS remains fully private with zero internet access.

## Tech Stack

VPC · IAM · EC2 · Auto Scaling · Application Load Balancer · RDS (MySQL) · S3 · CloudWatch · SNS · GitHub Actions

---

## 1. Networking — VPC, Subnets, Internet Gateway

Custom VPC (`10.0.0.0/16`) spanning 2 Availability Zones, with public subnets for the load balancer/app tier and private subnets isolating the database.

![VPC Overview](screenshots/01-vpc-overview.png)
![VPC Details](screenshots/02-vpc-details.png)
![Subnets](screenshots/03-subnets.png)
![Internet Gateway Attached](screenshots/04-internet-gateway.png)

## 2. Route Tables

Public subnets route `0.0.0.0/0` traffic through the Internet Gateway; private subnets have no such route, making them unreachable from the internet by design.

![Route Tables](screenshots/05-route-tables.png)

## 3. Security Groups

A layered access chain — nothing skips a tier:

| Security Group | Inbound Rule | Enforces |
|---|---|---|
| `alb-sg` | HTTP 80 from `0.0.0.0/0` | Only entry point from the internet |
| `ec2-sg` | HTTP 80 from `alb-sg` | App servers only accept traffic that passed through the ALB |
| `rds-sg` | MySQL 3306 from `ec2-sg` | Database only accepts traffic from app servers — nothing else |

![Security Groups](screenshots/06-security-groups.png)

## 4. IAM — Least-Privilege Access

EC2 instances use an IAM role (not hardcoded credentials) scoped to only what they need: read/write on one specific S3 bucket, plus SSM for remote management.

![EC2 IAM Role Permissions](screenshots/07-iam-ec2-role.png)
![IAM Roles List](screenshots/08-iam-roles-list.png)
![IAM Policy Created](screenshots/09-iam-policy.png)

## 5. Compute — Auto Scaling Group + Load Balancer

The Application Load Balancer distributes traffic across EC2 instances managed by an Auto Scaling Group (min 1, desired 2, max 3), scaling automatically on CPU > 70%.

**Live proof — the deployed application responding through the load balancer:**

![ALB Serving Traffic](screenshots/10-alb-hello-world.png)

## 6. Database — RDS (Private, Single-AZ)

MySQL database with zero public exposure — accessible only from the app tier via `rds-sg`.

![RDS Dashboard](screenshots/11-rds-dashboard.png)
![RDS Instance Summary](screenshots/12-rds-summary.png)
![RDS Connectivity — Security Group](screenshots/13-rds-connectivity.png)
![Successful Connection via Session Manager](screenshots/14-rds-session-manager.png)
![RDS Manual Snapshot](screenshots/15-rds-snapshot.png)

## 7. Storage — S3 with Lifecycle Policy

Private S3 bucket (Block Public Access enabled) with a lifecycle rule transitioning objects to Glacier after 30 days for cost optimization.

![S3 Object](screenshots/16-s3-object.png)
![S3 Lifecycle Rule](screenshots/17-s3-lifecycle.png)

## 8. Monitoring — CloudWatch + SNS

CloudWatch alarms on CPU utilization and unhealthy host count, notifying via SNS email on trigger.

![CloudWatch Alarms](screenshots/18-cloudwatch-alarms.png)

## 9. CI/CD — GitHub Actions

Pushing to `main` automatically deploys to the EC2 Auto Scaling Group via AWS Systems Manager, using scoped IAM credentials stored as encrypted GitHub secrets.

![GitHub Actions Successful Run](screenshots/19-github-actions-success.png)
![Deploy Workflow File](screenshots/20-deploy-yml.png)

---

## What I'd Change With a Production Budget

- **NAT Gateway** to move EC2 instances into fully private subnets, with the ALB as the only internet-facing tier
- **Multi-AZ RDS** for automatic failover (currently Single-AZ to stay within Free Tier)
- **Route 53 + ACM** for a custom domain and HTTPS (currently HTTP-only via the ALB's default DNS name)
- **OIDC for GitHub Actions** instead of long-lived IAM access keys — I initially implemented this but hit an unresolved account-level restriction during testing and fell back to scoped access keys as a working solution; I'd revisit this in an unrestricted account, since it removes stored credentials entirely

## Key Things I Learned Building This

- Sizing and dividing CIDR blocks (VPC `/16` vs. subnet `/24`) and why the distinction matters
- Why security groups should reference other security groups instead of static IPs
- Debugging a 502 Bad Gateway back to a networking root cause (private subnets with no NAT Gateway blocking instance initialization) and redesigning the architecture around it
- Diagnosing an OIDC federation failure systematically — testing the trust policy, the identity provider, the audience, and the account boundary independently — before deciding to pivot to a working fallback under time constraints
