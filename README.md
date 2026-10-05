# Assignment 4 – Terraform Infrastructure for Redis

## 👨‍💻 Submitted By

**Jeetendra Singh**

---

## 📌 Objective

The objective of this assignment is to design and implement AWS infrastructure for a Redis server using **Terraform Infrastructure as Code (IaC)**.

The infrastructure includes:

* AWS VPC
* Public and Private Subnets
* Internet Gateway
* Route Table
* Security Group
* Redis EC2 Instance
* Amazon S3 Remote Terraform State
* S3 Versioning
* S3 Server-Side Encryption
* Terraform Outputs
* IAM Role-based AWS authentication

The Terraform state is stored remotely in an **Amazon S3 bucket** instead of keeping the state only on the local machine.

---

# 🏗️ Architecture

```text
                         AWS Cloud
                            |
                            |
                    +----------------+
                    |      VPC       |
                    |  10.0.0.0/16   |
                    +----------------+
                       /          \
                      /            \
                     /              \
          Public Subnet          Private Subnet
           10.0.1.0/24           10.0.2.0/24
          ap-south-1a            ap-south-1a
                |                      |
                |                      |
         Internet Gateway       Redis EC2 Instance
                |                 Private IP:
                |                  10.0.2.158
                |                      |
                |                    :6379
                |
         Public Route Table
```

### Remote Terraform State

```text
Terraform
    |
    |
    v
Amazon S3
jeetendra-redis-tf-state-2026
    |
    |
    +-- redis/terraform.tfstate
```

---

# ☁️ AWS Region

```text
ap-south-1
```

---

# 📁 Project Structure

```text
terrafrom/
│
├── backend.tf
├── provider.tf
├── vpc.tf
├── subnet.tf
├── network.tf
├── security_group.tf
├── redis.tf
├── outputs.tf
├── .terraform.lock.hcl
└── README.md
```

---

# 🔐 IAM Authentication

Terraform is running on an EC2 instance with an IAM Role attached.

### IAM Role

```text
terraform-server-role
```

The Terraform server uses the IAM role to authenticate with AWS instead of storing AWS Access Key and Secret Key directly inside the Terraform configuration.

This is safer than hard-coding AWS credentials in `.tf` files.

---

# ⚙️ Terraform Provider

The AWS provider is configured for the `ap-south-1` region.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.66"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

---

# 🌐 VPC Configuration

File:

```text
vpc.tf
```

VPC CIDR:

```text
10.0.0.0/16
```

Terraform configuration:

```hcl
resource "aws_vpc" "redis_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "redis-vpc"
  }
}
```

### VPC ID

```text
vpc-028813ae83e90aef2
```

---

# 🌍 Public Subnet

File:

```text
subnet.tf
```

Configuration:

```text
CIDR: 10.0.1.0/24
AZ: ap-south-1a
Public IP: Enabled
```

Resource:

```hcl
resource "aws_subnet" "redis_public_subnet" {
  vpc_id                  = aws_vpc.redis_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "redis-public-subnet"
  }
}
```

---

# 🔒 Private Subnet

Configuration:

```text
CIDR: 10.0.2.0/24
AZ: ap-south-1a
Public IP: Disabled
```

Resource:

```hcl
resource "aws_subnet" "redis_private_subnet" {
  vpc_id                  = aws_vpc.redis_vpc.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = false

  tags = {
    Name = "redis-private-subnet"
  }
}
```

Redis is deployed inside the private subnet.

---

# 🌐 Internet Gateway

File:

```text
network.tf
```

The Internet Gateway provides internet connectivity to the public subnet through the public route table.

```hcl
resource "aws_internet_gateway" "redis_igw" {
  vpc_id = aws_vpc.redis_vpc.id

  tags = {
    Name = "redis-igw"
  }
}
```

### Internet Gateway ID

```text
igw-058413c7bb6a71727
```

---

# 🛣️ Route Table

Public route table:

```text
redis-public-rt
```

Default route:

```text
0.0.0.0/0 → Internet Gateway
```

Configuration:

```hcl
resource "aws_route_table" "redis_public_rt" {
  vpc_id = aws_vpc.redis_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id  = aws_internet_gateway.redis_igw.id
  }

  tags = {
    Name = "redis-public-rt"
  }
}
```

The public subnet is associated with this route table.

---

# 🔐 Security Group

File:

```text
security_group.tf
```

Security Group:

```text
redis-sg
```

Redis port:

```text
6379/TCP
```

The Redis port is allowed only from the VPC CIDR:

```text
10.0.0.0/16
```

Configuration:

```hcl
resource "aws_security_group" "redis_sg" {
  name   = "redis-sg"
  vpc_id = aws_vpc.redis_vpc.id

  ingress {
    description = "Redis"
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "redis-sg"
  }
}
```

### Security Group ID

```text
sg-03cc89a96ed704c78
```

---

# 🗄️ Redis EC2 Instance

File:

```text
redis.tf
```

Redis is deployed in the private subnet.

Configuration:

```hcl
resource "aws_instance" "redis_server" {
  ami                         = "ami-0af7e19d7f4a36103"
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.redis_private_subnet.id
  associate_public_ip_address = false
  vpc_security_group_ids      = [aws_security_group.redis_sg.id]
  key_name                    = "Jeet11"

  user_data = <<-EOF
              #!/bin/bash
              dnf install -y redis
              systemctl enable redis
              systemctl start redis
              EOF

  tags = {
    Name = "redis-server"
  }
}
```

### Redis Instance Details

```text
Instance ID : i-03fa83dc600bc72d7
Instance Type: t3.micro
Private IP  : 10.0.2.158
Port        : 6379
Subnet      : Private Subnet
Public IP   : None
```

The Redis EC2 instance was verified as:

```text
running
```

---

# 📦 S3 Remote State

Terraform state is stored remotely in Amazon S3.

### S3 Bucket

```text
jeetendra-redis-tf-state-2026
```

### Backend Key

```text
redis/terraform.tfstate
```

### Region

```text
ap-south-1
```

Backend configuration:

```hcl
terraform {
  backend "s3" {
    bucket = "jeetendra-redis-tf-state-2026"
    key    = "redis/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

---

# 🔄 S3 State Management

The S3 bucket has:

* Versioning enabled
* Public access blocked
* Server-side encryption enabled
* Remote Terraform state

The existing local Terraform state was migrated to S3 using:

```bash
terraform init -migrate-state
```

When Terraform asked:

```text
Do you want to copy existing state to the new backend?
```

the following option was selected:

```text
yes
```

---

# 🔒 S3 Security

Public access is blocked on the Terraform state bucket.

Configuration:

```text
Block Public ACLs       : Enabled
Block Public Policy     : Enabled
Ignore Public ACLs      : Enabled
Restrict Public Buckets: Enabled
```

Server-side encryption:

```text
AES256
```

Versioning:

```text
Enabled
```

---

# 🚀 Terraform Commands Used

## Initialize Terraform

```bash
terraform init
```

For migrating local state to S3:

```bash
terraform init -migrate-state
```

---

## Validate Configuration

```bash
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

---

## Format Terraform Files

```bash
terraform fmt
```

---

## Create Execution Plan

```bash
terraform plan
```

The final plan returned:

```text
No changes. Your infrastructure matches the configuration.
```

---

## Apply Infrastructure

```bash
terraform apply
```

---

## Check Terraform State

```bash
terraform state list
```

Resources:

```text
aws_instance.redis_server
aws_internet_gateway.redis_igw
aws_route_table.redis_public_rt
aws_route_table_association.redis_public_rta
aws_security_group.redis_sg
aws_subnet.redis_private_subnet
aws_subnet.redis_public_subnet
aws_vpc.redis_vpc
```

---

# 📤 Terraform Outputs

File:

```text
outputs.tf
```

Configuration:

```hcl
output "vpc_id" {
  value = aws_vpc.redis_vpc.id
}

output "redis_instance_id" {
  value = aws_instance.redis_server.id
}

output "redis_private_ip" {
  value = aws_instance.redis_server.private_ip
}

output "redis_security_group_id" {
  value = aws_security_group.redis_sg.id
}
```

View outputs:

```bash
terraform output
```

### Output

```text
redis_instance_id = "i-03fa83dc600bc72d7"
redis_private_ip = "10.0.2.158"
redis_security_group_id = "sg-03cc89a96ed704c78"
vpc_id = "vpc-028813ae83e90aef2"
```

---

# ✅ Verification

## EC2 Instance Status

```bash
aws ec2 describe-instances \
  --instance-ids i-03fa83dc600bc72d7 \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

Output:

```text
running
```

---

## Terraform Plan Verification

```bash
terraform plan
```

Output:

```text
No changes. Your infrastructure matches the configuration.
```

This confirms that the Terraform configuration matches the currently managed AWS infrastructure.

---

# 📸 Screenshots

The following screenshots can be added to document the assignment:

### 1. Terraform Directory

```text
Screenshot:
terraform project files
```

### 2. VPC



<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 30 14 AM" src="https://github.com/user-attachments/assets/292c77cd-35bf-41a6-832c-fc4df2143a72" />

AWS VPC showing redis-vpc
```

### 3. Subnets
```

Screenshot:
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 31 05 AM" src="https://github.com/user-attachments/assets/98426f37-ef7b-4983-8e05-d0f54da939f2" />
Public and Private Subnets
```

### 4. Internet Gateway

```text
Screenshot:
redis-igw
```

<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 31 50 AM" src="https://github.com/user-attachments/assets/fe606ba8-a887-4557-bb60-d09724a39b98" />


### 5. Route Table

```text
Screenshot:
redis-public-rt
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 32 35 AM" src="https://github.com/user-attachments/assets/b1e1be16-c58b-4e22-a976-4725ebf9bd62" />


### 6. Security Group

```text
Screenshot:
redis-sg with TCP 6379
```

<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 33 45 AM" src="https://github.com/user-attachments/assets/6d4eafb4-e4c0-4e74-b986-c4602b0e29fa" />

### 7. Redis EC2

```text
Screenshot:
redis-server instance
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 34 26 AM" src="https://github.com/user-attachments/assets/e6a57eda-fabf-4577-833a-17d461ec902f" />


### 8. S3 Remote State

```text
Screenshot:
jeetendra-redis-tf-state-2026
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 35 22 AM" src="https://github.com/user-attachments/assets/26f5bfb6-3881-4d8c-ad69-87d3cdd911b4" />


### 9. S3 Versioning

```text
Screenshot:
S3 bucket versioning enabled
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 39 35 AM" src="https://github.com/user-attachments/assets/61d1257e-6228-438d-b74f-7c8b1c2e7d4b" />


### 10. Terraform Plan

```text
Screenshot:
No changes. Your infrastructure matches the configuration.
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 40 34 AM" src="https://github.com/user-attachments/assets/b70b269a-2847-4b4e-bcda-c9c38e29d9ab" />

### 11. Terraform Output

```text
Screenshot:
terraform output
```
<img width="1440" height="900" alt="Screenshot 2026-09-30 at 1 42 44 AM" src="https://github.com/user-attachments/assets/e91ab6c5-e148-4a3b-acfb-b1ad5cd49766" />

---

# 🧪 Verification Checklist



<img width="1440" height="900" alt="Screenshot 2026-10-01 at 1 27 13 PM" src="https://github.com/user-attachments/assets/2eef47e3-98d7-4021-b968-184abaaca354" />

| Component                  | Status |
| -------------------------- | ------ |
| Terraform Provider         | ✅      |
| IAM Role Authentication    | ✅      |
| VPC                        | ✅      |
| Public Subnet              | ✅      |
| Private Subnet             | ✅      |
| Internet Gateway           | ✅      |
| Route Table                | ✅      |
| Security Group             | ✅      |
| Redis EC2                  | ✅      |
| S3 Remote State            | ✅      |
| S3 Versioning              | ✅      |
| S3 Encryption              | ✅      |
| State Migration            | ✅      |
| Terraform Outputs          | ✅      |
| Terraform Plan             | ✅      |
| Redis Service Verification |         |
| Redis Connectivity Test    |  DONE   |
| Git Push                   |         |

---

# 🧹 Destroy Infrastructure

If the infrastructure is no longer required, it can be removed using:

```bash
terraform destroy
```

**Note:** Do not run this command until the assignment is completely finished and the resources are no longer required.

---

# 📚 Key Concepts Learned

During this assignment, the following concepts were practiced:

* Infrastructure as Code (IaC)
* Terraform
* AWS Provider
* IAM Role authentication
* VPC
* Public and Private Subnets
* Internet Gateway
* Route Tables
* Security Groups
* EC2
* Redis
* Amazon S3
* Terraform Remote Backend
* Terraform State Management
* S3 Versioning
* S3 Encryption
* Terraform Outputs
* Terraform Plan and Apply

---

# 🎯 Conclusion

This assignment demonstrates how to provision AWS infrastructure for a Redis server using Terraform.

The Redis server is deployed inside a private subnet, while Terraform state is maintained remotely in an encrypted and versioned S3 bucket.

The infrastructure can be recreated, managed, and updated using Terraform configuration files instead of manually creating each AWS resource.

---

## 👨‍💻 Author

**Jeetendra Singh**

**AWS | Linux | Terraform | DevOps**
