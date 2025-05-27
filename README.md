# Terraform-Practice-Questions

## 🛠️ Intermediate-Level Terraform Scenarios

### 1. Create a Reusable VPC Module
- Build a module for VPC creation  
- Accept variables for CIDR blocks, subnet count, etc.  

### Answer
### a. modules/vpc

### a.1. main.tf

```hcl

resource "aws_vpc" "deepak-vpc" {
    cidr_block = var.cidr_block
    enable_dns_hostnames = true
    enable_dns_support = true
    
    tags = merge(var.tags, {
        Name = var.vpc_name
    })

}

# resource "aws_subnet" "public-subnet" {
#     count = length(var.public_subnets)
#     vpc_id = aws_vpc.deepak-vpc.id
#     cidr_block = var.public_subnets[count.index]
#     map_public_ip_on_launch = true

#     tags = merge(var.tags, {
#         Name = "subnet-public"
#     })

# }

# resource "aws_subnet" "pvt-subnet" {
#     vpc_id = aws_vpc.deepak-vpc.id
#     cidr_block = var.pvt_subnets[count.index]
#     map_public_ip_on_launch = false

#     tags = merge(var.tags, {
#         Name = var.pvt_subnet_name
#     })

# }

resource "aws_subnet" "subnets" {
  for_each = { for subnet in var.subnets : subnet.name => subnet }

  vpc_id                  = aws_vpc.deepak-vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = can(regex("public", each.key))

  tags = {
    Name = each.key
  }

}


resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.deepak-vpc.id

    tags = merge(var.tags, {
        Name = var.igw_name
    })
  
}

resource "aws_eip" "deepak-eip" {
    domain                    = "vpc"
}

resource "aws_nat_gateway" "deepak-nat" {
    subnet_id = element(
    [
      for subnet in aws_subnet.subnets : subnet.id
      if can(regex("public", subnet.tags["Name"]))  # Match against the Name tag
    ],
    0
  )
    # subnet_id = [for k, subnet in aws_subnet.subnets : subnet.id if can(regex("public", k))][0]
    allocation_id = aws_eip.deepak-eip.id

    tags = merge(var.tags, {
        Name = var.nat_name
    })

    # To ensure proper ordering, it is recommended to add an explicit dependency
    # on the Internet Gateway for the VPC.
    depends_on = [aws_internet_gateway.igw]
  
}

resource "aws_route_table" "public-rt" {
    vpc_id = aws_vpc.deepak-vpc.id
    
    route {
        cidr_block = var.public_route_cidr
        gateway_id = aws_internet_gateway.igw.id
  }
  tags = merge(var.tags,{
    Name = "public-rt"
  })

}

resource "aws_route_table_association" "public-rt-association" {
    for_each = {
        for subnet in var.subnets : subnet.name => subnet
            if can(regex("public", subnet.name))
        }
    
    subnet_id = aws_subnet.subnets[each.key].id
    route_table_id = aws_route_table.public-rt.id

}
resource "aws_route_table" "pvt-rt" {
    vpc_id = aws_vpc.deepak-vpc.id
    
    route {
        cidr_block = var.pvt_route_cidr
        nat_gateway_id= aws_nat_gateway.deepak-nat.id
  }
  tags = merge(var.tags,{
    Name = "pvt-rt"
  })

}

resource "aws_route_table_association" "pvt-rt-association" {
    for_each = {
        for subnet in var.subnets : subnet.name => subnet
            if can(regex("private", subnet.name))
        }
    
    subnet_id = aws_subnet.subnets[each.key].id
    route_table_id = aws_route_table.pvt-rt.id

}

resource "aws_security_group" "deepak_sg" {
  name   = "deepak-sg"
  vpc_id = aws_vpc.deepak-vpc.id

  tags = merge(var.tags,{
    Name = "deepak-sg"
  })
}

resource "aws_vpc_security_group_ingress_rule" "custom_ingress" {
  for_each = {
    for rule in var.ingress_rules :
    "${rule.from_port}-${rule.to_port}-${rule.cidr}" => rule
  }

  security_group_id = aws_security_group.deepak_sg.id
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  ip_protocol       = each.value.protocol
  cidr_ipv4         = each.value.cidr
  description       = each.value.description
}

resource "aws_vpc_security_group_egress_rule" "custom_egress" {
  for_each = {
    for rule in var.egress_rules :
    "${rule.protocol}-${rule.cidr}" => rule
  }

  security_group_id = aws_security_group.deepak_sg.id
  ip_protocol       = each.value.protocol
  cidr_ipv4         = each.value.cidr
  description       = each.value.description
}

```
### a.2 variables.tf

```hcl
variable "cidr_block" {
    type = string
    description = "cidr_block"
  
}

variable "tags" {
  type = map(string)
}

variable "vpc_name" {
    type = string
    description = "vpc_name"
  
}

# variable "pvt_subnets" {
#     type = list(string)
#     description = "pvt_subnets"
  
# }

# variable "pvt_subnet_name" {
#     type = string
#     description = "pvt_subnet_name"
  
# }

variable "igw_name" {
    type = string
    description = "igw_name"
  
}

variable "subnets" {
  type = list(object({
    name = string
    cidr = string
    az   = string
  }))
  description = "List of subnets to create"
}

variable "nat_name" {
    type = string
    description = "nat_name"
  
}

variable "public_route_cidr" {
    type = string
    description = "public_route_cidr"
  
}

variable "pvt_route_cidr" {
    type = string
    description = "pvt_route_cidr"
  
}

variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr        = string
    description = string
  }))
  default = []
}

variable "egress_rules" {
  description = "List of egress rules"
  type = list(object({
    protocol    = string
    cidr        = string
    description = string
  }))
  default = []
}

```

### a.3 outputs.tf

```hcl
output "vpc_id" {
  value = aws_vpc.deepak-vpc.id
}

output "subnets" {
  description = "Details of all created subnets"
  value       = aws_subnet.subnets
}

```

### Root modules

### a. main.tf

```hcl
provider "aws" {

  region = var.aws_region

}

module "vpc" {
  source            = "./modules/vpc"
  cidr_block        = var.cidr_block
  subnets           = var.subnets
  vpc_name          = var.vpc_name
  nat_name          = var.nat_name
  igw_name          = var.igw_name
  tags              = local.common_tags
  public_route_cidr = var.public_route_cidr
  pvt_route_cidr    = var.pvt_route_cidr

}
```
### b.variables.tf

```hcl
variable "aws_region" {
  type        = string
  description = "AWS Region"

}

variable "cidr_block" {
  type        = string
  description = "cidr_block"

}

variable "subnets" {
  type = list(object({
    name = string
    cidr = string
    az   = string
  }))
  description = "List of subnets to create"
}

variable "nat_name" {
  type        = string
  description = "nat_name"

}

variable "igw_name" {
  type        = string
  description = "igw_name"

}


variable "vpc_name" {
  type        = string
  description = "vpc_name"

}

variable "public_route_cidr" {
  type        = string
  description = "public_route_cidr"

}

variable "pvt_route_cidr" {
  type        = string
  description = "pvt_route_cidr"

}

```
### c. terraform.tfvars

```hcl
aws_region = "ap-south-1"
nat_name   = "deepak-nat"
vpc_name   = "deepak-vpc"
cidr_block = "10.0.0.0/16"
subnets = [
  {
    name = "subnet-public-1"
    cidr = "10.0.1.0/24"
    az   = "ap-south-1a"
  },
  {
    name = "subnet-public-2"
    cidr = "10.0.2.0/24"
    az   = "ap-south-1b"
  },
  {
    name = "subnet-private-1"
    cidr = "10.0.101.0/24"
    az   = "ap-south-1a"
  },
  {
    name = "subnet-private-2"
    cidr = "10.0.102.0/24"
    az   = "ap-south-1b"
  }
]

public_route_cidr = "0.0.0.0/0"
pvt_route_cidr    = "0.0.0.0/0"
igw_name          = "deepak-igw"
```
### d.locals.tf

```hcl
locals {
  common_tags = {
    Project     = "terraform-vpc-module"
    Environment = "Dev"
    Owner       = "Deepak"
  }
}
```

### e.outputs.tf

```hcl
output "aws_vpc" {
  value = module.vpc.vpc_id
}

output "subnets" {
  description = "Details of all created subnets"
  value       = module.vpc.subnets
}

```
<hr style="height:2px; border-width:0; color:black; background-color:black">

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
