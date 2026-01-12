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

# 12345

Great — below is the **complete, final reference** that ties everything together.
I’ll do this in **three clear sections**, exactly as you asked:

1. **Final Reference Architecture (visual + explanation)**
2. **Exact EKS Creation Steps (console-first, no gaps)**
3. **MSK Networking Prepared Alongside EKS (no rework later)**

This assumes everything you built so far is in place.

---

# 1️⃣ Final Reference Architecture (EKS + MSK + VPC Endpoints)

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/05/19/bdb1186-msk-connectivity-patterns001.png)

![Image](https://docs.aws.amazon.com/images/eks/latest/best-practices/images/networking/subnet_image.png)

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/07/30/BDB-1216-image001.png)

## Architecture Overview (Plain English)

### Networking Layer

* **1 VPC (10.0.0.0/16)**
* **2 AZs**
* **Public subnets**

  * NAT Gateway
  * Load Balancers (ALB/NLB)
* **Private subnets**

  * EKS worker nodes
  * MSK brokers
* **No public IPs** for EKS or MSK

### Connectivity

* Internet Gateway → Public subnets only
* NAT Gateway → outbound access for private subnets
* **VPC Endpoints**

  * S3 (Gateway)
  * ECR API + DKR (Interface)
  * STS (Interface)

### Platform Layer

* EKS (private API endpoint)
* MSK (private brokers only)
* Internal traffic stays **inside VPC**

---

# 2️⃣ Exact EKS Creation Steps (Console – From Zero to Ready)

This is **production-correct**, even for test.

---

## Step 1 – Create IAM Role for EKS Cluster

### IAM → Roles → Create role

* Trusted entity: **AWS service**
* Service: **EKS**
* Use case: **EKS - Cluster**

Attach policies:

* `AmazonEKSClusterPolicy`

Role name:

```
eks-cluster-role
```

---

## Step 2 – Create IAM Role for EKS Nodes

### IAM → Roles → Create role

* Trusted entity: **EC2**

Attach policies:

* `AmazonEKSWorkerNodePolicy`
* `AmazonEKS_CNI_Policy`
* `AmazonEC2ContainerRegistryReadOnly`

Role name:

```
eks-node-role
```

---

## Step 3 – Create the EKS Cluster

### Go to: EKS → Clusters → Create cluster

#### Cluster Configuration

* Name: `test-eks`
* Kubernetes version: latest stable
* Cluster service role: `eks-cluster-role`

---

### Networking Configuration (CRITICAL)

* VPC: `test-vpc`
* Subnets:

  * ✅ `private-subnet-1`
  * ✅ `private-subnet-2`
* Security group:

  * `eks-control-plane-sg`

#### Endpoint access

* **Private access:** ✅ Enabled
* **Public access:** ❌ Disabled

This ensures:

* API server is **only reachable inside VPC**

Create cluster → wait until **ACTIVE**

⏳ ~10–15 minutes

---

## Step 4 – Create Managed Node Group

### EKS → test-eks → Compute → Add node group

#### Node Group Configuration

* Name: `test-eks-ng`
* Node IAM role: `eks-node-role`

#### Networking

* Subnets:

  * `private-subnet-1`
  * `private-subnet-2`
* Security group:

  * `eks-node-sg`

#### Scaling

* Desired: 2
* Min: 2
* Max: 4

#### Instance Type

* `t3.medium` (test)
* AMI: Amazon Linux 2

Create node group → wait until **ACTIVE**

---

## Step 5 – Validate EKS Networking

From a bastion or VPN-connected machine:

```bash
aws eks update-kubeconfig --name test-eks --region <region>
kubectl get nodes
```

✅ Nodes should be **Ready**
✅ No public IPs on nodes
✅ Image pulls work without NAT dependency

---

# 3️⃣ Prepare MSK Networking Alongside EKS

MSK is **network-heavy**, so getting this right now saves major pain later.

---

## MSK Networking Principles

![Image](https://docs.aws.amazon.com/images/lambda/latest/dg/images/MSK-esm-network-overview.png)

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/05/19/bdb1186-msk-connectivity-patterns001.png)

* MSK **must be in private subnets**
* No internet access required
* Kafka traffic is **east–west** (inside VPC)
* EKS producers/consumers talk to MSK privately

---

## Step 6 – Create MSK Security Group

### Security Group: `msk-broker-sg`

**Inbound Rules**

| Port | Source                              |
| ---- | ----------------------------------- |
| 9092 | `eks-node-sg`                       |
| 9092 | `msk-broker-sg` (self)              |
| 2181 | `msk-broker-sg` (if ZooKeeper used) |

**Outbound**

* All traffic → `0.0.0.0/0`

---

## Step 7 – Verify Subnets for MSK

MSK requires:

* At least **2 subnets in different AZs**
* Same ones used by EKS is OK

You will use:

* `private-subnet-1`
* `private-subnet-2`

No route changes needed.

---

## Step 8 – (Optional but Recommended) Add MSK-Specific Endpoints

If you want **fully private AWS API access** for MSK operations:

Add Interface Endpoints:

* `kafka`
* `kafka-cluster`

(These are optional for test, recommended for prod.)

---

## Step 9 – Traffic Flow (EKS ↔ MSK)

**Producer / Consumer Path**

```
Pod → Node ENI → VPC → MSK Broker
```

No:

* NAT
* Internet Gateway
* Public DNS

Latency stays low, security stays high.

---

# Final State Checklist (Everything Done Right)

| Area                         | Status |
| ---------------------------- | ------ |
| VPC multi-AZ                 | ✅      |
| Public / Private subnets     | ✅      |
| NAT + IGW                    | ✅      |
| VPC Endpoints                | ✅      |
| EKS private cluster          | ✅      |
| EKS node groups              | ✅      |
| MSK-ready private networking | ✅      |
| Zero public exposure         | ✅      |
| Event-driven ready           | ✅      |

---

## What You Can Do Next (Natural Progression)

1️⃣ Install **AWS Load Balancer Controller**
2️⃣ Enable **IRSA**
3️⃣ Create **MSK cluster**
4️⃣ Deploy sample Kafka producer/consumer on EKS
5️⃣ Add **observability** (Prometheus / CloudWatch)


