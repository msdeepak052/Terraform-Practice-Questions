# Terraform-Practice-Questions

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
