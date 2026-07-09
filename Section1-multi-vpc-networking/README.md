# Section1 - Multi account, Multi Region, and Multi VPC Networking

Hands-on lab covering advanced multi-VPC connectivity patterns: PrivateLink services, VPC endpoint architectures, and Transit Gateway (including multicast) across accounts and regions.

This README is the only document you need to follow, start to finish. The
full technical design and implementation plan live in
`docs/superpowers/specs/2026-07-05-section1-multi-vpc-networking-design.md` and
`docs/superpowers/plans/2026-07-05-section1-multi-vpc-networking.md` - those are reference material
for whoever implements the remaining code, not something you need to read.

## Step 0: Base VPCs, Subnets, and IAM Role

### Create the 3 VPCs
For each of the 3 VPCs below, repeat:
1. Open the **VPC console** → left sidebar **Your VPCs** → **Create VPC**.
2. Resources to create: select **VPC only**.
3. **Name tag**: enter the VPC name from the table below.
4. **IPv4 CIDR block**: select **IPv4 CIDR manual input**, enter the CIDR from the table below.
5. Leave **IPv6 CIDR block** as **No IPv6 CIDR block**.
6. **Tenancy**: Default.
7. Click **Create VPC**.
8. After creation, select the new VPC → **Actions** → **Edit VPC settings** → check **Enable DNS hostnames** → **Save** (required later for PrivateLink/Interface Endpoint private DNS names).

| VPC Name | IPv4 CIDR |
|---|---|
| `Service-VPC` | `10.0.0.0/16` |
| `Consumer-A-VPC` | `10.1.0.0/16` |
| `Consumer-B-VPC` | `10.2.0.0/16` |

### Create one subnet per VPC
For each VPC, repeat:
1. VPC console → left sidebar **Subnets** → **Create subnet**.
2. **VPC ID**: select the matching VPC.
3. **Subnet name**: enter the subnet name from the table below.
4. **Availability Zone**: `us-east-1a` (same AZ for all 3, keeps the lab single-AZ).
5. **IPv4 subnet CIDR block**: enter the CIDR from the table below.
6. Click **Create subnet**.

| VPC Name | Subnet Name | Subnet CIDR |
|---|---|---|
| `Service-VPC` | `Service-Subnet` | `10.0.1.0/24` |
| `Consumer-A-VPC` | `Consumer-A-Subnet` | `10.1.1.0/24` |
| `Consumer-B-VPC` | `Consumer-B-Subnet` | `10.2.1.0/24` |

### Create the shared IAM role for SSM
1. Open the **IAM console** → left sidebar **Roles** → **Create role**.
2. **Trusted entity type**: AWS service.
3. **Use case**: EC2 → **Next**.
4. In the policy search box, type `AmazonSSMManagedInstanceCore`, check the box next to it → **Next**.
5. **Role name**: `EC2-SSM-Role`.
6. Click **Create role**.

### Launch the 3 EC2 instances
For each instance, repeat:
1. Open the **EC2 console** → **Instances** → **Launch instances**.
2. **Name**: enter the instance name from the table below.
3. **Application and OS Images**: Amazon Linux → **Amazon Linux 2023 AMI** (default, free-tier eligible).
4. **Instance type**: `t3.micro`.
5. **Key pair (login)**: select **Proceed without a key pair (Not recommended)** — this lab uses SSM only, no SSH.
6. **Network settings** → click **Edit**:
   - **VPC**: select the matching VPC.
   - **Subnet**: select the matching subnet.
   - **Auto-assign public IP**: **Disable**.
   - **Firewall (security groups)**: **Create security group**, name it `<InstanceName>-SG`, leave the default outbound rule (All traffic, 0.0.0.0/0) and **remove/leave empty all inbound rules**.
7. **Advanced details** → **IAM instance profile**: select `EC2-SSM-Role`.
8. Click **Launch instance**.

| VPC Name | Instance Name |
|---|---|
| `Service-VPC` | `Service-Backend` |
| `Consumer-A-VPC` | `Consumer-A-Test` |
| `Consumer-B-VPC` | `Consumer-B-Test` |

**Note:** these instances will NOT be reachable via Session Manager yet —
SSM Interface Endpoints don't exist until Step 2. That's expected.

**Captured values:**
_Pending — fill in after you complete the steps above._

## Progress

| # | Step | Status |
|---|------|--------|
| 1 | Using AWS PrivateLink for Services | Not started |
| 2 | Advanced VPC Endpoint Architectures | Not started |
| 3 | Advanced Transit Gateway Concepts | Not started |
| 4 | Multicast and Transit Gateways | Not started |
| 5 | Advanced Transit Gateway Architectures | Not started |

**Captured values so far:**
_None yet._

## Gotchas

_None yet._
