# AWS Three-Tier Web Application

A production-style, highly available three-tier web application on AWS: a React/NGINX web tier, a Node.js application tier, and an Amazon Aurora MySQL database tier, deployed across two Availability Zones behind internal and external Application Load Balancers with Auto Scaling. The environment is hardened with CloudWatch monitoring, AWS CloudTrail, Amazon GuardDuty and AWS Secrets Manager, and can be deployed either by hand in the console or as code with nested CloudFormation stacks.

![AWS](https://img.shields.io/badge/AWS-Cloud-orange) ![IaC](https://img.shields.io/badge/IaC-CloudFormation-blue) ![Database](https://img.shields.io/badge/DB-Aurora%20MySQL-4479A1) ![Runtime](https://img.shields.io/badge/App-Node.js-339933) ![Web](https://img.shields.io/badge/Web-React%20%2B%20NGINX-61DAFB)

---

## Table of Contents

- [Scenario](#scenario)
- [Architecture](#architecture)
- [AWS Services Used](#aws-services-used)
- [Network Design](#network-design)
- [Security Groups](#security-groups)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Estimated Cost](#estimated-cost)
- [Deployment Option A: Console Build](#deployment-option-a-console-build)
- [Deployment Option B: CloudFormation](#deployment-option-b-cloudformation)
- [Testing the Application](#testing-the-application)
- [Monitoring](#monitoring)
- [Security Hardening](#security-hardening)
- [Optional: AI Services](#optional-ai-services)
- [Clean Up](#clean-up)
- [Troubleshooting](#troubleshooting)
- [What I Learned](#what-i-learned)
- [Credits](#credits)

---

## Scenario

Company X is preparing for its annual event and needs a website that can handle thousands of attendees registering at once and provide real-time session updates throughout the event.

**Client requirements**

- A scalable web application that handles high traffic during registration
- Real-time session updates, delivered securely and reliably
- Amazon S3 for secure storage of application assets
- AWS IAM for instance access control
- A custom VPC with public and private subnets and an Internet Gateway for public connectivity

---

## Architecture

```
                          Internet
                             |
                    [Internet Gateway]
                             |
            +----------------+----------------+
            |   External ALB (internet-facing) |
            +----------------+----------------+
                 AZ-1        |        AZ-2
   Public    [Web EC2: NGINX + React]  [Web EC2: NGINX + React]   <- Web Tier ASG
   subnets       |  NAT Gateway AZ-1      |  NAT Gateway AZ-2
                 +-----------+------------+
                             |
            +----------------+----------------+
            |        Internal ALB (port 80)    |
            +----------------+----------------+
   Private   [App EC2: Node.js :4000]  [App EC2: Node.js :4000]   <- App Tier ASG
   subnets       |                        |
                 +-----------+------------+
                             |
   Private   [Aurora MySQL writer]  <---->  [Aurora reader]       <- Data Tier
   DB subnets
```
<img width="398" height="179" alt="image" src="https://github.com/user-attachments/assets/8ac33a01-cf3e-460d-a917-718f81876ba9" />


**Request path:** Browser → External ALB → NGINX (web tier) → Internal ALB → Node.js (app tier, port 4000) → Aurora MySQL (port 3306) → back.

| Tier | Purpose | Runs on |
|---|---|---|
| **Web** | Serves the React front end and proxies API calls to the app tier | EC2 (Amazon Linux 2023) with NGINX, in public subnets |
| **App** | Business logic and database access | EC2 with Node.js 16 and PM2, in private subnets |
| **Data** | Stores transactions | Amazon Aurora MySQL, writer and reader in two AZs |

Instances are reached with **AWS Systems Manager Session Manager**: no SSH keys and no bastion host.

---

## AWS Services Used

| Area | Services |
|---|---|
| Networking | Amazon VPC, subnets, route tables, Internet Gateway, NAT Gateways, Elastic IPs, security groups |
| Compute | Amazon EC2, AMIs, launch templates, Auto Scaling groups |
| Load balancing | Application Load Balancers (internal and internet-facing), target groups |
| Database | Amazon Aurora MySQL-Compatible (Multi-AZ) |
| Storage | Amazon S3 (application code, ALB access logs, CloudTrail logs) |
| Identity | AWS IAM (EC2 instance role), AWS Systems Manager Session Manager |
| Monitoring | Amazon CloudWatch (agent, alarms, dashboard, Logs Insights), VPC Flow Logs, Amazon SNS |
| Security | AWS CloudTrail, Amazon GuardDuty, Amazon EventBridge, AWS Secrets Manager |
| Infrastructure as code | AWS CloudFormation (nested stacks) |

---

## Network Design

VPC CIDR: `10.0.0.0/16`, region `us-east-1`, two Availability Zones.

| Subnet | CIDR | AZ | Route |
|---|---|---|---|
| Public-Web-Subnet-AZ-1 | 10.0.0.0/24 | us-east-1a | 0.0.0.0/0 → Internet Gateway |
| Public-Web-Subnet-AZ-2 | 10.0.1.0/24 | us-east-1b | 0.0.0.0/0 → Internet Gateway |
| Private-App-Subnet-AZ-1 | 10.0.2.0/24 | us-east-1a | 0.0.0.0/0 → NAT Gateway AZ-1 |
| Private-App-Subnet-AZ-2 | 10.0.3.0/24 | us-east-1b | 0.0.0.0/0 → NAT Gateway AZ-2 |
| Private-DB-Subnet-AZ-1 | 10.0.4.0/24 | us-east-1a | Local only |
| Private-DB-Subnet-AZ-2 | 10.0.5.0/24 | us-east-1b | Local only |

One NAT Gateway per AZ keeps the app tier's outbound access available if an AZ fails.

---

## Security Groups

Each tier accepts traffic only from the tier in front of it (security-group chaining).

| Security group | Inbound rule | Source |
|---|---|---|
| `internet-facing-lb-sg` | HTTP 80 | 0.0.0.0/0 |
| `WebTier-sg` | HTTP 80 | `internet-facing-lb-sg` (and My IP for testing) |
| `internal-lb-sg` | HTTP 80 | `WebTier-sg` |
| `Privateinstance-sg` | TCP 4000 | `internal-lb-sg` (and My IP for testing) |
| `DB-sg` | MySQL/Aurora 3306 | `Privateinstance-sg` |

---

## Repository Structure

```
.
├── application-code/
│   ├── app-tier/          # Node.js API (index.js, DbConfig.js, package.json)
│   ├── web-tier/          # React front end
│   └── nginx.conf         # NGINX config; proxies /api to the internal ALB
├── cloudformation/        # Nested stack templates (main-stack.yaml + child stacks)
├── docs/                  # Architecture diagram and screenshots
└── README.md
```

---

## Prerequisites

- An AWS account, with region set to **US East (N. Virginia) us-east-1**
- A **billing alarm or AWS Budget** set before you start
- The application code from the source workshop repository (see [Credits](#credits))

---

## Estimated Cost

Approximate total for the full console build, including monitoring and security: **about $18–28**.

| Item | Approx. cost |
|---|---|
| Base build (EC2, Aurora, NAT Gateways, ALBs) | $15–20 |
| CloudWatch (alarms, dashboard, Enhanced Monitoring, Flow Logs, logs) | $1–3 |
| CloudTrail | Free for one trail of management events |
| GuardDuty | Free trial for most new accounts; about $0–1 for a short project |
| Secrets Manager | About $0.40 per secret per month (prorated) |
| S3 storage | A few cents |

The CloudFormation deployment costs about $0.60–0.70 per hour while running. NAT Gateways and Aurora charge by the hour, so finish in as few sessions as you can and **always run the clean-up steps**.

---

## Deployment Option A: Console Build

### 1. Set up resources

1. Download the application code.
2. Create a private **S3 bucket** (Block Public Access on) for the code.
3. Create the IAM role **`ThreeTier-IAMRole`** for EC2 with:
   - `AmazonSSMManagedInstanceCore` (Session Manager access)
   - `AmazonS3ReadOnlyAccess` (download code from S3)

### 2. Configure the VPC

1. Create the VPC `threetierapp-vpc` (`10.0.0.0/16`) and the six subnets in the [table above](#network-design).
2. Create and attach an **Internet Gateway**.
3. Create one **NAT Gateway** in each public subnet, each with an Elastic IP.
4. Create route tables: one public (to the Internet Gateway) for both web subnets, and one private per AZ (to that AZ's NAT Gateway) for the app subnets.
5. Create the five [security groups](#security-groups).

### 3. Deploy the database

1. Create the DB subnet group `three-tier-db-subnet-group` with the two DB subnets.
2. Create an **Aurora MySQL-Compatible** cluster (Standard create, Dev/Test template) with a reader in a different AZ, no public access, and security group `DB-sg`. Turn off Performance Insights for now.
3. Note the **writer endpoint**, username and password.

### 4. Deploy the app tier

1. Launch `MYAppServer1` (Amazon Linux 2023, t2.micro, no key pair) in `Private-App-Subnet-AZ-1` with `Privateinstance-sg` and `ThreeTier-IAMRole`.
2. Connect with **Session Manager** and check internet access through the NAT Gateway: `ping 8.8.8.8`.
3. Install the MySQL client, connect to the writer endpoint, and create the schema:

   ```sql
   CREATE DATABASE webappdb;
   USE webappdb;
   CREATE TABLE IF NOT EXISTS transactions (
     id INT NOT NULL AUTO_INCREMENT,
     amount DECIMAL(10,2),
     description VARCHAR(100),
     PRIMARY KEY (id)
   );
   INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
   ```

4. Fill in `application-code/app-tier/DbConfig.js` (writer endpoint, user, password, `webappdb`), and upload the `app-tier` folder to S3.
5. On the instance, install Node.js and PM2, download the code and start the app:

   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16 && nvm use 16
   npm install -g pm2
   cd ~/ && aws s3 cp s3://<BUCKET_NAME>/app-tier/ app-tier --recursive
   cd ~/app-tier && npm install
   pm2 start index.js
   pm2 startup     # then run the command it prints
   pm2 save
   ```

6. Test locally:

   ```bash
   curl http://localhost:4000/health
   curl http://localhost:4000/transaction
   ```

### 5. Internal load balancer and app-tier Auto Scaling

1. Create the AMI `AppTierImage` from the app instance.
2. Create target group `AppTierTargetGroup` (HTTP 4000, health check `/health`).
3. Create the **internal ALB** `app-tier-internal-lb` in both private app subnets with `internal-lb-sg`, listener HTTP 80 → `AppTierTargetGroup`.
4. Create launch template `App-Tier-LaunchTemplate` (AMI `AppTierImage`, `Privateinstance-sg`, `ThreeTier-IAMRole`).
5. Create Auto Scaling group `AppTierASG` across both private app subnets, attached to `AppTierTargetGroup`, with desired/min/max of 2.

### 6. Deploy the web tier

1. In `application-code/nginx.conf`, replace `[INTERNAL-LOADBALANCER-DNS]` with the internal ALB's DNS name, then upload `web-tier/` and `nginx.conf` to S3.
2. Launch `MyWebApp1` in `Public-Web-Subnet-AZ-1` with a public IP, `WebTier-sg` and `ThreeTier-IAMRole`.
3. Connect with Session Manager, then build the front end and configure NGINX:

   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16 && nvm use 16
   cd ~/ && aws s3 cp s3://<BUCKET_NAME>/web-tier/ web-tier --recursive
   cd ~/web-tier && npm install && npm run build
   sudo yum install nginx -y
   cd /etc/nginx && sudo cp nginx.conf nginx.conf_bkp
   sudo aws s3 cp s3://<BUCKET_NAME>/nginx.conf .
   sudo service nginx restart
   chmod -R 755 /home/ec2-user
   sudo chkconfig nginx on
   ```

### 7. External load balancer and web-tier Auto Scaling

1. Create the AMI `WebServerImage` from the web instance.
2. Create target group `WebTierTargetGroup` (HTTP 80, health check `/health`).
3. Create the **internet-facing ALB** `web-tier-external-lb` in both public subnets with `internet-facing-lb-sg`, listener HTTP 80 → `WebTierTargetGroup`.
4. Create launch template `WebServerLaunchTemplate` (AMI `WebServerImage`, `WebTier-sg`, `ThreeTier-IAMRole`).
5. Create Auto Scaling group `WebServerASG` across both public subnets, attached to `WebTierTargetGroup`, with desired/min/max of 2.

---

## Deployment Option B: CloudFormation

The same environment, including monitoring and security, defined as **nested stacks**. A parent template orchestrates seven child stacks in dependency order:

```
main-stack
 ├── VPCStack
 ├── SecurityGroupsStack
 ├── DBStack               (about 10 minutes)
 ├── ComputeStack          (load balancers + launch templates)
 ├── AutoScalingStack
 ├── MonitoringStack
 └── SecurityControlsStack (runs in parallel)
```

### Part 1: Foundation (once)

1. Choose three globally unique bucket names, for example:
   - `<initials>-cfn-templates-<4 digits>`
   - `<initials>-cloudtrail-logs-<4 digits>`
   - `<initials>-appcode-<4 digits>`
2. Create the **templates bucket** and upload all the YAML templates.
3. Create the **CloudTrail logs bucket** and apply a bucket policy that lets `cloudtrail.amazonaws.com` run `s3:GetBucketAcl` on the bucket and `s3:PutObject` to `AWSLogs/<ACCOUNT_ID>/*` (with `aws:SourceAccount` conditions).
4. Create the **code bucket** and upload `app-tier/`, `web-tier/` and `nginx.conf` unedited; placeholders are filled in automatically at boot.
5. Create the IAM role `ThreeTier-IAMRole` (as in Option A) and copy its **instance profile ARN**.

### Part 2: Deploy

Create a stack from the S3 URL of `main-stack.yaml` with these parameters:

| Parameter | Value |
|---|---|
| `AlertEmail` | Your email for alarm notifications |
| `CodeBucketName` | Code bucket name (no `s3://`) |
| `DBPassword` | Letters and numbers only, at least 8 characters |
| `InstanceProfileArn` | ARN of `ThreeTier-IAMRole`'s instance profile |
| `LogBucketName` | CloudTrail logs bucket name |
| `TemplatesBucketURL` | `https://<templates-bucket>.s3.amazonaws.com` (no trailing slash) |

Acknowledge **IAM resources** and **CAPABILITY_AUTO_EXPAND**, then submit. Deployment takes about 25 minutes, and instances finish building themselves 5–8 minutes after the stacks turn green. Confirm the SNS subscription email while you wait.

### Part 3: Verify

Open the parent stack's **Outputs → ApplicationURL** over `http://`.

---

## Testing the Application

1. Open the external ALB's DNS name (or the stack's `ApplicationURL`) over **http://**.
2. The demo home page should load through the web tier.
3. Open the menu → **DB DEMO**, then add and delete rows. Each write travels browser → external ALB → NGINX → internal ALB → Node.js → Aurora and back, which proves all three tiers work.

---

## Monitoring

| Component | What it adds |
|---|---|
| **VPC Flow Logs** | All accepted and rejected traffic, to `/vpc/threetierapp-flowlogs` |
| **CloudWatch agent** | Memory and disk metrics (namespace `ThreeTierApp`), plus Node.js and NGINX logs (`/app/nodejs`, `/web/nginx-access`, `/web/nginx-error`) |
| **ALB access logs** | Every request to both load balancers, stored in S3 |
| **Aurora** | Performance Insights (7-day retention) and Enhanced Monitoring (60 s) |
| **SNS** | Topic `ThreeTierApp-Alerts`, email subscription |
| **CloudWatch alarms** | High CPU on both ASGs (> 70%), unhealthy hosts on both ALBs (> 0), Aurora connections (> 80) and CPU (> 75%), ALB 5xx errors (> 10 in 5 min) |
| **Dashboard** | `ThreeTierApp-Dashboard`: CPU, request counts, healthy hosts, DB connections, and a Logs Insights error table |
| **Target tracking scaling** | Both ASGs scale on average CPU at a 60% target |

Logs Insights query for recent application errors:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

---

## Security Hardening

- **AWS CloudTrail:** multi-region trail `ThreeTierApp-Trail`, read and write management events, with CloudWatch Logs integration (`/cloudtrail/threetierapp`).
- **Amazon GuardDuty:** threat detection across VPC Flow Logs, DNS logs and CloudTrail. An EventBridge rule (`GuardDuty-Findings-Alert`) sends findings to the SNS topic:

  ```json
  { "source": ["aws.guardduty"], "detail-type": ["GuardDuty Finding"] }
  ```

- **AWS Secrets Manager:** the database credentials move out of `DbConfig.js` into the secret `threetierapp/aurora/credentials`. The app reads it at runtime, and the instance role is granted `secretsmanager:GetSecretValue` on that secret only.
- **No SSH:** all instance access goes through Session Manager.
- **Private by default:** the app and database tiers have no public IPs, and the code bucket blocks all public access.

---

## Optional: AI Services

The Node.js app tier can be extended with managed AI services through new API routes:

| Service | Use | Route |
|---|---|---|
| Amazon Rekognition | Moderate product images and suggest tags | `POST /api/product/analyze-image` |
| Amazon Comprehend | Review sentiment and key phrases | `POST /api/review/analyze` |
| Amazon Bedrock (Claude) | Conversational shopping assistant | `POST /api/assistant/chat` |
| Amazon Personalize | "Customers also bought" recommendations | `GET /api/recommendations/:userId` |

To roll out code changes: sync `app-tier/` to S3, update the instance, create a new AMI, add a new launch template version, and run an **instance refresh** on `AppTierASG` (50% minimum healthy) for a zero-downtime rollout.

---

## Clean Up

**CloudFormation:** delete the parent stack (the one without the NESTED badge). All eight stacks are removed in about 15 minutes. Then delete the three S3 buckets and the IAM role.

**Console build:** delete in this order to avoid dependency errors:

1. Auto Scaling groups
2. Load balancers (listeners first), then target groups
3. Launch templates, AMIs (deregister) and snapshots
4. Remaining EC2 instances
5. Aurora: turn off deletion protection, delete the reader, the writer, then the cluster
6. NAT Gateways, then release the Elastic IPs
7. Route tables (remove routes and subnet associations first)
8. Detach and delete the Internet Gateway, then delete the VPC
9. S3 buckets (empty first), Secrets Manager secret, IAM role
10. Disable GuardDuty; delete the CloudTrail trail, CloudWatch alarms, dashboard and log groups, the SNS topic and the EventBridge rule

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `pm2 list` shows **errored** | Wrong values in `DbConfig.js` | Correct the endpoint, user, password and database name, re-upload, and restart the app |
| **Module not found** | `npm install` skipped in `app-tier` | Run `npm install` in `~/app-tier`, then `pm2 restart index` |
| **Empty reply from server** on `/transaction` | Bad credentials in the uploaded `DbConfig.js` | Fix the file, delete the old S3 copy, and redeploy the app instance |
| Page loads but **no data / can't add rows** | App-tier targets unhealthy (often `pm2 save` was missed before the AMI) | Re-run `pm2 startup` and `pm2 save`, recreate the AMI, launch template and ASG |
| **Session Manager can't connect** | Wrong subnet, routing or IAM role | Check the subnet, its route to a NAT Gateway or Internet Gateway, and `ThreeTier-IAMRole` |
| **404 Not Found** | NGINX `root` path wrong, missing build files, or bad health check | Check `root` points to `web-tier/build`, files exist with 755 permissions, then `sudo nginx -t` and reload |
| **Refused to connect** | Browser forced `https://`; ALB has only HTTP | Type `http://` explicitly or use a private window |
| **502 / unhealthy targets** right after deploy | Instances still building | Wait about 8 minutes after the stacks turn green |
| **502 persists** | A boot step failed | Session Manager → `sudo cat /var/log/userdata.log`; the last lines name the failure |
| **ComputeStack fails** | Typo in `CodeBucketName` or `TemplatesBucketURL` | Check the failed nested stack's Events tab |
| **SecurityControlsStack fails** | CloudTrail bucket policy missing or wrong | Re-check the bucket name and account ID in the policy |
| **DB DEMO errors, home page fine** | App tier or DB bootstrap failed | Check `AppTierTargetGroup` health and `userdata.log`; `DBPassword` must be letters and numbers only |

---

## What I Learned

- Designing a VPC with public and private subnets across two AZs, with per-AZ NAT Gateways for resilience
- Restricting each tier to traffic from the tier in front of it with security-group chaining
- Making stateless tiers highly available with AMIs, launch templates, Auto Scaling groups and internal/external ALBs
- Removing SSH entirely with Session Manager and instance roles
- Building observability with metrics, logs, alarms, dashboards and target-tracking scaling
- Hardening with CloudTrail, GuardDuty and Secrets Manager
- Turning a console build into repeatable infrastructure as code with nested CloudFormation stacks

---

## Credits

Based on the [AWS Three-Tier Web Architecture Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/85cd2bb2-7f79-4e96-bdee-8078e469752a/en-US) and its application code ([k21academyuk/aws-three-tier-web-architecture-workshop](https://github.com/k21academyuk/aws-three-tier-web-architecture-workshop)), extended with monitoring, security hardening and CloudFormation automation.

**Author:** Peter Kalisa, Network & Cloud Infrastructure Engineer · [LinkedIn](<your-linkedin-url>) · [GitHub](<your-github-url>)
