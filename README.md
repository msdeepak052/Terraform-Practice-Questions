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

### folder structure

```sh
EC2-PVT-NAT/
├── main.tf                 # Root module - contains provider and module calls
├── variables.tf            # Root module variables
├── outputs.tf              # Root module outputs
├── terraform.tfvars        # Variable definitions (optional)
├── locals.tf               # Local values (if using common_tags)
│
├── modules/
│   ├── vpc/
│   │   ├── main.tf         # VPC, subnets, NAT, IGW, routes, etc.
│   │   ├── variables.tf    # VPC module variables
│   │   ├── outputs.tf      # VPC module outputs (vpc_id, subnets, sg_id, etc.)
│   │
│   └── ec2/
│       ├── main.tf         # EC2 instance resource
│       ├── variables.tf    # EC2 module variables
│       └── outputs.tf      # EC2 module outputs (instance_id, etc.)
└── 
```

### ec2-module

#### a. main.tf

```hcl
resource "aws_instance" "deepak-ec2" {
    ami = var.ami_id
    instance_type = var.instance_type
    subnet_id = var.subnet_id
    vpc_security_group_ids = var.ec2_sg
    key_name = "newawss"
    tags = var.tags
}
```
#### b. outputs.tf
```hcl
output "instance_id" {
  value = aws_instance.deepak-ec2.id
}

output "instance_public_ip" {
  value = aws_instance.deepak-ec2.public_ip

}
```
#### c. variablestf

```hcl
variable "ami_id" {
  type = string
  description = "ami id"
}
variable "instance_type" {
  type = string
  description = "instance_type"
}

variable "subnet_id" {
  type = string
  description = "subnet_id"
}

variable "ec2_sg" {
  type = list(string)
  description = "ec2_sg"
}
variable "tags" {
  type = map(string)
  description = "Tags"
}

```

### vpc-module

#### a. main.tf

```hcl
resource "aws_vpc" "deepak-vpc" {
    cidr_block = var.cidr_block
    enable_dns_hostnames = true
    enable_dns_support = true

    tags = merge(var.tags,{
        Name = var.vpc_name
    })
}

resource "aws_subnet" "subnets" {
  for_each = {
    for subnet in var.subnets : 
      subnet.name => subnet
  }

  vpc_id = aws_vpc.deepak-vpc.id
  cidr_block = each.value.cidr
  availability_zone = each.value.az
  map_public_ip_on_launch = can(regex("public", each.key))

  tags = merge(var.tags,{
    Name = each.key
  })
}

resource "aws_internet_gateway" "deepak-igw" {
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
    depends_on = [aws_internet_gateway.deepak-igw]
}

resource "aws_route_table" "public-rt" {

  vpc_id = aws_vpc.deepak-vpc.id

  tags = merge(var.tags,{
    Name = "public-rt"
  })
  
}

resource "aws_route" "public-rt-route" {
  route_table_id = aws_route_table.public-rt.id
  destination_cidr_block = var.public_destination_route_cidr
  gateway_id = aws_internet_gateway.deepak-igw.id

}

resource "aws_route_table_association" "public-rt-saasociation" {
  for_each = {
        for subnet in var.subnets : subnet.name => subnet
            if can(regex("public", subnet.name))
        }
  subnet_id = aws_subnet.subnets[each.key].id
  route_table_id = aws_route_table.public-rt.id
  
}

resource "aws_route_table" "pvt-rt" {

  vpc_id = aws_vpc.deepak-vpc.id

  tags = merge(var.tags,{
    Name = "pvt-rt"
  })
  
}

resource "aws_route" "pvt-rt-route" {
  route_table_id = aws_route_table.pvt-rt.id
  destination_cidr_block = var.pvt_destination_route_cidr
  nat_gateway_id = aws_nat_gateway.deepak-nat.id

}

resource "aws_route_table_association" "pvt-rt-saasociation" {
  for_each = {
        for subnet in var.subnets : subnet.name => subnet
            if can(regex("private", subnet.name))
        }
  subnet_id = aws_subnet.subnets[each.key].id
  route_table_id = aws_route_table.pvt-rt.id
  
}

resource "aws_security_group" "deepak_sg" {
  vpc_id = aws_vpc.deepak-vpc.id

    tags = merge(var.tags,{
    Name = "deepak_sg"
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

#### b. variables.tf

```hcl
variable "cidr_block" {
    type = string
    description = "cidr_block"
}

variable "vpc_name" {
  type = string
  description = "VPC Name"
}

variable "tags" {
  type = map(string)
  description = "Tags"
}

variable "subnets" {
  type = list(object({
    name = string
    cidr = string
    az   = string
  }))
  description = "List of subnets to create"
}

variable "igw_name" {
  type = string
  description = "IGW Name"
}

variable "nat_name" {
  type = string
  description = "NAT Name"
}

variable "public_destination_route_cidr" {
    type = string
    description = "public_route_cidr"
  
}

variable "pvt_destination_route_cidr" {
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

#### c. outputs.tf
```hcl
output "vpc_id" {
  value = aws_vpc.deepak-vpc.id
}

output "subnets" {
  description = "Details of all created subnets"
  value       = aws_subnet.subnets
}

output "sg_id" {
  description = "Security group ID for EC2 instances"
  value = aws_security_group.deepak_sg.id
}

output "private_subnet_ids" {
  value = {
    for name, subnet in aws_subnet.subnets : 
    name => subnet.id
    if can(regex("private", name))
  }
}

output "public_subnet_ids" {
  value = {
    for name, subnet in aws_subnet.subnets : 
    name => subnet.id
    if can(regex("public", name))
  }
}

```

### root tfs

#### a.main.tf

```hcl
provider "aws" {
  region = var.aws_region
}

module "nat-deepak-vpc" {
  source                        = "./modules/vpc"
  cidr_block                    = var.cidr_block
  vpc_name                      = var.vpc_name
  subnets                       = var.subnets
  nat_name                      = var.nat_name
  igw_name                      = var.igw_name
  public_destination_route_cidr = var.public_destination_route_cidr
  pvt_destination_route_cidr    = var.pvt_destination_route_cidr
  ingress_rules                 = var.ingress_rules
  egress_rules                  = var.egress_rules
  tags                          = local.common_tags
}

module "ec2-pvt-nat" {
  source = "./modules/ec2"

  for_each = module.nat-deepak-vpc.private_subnet_ids

  ami_id        = var.ami_id
  instance_type = var.instance_type
  subnet_id     = each.value
  ec2_sg        = [module.nat-deepak-vpc.sg_id]
  tags = merge(local.common_tags, {
    Name = "deepak-web-server-${each.key}"
  })

}


```

#### b. variables.tf

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

variable "vpc_name" {
  type        = string
  description = "vpc_name"

}

variable "igw_name" {
  type        = string
  description = "IGW Name"
}
variable "nat_name" {
  type        = string
  description = "NAT Name"
}
variable "public_destination_route_cidr" {
  type        = string
  description = "public_route_cidr"

}

variable "pvt_destination_route_cidr" {
  type        = string
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

variable "ami_id" {
  type        = string
  description = "ami id"
}

variable "instance_type" {
  type        = string
  description = "instance_type"
}
```

#### c. locals.tf

```hcl
locals {
  common_tags = {
    Project     = "terraform-vpc-module"
    Environment = "Dev"
    Owner       = "Deepak"
  }
}
```

#### d. outputs.tf

```hcl
output "vpc_id" {
  value = module.nat-deepak-vpc.vpc_id
}

output "subnets" {
  value = { for k,subnet in module.nat-deepak-vpc.subnets : k=>subnet.id }
  
}

output "ec2_ids" {
  value = { for k, ec2 in module.ec2-pvt-nat : k => ec2.instance_id }
  # value = [for ec2 in module.ec2-pvt-nat : ec2.instance_id]
  # value = values(module.ec2-pvt-nat)[*].instance_id

}
```


#### e. terraform.tfvars

```hcl
aws_region = "ap-south-1"
cidr_block = "10.0.0.0/16"
vpc_name   = "deepak-vpc"
nat_name   = "deepak-nat"
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

igw_name = "deepak-igw"

public_destination_route_cidr = "0.0.0.0/0"
pvt_destination_route_cidr    = "0.0.0.0/0"
ingress_rules = [
  {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "Allow SSH access from any IP"
  },
  {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "Allow HTTP access from any IP"
  },
  {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "Allow HTTPS access from any IP"
  },
  {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "Allow alternative HTTP port access from any IP"
  },
  {
    from_port   = 8443
    to_port     = 8443
    protocol    = "tcp"
    cidr        = "0.0.0.0/0"
    description = "Allow alternative HTTPS port access from any IP"
  }
]

egress_rules = [
  {
    protocol    = "-1"
    cidr        = "0.0.0.0/0"
    description = "Allow all outbound traffic"
  }
]

ami_id        = "ami-0e35ddab05955cf57"
instance_type = "t2.micro"


```

### Bonus

###  dynamically create EC2 instances based on requirements by making these improvements:
#### 1. Enhanced Variable Structure (in variables.tf)
```hcl

variable "ec2_instances" {
  description = "Map of EC2 instances to create"
  type = map(object({
    ami_id        = string
    instance_type = string
    subnet_type   = string # "private" or "public"
    extra_sgs     = list(string) # Additional security groups
    user_data     = string # Optional startup scripts
  }))
  default = {
    web-server = {
      ami_id        = "ami-123456"
      instance_type = "t3.medium"
      subnet_type   = "private"
      extra_sgs     = []
      user_data     = ""
    }
  }
}
```
#### 2. Dynamic EC2 Creation (in main.tf)
```hcl

module "ec2-instances" {
  source = "./modules/ec2"
  
  for_each = {
    for name, config in var.ec2_instances : 
    name => merge(config, {
      subnet_id = config.subnet_type == "private" ? 
        element(values(module.nat-deepak-vpc.private_subnet_ids), 0) : # First private subnet
        element(values(module.nat-deepak-vpc.public_subnet_ids), 0)   # First public subnet
    })
  }

  ami_id        = each.value.ami_id
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id
  ec2_sg        = concat([module.nat-deepak-vpc.sg_id], each.value.extra_sgs)
  user_data     = each.value.user_data
  
  tags = merge(local.common_tags, {
    Name = "${each.key}-${each.value.subnet_type}"
  })
}
```

#### 3. Enhanced EC2 Module (modules/ec2/main.tf)
```hcl

resource "aws_instance" "dynamic-ec2" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  vpc_security_group_ids = var.ec2_sg
  user_data     = var.user_data
  key_name      = "newawss"

  tags = var.tags
}

```

#### 4. Usage Example (terraform.tfvars)
```hcl

ec2_instances = {
  web-server = {
    ami_id        = "ami-0c55b159cbfafe1f0"
    instance_type = "t3.micro"
    subnet_type   = "private"
  },
  bastion-host = {
    ami_id        = "ami-0c55b159cbfafe1f0"
    instance_type = "t2.small"
    subnet_type   = "public"
    extra_sgs     = ["sg-12345678"]
  }
}
```
### Advanced Version (Multiple AZ Placement):
```hcl

module "ec2-instances" {
  source = "./modules/ec2"
  
  for_each = var.ec2_instances

  ami_id        = each.value.ami_id
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_type == "private" ? 
    module.nat-deepak-vpc.private_subnet_ids[keys(module.nat-deepak-vpc.private_subnet_ids)[each.value.az_index % length(module.nat-deepak-vpc.private_subnet_ids)]] :
    module.nat-deepak-vpc.public_subnet_ids[keys(module.nat-deepak-vpc.public_subnet_ids)[each.value.az_index % length(module.nat-deepak-vpc.public_subnet_ids)]]
    
  ec2_sg        = concat([module.nat-deepak-vpc.sg_id], each.value.extra_sgs)
  
  tags = merge(local.common_tags, {
    Name = "${each.key}-${each.value.subnet_type}"
  })
}

Would you like me to focus on any particular aspect of this dynamic approach?
for_each = {
    for name, config in var.ec2_instances : 
    name => merge(config, {
      subnet_id = config.subnet_type == "private" ? 
        element(values(module.nat-deepak-vpc.private_subnet_ids), 0) : # First private subnet
        element(values(module.nat-deepak-vpc.public_subnet_ids), 0)   # First public subnet
    })
  }
```

#### Assume we have:

#### VPC Outputs:
```hcl
module.nat-deepak-vpc.private_subnet_ids = {
  "private-subnet-1" = "subnet-111111"
  "private-subnet-2" = "subnet-222222"
}

module.nat-deepak-vpc.public_subnet_ids = {
  "public-subnet-1" = "subnet-333333"
  "public-subnet-2" = "subnet-444444"
}
```

#### Input Variable (terraform.tfvars):
```hcl

    ec2_instances = {
      web-server = {
        ami_id        = "ami-123456"
        instance_type = "t3.medium"
        subnet_type   = "private"
      },
      bastion-host = {
        ami_id        = "ami-789012"
        instance_type = "t2.small"
        subnet_type   = "public"
      }
    }
```

#### The for_each Logic Explained
```hcl

for_each = {
  for name, config in var.ec2_instances : 
  name => merge(config, {
    subnet_id = config.subnet_type == "private" ? 
      element(values(module.nat-deepak-vpc.private_subnet_ids), 0) : 
      element(values(module.nat-deepak-vpc.public_subnet_ids), 0)
  })
}
```
#### Step-by-Step Transformation:
#### Original Input:
```json

{
  "web-server" = { ami_id = "ami-123456", instance_type = "t3.medium", subnet_type = "private" }
  "bastion-host" = { ami_id = "ami-789012", instance_type = "t2.small", subnet_type = "public" }
}

```

#### After Processing:
```json

    {
      "web-server" = {
        ami_id = "ami-123456",
        instance_type = "t3.medium",
        subnet_type = "private",
        subnet_id = "subnet-111111"  # First private subnet
      },
      "bastion-host" = {
        ami_id = "ami-789012",
        instance_type = "t2.small",
        subnet_type = "public",
        subnet_id = "subnet-333333"  # First public subnet
      }
    }
```
#### Key Components:

#### for name, config in var.ec2_instances:

- Iterates through each instance in the ec2_instances map
- name = key ("web-server", "bastion-host")
- config = value (the object containing ami_id, etc.)
- merge(config, { subnet_id = ... }):
    - Takes the original config
    - Adds a new subnet_id field

#### Ternary Condition:
```hcl

    config.subnet_type == "private" ? 
      element(values(module.nat-deepak-vpc.private_subnet_ids), 0) : 
      element(values(module.nat-deepak-vpc.public_subnet_ids), 0)
```
#### Checks subnet_type
- If private: Takes first value from private subnets (subnet-111111)
- If public: Takes first value from public subnets (subnet-333333)
##### element(values(...), 0):
- values() extracts just the subnet IDs (ignores the keys)
- element(..., 0) takes the first one (index 0)

#### Resulting EC2 Instances:

    - web-server:

        - Placed in first private subnet (subnet-111111)

        - Gets all original config plus the computed subnet_id

    - bastion-host:

        - Placed in first public subnet (subnet-333333)

        - Gets all original config plus the computed subnet_id

#### Why This Is Powerful:

    - Dynamic Placement: Automatically chooses correct subnet type

    - Clean Configuration: Keeps subnet selection logic separate from instance definitions

    - Flexible: Easy to add more instances just by adding to ec2_instances map
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


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
