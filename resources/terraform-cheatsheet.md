# Terraform Cheatsheet

## Core Workflow

```bash
terraform init          # Download providers, initialise backend
terraform validate      # Check syntax and configuration validity
terraform fmt           # Format all .tf files to canonical style
terraform fmt -recursive  # Format all subdirectories too
terraform plan          # Preview changes (does not change anything)
terraform plan -out=tfplan  # Save plan to file
terraform apply         # Apply changes (prompts for confirmation)
terraform apply tfplan  # Apply a saved plan (no prompt)
terraform apply -auto-approve  # Skip confirmation prompt (CI/CD only)
terraform destroy       # Destroy all managed resources
terraform destroy -auto-approve  # Skip confirmation
```

---

## State

```bash
terraform show                          # Show current state in human-readable form
terraform state list                    # List all resources in state
terraform state show aws_instance.web   # Inspect a specific resource in state
terraform state rm aws_instance.web     # Remove a resource from state (does not destroy it)
terraform state mv old_name new_name    # Rename a resource in state
terraform import aws_instance.web i-abc123  # Import an existing resource into state
terraform force-unlock LOCK-ID          # Release a stuck state lock
```

---

## Outputs

```bash
terraform output                        # Show all outputs
terraform output alb_dns_name           # Show a specific output
terraform output -json                  # All outputs as JSON
terraform output -raw alb_dns_name      # Raw value (no quotes — useful in scripts)
```

---

## Workspaces

```bash
terraform workspace list                # List workspaces
terraform workspace new staging         # Create a new workspace
terraform workspace select staging      # Switch to a workspace
terraform workspace show                # Show current workspace
terraform workspace delete staging      # Delete a workspace
```

---

## Variables

```bash
# Pass a variable on the command line
terraform plan -var="environment=staging"

# Use a variable file
terraform plan -var-file="staging.tfvars"

# Environment variable override (prefix: TF_VAR_)
export TF_VAR_environment=production
terraform plan
```

---

## HCL Syntax Reference

### Resource

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type

  tags = {
    Name = "web-server"
  }
}
```

### Variable

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}
```

### Output

```hcl
output "public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}
```

### Local

```hcl
locals {
  name_prefix = "${var.environment}-${var.project}"
}

# Reference: local.name_prefix
```

### Data Source

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Reference: data.aws_ami.amazon_linux.id
```

### Count (create multiple resources)

```hcl
resource "aws_subnet" "public" {
  count      = 2
  cidr_block = var.public_subnet_cidrs[count.index]
}

# Reference: aws_subnet.public[0].id
```

### For Each

```hcl
resource "aws_s3_bucket" "logs" {
  for_each = toset(["app", "access", "error"])
  bucket   = "my-logs-${each.key}"
}

# Reference: aws_s3_bucket.logs["app"].id
```

---

## Remote State Backend (S3 + DynamoDB)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-YOURNAME"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## Reading terraform plan Output

| Symbol | Meaning |
|--------|---------|
| `+` | Resource will be created |
| `-` | Resource will be destroyed |
| `~` | Resource will be updated in-place |
| `-/+` | Resource will be destroyed and recreated |
| `(known after apply)` | Value set by the provider after creation |

---

## Provider Authentication (AWS)

```bash
# Option 1: Named profile
export AWS_PROFILE=my-profile

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/...
export AWS_DEFAULT_REGION=us-east-1

# Option 3: EC2 instance profile (no credentials needed — preferred in CI/CD on EC2)
# Configure the instance profile in the launch template
```

Never hardcode credentials in `.tf` files.

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Error acquiring the state lock` | Another apply is running or crashed | `terraform force-unlock LOCK-ID` |
| `Provider produced inconsistent result` | Provider bug or drift | Re-run `terraform plan` |
| `Resource already exists` | Resource was created outside Terraform | `terraform import` |
| `No such host` | Backend S3 bucket does not exist | Create the bucket first, then `terraform init` |
| `-/+ forces replacement` | Changed an immutable attribute | Accept the recreate or find an alternative attribute |

---

## .gitignore for Terraform

```
.terraform/
*.tfstate
*.tfstate.backup
terraform.tfvars
*.pem
.terraform.lock.hcl    # Commit this file — it locks provider versions
```
