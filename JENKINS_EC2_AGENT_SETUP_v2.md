# Jenkins EC2 Cloud Agent Setup Guide — v2 (Fixed)

> **Fixes applied in this version:**
> - 🔐 Credentials no longer visible in build logs
> - 🖥️ EC2 AMI Template fields — fully clarified, no garbled table
> - 🚀 Multiple ghost instances launching but not connecting — root cause + fix
> - ⌨️ `input` step no longer fails on EC2 agents

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Prerequisites](#prerequisites)
3. [Step 1 — AWS Infrastructure Setup](#step-1--aws-infrastructure-setup)
4. [Step 2 — IAM User & Permissions](#step-2--iam-user--permissions)
5. [Step 3 — EC2 Key Pair](#step-3--ec2-key-pair)
6. [Step 4 — Install Jenkins EC2 Plugin](#step-4--install-jenkins-ec2-plugin)
7. [Step 5 — Configure the Cloud in Jenkins](#step-5--configure-the-cloud-in-jenkins)
8. [Step 6 — Configure AMI / Agent Template (Fixed & Complete)](#step-6--configure-ami--agent-template-fixed--complete)
9. [Step 7 — Pipeline: Credentials Masking + input Fix](#step-7--pipeline-credentials-masking--input-fix)
10. [Fix — Multiple Ghost EC2 Instances](#fix--multiple-ghost-ec2-instances)
11. [Home Network Considerations](#home-network-considerations)
12. [Troubleshooting](#troubleshooting)
13. [Cost Optimization](#cost-optimization)
14. [Quick Reference Checklist](#quick-reference-checklist)

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
         │  3. Jenkins waits for SSH (up to 180s via init script)
         │  4. Jenkins SSHes to EC2 public IP
         │  5. Build runs on EC2
         │  6. Instance terminated after idle timeout
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

You need a **Public Subnet** so that your home Jenkins can reach EC2 agents via SSH.

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
VPC                :  jenkins-agents-vpc
Subnet name        :  jenkins-agents-public-subnet
Availability Zone  :  ap-south-1a
IPv4 CIDR          :  10.0.1.0/24
```

3. Click **Create Subnet**
4. Select the subnet → **Actions → Edit Subnet Settings**
5. Enable ✅ **Auto-assign public IPv4 address** → Save

> ⚠️ Critical — EC2 agents must get a public IP so your home Jenkins can SSH into them.

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
Target      :  jenkins-agents-igw
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
| SSH | TCP | 22 | `<YOUR_HOME_IP>/32` | Jenkins SSH access |

> 🔒 Find your IP at https://whatismyip.com — use `/32` not `0.0.0.0/0`

4. Outbound Rules — leave default (allow all outbound)
5. Click **Create Security Group**

---

## Step 2 — IAM User & Permissions

### 2.1 — Create IAM Policy

1. Go to **IAM → Policies → Create Policy → JSON tab** and paste:

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

2. Name it: `JenkinsEC2AgentPolicy` → **Create Policy**

---

### 2.2 — Create IAM User

1. **IAM → Users → Create User**
2. Username: `jenkins-ec2-agent-user`
3. Attach `JenkinsEC2AgentPolicy` directly
4. **Create User**

### 2.3 — Generate Access Keys

1. Click user → **Security Credentials tab → Create Access Key**
2. Select **Application running outside AWS**
3. Save both:
   - `Access Key ID`
   - `Secret Access Key`

> ⚠️ The secret key is shown only once — save it immediately.

---

## Step 3 — EC2 Key Pair

1. Go to **EC2 → Key Pairs → Create Key Pair**
2. Fill in:

```
Name              :  jenkins-agent-key
Key pair type     :  RSA
Private key format:  .pem
```

3. Click **Create** — `.pem` file downloads automatically
4. Keep this file — you will paste its contents into Jenkins

---

## Step 4 — Install Jenkins EC2 Plugin

1. **Jenkins → Manage Jenkins → Plugins → Available Plugins**
2. Search: `Amazon EC2`
3. Install **Amazon EC2** plugin
4. Restart Jenkins when prompted

---

## Step 5 — Configure the Cloud in Jenkins

1. **Manage Jenkins → Clouds → New Cloud**
2. Fill in:

```
Cloud name :  aws-ec2-agents
Type       :  Amazon EC2
```

3. Click **Create**

---

### 5.1 — Add AWS Credentials

> ⚠️ **Credential Visibility Fix** — Store credentials in Jenkins Credentials Store,
> NEVER paste raw keys into shell commands or `echo` them in pipelines.

1. On the cloud config page → **Amazon EC2 Credentials → Add → Jenkins**
2. Fill in:

```
Kind              :  AWS Credentials
ID                :  aws-jenkins-ec2-creds
Description       :  AWS credentials for EC2 agents
Access Key ID     :  <your IAM access key>
Secret Access Key :  <your IAM secret key>
```

3. Click **Add**, then select `aws-jenkins-ec2-creds` from the dropdown

---

### 5.2 — Set Region and Private Key

```
Region                    :  ap-south-1
EC2 Key Pair's Private Key:  (paste full .pem contents here)
```

```bash
# To get the key content:
cat jenkins-agent-key.pem
```

Paste the entire output including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`.

---

## Step 6 — Configure AMI / Agent Template (Fixed & Complete)

Click **Add AMI** on the cloud config page.

> ℹ️ The original doc had a corrupted/merged table at the end of this section.
> Below is the complete and correct field reference.

---

### 6.1 — Basic AMI Settings

| Field | Value |
|---|---|
| **AMI ID** | `ami-0f58b397bc5c1f2e8` *(Ubuntu 22.04, ap-south-1)* |
| **Instance Type** | `t3.medium` |
| **Security group names** | `jenkins-agent-sg` |
| **Subnet ID** | Paste your public subnet ID from AWS console |
| **AMI Type** | `unix` |

---

### 6.2 — Labels (Critical)

```
Labels  :  ec2-agent
```

> 🎯 Any pipeline with `agent { label 'ec2-agent' }` will trigger this template.
> Use multiple space-separated labels if needed: `ec2-agent linux build-agent`

---

### 6.3 — SSH Settings (Complete Field Reference)

| Field | Recommended Value | Why |
|---|---|---|
| **Remote user** | `ubuntu` | Default user for Ubuntu 22.04 AMI |
| **Remote FS root** | `/home/ubuntu/jenkins` | Jenkins workspace directory on agent |
| **SSH port** | `22` | Standard SSH port |
| **Java path** | `/usr/bin/java` | Where Java is installed by init script |
| **Host Key Verification Strategy** | `accept-new` | Accepts new host keys automatically — safest starting point |
| **Root command prefix** | *(leave blank for ubuntu)* | Use `sudo` only if remote user is non-root like `ec2-user` |

> 🔑 **Host Key Verification** is a common source of "unable to connect" errors.
> Set it to `accept-new` for a new setup. Do NOT use `Non verifying` in production.

---

### 6.4 — Connection Timeout (Ghost Instance Fix)

This is the **most important fix** for multiple instances launching but never connecting.

| Field | Recommended Value | Why |
|---|---|---|
| **Connect Timeout (seconds)** | `180` | Gives the init script enough time to complete before Jenkins retries |
| **Max number of retries** | `5` | Limits retry loops that spawn ghost instances |
| **Retry Wait Time (seconds)** | `30` | Gap between retries |

> ⚠️ **Default timeout is 30 seconds** — far too short when the init script installs Java.
> Jenkins declares the instance "unreachable", launches a new one, and repeats.
> This creates 5–10 ghost instances all running simultaneously.

---

### 6.5 — Lifecycle Settings

```
Instance Cap            :  3    ← hard limit on simultaneous agents
Idle termination time   :  30   ← minutes before auto-terminate
Min number of instances :  0    ← scale to zero when no jobs are running
```

> Set Instance Cap to a low number (3–5) as a safety guard against runaway launches.

---

### 6.6 — Init Script (Updated — Handles Slow Boot Gracefully)

```bash
#!/bin/bash
set -e

# Wait for apt lock (common cause of init script failure on fresh Ubuntu)
while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
  echo "Waiting for apt lock..."
  sleep 5
done

sudo apt-get update -y
sudo apt-get install -y \
  openjdk-17-jdk \
  git \
  maven \
  curl \
  unzip \
  docker.io

# Allow ubuntu user to use docker without sudo
sudo usermod -aG docker ubuntu

# Verify Java is available at expected path
java -version
echo "Java path: $(which java)"
```

> ✅ The `fuser` lock-wait loop prevents the init script from failing on fresh Ubuntu
> instances that have automatic updates running in the background.

---

### 6.7 — (Optional) Spot Instances

Check ✅ **Use Spot Instance** to save ~70% on cost:

```
Spot Max Bid Price  :  0.05   (USD/hr)
```

---

### Full AMI Configuration Summary

```
AMI ID                         :  ami-0f58b397bc5c1f2e8
Instance Type                  :  t3.medium
AMI Type                       :  unix
Labels                         :  ec2-agent
Remote user                    :  ubuntu
Remote FS root                 :  /home/ubuntu/jenkins
Java path                      :  /usr/bin/java
Security Group                 :  jenkins-agent-sg
Subnet ID                      :  <your-public-subnet-id>
SSH Port                       :  22
Host Key Verification Strategy :  accept-new
Connect Timeout (seconds)      :  180     ← critical fix
Max retries                    :  5
Retry wait (seconds)           :  30
Instance Cap                   :  3
Idle termination                :  30 min
Init Script                    :  (Java 17 + apt lock guard)
```

Click **Save** when done.

---

## Step 7 — Pipeline: Credentials Masking + input Fix

### 🔐 Fix 1 — Credentials Leaking Into Build Logs

**Why it happens:** If you use `sh "aws configure set aws_access_key_id $ACCESS_KEY"` or
`echo $SECRET` directly in shell steps, Jenkins prints the value before masking kicks in.

**Fix:** Always use `withCredentials` block — values are masked as `****` in all logs.

```groovy
pipeline {
    agent { label 'ec2-agent' }

    environment {
        // ✅ Bind credentials from Jenkins Credentials Store
        // DO NOT hardcode keys here or in sh steps
    }

    stages {
        stage('AWS Operation') {
            steps {
                // ✅ CORRECT — values masked in logs as ****
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-jenkins-ec2-creds',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                        aws s3 ls                          # uses masked env vars
                        aws sts get-caller-identity        # verify identity
                    '''
                }
            }
        }

        stage('Docker Login — Masked') {
            steps {
                // ✅ For username/password credentials
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }
    }
}
```

> ❌ **Never do this** — keys printed in plain text in logs:
> ```groovy
> sh "export AWS_ACCESS_KEY_ID=${env.MY_KEY}"   // BAD
> sh "echo ${params.SECRET_VALUE}"              // BAD
> ```

---

### ⌨️ Fix 2 — `input` Step Failing on EC2 Agents

**Why it happens:** The `input` step pauses and waits for a human. While waiting,
the EC2 agent's idle timeout (30 min) kicks in — Jenkins terminates the instance mid-build.
Also, `input` requires the Jenkins controller UI, not an agent executor.

**Fix:** Always run `input` on the **built-in node** (Jenkins controller), not on the EC2 agent.

```groovy
pipeline {
    // ✅ Default agent — jobs run on EC2
    agent { label 'ec2-agent' }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Approval Gate') {
            // ✅ Switch to built-in node for input — no EC2 agent needed here
            agent none
            steps {
                script {
                    // This runs on Jenkins controller — no agent timeout risk
                    def userInput = input(
                        message: 'Approve deployment to Production?',
                        ok: 'Deploy Now',
                        parameters: [
                            choice(
                                name: 'ENVIRONMENT',
                                choices: ['staging', 'production'],
                                description: 'Target environment'
                            ),
                            booleanParam(
                                name: 'RUN_SMOKE_TESTS',
                                defaultValue: true,
                                description: 'Run smoke tests after deploy?'
                            )
                        ],
                        submitter: 'admin,devops-team',  // restrict who can approve
                        submitterParameter: 'APPROVER'
                    )
                    echo "Approved by: ${userInput.APPROVER}"
                    echo "Deploying to: ${userInput.ENVIRONMENT}"
                }
            }
        }

        stage('Deploy') {
            // ✅ Back on EC2 agent for actual deployment work
            agent { label 'ec2-agent' }
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

> 📌 **Key rule:** Any stage with `input`, `timeout`, or long waits → use `agent none`.
> Any stage doing actual build/test/deploy work → use `agent { label 'ec2-agent' }`.

---

## Fix — Multiple Ghost EC2 Instances

### Root Cause

When Jenkins tries to SSH into a newly launched EC2 agent, it uses a default timeout
of **30 seconds**. The init script (installing Java + packages) takes **60–120 seconds**.
Jenkins thinks the agent is dead, marks it offline, and launches a **new instance**.
This repeats, resulting in 5–10 running EC2 instances that all fail to connect.

### Fix Checklist

```
In AMI Template → SSH settings:
  ✅ Connect Timeout     →  180 seconds   (was default 30)
  ✅ Max retries         →  5
  ✅ Retry wait          →  30 seconds

In AMI Template → Init Script:
  ✅ Add apt lock guard  →  while fuser /var/lib/dpkg/lock-frontend; do sleep 5; done
  ✅ Verify Java path    →  echo "$(which java)" at end of init script

In AMI Template → Lifecycle:
  ✅ Instance Cap        →  3 (prevents runaway launches)

In Security Group:
  ✅ Port 22 open from your CURRENT home IP
     (your IP may have changed since you created the SG)

In Subnet:
  ✅ Auto-assign public IPv4 is still enabled
```

### How to Kill Ghost Instances

If you already have stale EC2 instances running:

1. Go to **AWS Console → EC2 → Instances**
2. Filter by tag: `jenkins` or check launch time
3. Select all ghost instances → **Actions → Terminate Instance**
4. In Jenkins → **Manage Jenkins → Nodes** → delete any offline/stuck agent nodes

---

## Home Network Considerations

### Static Home IP
Add your IP to the Security Group inbound rule. No further steps needed.

### Dynamic Home IP (Most Common)

**Option A — DuckDNS (Free, Recommended)**
1. Sign up at https://www.duckdns.org
2. Install DuckDNS client on your router or PC
3. Your IP auto-updates under `yourname.duckdns.org`
4. Update the Security Group rule when IP changes

**Option B — Elastic IP Bastion**
1. Launch a `t3.nano` with Elastic IP in the same VPC
2. Jenkins SSHes to bastion → agents via private IP
3. More stable, small extra cost

**Option C — AWS Site-to-Site VPN**
- Most secure — agents use private IPs
- Complex setup, suits production environments

---

## Troubleshooting

### Jenkins can't SSH to the agent

- Verify Security Group port 22 is open from **your current home IP** (it may have changed)
- Confirm subnet still has **Auto-assign public IP** enabled
- Manually test: `ssh -i jenkins-agent-key.pem ubuntu@<ec2-public-ip>`
- Check **Connect Timeout** is 180s, not the default 30s

### Instances launch and immediately go offline

- **Most likely cause:** Java not yet installed when Jenkins tried to connect
- Fix: Set **Connect Timeout to 180 seconds** in AMI SSH settings
- Add the `apt lock guard` to your init script (see Section 6.6)

### Build logs show `****` instead of my actual secret — is that a bug?

- No — that is the **correct behavior**. `****` means masking is working properly.
- If you see the actual key value in logs → you are not using `withCredentials` correctly

### Credentials still visible in logs

- Make sure you are NOT doing `echo $MY_SECRET` or `sh "export KEY=${env.KEY}"`
- Check for `set -x` in shell steps — this prints all expanded values before masking
- Use `set +x` before any step that handles secrets:
  ```groovy
  sh '''
    set +x
    echo "$SECRET_VALUE" | some-command
  '''
  ```

### `input` step times out or fails

- Confirm the stage with `input` has `agent none` set
- The `input` step must run on the Jenkins controller, not the EC2 agent

### EC2 instances not terminating after build

- Check **Idle termination time** is set (not 0)
- Go to **Manage Jenkins → Nodes → agent-name → Log** to see termination events

### AWS API errors

- Verify IAM policy has all `ec2:*` actions from Step 2
- Confirm Access Key / Secret Key are entered correctly (no trailing spaces)
- Check the correct **region** is selected (`ap-south-1` for Mumbai)

---

## Cost Optimization

| Strategy | Saving | How |
|---|---|---|
| **Spot Instances** | 60–70% | Enable in AMI config |
| **Right-size instance** | 20–40% | Use `t3.small` for light jobs |
| **Idle termination** | Big | Set to 15–30 min |
| **Low Instance Cap** | Safety | Max 3 prevents runaway billing |
| **Mumbai region** | Moderate | Lower latency from Andhra Pradesh |

```
On-Demand  :  ~$0.042 / hour   (t3.medium, ap-south-1)
Spot       :  ~$0.013 / hour   (varies)

A 20-minute build ≈ $0.004 on spot 🎉
```

---

## Quick Reference Checklist

```
AWS Setup
  [ ] VPC created (10.0.0.0/16)
  [ ] Public subnet (10.0.1.0/24) with auto-assign public IP enabled
  [ ] Internet Gateway attached, route table updated
  [ ] Security Group: port 22 from your home IP /32
  [ ] IAM user with JenkinsEC2AgentPolicy
  [ ] Access Key + Secret Key saved
  [ ] EC2 Key Pair (.pem) downloaded

Jenkins Cloud Config
  [ ] Amazon EC2 Plugin installed
  [ ] New Cloud: type Amazon EC2
  [ ] AWS credentials added (ID: aws-jenkins-ec2-creds)
  [ ] Region: ap-south-1
  [ ] Private key (.pem contents) pasted

Jenkins AMI Template Config
  [ ] AMI ID correct for ap-south-1
  [ ] Label: ec2-agent
  [ ] Remote user: ubuntu
  [ ] Remote FS root: /home/ubuntu/jenkins
  [ ] Java path: /usr/bin/java
  [ ] AMI Type: unix
  [ ] Host Key Verification: accept-new
  [ ] Connect Timeout: 180 seconds  ← ghost instance fix
  [ ] Max retries: 5
  [ ] Instance Cap: 3
  [ ] Idle termination: 30 min
  [ ] Init script: includes apt lock guard + Java install

Pipeline
  [ ] Credentials accessed via withCredentials{} block only
  [ ] No raw keys in sh steps or environment{} block
  [ ] input step runs under agent none (not on EC2 agent)
  [ ] Test run: EC2 instance launches, build completes, instance terminates
```

---

*Guide v2 — Jenkins 2.541.2 | AWS ap-south-1 (Mumbai) | Ubuntu 22.04 agents*
*Fixes: credential masking, AMI template fields, ghost instances, input step*
kubectl create secret generic aws-creds \
  --from-literal=AWS_ACCESS_KEY_ID=xxx \
  --from-literal=AWS_SECRET_ACCESS_KEY=yyy \
  -n jenkins


  Here is every field from that form with the exact value to enter, grouped by section. Java 21 is used throughout.

---

## 🖥️ AMI Template — Complete Field-by-Field Guide

---

### 🔷 Basic AMI Identity

| Field | Value | Notes |
|---|---|---|
| **Description** | `Ubuntu 22.04 Jenkins Build Agent` | Just a label for your reference |
| **AMI ID** | `ami-0f58b397bc5c1f2e8` | Ubuntu 22.04 LTS, ap-south-1 |
| **Check AMI** | Click it after entering AMI ID | Verifies the AMI exists in your region |
| **AMI Owners** | `099720109477` | Canonical's AWS account ID — ensures you only get official Ubuntu AMIs |
| **AMI Users** | *(leave blank)* | Only needed if using a shared private AMI |
| **AMI Filters** | *(leave blank)* | Only needed if you want dynamic AMI lookup instead of a fixed ID |

---

### 🔷 Instance Configuration

| Field | Value | Notes |
|---|---|---|
| **Instance Type** | `t3.medium` | 2 vCPU, 4GB RAM — good for most CI builds |
| **EBS Optimized** | ☑ Check it | Better disk I/O, free on t3.medium |
| **Monitoring** | ☐ Leave unchecked | CloudWatch detailed monitoring costs extra |
| **T2 Unlimited** | ☐ Leave unchecked | Only relevant for t2 family — you're using t3 |
| **Availability Zone** | *(leave blank)* | Let AWS pick — plugin handles this |

---

### 🔷 Spot Configuration

| Field | Value | Notes |
|---|---|---|
| **Spot configuration** | ☑ Enable (recommended) | ~70% cost savings |
| Spot max bid price | `0.05` | Slightly above on-demand (~$0.042/hr) to avoid interruption |

> Leave blank to pay current spot price automatically — safer than a fixed bid.

---

### 🔷 Network & Security

| Field | Value | Notes |
|---|---|---|
| **Security group names** | `jenkins-agent-sg` | The SG you created earlier |
| **Subnet IDs for VPC** | `<your-public-subnet-id>` | Paste the subnet ID from AWS Console — e.g. `subnet-0abc1234` |
| **Associate IP Strategy** | `Public IP` | ✅ Critical — your home Jenkins needs a public IP to SSH in |
| **Tenancy** | `Default` | Leave as Default unless you have Dedicated Host licensing |

---

### 🔷 SSH & Agent Settings

| Field | Value | Notes |
|---|---|---|
| **Remote FS root** | `/home/ubuntu/jenkins` | Jenkins workspace on the agent |
| **Remote user** | `ubuntu` | Default user for Ubuntu 22.04 AMI |
| **AMI Type** | `unix` | Select `unix` for Ubuntu/Linux |
| **Root command prefix** | *(leave blank)* | Leave blank — `ubuntu` user already has sudo via init script |
| **Agent command prefix** | *(leave blank)* | Not needed |
| **Agent command suffix** | *(leave blank)* | Not needed |
| **Remote ssh port** | `22` | Standard SSH |
| **Boot Delay** | `60` | Seconds to wait after launch before Jenkins tries SSH — gives init script a head start |
| **Connection Strategy** | `Public IP` | Your home network reaches EC2 via public IP |
| **Connect by SSH Process** | ☐ Leave unchecked | Default Java SSH is fine |
| **Host Key Verification Strategy** | `accept-new` | Accepts new host keys automatically — correct for dynamic EC2 agents |
| **Launch Timeout in seconds** | `300` | ✅ 5 minutes — gives init script time to install Java 21 before Jenkins gives up |

---

### 🔷 Labels & Usage

| Field | Value | Notes |
|---|---|---|
| **Labels** | `ec2-agent` | Used in `agent { label 'ec2-agent' }` in pipelines |
| **Usage** | `Only build jobs with label expressions matching this node` | Prevents random jobs from stealing EC2 agents |

---

### 🔷 Lifecycle & Capacity

| Field | Value | Notes |
|---|---|---|
| **Idle termination time** | `30` | Minutes before auto-terminate |
| **Terminate idle agents during shutdown** | ☑ Check it | Cleans up agents when Jenkins shuts down |
| **Number of Executors** | `1` | 1 build per agent — cleaner, simpler |
| **Minimum number of instances** | `0` | Scale to zero when idle — saves money |
| **Minimum number of spare instances** | `0` | No pre-warming needed |
| **Only apply minimum during time range** | ☐ Leave unchecked | Only relevant if you pre-warm during business hours |
| **Instance Cap** | `3` | Hard limit — prevents runaway instance launches |
| **Maximum Total Uses** | `0` | 0 = unlimited reuse. Set to `1` if you want fresh agents every build |
| **Avoid Using Orphaned Nodes** | ☑ Check it | Skips agents that lost connection instead of reusing them |
| **Stop/Disconnect on Idle Timeout** | ☐ Leave unchecked | Terminate fully rather than stop — cleaner |

---

### 🔷 IAM & Storage

| Field | Value | Notes |
|---|---|---|
| **IAM Instance Profile** | *(leave blank)* | Only needed if agents need to call AWS APIs themselves |
| **Delete root device on instance termination** | ☑ Check it | ✅ Prevents orphaned EBS volumes — saves cost |
| **Use ephemeral devices** | ☐ Leave unchecked | Not available on t3 family |
| **Encrypt EBS root volume** | `Based on AMI` | Fine for CI agents |
| **Block device mapping** | *(leave blank)* | Default 8GB root volume is sufficient |

---

### 🔷 Java & JVM

| Field | Value | Notes |
|---|---|---|
| **Java Path** | `/usr/bin/java` | Where Java 21 is installed by the init script |
| **JVM Options** | `-Xmx512m -Xms256m` | Limits agent JVM memory — fine for t3.medium |

---

### 🔷 Instance Metadata (IMDSv2)

| Field | Value | Notes |
|---|---|---|
| **Instance Metadata Supported** | ☑ Yes | Enable metadata access |
| **Enable Metadata HTTP Endpoint** | ☑ Yes | Required for metadata to work |
| **Metadata Require HTTP Tokens** | ☑ Yes (IMDSv2) | ✅ Recommended — IMDSv2 is more secure |
| **Metadata Put Response Hop Limit** | `1` | Default — fine for EC2 agents |
| **Enclave Enabled** | ☐ No | Only for Nitro Enclave workloads |

---

### 🔷 Misc

| Field | Value | Notes |
|---|---|---|
| **Override temporary dir location** | *(leave blank)* | Default `/tmp` is fine |
| **User Data** | *(leave blank)* | You're using Init Script instead — don't use both |
| **Tags → EC2 Tag/Value Pairs** | `Key: Name` / `Value: jenkins-ec2-agent` | Helps identify agents in AWS Console |

---

### 🔷 Init Script — Java 21

```bash
#!/bin/bash
set -e

# Wait for apt lock (common on fresh Ubuntu — background updates run at boot)
while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
  echo "Waiting for apt lock to release..."
  sleep 5
done

sudo apt-get update -y

# Install Java 21 (required — Jenkins agent needs Java 21+)
sudo apt-get install -y openjdk-21-jdk

# Install build tools
sudo apt-get install -y \
  git \
  maven \
  curl \
  unzip \
  docker.io

# Allow ubuntu user to run docker without sudo
sudo usermod -aG docker ubuntu

# Confirm Java version — must show 21
java -version
echo "Java installed at: $(which java)"
```

> ✅ Set **Java Path** to `/usr/bin/java` — the init script installs `openjdk-21-jdk` which places the binary there.

---

### Fields You Can Safely Leave Blank

`AMI Filters`, `AMI Users`, `Root command prefix`, `Agent command prefix`, `Agent command suffix`, `Override temporary dir location`, `User Data`, `Block device mapping`, `IAM Instance Profile`, `T2 Unlimited`, `Availability Zone`, `Disk Space Monitoring Thresholds`, `Tool Locations`, `Environment variables` — none of these are needed for a basic EC2 agent setup.