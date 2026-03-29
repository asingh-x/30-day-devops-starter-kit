# Week 4 — AWS Fundamentals, Terraform, and Production Deployment

This week moves from local tools into real cloud infrastructure. You will provision resources on AWS, manage access securely, understand networking, and automate everything with Terraform. The week ends with a capstone that deploys the Python hostname app from Week 2 into a production-grade AWS environment.

## What you will be able to do after this week

- Explain AWS global infrastructure and choose the right region for a workload
- Create IAM users, groups, and roles following least-privilege principles
- Launch and connect to EC2 instances and understand security groups
- Design a VPC with public and private subnets, an Internet Gateway, and a NAT Gateway
- Store objects in S3 and understand storage classes
- Set up an RDS database in Multi-AZ mode
- Build an Application Load Balancer with HTTPS termination using ACM
- Write Terraform code that provisions real AWS infrastructure
- Manage Terraform state remotely in S3 with DynamoDB locking
- Deploy a Python application as a systemd service on EC2 via Auto Scaling Groups

---

## Week Schedule

| Day | File | Topic | Key Concepts |
|-----|------|--------|--------------|
| 16 | [day-16-aws-basics-iam.md](./day-16-aws-basics-iam.md) | AWS Basics + IAM | Cloud models, regions, IAM users/roles/policies, AWS CLI, MFA |
| 17 | [day-17-ec2-vpc-networking.md](./day-17-ec2-vpc-networking.md) | EC2 + VPC Networking | Instance types, SSH, security groups, VPC, subnets, IGW, NAT |
| 18 | [day-18-aws-storage-databases.md](./day-18-aws-storage-databases.md) | Storage + Databases + Load Balancing | S3, RDS, ALB, ACM, ASG, Route 53 |
| 19 | [day-19-terraform.md](./day-19-terraform.md) | Terraform — IaC | HCL, providers, state, plan/apply, remote backend, real EC2 example |
| 20 | [exercises.md](./exercises.md) | Capstone Project | Full production deployment: VPC + ALB + ASG + Route 53 + Terraform |

---

## Prerequisites

You should be comfortable with everything from Weeks 1–3:
- Linux command line and shell scripting
- Python application structure (the hostname app)
- Git, GitHub, and branching workflows
- Docker and Docker Compose
- Jenkins pipelines

---

## AWS Account Setup Before Day 16

1. Create a free-tier AWS account at https://aws.amazon.com/free/
2. **Do not use the root account after initial setup.** Create an IAM user with AdministratorAccess and use that for everything.
3. Enable MFA on the root account immediately.
4. Install the AWS CLI:
   ```bash
   # macOS
   brew install awscli

   # Linux
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```
5. Verify: `aws --version`

---

## Cost Warning

All exercises use Free Tier resources wherever possible. The following can incur charges — **delete them when done**:

| Resource | Free Tier Limit | Charge if exceeded |
|----------|----------------|--------------------|
| EC2 t2.micro or t3.micro | 750 hours/month | ~$0.012/hour |
| RDS db.t3.micro | 750 hours/month | ~$0.017/hour |
| NAT Gateway | Not free | ~$0.045/hour + data |
| ALB | 750 hours/month | ~$0.008/hour |
| Route 53 hosted zone | Not free | $0.50/month per zone |

Run `terraform destroy` after every practice session.

---

## Folder Structure for This Week

```
week-4/
├── README.md                        # This file
├── day-16-aws-basics-iam.md
├── day-17-ec2-vpc-networking.md
├── day-18-aws-storage-databases.md
├── day-19-terraform.md
├── exercises.md                     # Capstone project
└── terraform/                       # Your Terraform code goes here
    ├── provider.tf
    ├── vpc.tf
    ├── security_groups.tf
    ├── alb.tf
    ├── asg.tf
    ├── route53.tf
    ├── variables.tf
    ├── outputs.tf
    └── terraform.tfvars             # gitignored — your actual values
```
