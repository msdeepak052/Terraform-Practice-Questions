# Terraform-Practice-Questions

# ✅ Beginner-Level Terraform Scenarios

# 1. Provision a Single EC2 Instance
- Create a Terraform script to launch one EC2 instance in AWS  
- Use variables for AMI ID, instance type, and tags  

## Answer

### a. main.tf
```hcl
provider "aws" {

  region = var.aws_region

}

resource "random_shuffle" "az" {
  input        = data.aws_availability_zones.available.names
  result_count = 1
}



resource "aws_instance" "ec2" {

  ami                         = var.ami_id
  availability_zone           = random_shuffle.az.result[0]
  instance_type               = var.instance_type
  key_name                    = var.key_name
  subnet_id                   = local.subnets_in_random_az[0] # First match in random AZ
  associate_public_ip_address = true

  tags = merge(local.tags, {
    Name = "ec2-instance"
  })

}
```
### b. variables.tf

```hcl

variable "aws_region" {
  type        = string
  description = "AWS Region"

}

variable "ami_id" {
  type        = string
  description = "ami_id"

}

variable "instance_type" {
  type        = string
  description = "instance type"

}
variable "key_name" {
  type        = string
  description = "Key Name"

}

```

### c. terraform.tfvars

```hcl

aws_region    = "ap-south-1"
ami_id        = "ami-0e35ddab05955cf57"
instance_type = "t2.micro"
key_name      = "lappynewawss"

```

### d. locals.tf

```hcl
locals {

  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  subnets_in_random_az = [
    for subnet_id in data.aws_subnets.selected.ids : subnet_id
    if data.aws_subnet.subnet[subnet_id].availability_zone == random_shuffle.az.result[0]
  ]
  tags = {
    Repo = "terraform-aws-vpc"
    Org  = "terraform-aws-modules"
  }
}

```

### e. data_source.tf

```hcl
# Fetch existing VPC by tag
data "aws_vpc" "selected" {
  id = "vpc-0b4d8673f9549d276"
}

# Fetch subnet in the selected VPC
data "aws_subnets" "selected" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
}

data "aws_subnet" "subnet" {
  for_each = toset(data.aws_subnets.selected.ids)
  id       = each.value
}


data "aws_availability_zones" "available" {}

```
### f. outputs.tf

```hcl
output "aws_instance" {
  value = aws_instance.ec2.id
}

output "subnet" {
  value = data.aws_subnets.selected.ids[0]
}

output "subnet" {
  value = data.aws_subnets.selected.ids[0]
}

output "subnet_selected" {
    value = data.aws_subnet.subnet.id
  
}

output "azs" {
  value = aws_instance.ec2.availability_zone
}

```

# Bonus Tip

## 🔀 random_shuffle
### 🔹 Purpose:
It randomly reorders items in a list and lets you use the result.

### ✅ Example: Randomly select an Availability Zone

```hcl

data "aws_availability_zones" "available" {
  state = "available"
}

resource "random_shuffle" "az" {
  input        = data.aws_availability_zones.available.names
  result_count = 1
}

```
### 🔸 Output:
If available.names = ["ap-south-1a", "ap-south-1b", "ap-south-1c"], the result could be:

```h
random_shuffle.az.result[0] = "ap-south-1b"
```
# You can then use it like:

``` hcl

availability_zone = random_shuffle.az.result[0]
```

# 🐶 random_pet
### 🔹 Purpose:
It generates a human-readable random name, e.g., playful-lion or awesome-dog.

## ✅ Example: Random EC2 Name
```hcl

resource "random_pet" "ec2_name" {
  length    = 2
  separator = "-"
}
```
### 🔸 Output:
```hcl

random_pet.ec2_name.id = "happy-tiger"
```
### Use it like:

``` h

resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = random_pet.ec2_name.id
  }
}
```
## 🔁 When to Use Each
### Resource	                 Use When You Want To...
random_shuffle	             Pick one or more random elements from a list
random_pet	                 Generate a fun and unique name for resources


## ✅ Randomly pick an AZ and subnet, and give the EC2 a fun random name

### 🧩 Use Case:
- You have a VPC with subnets across multiple AZs.

- You want to deploy a single EC2 instance:

- In a random AZ

- With a subnet in that AZ

- Tagged with a random, readable name

## ✅ Full Terraform Example
### a. main.tf
```hcl

provider "aws" {
  region = "ap-south-1"
}

# Get available AZs in the region
data "aws_availability_zones" "available" {
  state = "available"
}

# Shuffle AZs and pick one
resource "random_shuffle" "az" {
  input        = data.aws_availability_zones.available.names
  result_count = 1
}

# Get a specific VPC by name
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Name"
    values = ["my-vpc"]
  }
}

# Get subnets in the VPC
data "aws_subnets" "selected" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
}

# Get AZ info for each subnet
data "aws_subnet" "details" {
  for_each = toset(data.aws_subnets.selected.ids)
  id       = each.key
}

# Get subnets only in the random AZ
locals {
  subnets_in_random_az = [
    for id, subnet in data.aws_subnet.details : id
    if subnet.availability_zone == random_shuffle.az.result[0]
  ]
}

# Generate a fun EC2 name
resource "random_pet" "ec2_name" {
  length    = 2
  separator = "-"
}

# Create EC2 instance in random AZ with fun name
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = local.subnets_in_random_az[0]
  availability_zone = random_shuffle.az.result[0]
  key_name      = var.key_name

  tags = {
    Name = random_pet.ec2_name.id
  }
}
```
### b. variables.tf
```hcl

variable "ami_id" {
  description = "AMI ID for EC2"
  type        = string
}

variable "instance_type" {
  default     = "t2.micro"
  description = "Instance type"
}

variable "key_name" {
  description = "Key pair name"
  type        = string
}
```
### c.terraform.tfvars
```hcl

ami_id      = "ami-0abcdef1234567890"
key_name    = "my-keypair"

```
### 🔍 What This Does:
- Picks a random AZ from the region

- Filters subnets in your VPC that belong to that AZ

- Launches the EC2 in one of those subnets

- Gives it a fun tag like "happy-koala"

-------------------------------------------------------------------------------------------------------------------------------------------------------------------
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-------------------------------------------------------------------------------------------------------------------------------------------------------------------

### 2. Create a VPC with a Public Subnet
- Define a VPC with CIDR block `10.0.0.0/16`  
- Add a public subnet, internet gateway, and route table  

### 3. Use Input Variables
- Parameterize values like VPC name, CIDR, and subnet CIDR  

### 4. Define and Use Output Variables
- Output the instance ID and public IP of the EC2 instance  

### 5. Create a Simple Security Group
- Allow SSH (port 22) and HTTP (port 80) from any IP  

### 6. Destroy Specific Resources
- Use `terraform destroy -target=aws_instance.example` to destroy the EC2 instance only  

### 7. Terraform Formatting and Validation
- Run `terraform fmt`, `terraform validate`, and `terraform plan` as part of workflow  

---

## 🛠️ Intermediate-Level Terraform Scenarios

### 1. Create a Reusable VPC Module
- Build a module for VPC creation  
- Accept variables for CIDR blocks, subnet count, etc.  

### 2. Deploy EC2 Instances in Private Subnet with NAT Gateway
- Create private subnet and NAT Gateway in a public subnet  
- Launch EC2 instances with internet access via NAT  

### 3. Use Remote State with S3 and DynamoDB
- Store Terraform state in S3  
- Use DynamoDB for state locking  

### 4. Use Workspaces for Environment Isolation
- Create `dev`, `staging`, and `prod` workspaces  
- Use different `terraform.tfvars` files per workspace  

### 5. Conditionally Create Resources
- Use `count` or `for_each` to create resources based on conditions  

### 6. Use Data Sources to Reference AMIs
- Use `data "aws_ami"` to fetch the latest Amazon Linux 2 AMI  

### 7. Pass Variables Using terraform.tfvars
- Organize and load variables from `.tfvars` files  

### 8. Define Multiple Security Groups Dynamically
- Use `for_each` with a map to create multiple security groups  

### 9. Import Existing Resources
- Use `terraform import` to bring an existing S3 bucket under Terraform management  

### 10. Provision Multiple EC2 Instances Using for_each
- Launch EC2 instances from a map of instance names and types  

---

## 🔥 Advanced-Level Terraform Scenarios

### 1. Build a Multi-Tier Architecture Using Modules
- Separate modules for frontend, backend, and DB  
- Set up networking between the tiers  

### 2. Use Dynamic Blocks for Security Group Rules
- Use `dynamic` blocks to define rules based on a variable list  

### 3. Split Configuration into Multiple Files and Environments
- Organize code into folders like `modules/`, `env/dev/`, `env/prod/`  

### 4. Terraform CI/CD Integration (GitHub Actions / GitLab CI)
- Create CI pipelines that run `terraform init`, `plan`, and `apply`  
- Secure secrets and manage environments  

### 5. Use locals and Complex Data Structures
- Simplify code using `locals` with nested maps/lists  

### 6. Create and Manage an EKS Cluster
- Provision an EKS cluster  
- Deploy a sample Kubernetes app  

### 7. Use a Remote Module from GitHub
- Reference a public module and pass variables to it  

### 8. Terraform State Management: Move/Remove Resources
- Use `terraform state mv`, `rm`, and `taint` commands  

### 9. Handle Resource Dependencies with depends_on
- Manage dependencies explicitly using `depends_on`  

### 10. Multiple AWS Profiles/Regions
- Use provider aliasing for deployments across accounts or regions


## ➕ Additional Practice: Terraform Modules

### 1. Module for EC2 with Custom Security Group
- Create a reusable EC2 instance module.
- Accept variables for AMI ID, instance type, and custom security group rules.
- Output the instance ID and public IP.

### 2. Nested Module Usage
- Create a root module that uses a `vpc` module and an `ec2` module internally.
- Pass outputs from the `vpc` module as inputs to the `ec2` module.

### 3. Parameterize Resource Count in Module
- Modify a module to support launching multiple EC2 instances using `count` or `for_each`.
- Accept a list or map of instances with names and types.

### 4. Write a Module for S3 Bucket Creation
- Accept parameters for versioning, lifecycle rules, and bucket policy.
- Output bucket name and ARN.

### 5. Module with Conditional Logic
- Add a variable to enable or disable the creation of a resource (e.g., NAT Gateway) inside the module.
- Use `count` or `for_each` with conditions.

### 6. Module for CloudWatch Alarms
- Build a module that creates alarms for EC2 instance CPU utilization.
- Accept threshold and alarm actions as variables.

### 7. DRY Modules for Multi-Tier Architecture
- Create a base `network` module used by frontend/backend/db modules.
- Demonstrate reuse by deploying all 3 tiers in separate subnets with isolated security groups.

### 8. Use of `locals` in Modules
- Refactor your module to use `locals` for computed values like naming conventions or merged tags.

### 9. Output Propagation from Modules
- Capture module outputs in the root module and use them in another module.
- Example: Output `subnet_id` from VPC module, use it in EC2 module.

### 10. Use Public Modules (Terraform Registry or GitHub)
- Integrate a public module (e.g., `terraform-aws-modules/vpc/aws`) into your setup.
- Pass variables to customize its behavior for your environment.


## 🚀 Real-Life Industry Terraform Scenarios

### 1. Launch a Scalable Web Application
- Create a VPC with public and private subnets.
- Deploy an Auto Scaling Group of EC2 instances behind an Application Load Balancer.
- Use security groups and target groups.

### 2. Deploy a Multi-Tier Architecture
- Provision three tiers: Web (ALB + EC2), App (EC2), DB (RDS).
- Use private subnets for App and DB layers.
- Set up routing, security groups, and NAT Gateway.

### 3. Set Up Terraform CI/CD in GitHub Actions
- Create a pipeline that runs `terraform fmt`, `validate`, `plan`, and `apply`.
- Use environment secrets for AWS credentials.
- Separate workflows for staging and production.

### 4. Terraform with Remote State in S3 and State Locking in DynamoDB
- Configure `backend` block in Terraform.
- Use versioning and encryption for the S3 bucket.
- Enable locking and consistency with DynamoDB table.

### 5. Manage IAM Roles and Policies
- Create IAM roles for EC2, Lambda, and ECS services.
- Attach inline and managed policies.
- Implement least-privilege access.

### 6. Provision EKS Cluster with Worker Nodes
- Use `terraform-aws-modules/eks/aws`.
- Set up node groups with appropriate scaling.
- Output kubeconfig and authenticate with AWS CLI.

### 7. Deploy Serverless Application Using Lambda and API Gateway
- Create Lambda functions with IAM roles.
- Integrate with API Gateway to expose endpoints.
- Use CloudWatch for logging and monitoring.

### 8. Blue/Green Deployment Using Terraform Workspaces
- Create dev, staging, and prod workspaces.
- Use workspaces to deploy different versions of the same infrastructure.
- Automate promotion from staging to prod.

### 9. Provision Multi-Region Disaster Recovery Setup
- Deploy infrastructure in two AWS regions using provider aliasing.
- Sync state and DNS using Route 53 failover routing.
- Use `terraform_remote_state` for data sharing.

### 10. Infrastructure Tagging Strategy
- Implement a consistent tagging module.
- Automatically apply tags like `Owner`, `Environment`, `CostCenter`.
- Enforce tagging using pre-commit hooks or policies.

### 11. Manage Secrets with AWS SSM or Secrets Manager
- Store DB credentials in SSM Parameter Store or Secrets Manager.
- Reference them in Terraform using `data` sources.
- Pass securely to EC2 or Lambda functions.

### 12. Scheduled Auto Start/Stop of EC2 Instances
- Use `aws_instance` with Lambda and CloudWatch rules.
- Create schedule expressions to reduce costs in non-business hours.
- Use Terraform to deploy automation.

### 13. Monitor Infrastructure with CloudWatch Dashboards
- Create custom metrics and dashboards via Terraform.
- Monitor EC2, RDS, EKS metrics.
- Set alarms for thresholds.

### 14. Use Sentinel or OPA for Policy-as-Code
- Implement guardrails like “No public S3 buckets” or “Tag enforcement”.
- Run policies during CI/CD or `terraform apply`.

### 15. Automate SSL Certificate Management
- Provision ACM certificates using Terraform.
- Use Route 53 for DNS validation.
- Attach certificates to ALBs or CloudFront.

### 16. Use Terraform to Provision Azure/AWS Hybrid Infrastructure
- Configure providers for both AWS and Azure.
- Create VPC in AWS and VNet in Azure.
- Use shared secrets or monitoring solutions across clouds.

### 17. Import Legacy Resources to Terraform
- Use `terraform import` to bring existing infra under code.
- Clean up and map the state with configuration files.
- Validate with `terraform plan`.

### 18. Create an Audit-Ready Logging and Monitoring Setup
- Enable AWS Config and CloudTrail via Terraform.
- Stream logs to a central S3 bucket.
- Apply retention and lifecycle policies.

### 19. Onboard New Environments with a Single Command
- Parameterize environment creation (e.g., dev, qa, uat, prod).
- Use modules to deploy identical infrastructure across environments.
- Separate state and secrets per environment.

### 20. Implement Cost Optimization with Auto Scaling and Spot Instances
- Use mixed instance policies in Auto Scaling Group.
- Balance On-Demand and Spot instances with fallback strategies.
- Use predictive scaling or schedules.


