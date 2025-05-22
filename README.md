# Terraform-Practice-Questions

## ✅ Beginner-Level Terraform Scenarios

### 1. Provision a Single EC2 Instance
- Create a Terraform script to launch one EC2 instance in AWS  
- Use variables for AMI ID, instance type, and tags  

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
