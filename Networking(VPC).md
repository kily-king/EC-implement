Design it **Multi-AZ ready** (minimum effort now, no redesign later).

---

## Target Architecture (What You’ll End Up With)

**High-level components**

* 1 VPC
* 2 Public subnets (NAT + future ALB)
* 2 Private subnets (EKS / MSK)
* 1 Internet Gateway
* 1 NAT Gateway
* Route tables (public & private)
* Baseline security groups

---

## Step 0 – Decide Your CIDR Plan (Do This First)

Choose a CIDR that:

* Is private
* Has room to grow

**Recommended for test:**

```
VPC CIDR: 10.0.0.0/16
```

Subnet breakdown:

| Subnet Type | AZ   | CIDR          |
| ----------- | ---- | ------------- |
| Public-1    | AZ-a | 10.0.1.0/24   |
| Public-2    | AZ-b | 10.0.2.0/24   |
| Private-1   | AZ-a | 10.0.101.0/24 |
| Private-2   | AZ-b | 10.0.102.0/24 |

---

## Step 1 – Create the VPC

1. Go to **VPC → Your VPCs → Create VPC**
2. Choose:

   * **Resources to create:** VPC only
   * **Name:** `test-vpc`
   * **IPv4 CIDR:** `10.0.0.0/16`
   * **Tenancy:** Default
3. Click **Create VPC**

✅ **Acceptance:** VPC created with CIDR defined

---

## Step 2 – Create Public Subnets (2)

Go to **VPC → Subnets → Create subnet**

### Public Subnet 1

* VPC: `test-vpc`
* Subnet name: `public-subnet-1`
* AZ: `us-east-1a`
* CIDR: `10.0.1.0/24`

### Public Subnet 2

* Subnet name: `public-subnet-2`
* AZ: `us-east-1b`
* CIDR: `10.0.2.0/24`

After creation:

1. Select **each public subnet**
2. Click **Actions → Edit subnet settings**
3. Enable **Auto-assign public IPv4 address**

✅ Required for NAT Gateway and any public-facing service

---

## Step 3 – Create Private Subnets (2)

Repeat subnet creation:

### Private Subnet 1

* Name: `private-subnet-1`
* AZ: `us-east-1a`
* CIDR: `10.0.101.0/24`

### Private Subnet 2

* Name: `private-subnet-2`
* AZ: `us-east-1b`
* CIDR: `10.0.102.0/24`

❌ Do **NOT** enable public IP assignment

✅ **Acceptance:** Minimum 2 public + 2 private subnets

---

## Step 4 – Create & Attach Internet Gateway (IGW)

1. Go to **VPC → Internet Gateways → Create internet gateway**
2. Name: `test-igw`
3. Create
4. Select it → **Actions → Attach to VPC**
5. Choose `test-vpc`

✅ **Acceptance:** Internet Gateway attached

---

## Step 5 – Create Public Route Table

1. Go to **VPC → Route Tables → Create route table**
2. Name: `public-rt`
3. VPC: `test-vpc`

### Add Route

* Destination: `0.0.0.0/0`
* Target: `Internet Gateway`
* Select `test-igw`

### Associate Subnets

* Associate:

  * `public-subnet-1`
  * `public-subnet-2`

✅ Public subnets now have internet access

---

## Step 6 – Create NAT Gateway (For Private Subnets)

### Step 6.1 – Allocate Elastic IP

1. Go to **VPC → Elastic IPs**
2. Allocate Elastic IP
3. Keep it unassigned for now

### Step 6.2 – Create NAT Gateway

1. Go to **VPC → NAT Gateways → Create NAT gateway**
2. Name: `test-nat-gw`
3. Subnet: `public-subnet-1`
4. Elastic IP: select the one you created
5. Create NAT Gateway

⏳ Wait until **Status = Available**

✅ **Acceptance:** NAT Gateway available

---

## Step 7 – Create Private Route Table

1. **Route Tables → Create route table**
2. Name: `private-rt`
3. VPC: `test-vpc`

### Add Route

* Destination: `0.0.0.0/0`
* Target: `NAT Gateway`
* Select `test-nat-gw`

### Associate Subnets

* `private-subnet-1`
* `private-subnet-2`

✅ Private subnets can now access the internet **outbound only**

---

## Step 8 – Create Baseline Security Groups

### 1️⃣ VPC Baseline SG

* Name: `vpc-baseline-sg`
* Description: Base SG for internal communication
* VPC: `test-vpc`

**Inbound**

* All traffic
* Source: **self**

**Outbound**

* All traffic
* Destination: `0.0.0.0/0`

---

### 2️⃣ Private Workload SG (EKS / MSK)

* Name: `private-workload-sg`
* VPC: `test-vpc`

**Inbound**

* From `vpc-baseline-sg`
* (Later you’ll add EKS/MSK-specific ports)

**Outbound**

* All traffic → `0.0.0.0/0`

✅ **Acceptance:** Security groups baseline created

---

## Step 9 – Final Acceptance Checklist

| Requirement              | Status |
| ------------------------ | ------ |
| VPC with CIDR            | ✅      |
| 2 Public Subnets         | ✅      |
| 2 Private Subnets        | ✅      |
| Internet Gateway         | ✅      |
| NAT Gateway              | ✅      |
| Correct Route Tables     | ✅      |
| Baseline Security Groups | ✅      |
| Multi-AZ Ready           | ✅      |

---

## What This Enables Next (Roadmap)

This VPC is now **production-style**, even in test:

* EKS cluster creation (private subnets)
* MSK deployment
* ALB / NLB in public subnets
* Event-driven services with secure egress


