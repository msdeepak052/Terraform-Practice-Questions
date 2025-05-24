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


### 2. Create a VPC with a Public Subnet
- Define a VPC with CIDR block `10.0.0.0/16`  
- Add a public subnet, internet gateway, and route table  

### a. main.tf

```hcl

provider "aws" {

  region = var.aws_region

}

resource "aws_vpc" "deepak-vpc" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "deepak-vpc"
  }
}

# resource "aws_subnet" "public_subnet" {
#   vpc_id = aws_vpc.deepak-vpc.id
#   cidr_block = [var.public_subnets]
#   map_public_ip_on_launch = true

# }

resource "aws_subnet" "public_subnet" {
  for_each = { for subnet in var.subnets : subnet.name => subnet }

  vpc_id                  = aws_vpc.deepak-vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = can(regex("public", each.key))

  tags = {
    Name = each.key
  }

}

resource "aws_internet_gateway" "igw-main" {
  vpc_id = aws_vpc.deepak-vpc.id

  tags = {
    Name = "igw-main"
  }
}

resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.deepak-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw-main.id
  }
  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public-rt-association" {
  for_each = {
    for subnet in var.subnets : subnet.name => subnet
    if can(regex("public", subnet.name))
  }

  subnet_id      = aws_subnet.public_subnet[each.key].id
  route_table_id = aws_route_table.public-rt.id

}

```

### b. variables.tf

```hcl
variable "aws_region" {
  type        = string
  description = "AWS Region"

}

variable "cidr_block" {
  type        = string
  description = "cidr_block"

}

# variable "public_subnets" {
#   type        = list(string)
#   description = "public_subnets"

# }

variable "subnets" {
  type = list(object({
    name = string
    cidr = string
    az   = string
  }))
  description = "List of subnets to create"
}
```
### c. terraform.tfvars

```hcl

aws_region = "ap-south-1"
cidr_block = "10.0.0.0/16"
# public_subnets = ["10.0.0.0/24","10.0.1.0/24"]

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

```

### d. outputs.tf

```hcl

output "aws_vpc" {
  value = aws_vpc.deepak-vpc.id
}

output "public_subnets" {
  value = [
    for name, subnet in aws_subnet.public_subnet : subnet.id
    if can(regex("public", name))
  ]
}

output "igw-id" {
  value = aws_internet_gateway.igw-main.id
}

```

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

### 3. Use Input Variables
- Parameterize values like VPC name, CIDR, and subnet CIDR 

### Already followed  in Q1 & Q2 

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

### 4. Define and Use Output Variables
- Output the instance ID and public IP of the EC2 instance

 ### a.main.tf

 ```hcl
provider "aws" {
  region = var.aws_region
}

resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  tags = {
    Name = "MyEC2Instance"
  }
}

```

### b.variables.tf

```hcl
variable "aws_region" {
  description = "The AWS region to deploy into"
  type        = string
}

variable "ami_id" {
  description = "Amazon Machine Image ID"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}

variable "key_name" {
  description = "EC2 key pair name"
  type        = string
}

```

### c.terraform.tfvars

```hcl
aws_region     = "ap-south-1"
ami_id         = "ami-0e35ddab05955cf57"
instance_type  = "t2.micro"
key_name       = "lappynewawss"
```

### d. outputs.tf

```hcl
output "instance_public_ip" {
  value = aws_instance.example.public_ip
}

output "instance_id" {
  value = aws_instance.example.id
}

```

### Ouput from Terraform
```hcl
Outputs:

ec2-instance-id = "i-0b2f3397b10417de0"
ec2-instance-pub-ip = "13.201.22.20"
```

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

### 5. Create a Simple Security Group
- Allow SSH (port 22) and HTTP (port 80) from any IP

### a.main.tf
```hcl
provider "aws" {

  region = var.aws_region

}

data "aws_vpc" "selected" {
  id = "vpc-0b4d8673f9549d276"
}

resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = data.aws_vpc.selected.id

  tags = {
    Name = "web-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "custom_ingress" {
  for_each = {
    for rule in var.ingress_rules :
    "${rule.from_port}-${rule.to_port}-${rule.cidr}" => rule
  }

  security_group_id = aws_security_group.web_sg.id
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

  security_group_id = aws_security_group.web_sg.id
  ip_protocol       = each.value.protocol
  cidr_ipv4         = each.value.cidr
  description       = each.value.description
}

```

### b. variables.tf

```hcl
variable "aws_region" {
  type        = string
  description = "AWS Region"

}

variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    from_port = number
    to_port   = number
    protocol  = string
    cidr      = string
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

### c. terraform.tfvars

```hcl
aws_region    = "ap-south-1"
ingress_rules = [
  {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr        = "10.0.0.0/16"
    description = "SSH access"
  },
  {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "HTTP access"
  },
  {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "HTTPS access"
  }
]

egress_rules = [
  {
    protocol    = "-1"
    cidr        = "0.0.0.0/0"
    description = "Allow all outbound traffic"
  }
]
```
### d. outputs.tf
```hcl
output "sg" {
  value = aws_security_group.web_sg.id
}
```

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

### 6. Destroy Specific Resources
- Use `terraform destroy -target=aws_instance.example` to destroy the EC2 instance only
- 
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
  count                       = 3
  ami                         = var.ami_id
  availability_zone           = random_shuffle.az.result[0]
  instance_type               = var.instance_type
  key_name                    = var.key_name
  subnet_id                   = local.subnets_in_random_az[0] # First match in random AZ
  associate_public_ip_address = true

  tags = merge(local.tags, {
    Name = "ec2-instance-${count.index+1}"
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
### Results

### Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

### Outputs:
```hcl
instance_azs = [
  "ap-south-1b",
  "ap-south-1b",
  "ap-south-1b",
]
instance_ids = [
  "i-00313ba73e8a0ada1",
  "i-0dea2ceeb43eef1bf",
  "i-0d35288d19db8eb3a",
]
instance_subnet_ids = [
  "subnet-0d1e44b24dd3e9c69",
  "subnet-0d1e44b24dd3e9c69",
  "subnet-0d1e44b24dd3e9c69",
]
subnet_selected = [
  "subnet-060a8e055c719c184",
  "subnet-07d0f65b80ead8bb3",
  "subnet-0d1e44b24dd3e9c69",
]
ubuntu@ip-172-31-13-230:~/ec2_instance_destroy_specific$ terraform state list
data.aws_availability_zones.available
data.aws_subnet.subnet["subnet-060a8e055c719c184"]
data.aws_subnet.subnet["subnet-07d0f65b80ead8bb3"]
data.aws_subnet.subnet["subnet-0d1e44b24dd3e9c69"]
data.aws_subnets.selected
data.aws_vpc.selected
aws_instance.ec2[0]
aws_instance.ec2[1]
aws_instance.ec2[2]
random_shuffle.az
```
![image](https://github.com/user-attachments/assets/067339e7-c848-4178-a743-4c7386060cf2)
![image](https://github.com/user-attachments/assets/77e40bbd-ae7f-4fb8-ae0a-519280094195)
![image](https://github.com/user-attachments/assets/be6e2fb9-7a6b-4673-842d-29990893910e)
![image](https://github.com/user-attachments/assets/ec318494-2e7e-41b0-847a-ef6bb24313cc)




### 7. Terraform Formatting and Validation
- Run `terraform fmt`, `terraform validate`, and `terraform plan` as part of workflow

```hcl
terraform fmt; terraform validate; terraform plan
```

