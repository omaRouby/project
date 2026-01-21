Below is your content rewritten **cleanly and correctly in GitHub-ready `.md` (Markdown) syntax**.
You can copy-paste this directly into a `README.md` file.

---

# 🌐 VPC Configuration

## 📌 VPC Details

| Attribute      | Value                   |
| -------------- | ----------------------- |
| **VPC ID**     | `vpc-05ae083309505dd2d` |
| **CIDR Block** | `10.0.0.0/16`           |
| **Region**     | `us-east-1`             |
| **Tenancy**    | Default                 |

### 🔧 Commands Used

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)

# Tag VPC
aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=fintech-vpc
```

---

# 🏘️ Subnet Strategy

## 🌐 Public Subnets (Internet-Facing)

| Subnet          | AZ         | Subnet ID                  | Purpose          |
| --------------- | ---------- | -------------------------- | ---------------- |
| Public Subnet A | us-east-1a | `subnet-01a0ea892bddbe272` | ALB, NAT Gateway |
| Public Subnet B | us-east-1b | `subnet-01e465ff29dc73d6e` | ALB, NAT Gateway |

## 🔒 Private Subnets (Application Tier)

| Subnet           | AZ         | Subnet ID                  | Purpose                 |
| ---------------- | ---------- | -------------------------- | ----------------------- |
| Private Subnet A | us-east-1a | `subnet-01808abced051ff76` | ECS Fargate, RDS, Redis |
| Private Subnet B | us-east-1b | `subnet-08f5e862854552155` | ECS Fargate, RDS, Redis |

## 🗄️ Database Subnets (Data Tier)

| Subnet      | AZ         | Subnet ID                  | Purpose      |
| ----------- | ---------- | -------------------------- | ------------ |
| DB Subnet A | us-east-1a | `subnet-0ad4497bc86861f6a` | RDS Multi-AZ |
| DB Subnet B | us-east-1b | `subnet-0008720550e19af4a` | RDS Multi-AZ |

### 🔧 Commands Used

```bash
# Create Public Subnets
PUBLIC_SUBNET_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

PUBLIC_SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --query 'Subnet.SubnetId' --output text)
```

---

# 🌍 Internet Gateway & NAT Gateway

## 🌐 Internet Gateway

| Attribute        | Value                              |
| ---------------- | ---------------------------------- |
| **IGW ID**       | `igw-0314349d8ac8eed51`            |
| **Attached VPC** | `vpc-05ae083309505dd2d`            |
| **Purpose**      | Internet access for public subnets |

## 🔁 NAT Gateway

| Attribute          | Value                                 |
| ------------------ | ------------------------------------- |
| **NAT Gateway ID** | `nat-01970a185ce853dff`               |
| **Elastic IP**     | `eipalloc-0b0795b25304316a4`          |
| **Subnet**         | Public Subnet A                       |
| **Purpose**        | Outbound internet for private subnets |

### 🔧 Commands Used

```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

aws ec2 create-tags \
  --resources $IGW_ID \
  --tags Key=Name,Value=fintech-igw

# Allocate Elastic IP for NAT Gateway
EIP_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' --output text)
```

---

# 🛣️ Route Tables

## 🌐 Public Route Table

| Attribute              | Value                                      |
| ---------------------- | ------------------------------------------ |
| **Route Table ID**     | `rtb-02ae44cf1ae9b71c7`                    |
| **Routes**             | `10.0.0.0/16 → local`<br>`0.0.0.0/0 → IGW` |
| **Associated Subnets** | Public Subnets A & B                       |

## 🔒 Private Route Table

| Attribute              | Value                                              |
| ---------------------- | -------------------------------------------------- |
| **Route Table ID**     | `rtb-0c80d8bcfa564bc25`                            |
| **Routes**             | `10.0.0.0/16 → local`<br>`0.0.0.0/0 → NAT Gateway` |
| **Associated Subnets** | Private Subnets A & B                              |

### 🔧 Commands Used

```bash
# Create Public Route Table
PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $PUBLIC_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT_ID \
  --subnet-id $PUBLIC_SUBNET_A

aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT_ID \
  --subnet-id $PUBLIC_SUBNET_B
```

---

# 🔒 Security Groups

## 🌐 Application Load Balancer SG

* **SG ID:** `sg-08990648c310029a9`
* **Inbound**

  * HTTP (80) from CloudFront IP ranges
  * HTTPS (443) from CloudFront IP ranges
* **Outbound**

  * All traffic allowed

## 🚀 ECS Fargate SG

* **SG ID:** `sg-00f1816433ac5e19b`
* **Inbound**

  * HTTP (8080) from ALB SG
* **Outbound**

  * All traffic allowed

## 🔗 VPC Endpoint SG

* **SG ID:** `sg-06df4166ed23f18c5`
* **Inbound**

  * HTTPS (443) from ECS SG
* **Outbound**

  * Default (All)

---

# 🔗 VPC Endpoints

## 📡 Interface Endpoints

| Service         | Type      | Purpose                      |
| --------------- | --------- | ---------------------------- |
| Secrets Manager | Interface | Secure secret retrieval      |
| ECR API         | Interface | Secure image metadata access |
| ECR Docker      | Interface | Secure image pulls           |
| STS             | Interface | Secure token service access  |

### 🔧 Commands Used

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids $PRIVATE_SUBNET_A $PRIVATE_SUBNET_B \
  --security-group-ids $VPC_ENDPOINT_SG \
  --private-dns-enabled
```

---

# 🏢 Multi-AZ High Availability

## 🧭 AZ Distribution Strategy

* **Compute:** ECS Fargate across multiple AZs
* **Load Balancing:** ALB across public subnets
* **Database:** RDS Aurora Multi-AZ
* **Cache:** ElastiCache Redis Multi-AZ
* **Storage:** DynamoDB (native Multi-AZ)

## ✅ Benefits Achieved

* **Fault Tolerance:** Survives AZ failure
* **High Availability:** No single point of failure
* **Load Distribution:** Balanced traffic
* **Disaster Recovery:** Automatic failover

---

If you want, next I can:

* Add **architecture ASCII diagrams**
* Turn this into a **professional GitHub README**
* Add **Terraform / CloudFormation references**
* Add **security & cost-optimization sections**

Just tell me 👍
