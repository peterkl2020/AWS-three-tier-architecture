# AWS Three-Tier Web Architecture

A hands-on build of a highly available three-tier web application on AWS. The network, security groups, load balancers, Auto Scaling groups, application servers and database are all created by hand in the AWS Console, so every moving part is visible.

> Based on the open-source [AWS Three Tier Web Architecture Workshop](https://github.com/aws-samples/aws-three-tier-web-architecture-workshop) (MIT-0 license). This repository documents my completed deployment, with a full step-by-step guide.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [What gets built](#2-what-gets-built)
3. [Repository structure](#3-repository-structure)
4. [Prerequisites](#4-prerequisites)
5. [Part 0: S3 bucket and IAM role](#part-0-s3-bucket-and-iam-role)
6. [Part 1: Networking and security](#part-1-networking-and-security)
7. [Part 2: Database tier (Aurora MySQL)](#part-2-database-tier-aurora-mysql)
8. [Part 3: App tier (Node.js)](#part-3-app-tier-nodejs)
9. [Part 4: Web tier (React and Nginx)](#part-4-web-tier-react-and-nginx)
10. [Testing the full stack](#testing-the-full-stack)
11. [Code walkthrough](#code-walkthrough)
12. [Troubleshooting](#troubleshooting)
13. [Clean up](#clean-up)
14. [Known limitations](#known-limitations)
15. [License](#license)

---

## 1. Architecture

![Architecture Diagram](application-code/web-tier/src/assets/3TierArch.png)

Request flow:

1. A user opens the **public Application Load Balancer** (ALB) in a browser.
2. The ALB forwards traffic to **web tier** EC2 instances running **Nginx**, which serve the **React.js** site.
3. Requests to `/api/` are proxied by Nginx to the **internal ALB**.
4. The internal ALB forwards traffic to the **app tier**, a **Node.js / Express** API on port 4000.
5. The app tier reads and writes an **Amazon Aurora MySQL** database (Multi-AZ) and returns the result.

Load balancing, health checks and Auto Scaling groups are used at the web and app layers so the application stays available if an instance or an Availability Zone fails.

## 2. What gets built

| Layer | AWS services | Purpose |
|---|---|---|
| Network | VPC, 6 subnets, Internet Gateway, 2 NAT Gateways, route tables | Isolate tiers across 2 Availability Zones |
| Security | 5 security groups, IAM role | Each tier only accepts traffic from the tier above it |
| Web tier | Public ALB, Auto Scaling group, EC2 (Nginx + React) | Serves the UI and proxies API calls |
| App tier | Internal ALB, Auto Scaling group, EC2 (Node.js) | Business logic and REST API |
| Database tier | Aurora MySQL (Multi-AZ) | Persistent storage |
| Storage | S3 bucket | Holds application code for EC2 instances to download |

## 3. Repository structure

```
aws-three-tier-web-architecture-workshop/
├── README.md                     <- this guide
├── LICENSE                       <- MIT-0
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── .gitignore
├── docs/
│   └── screenshots/              <- add your own console screenshots here
└── application-code/
    ├── nginx.conf                <- Nginx config for the web tier
    ├── app-tier/                 <- Node.js API
    │   ├── index.js              <- Express routes
    │   ├── TransactionService.js <- MySQL queries
    │   ├── DbConfig.js           <- DB credentials (placeholders only)
    │   ├── package.json
    │   └── package-lock.json
    └── web-tier/                 <- React frontend
        ├── package.json
        ├── public/
        └── src/
            ├── App.js
            ├── index.js
            ├── components/       <- Menu, Burger, Home, DatabaseDemo
            └── assets/3TierArch.png
```

## 4. Prerequisites

- An AWS account with permission to create VPC, EC2, RDS, ELB, IAM and S3 resources
- A region of your choice with at least 2 Availability Zones (steps below use `us-east-1`)
- A text editor (VS Code or similar)
- Estimated cost: a few dollars per day (NAT Gateways, Aurora and load balancers bill hourly). **Run the [clean up](#clean-up) when finished.**

---

## Part 0: S3 bucket and IAM role

### 0.1 Create an S3 bucket

1. Open **S3** in the console and choose **Create bucket**.
2. Give it a globally unique name, for example `three-tier-code-<yourname>`.
3. Keep **Block all public access** on, and create the bucket.
4. Upload the whole `application-code` folder contents so the bucket looks like this:

```
three-tier-code-<yourname>/
├── app-tier/
├── web-tier/
└── nginx.conf
```

> Do the upload of `app-tier/DbConfig.js` only after you fill in your DB details (Part 3), or re-upload it later.

### 0.2 Create an IAM role for the EC2 instances

1. Open **IAM > Roles > Create role**.
2. Trusted entity: **AWS service**, use case **EC2**.
3. Attach these managed policies:
   - `AmazonSSMManagedInstanceCore` (lets you connect through Session Manager, no SSH keys needed)
   - `AmazonS3ReadOnlyAccess` (lets instances download code from your bucket)
4. Name it `3TierEC2Role` and create it.

---

## Part 1: Networking and security

### 1.1 Create the VPC

**VPC > Create VPC > VPC only**

- Name: `three-tier-vpc`
- IPv4 CIDR: `10.0.0.0/16`

### 1.2 Create six subnets (2 per tier, across 2 AZs)

| Name | AZ | CIDR |
|---|---|---|
| `public-web-subnet-az1` | AZ 1 | `10.0.0.0/24` |
| `public-web-subnet-az2` | AZ 2 | `10.0.1.0/24` |
| `private-app-subnet-az1` | AZ 1 | `10.0.2.0/24` |
| `private-app-subnet-az2` | AZ 2 | `10.0.3.0/24` |
| `private-db-subnet-az1` | AZ 1 | `10.0.4.0/24` |
| `private-db-subnet-az2` | AZ 2 | `10.0.5.0/24` |

For both `public-web-subnet-*`, open **Actions > Edit subnet settings** and enable **Auto-assign public IPv4 address**.

### 1.3 Internet Gateway

1. **Internet gateways > Create** named `three-tier-igw`.
2. **Actions > Attach to VPC** and pick `three-tier-vpc`.

### 1.4 NAT Gateways (one per AZ for high availability)

1. **NAT gateways > Create**: name `nat-az1`, subnet `public-web-subnet-az1`, allocate an Elastic IP.
2. Repeat for `nat-az2` in `public-web-subnet-az2`.

### 1.5 Route tables

| Route table | Associated subnets | Routes |
|---|---|---|
| `public-rt` | both public web subnets | `0.0.0.0/0` to Internet Gateway |
| `private-app-rt-az1` | `private-app-subnet-az1` | `0.0.0.0/0` to `nat-az1` |
| `private-app-rt-az2` | `private-app-subnet-az2` | `0.0.0.0/0` to `nat-az2` |

The database subnets use the default local-only route table (no internet access needed).

### 1.6 Security groups

Create these in `three-tier-vpc`. Each tier only accepts traffic from the tier in front of it.

| Security group | Inbound rule | Source |
|---|---|---|
| `internet-facing-lb-sg` | HTTP, port 80 | My IP (or `0.0.0.0/0` for public access) |
| `web-tier-sg` | HTTP, port 80 | `internet-facing-lb-sg` |
| `internal-lb-sg` | HTTP, port 80 | `web-tier-sg` |
| `private-instance-sg` | Custom TCP, port 4000 | `internal-lb-sg` |
| `db-sg` | MySQL/Aurora, port 3306 | `private-instance-sg` |

> For quick testing from your own machine, `web-tier-sg` can also allow HTTP from My IP. Remove it afterwards.

---

## Part 2: Database tier (Aurora MySQL)

### 2.1 Subnet group

**RDS > Subnet groups > Create**

- Name: `three-tier-db-subnet-group`
- VPC: `three-tier-vpc`
- Add both AZs and select `private-db-subnet-az1` and `private-db-subnet-az2`

### 2.2 Create the database

**RDS > Create database**

- Creation method: **Standard create**
- Engine: **Aurora (MySQL Compatible)**
- Template: **Dev/Test**
- DB cluster identifier: `three-tier-db`
- Master username: `admin`, and set a strong password (note it down, you need it in Part 3)
- Instance class: a small burstable class (for example `db.t3.medium`)
- Availability: **Create an Aurora Replica in a different AZ** (Multi-AZ)
- VPC: `three-tier-vpc`, subnet group: `three-tier-db-subnet-group`
- Public access: **No**
- VPC security group: `db-sg`

Wait until the status is **Available**, then copy the **Writer endpoint** from the cluster page. You will need it twice.

---

## Part 3: App tier (Node.js)

### 3.1 Launch the app server

**EC2 > Launch instance**

- Name: `app-tier-server`
- AMI: Amazon Linux 2023
- Instance type: `t2.micro` (or `t3.micro`)
- Key pair: none (we use Session Manager)
- Network: `three-tier-vpc`, subnet `private-app-subnet-az1`, auto-assign public IP **disabled**
- Security group: `private-instance-sg`
- Advanced details > IAM instance profile: `3TierEC2Role`

### 3.2 Connect with Session Manager

Select the instance > **Connect > Session Manager > Connect**, then:

```bash
sudo -su ec2-user
ping 8.8.8.8        # confirms internet access through the NAT Gateway (Ctrl+C to stop)
```

### 3.3 Install a MySQL client

```bash
sudo dnf install -y mariadb105
```

> This installs a MySQL-compatible client. It works with Aurora MySQL.

### 3.4 Create the database and table

```bash
mysql -h <YOUR-DB-WRITER-ENDPOINT> -u admin -p
```

Enter your password, then run:

```sql
CREATE DATABASE webappdb;
SHOW DATABASES;
USE webappdb;

CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);

SHOW TABLES;
INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
SELECT * FROM transactions;
EXIT;
```

If you see the row come back, the database tier works.

### 3.5 Configure database credentials

On your **local machine**, open `application-code/app-tier/DbConfig.js` and fill in:

```js
module.exports = Object.freeze({
    DB_HOST : '<YOUR-DB-WRITER-ENDPOINT>',
    DB_USER : 'admin',
    DB_PWD : '<YOUR-PASSWORD>',
    DB_DATABASE : 'webappdb'
});
```

Upload this edited file to `s3://<your-bucket>/app-tier/DbConfig.js`.

> **Never commit real credentials to GitHub.** The copy in this repository intentionally contains empty placeholders. Edit your local copy for deployment, then reset it before you commit.

### 3.6 Install Node.js and run the app with PM2

Back in the Session Manager terminal:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
npm install -g pm2

cd ~/
aws s3 cp s3://<your-bucket>/app-tier/ app-tier --recursive
cd app-tier
npm install
pm2 start index.js
pm2 list          # status should be "online"
pm2 logs          # check for errors, Ctrl+C to exit
```

Make the app restart automatically after a reboot:

```bash
pm2 startup
# Copy and run the "sudo env PATH=..." command that pm2 prints, then:
pm2 save
```

Test the API locally:

```bash
curl http://localhost:4000/health
# "This is the health check"

curl http://localhost:4000/transaction
# {"result":[{"id":1,"amount":400,"description":"groceries"}]}
```

### 3.7 Create an image of the app server

**EC2 > Instances > select `app-tier-server` > Actions > Image and templates > Create image**

- Name: `app-tier-ami`

Wait until the AMI status is **Available**.

### 3.8 Target group and internal load balancer

**Target group** (EC2 > Target Groups > Create)

- Type: Instances, name: `app-tier-tg`
- Protocol/port: HTTP, `4000`
- VPC: `three-tier-vpc`
- Health check path: `/health`
- Skip registering targets (the Auto Scaling group does this)

**Internal load balancer** (EC2 > Load Balancers > Create > Application Load Balancer)

- Name: `app-tier-internal-lb`
- Scheme: **Internal**
- VPC: `three-tier-vpc`, subnets: `private-app-subnet-az1` and `private-app-subnet-az2`
- Security group: `internal-lb-sg`
- Listener: HTTP 80, forward to `app-tier-tg`

Copy the **DNS name** of this load balancer. You need it in Part 4.

### 3.9 Launch template and Auto Scaling group

**Launch template**

- Name: `app-tier-launch-template`
- AMI: `app-tier-ami` (My AMIs)
- Instance type: `t2.micro`
- Security group: `private-instance-sg`
- IAM instance profile: `3TierEC2Role`

**Auto Scaling group**

- Name: `app-tier-asg`
- Launch template: `app-tier-launch-template`
- VPC: `three-tier-vpc`, subnets: both `private-app-subnet-*`
- Load balancing: attach to existing target group `app-tier-tg`
- Health checks: turn on **ELB** health checks
- Capacity: desired `2`, minimum `2`, maximum `2`

Within a few minutes both targets should show **healthy** in `app-tier-tg`.

---

## Part 4: Web tier (React and Nginx)

### 4.1 Point Nginx at the internal load balancer

On your local machine, open `application-code/nginx.conf` and find:

```nginx
location /api/{
        proxy_pass http://[REPLACE-WITH-INTERNAL-LB-DNS]:80/;
}
```

Replace `[REPLACE-WITH-INTERNAL-LB-DNS]` with the internal ALB DNS name from step 3.8. Upload the edited file to `s3://<your-bucket>/nginx.conf`.

### 4.2 Launch the web server

**EC2 > Launch instance**

- Name: `web-tier-server`
- AMI: Amazon Linux 2023, type `t2.micro`
- Network: `three-tier-vpc`, subnet `public-web-subnet-az1`, auto-assign public IP **enabled**
- Security group: `web-tier-sg`
- IAM instance profile: `3TierEC2Role`

Connect with Session Manager and run:

```bash
sudo -su ec2-user
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

cd ~/
aws s3 cp s3://<your-bucket>/web-tier/ web-tier --recursive
cd web-tier
npm install
npm run build
```

### 4.3 Install and configure Nginx

```bash
sudo dnf install -y nginx
cd /etc/nginx
sudo rm nginx.conf
sudo aws s3 cp s3://<your-bucket>/nginx.conf .
sudo service nginx restart
chmod -R 755 /home/ec2-user
sudo chkconfig nginx on
```

Test it by opening `http://<web-server-public-ip>/` in a browser. You should see the app, and `http://<web-server-public-ip>/health` should return "Web Tier Health Check".

### 4.4 Create an image of the web server

**EC2 > Instances > select `web-tier-server` > Actions > Image and templates > Create image**, name it `web-tier-ami`.

### 4.5 Target group and public load balancer

**Target group**

- Name: `web-tier-tg`, HTTP port `80`, VPC `three-tier-vpc`
- Health check path: `/health`

**Internet-facing load balancer**

- Name: `web-tier-external-lb`
- Scheme: **Internet-facing**
- Subnets: `public-web-subnet-az1` and `public-web-subnet-az2`
- Security group: `internet-facing-lb-sg`
- Listener: HTTP 80, forward to `web-tier-tg`

### 4.6 Launch template and Auto Scaling group

**Launch template** `web-tier-launch-template`: AMI `web-tier-ami`, type `t2.micro`, security group `web-tier-sg`, IAM profile `3TierEC2Role`.

**Auto Scaling group** `web-tier-asg`:

- Subnets: both `public-web-subnet-*`
- Attach to existing target group `web-tier-tg`, ELB health checks on
- Capacity: desired `2`, minimum `2`, maximum `2`

---

## Testing the full stack

1. Copy the DNS name of `web-tier-external-lb` and open `http://<that-dns-name>` in a browser.
2. Open the **DB Demo** page from the menu.
3. Add a transaction (amount and description, then click **ADD**). It should appear in the list.
4. Use the **DEL** button to clear the table.
5. To test high availability, terminate one instance from `web-tier-asg` or `app-tier-asg` and watch Auto Scaling replace it while the site stays up.

## Code walkthrough

### App tier API (`application-code/app-tier/index.js`)

| Method and path | What it does |
|---|---|
| `GET /health` | Health check used by the internal ALB |
| `GET /transaction` | Returns all transactions |
| `POST /transaction` | Adds a transaction (`amount`, `desc` in the body) |
| `DELETE /transaction` | Deletes all transactions |
| `GET /transaction/id` | Gets one transaction by `id` in the body |
| `DELETE /transaction/id` | Deletes one transaction by `id` in the body |

- `TransactionService.js` opens a MySQL connection using the values in `DbConfig.js` and runs the queries.
- The server listens on port **4000**, which is why the target group, security group and ALB are set up for that port.

### Web tier

- `src/components/DatabaseDemo/DatabaseDemo.js` calls `/api/transaction`. Nginx strips `/api/` and forwards the request to the internal load balancer.
- `npm run build` produces static files in `web-tier/build`, which Nginx serves from `/home/ec2-user/web-tier/build`.

### Nginx (`application-code/nginx.conf`)

- `/health` returns a static 200 response for the public ALB health check
- `/` serves the React build and falls back to `index.html` for client-side routing
- `/api/` is proxied to the internal load balancer

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Cannot connect with Session Manager | Instance role missing `AmazonSSMManagedInstanceCore`, or no route to the internet through the NAT Gateway |
| `ping 8.8.8.8` fails on the app server | Private route table is missing the `0.0.0.0/0` route to the NAT Gateway |
| `mysql` command times out | `db-sg` does not allow port 3306 from `private-instance-sg` |
| `pm2 list` shows `errored` | Wrong values in `DbConfig.js`, run `pm2 logs` to see the MySQL error |
| App-tier targets are unhealthy | `private-instance-sg` must allow port 4000 from `internal-lb-sg`, and the target group health check path must be `/health` |
| Website loads but DB Demo shows no data | `proxy_pass` in `nginx.conf` still has the placeholder, or `internal-lb-sg` does not allow `web-tier-sg` |
| 502 Bad Gateway from the public ALB | Nginx is not running: `sudo service nginx status` |
| Web-tier targets are unhealthy | `web-tier-sg` must allow port 80 from `internet-facing-lb-sg` |

## Clean up

Delete resources in this order to stop charges:

1. Auto Scaling groups: `web-tier-asg`, `app-tier-asg`
2. Load balancers: `web-tier-external-lb`, `app-tier-internal-lb`
3. Target groups: `web-tier-tg`, `app-tier-tg`
4. Launch templates and the two AMIs (also delete their snapshots)
5. Original EC2 instances if still running
6. Aurora cluster and its subnet group
7. NAT Gateways, then release the Elastic IPs
8. Internet Gateway (detach, then delete)
9. Subnets, route tables, security groups, then the VPC
10. S3 bucket (empty it first) and the IAM role

## Known limitations

This is learning code and is not production ready:

- `TransactionService.js` builds SQL by string concatenation, which is open to SQL injection. Use parameterized queries (`con.query(sql, [values])`) in a real application.
- Database credentials sit in `DbConfig.js`. For production, use AWS Secrets Manager or SSM Parameter Store.
- The load balancers use HTTP only. Add an ACM certificate and an HTTPS listener for production.
- `POST /transaction` replies success before the insert finishes, so insert errors are not reported to the caller.

## License

This project is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
