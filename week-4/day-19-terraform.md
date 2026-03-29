# Day 19 — Terraform: Infrastructure as Code

## Learning Objectives

By the end of this day you will:
- Explain what IaC is and why it replaces manual console clicking
- Understand Terraform's architecture: providers, resources, state, and the plan/apply cycle
- Write HCL code with variables, outputs, locals, and data sources
- Configure a remote S3 backend with DynamoDB locking
- Deploy a real EC2 instance with a security group and key pair using Terraform
- Read and interpret `terraform plan` output
- Recover from common errors like state drift

---

## 1. What is Infrastructure as Code?

IaC means describing your infrastructure in code files that live in version control, rather than clicking through a web console or running ad-hoc CLI commands.

### The Problem With Clicking

Imagine you spend two hours clicking through the AWS Console to set up a VPC, subnets, security groups, ALB, two EC2 instances, and an RDS database. Now:

- A colleague needs to set up an identical staging environment — they repeat your two hours of clicking, with slightly different settings and different bugs
- Six months later, someone changes a security group rule manually "just to test" and never reverts it. Your actual infrastructure now differs from what you think it is. This is **drift**.
- The company needs to spin up the same stack in another region — another two hours of clicking
- The instance gets terminated by mistake — you have no record of its exact configuration

### IaC Fixes This

```
# Every resource is a code file
# Changes go through git: reviewed, approved, audited
# Anyone can reproduce the environment: terraform apply
# Drift is detected: terraform plan shows what changed outside Terraform
# Documentation is the code itself
```

---

## 2. Terraform vs CloudFormation vs Pulumi

| Tool | Language | Cloud | State | Best For |
|------|----------|-------|-------|---------|
| **Terraform** | HCL (HashiCorp Config Language) | Multi-cloud (AWS, GCP, Azure, GitHub, Datadog...) | External (S3, Terraform Cloud) | Industry standard, large ecosystem |
| **CloudFormation** | JSON or YAML | AWS only | Managed by AWS (in Stacks) | AWS-native shops, no external tools |
| **Pulumi** | TypeScript, Python, Go, C# | Multi-cloud | Pulumi Cloud or self-managed | Teams that prefer real programming languages over DSL |

**Why Terraform is the most common in the industry:**
- Provider ecosystem: 3,000+ providers covering nearly every cloud and SaaS tool
- Community: massive, well-documented
- HCL is readable even by non-developers
- Works across clouds — your Terraform skills transfer between jobs

---

## 3. Terraform Architecture

```mermaid
flowchart TD
    HCL[HCL Configuration Files] --> CORE[Terraform Core\nReads config and state\nBuilds dependency graph\nCalculates diff - plan\nExecutes changes - apply]
    CORE -->|calls| PROV[Provider Plugin\ne.g. AWS Provider\nTranslates HCL to API calls]
    PROV -->|API calls| AWS[AWS APIs\nEC2 - S3 - IAM - VPC - RDS - ALB]
```

### Plugin-Based Architecture

Terraform Core is thin. It does not know anything about AWS or any other cloud. All cloud-specific logic lives in **providers**, which are separate binary plugins. When you run `terraform init`, Terraform downloads the specified providers from the Terraform Registry.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"    # Downloads from registry.terraform.io/hashicorp/aws
      version = "~> 5.0"           # Use any 5.x version, not 6.x
    }
  }
}
```

---

## 4. Core Workflow

```mermaid
flowchart TD
    INIT[terraform init\nDownload providers\nInitialise backend] --> PLAN
    PLAN[terraform plan\nCompare desired vs current state\nShow changes - does NOT apply anything] --> APPLY
    APPLY[terraform apply\nExecute the changes\nUpdate state file] --> DESTROY
    DESTROY[terraform destroy\nDestroy all managed resources\nUpdate state file]
```

```bash
# Initialise a new working directory
terraform init

# Preview changes (dry run)
terraform plan

# Apply with auto-approval (skip interactive prompt — use in CI/CD only)
terraform apply -auto-approve

# Apply a specific plan file (ensures you apply exactly what you reviewed)
terraform plan -out=tfplan
terraform apply tfplan

# Destroy everything
terraform destroy

# Destroy with auto-approval
terraform destroy -auto-approve

# Show current state
terraform show

# List resources in state
terraform state list

# Show a specific resource in state
terraform state show aws_instance.web_server
```

---

## 5. HCL Syntax

### Resource Block

A resource block creates or manages one piece of infrastructure.

```hcl
resource "aws_instance" "web_server" {
  # resource type       # local name (used to reference this resource elsewhere)
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "production-web"
    Environment = "production"
  }
}
```

`aws_instance` is the resource type — it maps to the EC2 API. `web_server` is the local name you give this resource in Terraform — it is only meaningful within your Terraform code.

### Variable Block

Variables are inputs to your Terraform configuration.

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, production)"
  # No default — must be provided
}

variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "List of CIDR blocks allowed to access the ALB"
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  type = map(string)
  description = "Common tags for all resources"
  default = {
    Project = "devops-practice"
    ManagedBy = "terraform"
  }
}
```

**Reference a variable:**
```hcl
resource "aws_instance" "web_server" {
  instance_type = var.instance_type
}
```

### Output Block

Outputs print values after `terraform apply` and make them available to other Terraform configurations.

```hcl
output "web_server_public_ip" {
  description = "Public IP address of the web server"
  value       = aws_instance.web_server.public_ip
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "rds_endpoint" {
  description = "RDS endpoint (used in app config)"
  value       = aws_db_instance.main.endpoint
  sensitive   = true    # Value is hidden in plan/apply output, still accessible
}
```

```bash
# View outputs after apply
terraform output
terraform output web_server_public_ip
terraform output -json    # All outputs as JSON
```

### Locals Block

Locals are computed values within a configuration — they reduce repetition and improve readability.

```hcl
locals {
  name_prefix = "${var.environment}-${var.project}"
  common_tags = merge(var.tags, {
    Environment = var.environment
    CreatedAt   = "2025-01-01"
  })
}

resource "aws_instance" "web_server" {
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-server"
  })
}
```

### Data Source Block

Data sources query existing infrastructure — things not managed by your Terraform code.

```hcl
# Look up the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}

# Reference it in a resource
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
}
```

```hcl
# Look up an existing VPC by tag
data "aws_vpc" "existing" {
  tags = {
    Name = "production-vpc"
  }
}

# Use its ID
resource "aws_subnet" "new_subnet" {
  vpc_id     = data.aws_vpc.existing.id
  cidr_block = "10.0.20.0/24"
}
```

---

## 6. Variable Files (.tfvars)

You pass variable values via `.tfvars` files. Never commit files containing secrets to git.

```hcl
# terraform.tfvars (gitignored)
environment   = "production"
instance_type = "t3.small"
aws_region    = "us-east-1"
key_name      = "my-ec2-key"
```

```bash
# Terraform automatically loads terraform.tfvars
terraform plan

# Or specify a file explicitly
terraform plan -var-file="production.tfvars"

# Or pass individual variables
terraform plan -var="environment=staging"

# Environment variables: TF_VAR_ prefix overrides everything
export TF_VAR_environment=production
terraform plan
```

---

## 7. State File

The state file (`terraform.tfstate`) is Terraform's source of truth about what it manages. It maps your HCL resource blocks to real infrastructure IDs.

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web_server",
      "instances": [
        {
          "attributes": {
            "id": "i-0abcdef1234567890",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t3.micro",
            "public_ip": "54.210.123.45",
            ...
          }
        }
      ]
    }
  ]
}
```

### Rules About State

1. **Never edit the state file manually.** Use `terraform state` commands if you must manipulate state.
2. **Never commit the state file to git.** It may contain secrets (database passwords, private key material). Add `*.tfstate` and `*.tfstate.backup` to `.gitignore`.
3. **Use remote state** for any team or production environment.

```bash
# If you need to remove a resource from state without destroying it:
terraform state rm aws_instance.web_server

# If you need to import an existing resource into state:
terraform import aws_instance.web_server i-0abcdef1234567890

# If state gets corrupted, roll back to a previous version from S3 versioning
```

---

## 8. Remote State — S3 Backend with DynamoDB Locking

Local state breaks in teams. Remote state in S3 means everyone shares the same state file. DynamoDB locking prevents two `terraform apply` runs from conflicting.

```hcl
# provider.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state-yourinitials-2025"
    key            = "production/week4-app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = "devops-zero-to-hero"
      Environment = var.environment
    }
  }
}
```

**Why separate `provider.tf` from `main.tf`:**
- `provider.tf` — provider configuration, backend, required versions. Rarely changes.
- `main.tf` — actual resources. Changes frequently.
- This split makes code review easier and reduces merge conflicts.

---

## 9. Authentication — Never Hardcode Credentials

```hcl
# WRONG — Never do this
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"         # Hardcoded — terrible
  secret_key = "wJalrXUtnFEMI/K7MDENG/..."   # Hardcoded — terrible
}
```

```hcl
# CORRECT — Use environment variables or instance profile
provider "aws" {
  region = var.aws_region
  # No credentials here. Terraform uses the same credential chain as the AWS CLI:
  # 1. TF_VAR or AWS_ environment variables
  # 2. ~/.aws/credentials file
  # 3. EC2 instance profile (when running in CI/CD on EC2)
}
```

```bash
# For local development — credentials from ~/.aws/credentials
export AWS_PROFILE=my-profile
terraform plan

# For CI/CD — credentials from environment variables
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/...
export AWS_DEFAULT_REGION=us-east-1
terraform apply -auto-approve
```

---

## 10. Real Example — EC2 + Security Group + Key Pair

### File Structure

```
week4-ec2/
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

### provider.tf

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state-yourinitials-2025"
    key            = "practice/ec2-demo/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}
```

### variables.tf

```hcl
variable "aws_region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "us-east-1"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "my_ip" {
  type        = string
  description = "Your public IP in CIDR notation (e.g., 203.0.113.5/32)"
}
```

### main.tf

```hcl
# Fetch the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}

# Use the default VPC for simplicity in this example
data "aws_vpc" "default" {
  default = true
}

# Fetch public subnets in the default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Generate an RSA key pair using the TLS provider
resource "tls_private_key" "web_server_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Upload the public key to AWS
resource "aws_key_pair" "web_server" {
  key_name   = "${var.environment}-web-server-key"
  public_key = tls_private_key.web_server_key.public_key_openssh
}

# Save the private key locally
resource "local_sensitive_file" "private_key" {
  content         = tls_private_key.web_server_key.private_key_pem
  filename        = "${path.module}/${var.environment}-web-server-key.pem"
  file_permission = "0400"
}

# Security group — allow SSH from your IP, HTTP from everywhere
resource "aws_security_group" "web_server" {
  name        = "${var.environment}-web-server-sg"
  description = "Security group for the web server"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]
  }

  ingress {
    description = "App port from internet"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-web-server-sg"
    Environment = var.environment
  }
}

# User data script to install and start a Python web server
locals {
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y python3 python3-pip
    pip3 install flask gunicorn

    mkdir -p /opt/myapp
    cat > /opt/myapp/app.py << 'PYEOF'
    from flask import Flask
    import socket

    app = Flask(__name__)

    @app.route('/')
    def index():
        return f"Hello from {socket.gethostname()}\n"

    @app.route('/health')
    def health():
        return "OK", 200
    PYEOF

    cat > /etc/systemd/system/myapp.service << 'SVCEOF'
    [Unit]
    Description=My Python App
    After=network.target

    [Service]
    User=ec2-user
    WorkingDirectory=/opt/myapp
    ExecStart=/usr/local/bin/gunicorn -w 2 -b 0.0.0.0:8080 app:app
    Restart=always
    RestartSec=3

    [Install]
    WantedBy=multi-user.target
    SVCEOF

    systemctl daemon-reload
    systemctl enable myapp
    systemctl start myapp
  EOF
}

# EC2 Instance
resource "aws_instance" "web_server" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  key_name                    = aws_key_pair.web_server.key_name
  vpc_security_group_ids      = [aws_security_group.web_server.id]
  subnet_id                   = data.aws_subnets.default.ids[0]
  associate_public_ip_address = true

  user_data = base64encode(local.user_data)

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    delete_on_termination = true
  }

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
  }

  # Wait for the instance to pass both system and instance status checks
  depends_on = [aws_security_group.web_server]
}
```

### outputs.tf

```hcl
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web_server.id
}

output "public_ip" {
  description = "Public IP address"
  value       = aws_instance.web_server.public_ip
}

output "public_dns" {
  description = "Public DNS hostname"
  value       = aws_instance.web_server.public_dns
}

output "ssh_command" {
  description = "Command to SSH into the instance"
  value       = "ssh -i ${local_sensitive_file.private_key.filename} ec2-user@${aws_instance.web_server.public_ip}"
}

output "app_url" {
  description = "URL to access the web app"
  value       = "http://${aws_instance.web_server.public_ip}:8080"
}
```

### terraform.tfvars

```hcl
# terraform.tfvars — gitignore this file
aws_region    = "us-east-1"
environment   = "dev"
instance_type = "t3.micro"
my_ip         = "203.0.113.5/32"   # Replace with your actual IP
```

### Running It

```bash
# Format code (fix whitespace and alignment)
terraform fmt

# Validate syntax
terraform validate

# Initialise (download providers, connect to backend)
terraform init

# Preview
terraform plan

# Apply
terraform apply

# After apply, test it
curl $(terraform output -raw app_url)

# SSH in
$(terraform output -raw ssh_command)

# Destroy when done
terraform destroy
```

---

## 11. Understanding terraform plan Output

```
Terraform will perform the following actions:

  # aws_instance.web_server will be created        ← (+) means new resource
  + resource "aws_instance" "web_server" {
      + ami                         = "ami-0c55b159cbfafe1f0"
      + arn                         = (known after apply)  ← set by AWS after creation
      + instance_type               = "t3.micro"
      + public_ip                   = (known after apply)
      + tags                        = {
          + "Environment" = "dev"
          + "Name"        = "dev-web-server"
        }
    }

  # aws_security_group.web_server will be updated  ← (~) means in-place update
  ~ resource "aws_security_group" "web_server" {
      ~ description = "Old description" -> "New description"
        id          = "sg-0abc123"           ← unchanged values shown without prefix
    }

  # aws_instance.old_server will be destroyed       ← (-) means deleted
  - resource "aws_instance" "old_server" {
      - ami           = "ami-old" -> null
      - instance_type = "t3.nano" -> null
    }

  # aws_instance.big_server must be replaced        ← (-/+) means destroy and recreate
  -/+ resource "aws_instance" "big_server" {
      ~ instance_type = "t3.micro" -> "t3.large"  # forces replacement
    }

Plan: 2 to add, 1 to change, 1 to destroy.
```

**Key symbols:**
- `+` New resource will be created
- `-` Resource will be destroyed
- `~` Resource will be updated in-place (no downtime for most attributes)
- `-/+` Resource must be replaced (destroyed then recreated — causes downtime)
- `(known after apply)` The value is determined by AWS at creation time

When you see `-/+`, check if there is any way to avoid the replacement. Sometimes changing a computed attribute in a different way avoids the recreate.

---

## 12. Terraform fmt and validate

```bash
# Format all .tf files in the current directory (and subdirectories with -recursive)
terraform fmt
terraform fmt -recursive

# Validate syntax and configuration consistency
terraform validate
# Success! The configuration is valid.
# Error: Missing required argument "ami" in resource "aws_instance" "web_server"

# Check if state is in sync with real infrastructure
terraform plan -detailed-exitcode
# Exit code 0: no changes
# Exit code 1: error
# Exit code 2: changes present
```

---

## 13. Common Errors

### State Drift

Drift occurs when someone changes infrastructure outside of Terraform (e.g., manually in the console).

```bash
# Detect drift
terraform plan
# Terraform will show: "aws_security_group.web_server will be updated in-place"
# even though you did not change any code

# Options:
# 1. Let Terraform revert it (re-run terraform apply)
# 2. Update your code to match the manual change
# 3. Import the new resource if it was created manually
```

### Dependency Errors

Terraform builds a dependency graph from explicit references. If you create a circular reference, Terraform will error.

```hcl
# Implicit dependency (preferred) — Terraform auto-detects this
resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.web.id]
  # Terraform knows: create the security group before the instance
}

# Explicit dependency — use when no attribute reference exists
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy_attachment.web_policy]
}
```

### Lock Errors

```
Error: Error acquiring the state lock
```

This means another `terraform apply` is running (or a previous one crashed without releasing the lock).

```bash
# Force-unlock (only do this if you are sure no apply is actually running)
terraform force-unlock LOCK-ID
```

### Provider Version Conflicts

```bash
# Upgrade providers (respects version constraints)
terraform init -upgrade

# Lock providers to specific versions (commit this file)
terraform providers lock -platform=linux_amd64 -platform=darwin_amd64
```

---

## Exercises

### Exercise 1 — Install Terraform and Run the EC2 Example

```bash
# macOS (Homebrew)
brew install terraform

# Linux (Debian/Ubuntu)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify
terraform version
```

Copy the four files from section 10 into a new directory. Fill in `terraform.tfvars` with your actual IP. Run the full workflow: `init → plan → apply`. Access the app URL from the output, then run `terraform destroy`.

### Exercise 2 — Add a Second EC2 Instance

Modify `main.tf` to add a second EC2 instance (`web_server_2`) in a different subnet. Add an output for its IP.

Run `terraform plan` — you should see `1 to add`. Observe how Terraform only creates the new resource without touching the existing one.

### Exercise 3 — Introduce Drift and Detect It

1. Run `terraform apply` to create your EC2 instance
2. Go to the AWS Console and manually add an inbound rule to the security group (allow port 8080 from 0.0.0.0/0)
3. Run `terraform plan` — what does it say?
4. Run `terraform apply` — does it revert the manual change?

### Exercise 4 — Use Data Sources

Remove the hardcoded `aws_region` default and instead use a data source to dynamically find the region:

```hcl
data "aws_region" "current" {}

locals {
  region = data.aws_region.current.name
}

output "current_region" {
  value = local.region
}
```

Add a data source that looks up your current IAM user and outputs their ARN:
```hcl
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "current_user_arn" {
  value = data.aws_caller_identity.current.arn
}
```

### Exercise 5 — Variables and Environments

Create two `.tfvars` files:
- `dev.tfvars` — t3.micro, environment = "dev"
- `staging.tfvars` — t3.small, environment = "staging"

Use different state keys in the backend for each:
```hcl
# In dev
key = "environments/dev/terraform.tfstate"

# In staging
key = "environments/staging/terraform.tfstate"
```

Show that `terraform workspace` is another (though less recommended for beginners) approach:
```bash
terraform workspace new staging
terraform workspace list
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
```

### Exercise 6 — Terraform Import

Launch an EC2 instance manually in the AWS Console. Then import it into Terraform without destroying and recreating it:

```bash
# Add an empty resource block to main.tf
# resource "aws_instance" "imported" {}

# Import the existing instance
terraform import aws_instance.imported i-0abcdef1234567890

# Now run plan — Terraform will show differences between the imported state and your empty resource block
terraform plan

# Fill in the resource block to match the actual configuration
# Keep running plan until you see "No changes"
```

This is a critical skill for teams adopting IaC with existing infrastructure.
