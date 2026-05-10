# Web-app-deployment-aws-asg-alb
# 🍽️ Spice Garden — AWS Production Web App Deployment
![image alt](https://github.com/Y-Vishanth/Web-app-deployment-aws-asg-alb/blob/baee4f8b3492cf8ffb236b12d4d5d5606cc0f42f/screenshot.jpeg)
A complete step-by-step guide to deploying a production-ready web application on AWS from scratch using only the AWS Console. This guide covers VPC, EC2, Application Load Balancer, Auto Scaling, CloudWatch monitoring, and SNS notifications — all on the AWS Free Tier.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Phase 1 — AWS Account & IAM](#phase-1--aws-account--iam)
- [Phase 2 — VPC & Networking](#phase-2--vpc--networking)
- [Phase 3 — EC2 Instances](#phase-3--ec2-instances)
- [Phase 4 — Application Load Balancer](#phase-4--application-load-balancer)
- [Phase 5 — Auto Scaling Group](#phase-5--auto-scaling-group)
- [Phase 6 — CloudWatch & SNS](#phase-6--cloudwatch--sns)
- [Phase 7 — Testing & Verification](#phase-7--testing--verification)
- [Common Errors & Fixes](#common-errors--fixes)
---

## 🏗️ Architecture Overview

```
                          ┌─────────────────────────────────────────┐
                          │              AWS Region                  │
  User ──► Internet ──►  │                                          │
          Gateway         │   ┌──────────────┐  ┌──────────────┐   │
                          │   │  AZ 1a       │  │  AZ 1b       │   │
                          │   │  Public      │  │  Public      │   │
                          │   │  Subnet      │  │  Subnet      │   │
                          │   │  10.0.1.0/24 │  │  10.0.2.0/24│   │
                          │   │              │  │              │   │
                          │   │  ┌────────┐  │  │  ┌────────┐ │   │
                          │   │  │  EC2   │  │  │  │  EC2   │ │   │
                          │   │  │Server 1│◄─┼──┼─►│Server 2│ │   │
                          │   │  └────────┘  │  │  └────────┘ │   │
                          │   └──────────────┘  └─────────────┘   │
                          │              │    ALB    │              │
                          │              └─────┬─────┘              │
                          │                    │                    │
                          │               ┌────▼────┐              │
                          │               │   ASG   │              │
                          │               └─────────┘              │
                          │                                         │
                          │   CloudWatch Alarm ──► SNS ──► Email   │
                          └─────────────────────────────────────────┘
```

| Phase | Service | Purpose |
|-------|---------|---------|
| 1 | IAM | Secure admin user, MFA, access control |
| 2 | VPC + Subnets + IGW | Isolated network across 2 Availability Zones |
| 3 | EC2 (x2) | Web servers running Apache/httpd |
| 4 | Application Load Balancer | Distributes traffic between EC2 instances |
| 5 | Auto Scaling Group | Automatically scales EC2 count based on load |
| 6 | CloudWatch + SNS | Monitors CPU and sends email alerts |
| 7 | Testing & Cleanup | Verify everything works, then delete resources |

**Estimated time:** 3–4 hours  
**Cost:** $0–5/month (AWS Free Tier eligible)  
**Region used:** `ap-south-1` (Mumbai)

---

## ✅ Prerequisites

- A computer with internet access
- A valid email address
- A credit card (won't be charged if you stay within Free Tier)
- Basic comfort using a web browser

No prior AWS knowledge required. This guide is written for complete beginners.

---

## Phase 1 — AWS Account & IAM

### Step 1.1 — Create Your AWS Account

Go to [aws.amazon.com](https://aws.amazon.com) and click "Create an AWS Account". Enter your email, set a password, and give your account a name. You'll need to enter a credit card for identity verification — you won't be charged as long as you stay within the Free Tier limits. Complete the phone verification and select the "Basic support" (free) plan.

### Step 1.2 — Enable MFA on Root Account

After logging in, click your account name in the top right corner and go to Security credentials. Scroll down to "Multi-factor authentication (MFA)" and click "Assign MFA device". Choose "Authenticator app", scan the QR code using Google Authenticator or Authy on your phone, and enter two consecutive codes to confirm.

> ⚠️ **Important:** Your root account is the most powerful account in AWS. Always protect it with MFA and never use it for daily work.

### Step 1.3 — Create an IAM Admin User

Search for "IAM" in the AWS Console search bar and click on it. Go to Users → Create user and fill in the following:

- **Username:** anything you like (e.g. `admin-user`)
- **Console access:** Enabled — select "I want to create an IAM user"
- **Password:** set a strong password
- **Permissions:** click "Attach policies directly" → search and select `AdministratorAccess`

Click through to Create user and download the CSV file with the login credentials. Keep this file safe.

### Step 1.4 — Log In as IAM Admin

Sign out of the root account. Go to IAM → Dashboard and copy the "Sign-in URL for IAM users". It looks like `https://123456789.signin.aws.amazon.com/console`. Bookmark this URL — it's your daily login from now on. Log in using your IAM admin username and password.

> ✅ From this point forward, always use the IAM admin user — never the root account.

---

## Phase 2 — VPC & Networking

A VPC (Virtual Private Cloud) is your own private network on AWS. You'll create one VPC with two public subnets spread across two Availability Zones so that if one data center has issues, your app still runs in the other.

### Step 2.1 — Create the VPC

Search "VPC" in the console and click "Your VPCs" → Create VPC. Select "VPC only" (not "VPC and more").

| Setting | Value |
|---------|-------|
| Name tag | `my-app-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 CIDR | No IPv6 CIDR block |
| Tenancy | Default |

Click Create VPC.

### Step 2.2 — Create Two Public Subnets

Go to VPC → Subnets → Create subnet. Select `my-app-vpc` as the VPC, then add two subnets:

| Setting | Subnet 1 | Subnet 2 |
|---------|----------|----------|
| Name | `public-subnet-1a` | `public-subnet-1b` |
| Availability Zone | `ap-south-1a` | `ap-south-1b` |
| IPv4 CIDR | `10.0.1.0/24` | `10.0.2.0/24` |

After creating both subnets, enable auto-assign public IP on each one. Select each subnet → Actions → Edit subnet settings → check "Enable auto-assign public IPv4 address" → Save.

### Step 2.3 — Create & Attach Internet Gateway

Go to VPC → Internet Gateways → Create internet gateway. Name it `my-app-igw` and click Create. Then select it → Actions → Attach to VPC → choose `my-app-vpc` → Attach internet gateway.

The Internet Gateway is what connects your VPC to the public internet. Without it, no traffic can reach your website from outside.

### Step 2.4 — Update the Route Table

Go to VPC → Route Tables and find the route table associated with `my-app-vpc`. Click on it and go to the Routes tab → Edit routes → Add route:

- **Destination:** `0.0.0.0/0`
- **Target:** Internet Gateway → `my-app-igw`

Save changes. Then go to the Subnet Associations tab → Edit subnet associations → check both `public-subnet-1a` and `public-subnet-1b` → Save associations.

---

## Phase 3 — EC2 Instances

### Step 3.1 — Create a Key Pair

Go to EC2 → Key Pairs (left sidebar under Network & Security) → Create key pair.

| Setting | Value |
|---------|-------|
| Name | `my-app-key` |
| Key pair type | RSA |
| Format (Mac/Linux) | `.pem` |
| Format (Windows) | `.ppk` |

The file will download automatically. Save it somewhere safe on your computer — you cannot download it again.

### Step 3.2 — Create Security Groups

Create the ALB security group first because you'll need it when setting up the EC2 security group.

Go to EC2 → Security Groups → Create security group.

**ALB Security Group (`alb-sg`):**

| Setting | Value |
|---------|-------|
| Name | `alb-sg` |
| Description | Allow HTTP from internet |
| VPC | `my-app-vpc` |
| Inbound rule | HTTP, Port 80, Source: `0.0.0.0/0` |

**EC2 Security Group (`ec2-sg`):**

| Setting | Value |
|---------|-------|
| Name | `ec2-sg` |
| Description | Allow HTTP from ALB and SSH from my IP |
| VPC | `my-app-vpc` |
| Inbound rule 1 | HTTP, Port 80, Source: `alb-sg` |
| Inbound rule 2 | SSH, Port 22, Source: My IP |

> ⚠️ **Important:** When adding the Source for the HTTP rule in `ec2-sg`, type `sg-` and select `alb-sg` from the dropdown. Make sure only ONE security group ID appears in the source field. Having two entries will cause a validation error.

### Step 3.3 — Launch EC2 Instance 1

Go to EC2 → Instances → Launch instances and fill in the following:

| Setting | Value |
|---------|-------|
| Name | `app-server-1` |
| AMI | Amazon Linux 2023 (Free Tier) |
| Instance type | `t2.micro` (Free Tier) |
| Key pair | `my-app-key` |
| VPC | `my-app-vpc` |
| Subnet | `public-subnet-1a` |
| Auto-assign Public IP | Enable |
| Security group | `ec2-sg` (existing) |

Scroll to the bottom, expand "Advanced details", and paste this in the User data field:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

Click Launch instance.

### Step 3.4 — Launch EC2 Instance 2

Repeat the same steps but with these differences:

- **Name:** `app-server-2`
- **Subnet:** `public-subnet-1b`
- In the User data script, change `Server 1` to `Server 2`

### Step 3.5 — Verify Instances are Running

Go to EC2 → Instances. Both instances should show "Running" with a green dot. Wait 2–3 minutes until both show "2/2 checks passed" in the Status check column before moving to the next phase.

> 💡 **Free Tier note:** `t2.micro` gives you 750 free hours per month. Stop instances when you're not actively working on this project.

---

## Phase 4 — Application Load Balancer

The ALB sits in front of your EC2 instances and distributes incoming traffic between them so neither server gets overwhelmed.

### Step 4.1 — Create a Target Group

The target group is the list of servers the ALB will send traffic to. Go to EC2 → Target Groups → Create target group.

| Setting | Value |
|---------|-------|
| Target type | Instances |
| Name | `app-target-group` |
| Protocol | HTTP |
| Port | 80 |
| VPC | `my-app-vpc` |
| Health check path | `/` |
| Healthy threshold | 2 |
| Unhealthy threshold | 2 |
| Interval | 30 seconds |

On the next screen, select both `app-server-1` and `app-server-2`, click "Include as pending below", then Create target group.

### Step 4.2 — Create the Application Load Balancer

Go to EC2 → Load Balancers → Create load balancer → Application Load Balancer.

| Setting | Value |
|---------|-------|
| Name | `my-app-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `my-app-vpc` |
| Availability Zones | `ap-south-1a` → `public-subnet-1a` |
| | `ap-south-1b` → `public-subnet-1b` |
| Security group | `alb-sg` (remove the default one) |
| Listener Protocol | HTTP |
| Listener Port | 80 |
| Default action | Forward to `app-target-group` |

Click Create load balancer. It takes about 2–3 minutes to become Active.

> 💡 After creation, go to EC2 → Load Balancers, select your ALB, and copy the DNS name. It looks like `my-app-alb-123456789.ap-south-1.elb.amazonaws.com`. This is your website URL.

### Step 4.3 — Verify Target Group Health

Go to EC2 → Target Groups → `app-target-group` → Targets tab. Both instances should show "healthy" status. If they show "unhealthy", SSH into the instance and run `sudo systemctl start httpd`, then wait 30 seconds and refresh.

### Step 4.4 — Test the Website

Paste the ALB DNS name into your browser. You should see your web page. Refresh a few times — you'll notice the response alternates between Server 1 and Server 2, which means the load balancer is working correctly.

---

## Phase 5 — Auto Scaling Group

The Auto Scaling Group automatically adds more EC2 instances when traffic is high and removes them when traffic drops — so your app handles spikes without any manual work.

### Step 5.1 — Create a Launch Template

The launch template is the blueprint ASG uses to spin up new EC2 instances. Go to EC2 → Launch Templates → Create launch template.

| Setting | Value |
|---------|-------|
| Template name | `my-app-launch-template` |
| Auto Scaling guidance | Check this box ✓ |
| AMI | Amazon Linux 2023 (Free Tier) |
| Instance type | `t2.micro` (Free Tier) |
| Key pair | `my-app-key` |
| Security group | `ec2-sg` |
| Subnet | Do NOT include in template |

In Advanced details → User data, paste the same startup script from Phase 3, then click Create launch template.

### Step 5.2 — Create the Auto Scaling Group

Go to EC2 → Auto Scaling Groups → Create Auto Scaling group and go through each page:

**Page 1:** Name it `my-app-asg` and select `my-app-launch-template`.

**Page 2:** Set VPC to `my-app-vpc` and select both `public-subnet-1a` and `public-subnet-1b`.

**Page 3:** Under Load balancing, choose "Attach to an existing load balancer" and select `app-target-group`. Enable ELB health checks.

**Page 4 — Group size and scaling:**

| Setting | Value |
|---------|-------|
| Desired capacity | 2 |
| Minimum capacity | 1 |
| Maximum capacity | 3 |
| Scaling policy | Target tracking scaling |
| Metric type | Average CPU utilization |
| Target value | 50% |
| Instance warmup | 180 seconds |

**Page 5 — Tags:** Add Key: `Name`, Value: `asg-instance` so new ASG instances get a proper name in the console.

Click Create Auto Scaling group.

> ⚠️ **Free Tier note:** Keep maximum capacity at 3. After creating the ASG, stop your original `app-server-1` and `app-server-2` manually — the ASG will manage its own instances going forward.

---

## Phase 6 — CloudWatch & SNS

This phase sets up monitoring so you get an email alert whenever your servers are under high load.

### Step 6.1 — Create an SNS Topic

Search "SNS" in the console and go to Topics → Create topic.

| Setting | Value |
|---------|-------|
| Type | **Standard** (NOT FIFO) |
| Name | `app-alerts` |
| Display name | `AppAlerts` |

> ⚠️ **Important:** You must choose "Standard" type. FIFO topics do not support email subscriptions. If your topic name ends with `.fifo`, delete it and start over with a Standard topic.

### Step 6.2 — Subscribe Your Email

Inside the `app-alerts` topic, click Create subscription.

| Setting | Value |
|---------|-------|
| Protocol | Email |
| Endpoint | your-email@gmail.com |

After creating the subscription, check your email inbox for a message with subject "AWS Notification - Subscription Confirmation". Check your Spam/Junk folder if you don't see it in the inbox — AWS emails often land there. Click "Confirm subscription" in the email. The subscription status on the SNS page should change from "Pending confirmation" to "Confirmed".

### Step 6.3 — Create a CloudWatch Alarm

Go to CloudWatch → Alarms → All alarms → Create alarm → Select metric → EC2 → By Auto Scaling Group → CPUUtilization → select `my-app-asg` → Select metric.

| Setting | Value |
|---------|-------|
| Statistic | Average |
| Period | 5 minutes |
| Threshold type | Static |
| Condition | Greater than |
| Threshold value | 70% |
| Notification | Send to `app-alerts` SNS topic |
| Alarm name | `high-cpu-alarm` |

This alarm fires when average CPU stays above 70% for 5 consecutive minutes and sends you an email.

### Step 6.4 — Create a Scaling Policy

Go to EC2 → Auto Scaling Groups → `my-app-asg` → Automatic scaling tab → Create dynamic scaling policy.

| Setting | Value |
|---------|-------|
| Policy type | Target tracking scaling |
| Policy name | `scale-out-cpu-tracking` |
| Metric type | Average CPU utilization |
| Target value | 50% |
| Instance warmup | 180 seconds |

Click Create. AWS automatically creates both scale-out and scale-in alarms for you — no need to configure them separately.

### Understanding Your CloudWatch Alarms

After setup you'll see 3 alarms in CloudWatch. This is completely normal:

| Alarm | Expected State | What It Means |
|-------|----------------|---------------|
| `TargetTracking-...-AlarmLow` | In Alarm | CPU is low (servers are idle) — normal with no real traffic |
| `TargetTracking-...-AlarmHigh` | OK | CPU is normal, no scale-out needed |
| `high-cpu-alarm` | OK | Manual alarm, CPU not exceeding 70% |

The "AlarmLow In Alarm" state is expected. It means CPU is low and auto scaling may reduce the instance count to save costs — this is the system working exactly as designed.

---

## Phase 7 — Testing & Verification

### Step 7.1 — Upload Your Website HTML

This project deploys a Spice Garden restaurant website. Each server gets a slightly different version with a colored badge in the bottom-right corner so you can tell which server is responding:

- **app-server-1** → 🟢 Green badge: "Server 1 — AZ ap-south-1a"
- **app-server-2** → 🔵 Blue badge: "Server 2 — AZ ap-south-1b"
- **ASG instances** → 🟣 Purple badge: "ASG Instance — Auto Scaled"

Upload the HTML files from your local terminal. Open Command Prompt or Terminal in the folder where your `.pem` file and HTML files are saved:

```bash
# Upload to app-server-1
scp -i "my-app-key.pem" server1-index.html ec2-user@<SERVER-1-PUBLIC-IP>:/tmp/
ssh -i "my-app-key.pem" ec2-user@<SERVER-1-PUBLIC-IP> "sudo cp /tmp/server1-index.html /var/www/html/index.html"

# Upload to app-server-2
scp -i "my-app-key.pem" server2-index.html ec2-user@<SERVER-2-PUBLIC-IP>:/tmp/
ssh -i "my-app-key.pem" ec2-user@<SERVER-2-PUBLIC-IP> "sudo cp /tmp/server2-index.html /var/www/html/index.html"
```

Replace `<SERVER-1-PUBLIC-IP>` and `<SERVER-2-PUBLIC-IP>` with the public IPs shown in EC2 → Instances.

### Step 7.2 — Test Load Balancing

Open the ALB DNS name in your browser and press `Ctrl + Shift + R` (hard refresh) several times. The badge in the bottom-right corner should alternate between green and blue — this confirms the ALB is successfully splitting traffic between both servers.

> 💡 If the badge doesn't switch, try opening the URL in an Incognito/Private window. Browsers cache pages aggressively which can make it seem like only one server is responding.

### Step 7.3 — Full Verification Checklist

Go through each of these in the AWS Console before considering the setup complete:

- [ ] VPC `my-app-vpc` exists with CIDR `10.0.0.0/16`
- [ ] Two subnets exist in `ap-south-1a` and `ap-south-1b`
- [ ] Internet Gateway is attached to the VPC (State: Attached)
- [ ] Route table has `0.0.0.0/0 → my-app-igw` and both subnets associated
- [ ] Both EC2 instances show 2/2 status checks passed
- [ ] `ec2-sg` HTTP inbound source is `alb-sg` (not `0.0.0.0/0`)
- [ ] ALB state is Active
- [ ] All instances show Healthy in the target group
- [ ] ASG shows Desired: 2, Min: 1, Max: 3
- [ ] Scaling policy is listed in ASG → Automatic scaling tab
- [ ] `high-cpu-alarm` state is OK in CloudWatch
- [ ] SNS subscription status is Confirmed
- [ ] Website loads correctly via ALB DNS URL in browser
- [ ] Green and blue badges alternate on hard refresh

---

---

## Common Errors & Fixes

**Target group shows Unhealthy instances**
SSH into the EC2 instance and run `sudo systemctl start httpd`. Wait 30 seconds and refresh the target group health status. If still unhealthy, check that `ec2-sg` allows HTTP traffic from `alb-sg`.

**ALB returns 502 Bad Gateway**
This usually means the EC2 security group is blocking traffic. Check that `ec2-sg` has an HTTP inbound rule with source set to `alb-sg`, not `0.0.0.0/0`. Also confirm Apache is running on the instance with `sudo systemctl status httpd`.

**EC2 security group error: "You may not specify a referenced group id for an existing IPv4 CIDR rule"**
You accidentally added two entries in the Source field. Click the X on both entries to clear the field completely, then type `sg-` and select `alb-sg` from the dropdown — only one entry should appear in the source.

**SNS topic only shows "Amazon SQS" as the protocol option**
You created a FIFO topic by mistake. Delete it and create a new Standard topic. The topic name should NOT end with `.fifo`. Standard topics support Email, HTTP, Lambda, and more.

**Step scaling policy error about lower bound being null**
Switch from Step scaling to Target tracking scaling instead. Target tracking is simpler, handles the step configuration automatically, and is the recommended approach for CPU-based scaling.

**Internet Gateway cannot be deleted**
You must detach it from the VPC before deleting. Go to Actions → Detach from VPC, confirm the detachment, then go to Actions → Delete internet gateway.

**Security group shows "in use" error when deleting**
The ALB or EC2 instances haven't fully terminated yet. Wait 2–3 minutes and try again.

**Plain text "Hello from hostname" page still appears on some refreshes**
Some ASG instances were launched with the old user data script. Update the launch template to the latest version, then terminate the instances named `asg-instance` in EC2 → Instances. The ASG will replace them with fresh instances using the updated template automatically.

**Badge doesn't switch between servers on refresh**
Your browser is caching the page. Open the ALB URL in an Incognito/Private window, or press `Ctrl + Shift + R` for a hard refresh. The ALB uses round-robin routing so it won't always alternate perfectly on every single refresh.

---


## 🙏 Acknowledgements

Built and documented by **Vishanth** as a hands-on learning project covering real-world AWS infrastructure from scratch.

Deployed in region `ap-south-1` (Mumbai, India) using the AWS Console — no CLI or Infrastructure as Code tools required. Suitable for anyone learning AWS cloud architecture for the first time.

---

*If this guide helped you, consider giving the repo a ⭐*
