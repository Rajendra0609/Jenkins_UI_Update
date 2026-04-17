# Jenkins EC2 Cloud Agent Setup Guide

> **Use Case:** Jenkins controller running on a **home network** → dynamically spin up **AWS EC2 instances** as build agents when a job label matches.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Prerequisites](#prerequisites)
3. [Step 1 — AWS Infrastructure Setup](#step-1--aws-infrastructure-setup)
4. [Step 2 — IAM User & Permissions](#step-2--iam-user--permissions)
5. [Step 3 — EC2 Key Pair](#step-3--ec2-key-pair)
6. [Step 4 — Install Jenkins EC2 Plugin](#step-4--install-jenkins-ec2-plugin)
7. [Step 5 — Configure the Cloud in Jenkins](#step-5--configure-the-cloud-in-jenkins)
8. [Step 6 — Configure AMI / Agent Template](#step-6--configure-ami--agent-template)
9. [Step 7 — Use the Label in Your Pipeline](#step-7--use-the-label-in-your-pipeline)
10. [Home Network Considerations](#home-network-considerations)
11. [End-to-End Flow](#end-to-end-flow)
12. [Cost Optimization Tips](#cost-optimization-tips)
13. [Troubleshooting](#troubleshooting)

---

## How It Works

```
┌─────────────────────────┐          ┌──────────────────────────────┐
│  Home Network           │          │  AWS Cloud (ap-south-1)      │
│                         │          │                              │
│  Jenkins Controller     │──SSH────►│  EC2 Agent (spun on demand)  │
│  (your PC/server)       │          │  Public Subnet               │
│                         │◄─────────│  Auto-terminated after idle  │
└─────────────────────────┘          └──────────────────────────────┘
         │
         │  1. Job queued with label 'ec2-agent'
         │  2. EC2 Plugin calls AWS API → launches instance
         │  3. Jenkins SSHes to EC2 public IP
         │  4. Build runs on EC2
         │  5. Instance terminated after idle timeout
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Jenkins version | 2.300+ (you have 2.541.2 ✅) |
| AWS Account | Active account with billing enabled |
| Home internet | Public IP (static preferred) or Dynamic DNS |
| Java on agent AMI | Java 17 (installed via init script) |

---

## Step 1 — AWS Infrastructure Setup

You need a **Public Subnet** so that your home Jenkins can reach EC2 agents over the internet via SSH.

### 1.1 — Create a VPC

1. Go to **AWS Console → VPC → Create VPC**
2. Fill in:

```
Name        :  jenkins-agents-vpc
IPv4 CIDR   :  10.0.0.0/16
Tenancy     :  Default
```

3. Click **Create VPC**

---

### 1.2 — Create a Public Subnet

1. Go to **VPC → Subnets → Create Subnet**
2. Fill in:

```
VPC            :  jenkins-agents-vpc
Subnet name    :  jenkins-agents-public-subnet
Availability Zone : ap-south-1a   (or any)
IPv4 CIDR      :  10.0.1.0/24
```

3. Click **Create Subnet**
4. Select the subnet → **Actions → Edit Subnet Settings**
5. Enable ✅ **Auto-assign public IPv4 address** → Save

> ⚠️ This is critical — EC2 agents must get a public IP so Jenkins at home can SSH into them.

---

### 1.3 — Create and Attach an Internet Gateway

1. Go to **VPC → Internet Gateways → Create Internet Gateway**
2. Name it: `jenkins-agents-igw`
3. Click **Create**, then **Actions → Attach to VPC**
4. Select `jenkins-agents-vpc` → **Attach**

---

### 1.4 — Configure the Route Table

1. Go to **VPC → Route Tables**
2. Find the route table associated with `jenkins-agents-vpc`
3. Click **Edit Routes → Add Route**:

```
Destination :  0.0.0.0/0
Target      :  jenkins-agents-igw   (your Internet Gateway)
```

4. Click **Save Routes**
5. Go to **Subnet Associations → Edit** → associate `jenkins-agents-public-subnet`

---

### 1.5 — Create a Security Group

1. Go to **EC2 → Security Groups → Create Security Group**
2. Fill in:

```
Name        :  jenkins-agent-sg
Description :  Allow Jenkins SSH from home
VPC         :  jenkins-agents-vpc
```

3. Add **Inbound Rules**:

| Type | Protocol | Port | Source | Description |
|---|---|---|---|---|
| SSH | TCP | 22 | Your home public IP/32 | Jenkins SSH access |

> 🔒 **Security tip:** Use `YOUR_HOME_IP/32` (not `0.0.0.0/0`) to restrict SSH to only your home IP. Find your IP at https://whatismyip.com

4. **Outbound Rules** — leave default (allow all outbound)
5. Click **Create Security Group**

---

### AWS Infrastructure Summary

```
VPC: jenkins-agents-vpc (10.0.0.0/16)
  │
  ├── Internet Gateway: jenkins-agents-igw
  │         │
  ├── Public Subnet: jenkins-agents-public-subnet (10.0.1.0/24)
  │         ├── Auto-assign public IP: Enabled ✅
  │         └── Route: 0.0.0.0/0 → jenkins-agents-igw
  │
  └── Security Group: jenkins-agent-sg
            └── Inbound: Port 22 from <your-home-ip>/32
```

---

## Step 2 — IAM User & Permissions

Jenkins needs AWS credentials to call the EC2 API to launch and terminate instances.

### 2.1 — Create IAM Policy

1. Go to **IAM → Policies → Create Policy**
2. Choose **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:TerminateInstances",
        "ec2:RequestSpotInstances",
        "ec2:DeleteTags",
        "ec2:CreateTags",
        "ec2:DescribeRegions",
        "ec2:RunInstances",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeImages",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeKeyPairs",
        "ec2:GetConsoleOutput",
        "ec2:DescribeInstanceStatus",
        "ec2:StopInstances",
        "ec2:StartInstances",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    }
  ]
}
```

3. Name it: `JenkinsEC2AgentPolicy`
4. Click **Create Policy**

---

### 2.2 — Create IAM User

1. Go to **IAM → Users → Create User**
2. Username: `jenkins-ec2-agent-user`
3. **Permissions**: Attach `JenkinsEC2AgentPolicy` directly
4. Click **Create User**

### 2.3 — Generate Access Keys

1. Click on the user → **Security Credentials** tab
2. **Create Access Key** → Select **Application running outside AWS**
3. Download or note:
   - `Access Key ID`
   - `Secret Access Key`

> ⚠️ Save these — the secret key is shown only once.

---

## Step 3 — EC2 Key Pair

Jenkins SSHes into the agent using a key pair.

1. Go to **EC2 → Key Pairs → Create Key Pair**
2. Fill in:

```
Name            :  jenkins-agent-key
Key pair type   :  RSA
Private key format : .pem
```

3. Click **Create** — the `.pem` file downloads automatically
4. Keep this file safe — you will paste its contents into Jenkins

---

## Step 4 — Install Jenkins EC2 Plugin

1. Go to **Jenkins → Manage Jenkins → Plugins → Available Plugins**
2. Search for: `Amazon EC2`
3. Install **Amazon EC2** plugin
4. Restart Jenkins when prompted

---

## Step 5 — Configure the Cloud in Jenkins

1. Go to **Manage Jenkins → Clouds → New Cloud**
2. Fill in:

```
Cloud name :  aws-ec2-agents
Type       :  Amazon EC2   ✅ (select this radio button)
```

3. Click **Create**

---

### 5.1 — Add AWS Credentials

On the cloud configuration page:

1. Next to **Amazon EC2 Credentials** → click **Add → Jenkins**
2. Fill in:

```
Kind             :  AWS Credentials
ID               :  aws-jenkins-ec2-creds
Description      :  AWS credentials for EC2 agents
Access Key ID    :  <your IAM access key>
Secret Access Key:  <your IAM secret key>
```

3. Click **Add**, then select `aws-jenkins-ec2-creds` from the dropdown

---

### 5.2 — Set Region and Private Key

```
Region                    :  ap-south-1   (Mumbai — closest to Andhra Pradesh)
EC2 Key Pair's Private Key:  (paste full contents of jenkins-agent-key.pem)
```

To get the key content:
```bash
cat jenkins-agent-key.pem
```
Paste the entire output including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`.

---

## Step 6 — Configure AMI / Agent Template

Click **Add AMI** on the cloud config page and fill in:

### 6.1 — Basic AMI Settings

| Field | Value |
|---|---|
| **AMI ID** | `ami-0f58b397bc5c1f2e8` *(Ubuntu 22.04, ap-south-1)* |
| **Instance Type** | `t3.medium` |
| **Security group names** | `jenkins-agent-sg` |
| **Subnet ID** | *(paste your public subnet ID from AWS)* |

### 6.2 — Label (Most Important)

```
Labels  :  ec2-agent
```

> 🎯 This is what jobs use to request this agent. Any pipeline with `agent { label 'ec2-agent' }` will trigger an EC2 instance launch.

### 6.3 — SSH Settings

```
Remote user          :  ubuntu
Remote FS root       :  /home/ubuntu/jenkins
SSH port             :  22
```

### 6.4 — Lifecycle Settings

```
Instance Cap             :  5      (max simultaneous agents)
Idle termination time    :  30     (minutes before auto-terminate)
Min number of instances  :  0      (scale to zero when idle)
```

### 6.5 — Init Script

This runs once when the agent boots. It installs Java and other tools:

```bash
#!/bin/bash
set -e
sudo apt-get update -y
sudo apt-get install -y \
  openjdk-17-jdk \
  git \
  maven \
  curl \
  unzip \
  docker.io

# Allow ubuntu user to use docker
sudo usermod -aG docker ubuntu

# Verify Java installation
java -version
```

> Jenkins requires Java to be installed on the agent to run the agent JAR.

### 6.6 — (Optional) Use Spot Instances

Check ✅ **Use Spot Instance** to save up to 70% on compute costs:

```
Spot Max Bid Price  :  0.05   (USD/hr — slightly above on-demand)
```

---

### Full AMI Configuration Summary

```
AMI ID               :  ami-0f58b397bc5c1f2e8
Instance Type        :  t3.medium
Labels               :  ec2-agent
Remote user          :  ubuntu
Remote FS root       :  /home/ubuntu/jenkins
Security Group       :  jenkins-agent-sg
Subnet ID            :  <your-public-subnet-id>
SSH Port             :  22
Instance Cap         :  5
Idle termination     :  30 min
Init Script          :  (Java + Git + Maven install)
``

FieldRecommended ValueRemote FS root/var/lib/jenkinsRemote userec2-user (AL2023 default) or jenkinsAMI TypeunixJava Path/usr/bin/javaLabelse.g. linux al2023 (to target this agent in jobs)Idle termination time30 (minutes) — avoids runaway costsHost Key Verification Strategyaccept-new (easiest to start)Root command prefixsudo (if Remote user is ec2-user)`

Click **Save** when done.

---

## Step 7 — Use the Label in Your Pipeline

### Declarative Pipeline

```groovy
pipeline {
    agent {
        label 'ec2-agent'   // Jenkins spins up EC2 when this job is queued
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-org/your-repo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }

    post {
        always {
            junit '**/target/surefire-reports/*.xml'
        }
    }
}
```

### Scripted Pipeline

```groovy
node('ec2-agent') {
    stage('Build') {
        checkout scm
        sh 'mvn clean install'
    }
}
```

---

## Home Network Considerations

### If your home IP is Static
- Simply add your static IP to the Security Group inbound rule
- No further changes needed

### If your home IP is Dynamic (most common)

**Option A — Dynamic DNS (Free)**
1. Sign up at https://www.duckdns.org
2. Install the DuckDNS client on your router or home PC
3. Your home IP updates automatically under `yourname.duckdns.org`
4. In the Security Group, use your DDNS hostname (or update the IP rule when it changes)

**Option B — Use Elastic IP on a Bastion**
1. Launch a tiny `t3.nano` EC2 in the same VPC with an **Elastic IP**
2. Jenkins SSHes to bastion first, then to agents via private IP
3. More complex but very stable

**Option C — AWS VPN (Advanced)**
- Set up AWS Site-to-Site VPN between home router and AWS VPC
- Agents get private IPs — most secure setup

---

## End-to-End Flow

```
1.  Developer pushes code to GitHub/GitLab

2.  Jenkins job is triggered (webhook or polling)

3.  Job has:  agent { label 'ec2-agent' }

4.  Jenkins EC2 Plugin → calls AWS RunInstances API
      └── Launches t3.medium in jenkins-agents-public-subnet
      └── Instance gets a public IP automatically

5.  Jenkins waits for SSH to become available (~60-90 seconds)

6.  Jenkins SSHes from home → EC2 public IP:22
      └── Copies agent.jar to /home/ubuntu/jenkins
      └── Starts the agent process

7.  Build runs on EC2
      └── Checkout, compile, test, package, deploy

8.  Build finishes

9.  Agent is idle for 30 minutes → Jenkins calls TerminateInstances API

10. Instance is gone. You pay ONLY for the build time. 💰
```

---

## Cost Optimization Tips

| Strategy | Saving | How |
|---|---|---|
| **Spot Instances** | 60–70% | Enable in AMI config, set max bid |
| **Right-size instance** | 20–40% | Use `t3.small` for lightweight jobs |
| **Idle termination** | Big | Set to 15–30 min, never leave running |
| **Mumbai region** | Moderate | Closer = lower latency from India |
| **Instance Cap** | Safety | Set max 5 to avoid runaway billing |

### Estimated Cost (ap-south-1, t3.medium)

```
On-Demand  :  ~$0.042 / hour
Spot       :  ~$0.013 / hour   (varies)

A 20-minute build = ~$0.004 on spot 🎉
```

---

## Troubleshooting

### Jenkins can't SSH to the agent

- Check Security Group inbound rule allows port 22 from **your current home IP**
- Verify subnet has **Auto-assign public IP** enabled
- Confirm the `.pem` key in Jenkins matches the EC2 key pair name
- Try SSHing manually: `ssh -i jenkins-agent-key.pem ubuntu@<ec2-public-ip>`

### Agent connects but build fails immediately

- Check the **Init Script** ran correctly (view system log in Jenkins)
- Verify Java is installed: `java -version` on the EC2 instance
- Ensure `/home/ubuntu/jenkins` directory is writable

### EC2 instances keep launching but never connect

- Jenkins controller needs **outbound internet access** to reach EC2
- Check your home router/firewall isn't blocking outbound SSH (port 22)
- Try increasing **"Connect Timeout"** in the AMI SSH settings to 120 seconds

### Instances not terminating

- Check **Idle termination time** is set (not 0)
- Look at Jenkins agent logs: **Manage Jenkins → Nodes → agent-name → Log**

### AWS API errors

- Verify IAM policy has all required `ec2:*` actions listed in Step 2
- Confirm Access Key / Secret Key are entered correctly in Jenkins credentials
- Check the correct **region** is selected (ap-south-1 for Mumbai)

---

## Quick Reference Checklist

```
AWS Setup
  [ ] VPC created with CIDR 10.0.0.0/16
  [ ] Public subnet created (10.0.1.0/24)
  [ ] Auto-assign public IP enabled on subnet
  [ ] Internet Gateway created and attached to VPC
  [ ] Route table has 0.0.0.0/0 → IGW route
  [ ] Route table associated with public subnet
  [ ] Security Group allows port 22 from home IP
  [ ] IAM user created with EC2 policy
  [ ] Access Key ID + Secret Key saved
  [ ] EC2 Key Pair created, .pem file downloaded

Jenkins Setup
  [ ] Amazon EC2 Plugin installed
  [ ] New Cloud created (type: Amazon EC2)
  [ ] AWS credentials added (Access Key + Secret)
  [ ] Region set to ap-south-1
  [ ] Private key (.pem contents) pasted
  [ ] AMI configured with correct AMI ID
  [ ] Label set to 'ec2-agent'
  [ ] Remote user set to 'ubuntu'
  [ ] Init script installs Java 17
  [ ] Idle termination time set (30 min)

Pipeline
  [ ] Job uses  agent { label 'ec2-agent' }
  [ ] Test run shows EC2 instance launching in AWS console
  [ ] Build completes successfully
  [ ] Instance terminates after idle timeout
```

---

*Guide written for Jenkins 2.541.2 | AWS Region: ap-south-1 (Mumbai) | Ubuntu 22.04 agents*
