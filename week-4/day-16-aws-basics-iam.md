# Day 16 — AWS Basics + IAM

## Learning Objectives

By the end of this day you will:
- Understand the three cloud service models and where AWS fits
- Know how AWS organises its global infrastructure and why it matters
- Create IAM users, groups, and policies from both the Console and the CLI
- Configure the AWS CLI and understand where credentials live
- Explain IAM Roles and why EC2 instances should use them instead of keys
- Know the five most common IAM mistakes and how to avoid them

---

## 1. Cloud Computing Models

Cloud computing sells computing resources as a service over the internet. There are three models. Understanding which layer you manage determines your operational burden.

### IaaS — Infrastructure as a Service

You rent raw infrastructure: virtual machines, storage, networking. You manage the operating system and everything above it.

**AWS examples:** EC2, EBS, VPC, S3

**You manage:** OS patching, runtime, middleware, applications, data
**AWS manages:** Physical hardware, hypervisor, data center facilities

### PaaS — Platform as a Service

You rent a managed platform. AWS runs the infrastructure and the runtime. You deploy your application code.

**AWS examples:** Elastic Beanstalk, RDS, Lambda (partially), Fargate

**You manage:** Application code, data
**AWS manages:** OS, runtime, scaling, patching

### SaaS — Software as a Service

You use fully managed software. No infrastructure or platform concerns.

**AWS examples:** Amazon WorkMail, Amazon Chime, AWS Config

**You manage:** Configuration and data
**AWS manages:** Everything else

### Why This Matters

When you run a database on EC2 (IaaS), you own patching, backups, failover, and performance tuning. When you use RDS (PaaS), AWS handles all of that. IaaS gives more control; PaaS trades control for operational simplicity. Choosing the right model for each component is a core architectural skill.

---

## 2. AWS Global Infrastructure

### Regions

A Region is a geographic area containing multiple isolated data centres. As of 2025, AWS has 33+ regions worldwide (us-east-1, eu-west-1, ap-southeast-1, etc.).

**Why regions matter:**
- **Latency:** Deploy close to your users
- **Compliance:** Data residency requirements (GDPR requires EU data to stay in EU)
- **Availability:** Isolate failures to a geographic area
- **Pricing:** Prices vary by region (us-east-1 is typically cheapest)

```bash
# List all available regions
aws ec2 describe-regions --output table

# Set a default region in your CLI config
aws configure set region us-east-1
```

### Availability Zones (AZs)

Each Region contains 2–6 Availability Zones. An AZ is one or more discrete data centres with redundant power, networking, and connectivity. AZs within a Region are connected by low-latency private fibre links but are physically separated (different flood plains, power grids).

**Why AZs matter:**

If you run everything in a single AZ and that data centre loses power, your application goes down. If you spread across two AZs and one fails, the other keeps serving traffic. This is the basis of High Availability (HA) architecture.

```
Region: us-east-1 (Northern Virginia)
├── us-east-1a  (AZ)
├── us-east-1b  (AZ)
├── us-east-1c  (AZ)
├── us-east-1d  (AZ)
├── us-east-1e  (AZ)
└── us-east-1f  (AZ)
```

### Edge Locations

Edge Locations are endpoints for AWS content delivery. They sit closer to end users than Regions. CloudFront (AWS CDN) uses over 400 edge locations to cache and serve static content with low latency. Route 53 also uses edge locations for DNS resolution.

---

## 3. AWS Free Tier

### What is Free

| Service | Free Tier Offer | Duration |
|---------|----------------|----------|
| EC2 | 750 hours/month of t2.micro or t3.micro | 12 months |
| S3 | 5 GB storage, 20,000 GET requests | 12 months |
| RDS | 750 hours/month of db.t2.micro or db.t3.micro, 20 GB storage | 12 months |
| Lambda | 1 million requests/month | Always free |
| CloudWatch | 10 custom metrics, 10 alarms | Always free |
| DynamoDB | 25 GB storage, 25 read/write capacity units | Always free |

### What to Watch Out For

- **NAT Gateway:** Not free. Costs $0.045/hour plus $0.045/GB of data processed. Destroy it when not in use.
- **Elastic IP not attached to a running instance:** Charged $0.005/hour.
- **Data transfer out:** First 100 GB/month free, then $0.09/GB.
- **Route 53 hosted zones:** $0.50/month per zone. Not covered by Free Tier.
- **Running instances past 750 hours:** If you run two t3.micros simultaneously, you hit 750 hours in 15 days.

**Set a billing alarm:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "billing-alert-10usd" \
  --alarm-description "Alert when estimated charges exceed $10" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:billing-alerts
```

---

## 4. IAM — Identity and Access Management

IAM controls who (identity) can do what (access) on which AWS resources. It is global — IAM is not region-specific.

### The Four Core IAM Objects

#### Users

An IAM User represents a person or application that needs long-term credentials to interact with AWS. Users get:
- A username and password (for Console access)
- Access key ID + secret access key (for programmatic access via CLI/SDK)

#### Groups

A Group is a collection of users. You attach policies to groups, not directly to individual users (when possible). When a user joins the group, they inherit all the group's permissions.

```
Group: developers
  Policy: ReadOnlyAccess to S3
  Policy: EC2FullAccess
  ├── user: alice
  ├── user: bob
  └── user: carol
```

#### Policies

A Policy is a JSON document that defines permissions. Policies are attached to users, groups, or roles.

**Two types:**
- **AWS Managed Policies:** Maintained by AWS (e.g., `AdministratorAccess`, `ReadOnlyAccess`, `AmazonS3FullAccess`)
- **Customer Managed Policies:** Written by you for custom permission sets

#### Roles

A Role is an IAM identity that can be assumed by AWS services, other AWS accounts, or federated users. Roles do not have long-term credentials — they issue temporary security tokens. EC2 instances, Lambda functions, and ECS tasks all use roles to access other AWS services.

---

## 5. Root Account vs IAM User

When you create an AWS account, you get a root account tied to your email address. The root account has unrestricted access to everything, including billing, account closure, and changing support plans.

**Never use the root account for daily work.** If root credentials are compromised, the attacker has complete control of your account.

### What to Do With Root Immediately

1. Enable MFA on the root account
2. Create an IAM user with `AdministratorAccess` for day-to-day work
3. Generate root access keys only if absolutely required — delete them after
4. Lock the root credentials in a password manager and do not use them again

```bash
# Check if root access keys exist (must be done in Console or via IAM credential report)
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d | column -t -s ','
# Look at the "access_key_1_active" and "access_key_2_active" columns for the root row
```

---

## 6. Principle of Least Privilege

Grant only the permissions required to perform a specific task — nothing more.

**Bad:** Giving a Lambda function that reads from one S3 bucket `AdministratorAccess`
**Good:** Giving it `s3:GetObject` on `arn:aws:s3:::my-specific-bucket/*` only

This limits blast radius when credentials are compromised or code has a bug.

---

## 7. IAM Policy Structure

Every policy is a JSON document. The key fields:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-assets",
        "arn:aws:s3:::my-app-assets/*"
      ]
    },
    {
      "Sid": "DenyDeleteOnProdBucket",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-app-assets-prod/*"
    }
  ]
}
```

### Field Breakdown

| Field | Purpose | Values |
|-------|---------|--------|
| `Version` | Policy language version | Always `"2012-10-17"` |
| `Sid` | Statement ID — optional human-readable label | Any string |
| `Effect` | Allow or deny the listed actions | `"Allow"` or `"Deny"` |
| `Action` | Which API calls are covered | e.g., `"s3:GetObject"`, `"ec2:*"`, `"*"` |
| `Resource` | Which AWS resources the actions apply to | ARN or `"*"` |
| `Condition` | Optional conditions (IP range, MFA, time) | Key-value conditions |

**Deny always overrides Allow.** Even if a user has `Allow` from one policy and `Deny` from another, the result is Deny.

### ARN Format

```
arn:partition:service:region:account-id:resource-type/resource-id

arn:aws:s3:::my-bucket                  # S3 bucket (no region or account in S3 ARNs)
arn:aws:ec2:us-east-1:123456789012:instance/i-0abcd1234efgh5678
arn:aws:iam::123456789012:user/alice    # IAM (no region)
```

---

## 8. Creating IAM Users and Groups

### Via AWS Console

1. Go to IAM → Users → Create user
2. Enter username, enable Console access if needed
3. Add to a group or attach policies directly
4. Download credentials CSV (you only see the secret access key once)

### Via AWS CLI

```bash
# Create a group
aws iam create-group --group-name developers

# Attach a managed policy to the group
aws iam attach-group-policy \
  --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Create a user
aws iam create-user --user-name alice

# Add user to group
aws iam add-user-to-group --user-name alice --group-name developers

# Create Console login profile (password)
aws iam create-login-profile \
  --user-name alice \
  --password "Temp1234!" \
  --password-reset-required

# Create programmatic access keys
aws iam create-access-key --user-name alice
# This returns AccessKeyId and SecretAccessKey — save them immediately

# List users
aws iam list-users --output table

# List groups a user belongs to
aws iam list-groups-for-user --user-name alice

# Get current caller identity (who am I?)
aws sts get-caller-identity
```

---

## 9. AWS CLI — Install, Configure, Credential Location

### Install

```bash
# macOS (Homebrew)
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# aws-cli/2.15.0 Python/3.11.6 ...
```

### Configure

```bash
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json
```

### Where Credentials Live

```bash
~/.aws/credentials    # Access keys
~/.aws/config         # Region, output format, profiles
```

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[production]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json

[profile production]
region = eu-west-1
output = table
```

### Using Named Profiles

```bash
# Use a specific profile for a command
aws s3 ls --profile production

# Set a default profile for the session
export AWS_PROFILE=production

# Override with environment variables (highest priority)
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
```

**Credential resolution order (CLI checks these in order):**
1. Command line options (`--profile`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, etc.)
3. AWS config file (`~/.aws/config`)
4. Credentials file (`~/.aws/credentials`)
5. Container credentials (ECS task role)
6. EC2 instance profile (IAM Role attached to EC2)

---

## 10. MFA — Multi-Factor Authentication

MFA adds a second factor beyond username/password. Even if credentials are stolen, the attacker cannot authenticate without the physical device.

**Enable MFA on:**
- Root account (mandatory, do it first)
- All IAM users with Console access
- Any user with admin permissions

### Enforce MFA via Policy

Attach this policy to your IAM users or group to deny all actions unless MFA is active:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptMFARelatedIfNoMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

---

## 11. IAM Roles — The Right Way for EC2 to Access S3

### The Wrong Way (Never Do This)

```bash
# Bad: Hardcoded credentials inside an EC2 instance
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/...

# Or worse, in your application code:
s3 = boto3.client(
    's3',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',
    aws_secret_access_key='wJalrXUtnFEMI/...'
)
```

If this EC2 instance is compromised, the attacker has your long-term credentials. If the code goes to GitHub by accident, same problem.

### The Right Way — IAM Roles

1. Create an IAM Role with a trust policy allowing EC2 to assume it
2. Attach a permissions policy (e.g., `AmazonS3ReadOnlyAccess`)
3. Attach the role to the EC2 instance (Instance Profile)
4. Your application code calls boto3 with no credentials — AWS SDK automatically retrieves temporary credentials from the EC2 metadata service

```bash
# Create a role for EC2
aws iam create-role \
  --role-name ec2-s3-read-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach permissions policy to the role
aws iam attach-role-policy \
  --role-name ec2-s3-read-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create an instance profile (wrapper for the role so EC2 can use it)
aws iam create-instance-profile --instance-profile-name ec2-s3-read-profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-s3-read-profile \
  --role-name ec2-s3-read-role

# Attach to a running EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abcd1234efgh5678 \
  --iam-instance-profile Name=ec2-s3-read-profile
```

**Application code (Python) — no credentials needed:**

```python
import boto3

# Credentials are automatically fetched from the instance metadata service
s3 = boto3.client('s3')
response = s3.list_buckets()
print(response['Buckets'])
```

The SDK fetches temporary credentials from `http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-read-role`. These expire and rotate automatically.

---

## 12. Common IAM Mistakes

### Mistake 1: Wildcard Actions

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

This is `AdministratorAccess`. Never give this to application roles or service accounts.

### Mistake 2: Not Enabling MFA

If a user's password or access key is leaked, MFA is the only thing preventing full account access. Enable it everywhere.

### Mistake 3: Using the Root Account

The root account has no permission boundaries. If root credentials are stolen, the attacker can lock you out of your own account. Use root only for initial setup and rare account-level tasks.

### Mistake 4: Sharing Access Keys

Access keys are individual credentials. Never share them between team members. Each person gets their own IAM user and their own keys. This allows you to revoke one person's access without affecting others.

### Mistake 5: Long-Lived Keys on EC2

Putting access keys on an EC2 instance means if the instance is compromised, so are the keys. Use IAM roles (instance profiles) instead. If the instance is terminated, the role disappears with it.

---

## Exercises

### Exercise 1 — Create an IAM Group and User via Console

1. Log in to the AWS Console as an IAM admin user (not root).
2. Create a group called `devops-learners`.
3. Attach the `AmazonEC2ReadOnlyAccess` and `AmazonS3ReadOnlyAccess` managed policies to the group.
4. Create a user called `student-01` with Console access.
5. Add `student-01` to `devops-learners`.
6. Log out and log in as `student-01`. Verify you can list EC2 instances but cannot launch them.

### Exercise 2 — Replicate Exercise 1 via AWS CLI

```bash
# Create the group
aws iam create-group --group-name devops-learners-cli

# Attach policies
aws iam attach-group-policy \
  --group-name devops-learners-cli \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

aws iam attach-group-policy \
  --group-name devops-learners-cli \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create user and add to group
aws iam create-user --user-name student-cli-01
aws iam add-user-to-group --user-name student-cli-01 --group-name devops-learners-cli

# Create access keys for programmatic access
aws iam create-access-key --user-name student-cli-01

# Verify group membership
aws iam list-groups-for-user --user-name student-cli-01
aws iam list-attached-group-policies --group-name devops-learners-cli
```

### Exercise 3 — Write a Custom IAM Policy

Write a policy JSON file that:
- Allows `s3:GetObject` and `s3:ListBucket` on a bucket named `my-devops-practice-YOURNAME`
- Allows `ec2:DescribeInstances` on all resources
- Denies `s3:DeleteObject` on all S3 resources

```bash
# Create the policy file
cat > my-custom-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-devops-practice-YOURNAME",
        "arn:aws:s3:::my-devops-practice-YOURNAME/*"
      ]
    },
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    },
    {
      "Sid": "DenyS3Delete",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
EOF

# Create the policy in AWS
aws iam create-policy \
  --policy-name my-s3-ec2-policy \
  --policy-document file://my-custom-policy.json
```

### Exercise 4 — Create an EC2 IAM Role

Create an IAM role that allows an EC2 instance to write objects to a specific S3 bucket. Do not use a managed policy — write the policy JSON yourself.

```bash
# Trust policy (who can assume the role)
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Permissions policy (what the role can do)
cat > s3-write-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:GetObject"],
    "Resource": "arn:aws:s3:::my-devops-practice-YOURNAME/*"
  }]
}
EOF

aws iam create-role \
  --role-name ec2-s3-writer \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam put-role-policy \
  --role-name ec2-s3-writer \
  --policy-name s3-write-inline \
  --policy-document file://s3-write-policy.json

aws iam create-instance-profile --instance-profile-name ec2-s3-writer-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-s3-writer-profile \
  --role-name ec2-s3-writer
```

### Exercise 5 — Generate and Review a Credential Report

```bash
# Request the report (takes a few seconds)
aws iam generate-credential-report

# Download and view it
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d > credential-report.csv

# Open it
column -t -s ',' credential-report.csv | less -S
```

Answer these questions from the report:
- Does the root account have MFA enabled?
- Does the root account have active access keys?
- Which users have not used their access keys in the last 90 days?
- Which users have passwords but no MFA?
