# Section1 - Multi account, Multi Region, and Multi VPC Networking

Hands-on lab covering advanced multi-VPC connectivity patterns: PrivateLink services, VPC endpoint architectures, VPC Lattice service networks, and Transit Gateway (including multicast) across accounts and regions.

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
8. After creation, select the new VPC → **Actions** → **Edit VPC settings** → check **Enable DNS resolution** and **Enable DNS hostnames** → **Save** (required later for Interface Endpoint Private DNS resolution).

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

## Step 1: Using AWS PrivateLink for Services

**Execution order after Step 0:**
1. Complete the PrivateLink infrastructure in this section.
2. Complete Step 2 so Session Manager has a working path.
3. Return to the bottom of this step and finish the live test.

Because the lab forbids public IPs, NAT Gateways, Internet Gateways, and SSH, the final Session Manager commands in this step must wait until the SSM Interface Endpoints from Step 2 exist in `Service-VPC` and `Consumer-A-VPC`.

### Create the internal Network Load Balancer
1. Open the **EC2 console** → left sidebar **Load Balancers** → **Create load balancer** → **Network Load Balancer** → **Create**.
2. **Load balancer name**: `Service-NLB`.
3. **Scheme**: **Internal**.
4. **VPC**: select `Service-VPC`.
5. **Mappings**: check `us-east-1a`, select `Service-Subnet`.
6. Under **Listeners and routing**: Listener protocol/port `TCP` / `80` → **Default action** → **Create target group**:
   - **Target type**: Instances.
   - **Target group name**: `Service-TG`.
   - **Protocol/Port**: `TCP` / `80`.
   - **VPC**: select `Service-VPC`.
   - Click **Next**, select the `Service-Backend` instance, click **Include as pending below**, then **Create target group**.
7. Back on the load balancer creation page, select `Service-TG` as the listener's target group.
8. Click **Create load balancer**.

### Allow the NLB to receive the PrivateLink traffic
If `Service-NLB` has a security group attached, make sure it allows the consumer VPC to reach the listener port. A default security group with only the self-referencing inbound rule is not enough for this lab.

1. Open the **EC2 console** → left sidebar **Load Balancers** → select `Service-NLB`.
2. Open the **Security** tab → click the linked security group.
3. Open the **Inbound rules** tab → **Edit inbound rules**.
4. Add a rule:
   - **Type**: `HTTP`
   - **Protocol**: `TCP`
   - **Port range**: `80`
   - **Source**: `10.1.0.0/16`
5. Click **Save rules**.

### Allow the backend instance to receive the NLB traffic
The `Service-Backend` instance was launched in Step 0 with no inbound rules. Before the PrivateLink path can work, open the backend instance security group for HTTP. In this lab, keep it explicit and allow both the provider-side VPC CIDR and the consumer VPC CIDR so the NLB health checks and the consumer-side PrivateLink traffic can both reach the backend cleanly.

1. Open the **EC2 console** → **Instances** → select `Service-Backend`.
2. Open the **Security** tab → click the linked security group (for example, `Service-Backend-SG`).
3. Open the **Inbound rules** tab → **Edit inbound rules**.
4. Add a rule:
   - **Type**: `HTTP`
   - **Protocol**: `TCP`
   - **Port range**: `80`
   - **Source**: `10.0.0.0/16`
5. Add a second rule:
   - **Type**: `HTTP`
   - **Protocol**: `TCP`
   - **Port range**: `80`
   - **Source**: `10.1.0.0/16`
6. Click **Save rules**.

### Confirm the target group is healthy
1. Open the **EC2 console** → left sidebar **Target Groups** → select `Service-TG`.
2. Open the **Targets** tab.
3. Confirm `Service-Backend` becomes `healthy` before continuing. If it is `unhealthy`, re-check the port `80` inbound rules above and confirm the Python HTTP server is running on `Service-Backend`.

### Create the VPC Endpoint Service
1. Open the **VPC console** → left sidebar **Endpoint services** → **Create endpoint service**.
2. **Load balancer type**: **Network**.
3. **Available load balancers**: select `Service-NLB`.
4. **Require acceptance for endpoint**: leave this **checked**.
5. Click **Create**.
6. Copy the new endpoint service **Service name** (it will look like `com.amazonaws.vpce.us-east-1.vpce-svc-...`) — this is a captured value.

### Create the consumer Interface Endpoint in Consumer-A-VPC
1. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
2. **Name tag**: `Consumer-A-Endpoint`.
3. **Service category**: **Other endpoint services**.
4. **Service name**: paste the endpoint service name you copied above → click **Verify service**.
5. **VPC**: select `Consumer-A-VPC`.
6. **Subnets**: check `us-east-1a`, then select `Consumer-A-Subnet`.
7. **Private DNS name**: leave this **disabled** for this lab. If the console says `Private DNS can't be enabled because the service has not verified the private DNS name`, that is expected for this custom endpoint service and not an error.
8. **Security groups**: create or select a security group that allows inbound `TCP` `80` from `10.1.0.0/16`.
9. Click **Create endpoint**.

### Accept the endpoint connection from the provider side
1. Open the **VPC console** → left sidebar **Endpoint services** → select your endpoint service.
2. Open the **Endpoint connections** tab.
3. Select the pending request from `Consumer-A-Endpoint`.
4. Click **Actions** → **Accept endpoint connection**.
5. Return to **Endpoints**, open `Consumer-A-Endpoint`, and wait until the state is **Available**.
6. Copy the first generated DNS name shown for the endpoint — this is a captured value and the hostname you will later use for `curl`.

### After you complete Step 2, return here and finish the live test
#### Start the backend HTTP responder
**If you see `Ping status: Offline` or an SSM timeout to `https://ssm.us-east-1.amazonaws.com/`, stop here and complete Step 2 first.** In this lab, that error means the instance still has no path to Systems Manager because the required VPC Interface Endpoints for `ssm`, `ssmmessages`, and `ec2messages` do not exist yet in `Service-VPC`.

1. In the **EC2 console**, select `Service-Backend` → **Connect** → **Session Manager** tab → **Connect**.
2. In the Session Manager terminal, run:
   ```bash
   sudo python3 -m http.server 80 &
   ```
3. Leave this running in the background for the rest of the lab.

#### Verify (live)
1. In the **EC2 console**, select `Consumer-A-Test` → **Connect** → **Session Manager** → **Connect**.
2. Run:
   ```bash
   curl -m 5 http://<consumer-a-endpoint-dns-name>
   ```
3. Expected result: an HTML directory listing produced by Python's `http.server`, such as a page titled `Directory listing for /`.
4. If the command times out but `nslookup <consumer-a-endpoint-dns-name>` resolves to an IP in `10.1.1.0/24`, DNS is working and the failure is on the PrivateLink data path. Check these in order:
   - The security group attached to `Consumer-A-Endpoint` allows inbound `TCP` `80` from `10.1.0.0/16`.
   - The security group attached to `Service-NLB` allows inbound `TCP` `80` from `10.1.0.0/16`.
   - `Consumer-A-Endpoint` is in state `Available`.
   - In **Endpoint services** → your service → **Endpoint connections**, the connection for `Consumer-A-Endpoint` is `Accepted`.
   - `Service-TG` still shows `Service-Backend` as `healthy`.

**Captured values:**
_Pending — fill in after you complete the steps above._

## Step 2: Advanced VPC Endpoint Architectures

**Complete this step after the infrastructure part of Step 1.** This step creates the Systems Manager network path that the Step 1 live test depends on.

### Add SSM Interface Endpoints to all 3 VPCs
For each of the 3 VPCs, repeat this 3 times (once each for `ssm`, `ssmmessages`, and `ec2messages`):
1. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
2. **Name tag**: `<VPCName>-<service>-endpoint` (for example, `Service-VPC-ssm-endpoint`).
3. **Service category**: **AWS services**.
4. **Services** search box: type the service name (`ssm`, `ssmmessages`, or `ec2messages`), then select `com.amazonaws.us-east-1.<service>`.
5. **VPC**: select the current VPC.
6. **Subnets**: check `us-east-1a`, then select the matching subnet.
7. **Enable DNS name**: leave this checked.
8. **Security group**: create or select one that allows inbound `TCP` `443` from the VPC's own CIDR.
9. Click **Create endpoint**.

That is 9 Interface Endpoints total: 3 services × 3 VPCs.

### Add a restricted S3 Gateway Endpoint in Consumer-A-VPC
1. Open the **S3 console** → **Create bucket** and create two test buckets:
   - one allowed bucket, such as `<your-name>-privatelink-lab-allowed`
   - one denied bucket, such as `<your-name>-privatelink-lab-denied`
2. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
3. **Name tag**: `Consumer-A-S3-Gateway-Endpoint`.
4. **Service category**: **AWS services**.
5. **Services** search box: type `s3`, then select the **Gateway** entry `com.amazonaws.us-east-1.s3`.
6. **VPC**: select `Consumer-A-VPC`.
7. **Route tables**: check the route table associated with `Consumer-A-Subnet`.
8. **Policy**: select **Custom** and paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": "*",
         "Action": ["s3:GetObject", "s3:ListBucket"],
         "Resource": [
           "arn:aws:s3:::<your-name>-privatelink-lab-allowed",
           "arn:aws:s3:::<your-name>-privatelink-lab-allowed/*"
         ]
       }
     ]
   }
   ```
9. Click **Create endpoint**.

### Add S3 read permissions to the EC2-SSM-Role
The VPC endpoint policy only restricts which buckets can be reached through the endpoint. It does **not** grant S3 permissions by itself. Add an IAM policy to the instance role first so the later `AccessDenied` on the denied bucket is caused by the endpoint policy rather than missing IAM permissions.

1. Open the **IAM console** → left sidebar **Roles** → select `EC2-SSM-Role`.
2. Open the **Permissions** tab → **Add permissions** → **Create inline policy**.
3. Switch to the **JSON** editor and paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowReadToLabBuckets",
         "Effect": "Allow",
         "Action": ["s3:ListBucket"],
         "Resource": [
           "arn:aws:s3:::<your-name>-privatelink-lab-allowed",
           "arn:aws:s3:::<your-name>-privatelink-lab-denied"
         ]
       },
       {
         "Sid": "AllowObjectReadToLabBuckets",
         "Effect": "Allow",
         "Action": ["s3:GetObject"],
         "Resource": [
           "arn:aws:s3:::<your-name>-privatelink-lab-allowed/*",
           "arn:aws:s3:::<your-name>-privatelink-lab-denied/*"
         ]
       }
     ]
   }
   ```
4. Click **Next**.
5. **Policy name**: `LabS3ReadPolicy`.
6. Click **Create policy**.

### Verify (live)
1. In the **EC2 console**, select `Consumer-A-Test` → **Connect** → **Session Manager** → **Connect**.
2. If Session Manager is still offline, wait 1-2 minutes for the new endpoints to register, then retry.
3. If it is still offline after that, verify all of the following before debugging anything else:
   - `Consumer-A-VPC` has **Enable DNS resolution** and **Enable DNS hostnames** turned on in **Actions** → **Edit VPC settings**.
   - The `ssm`, `ssmmessages`, and `ec2messages` Interface Endpoints in `Consumer-A-VPC` all have **Private DNS** enabled.
   - The security group attached to those three endpoints allows inbound `TCP` `443` from `10.1.0.0/16`.
   - The instance is using the `EC2-SSM-Role` instance profile.
4. Run:
   ```bash
   aws s3 ls s3://<your-name>-privatelink-lab-allowed --region us-east-1
   ```
5. Expected result: an empty bucket listing with exit code `0`.
6. Run:
   ```bash
   aws s3 ls s3://<your-name>-privatelink-lab-denied --region us-east-1
   ```
7. Expected result: an explicit `AccessDenied` error caused by the endpoint policy.
8. After this verification succeeds, return to the bottom of Step 1 and finish the PrivateLink live test.

**Captured values:**
_Pending — fill in after you complete the steps above._

## Step 3: VPC Lattice Services and Service Networks

This step builds the same "consumer VPC calls a service in another VPC" pattern again, but with VPC Lattice instead of a PrivateLink endpoint service. VPC Lattice introduces a service network: clients in VPCs associated with the service network can discover and call services associated with that network.

In this lab:

| Component | Role |
|---|---|
| `Service-Backend` | Existing HTTP backend target in `Service-VPC` |
| `Service-Lattice-TG` | VPC Lattice target group pointing to `Service-Backend` |
| `service-backend-lattice` | VPC Lattice service with an HTTP listener on port `80` |
| `Lab-Service-Network` | VPC Lattice service network |
| `Consumer-A-VPC` | Client VPC associated with the service network |

### Allow VPC Lattice to reach the backend target
VPC Lattice traffic to targets comes from the VPC Lattice managed prefix list, not directly from the client security group. Add that prefix list to the backend instance security group.

1. Open the **VPC console** -> left sidebar **Managed Prefix Lists**.
2. In the search box, search for `com.amazonaws.us-east-1.vpc-lattice`.
3. Copy the prefix list ID for `com.amazonaws.us-east-1.vpc-lattice`.
4. Open the **EC2 console** -> **Instances** -> select `Service-Backend`.
5. Open the **Security** tab -> click `Service-Backend-SG`.
6. Open **Inbound rules** -> **Edit inbound rules**.
7. Add a rule:
   - **Type**: `HTTP`
   - **Protocol**: `TCP`
   - **Port range**: `80`
   - **Source**: select the VPC Lattice managed prefix list (`com.amazonaws.us-east-1.vpc-lattice`)
8. Click **Save rules**.

### Create a VPC Lattice target group
1. Open the **VPC console** -> left sidebar under **VPC Lattice** -> **Target groups** -> **Create target group**.
2. **Target group name**: `Service-Lattice-TG`.
3. **Target type**: `Instances`.
4. **Protocol**: `HTTP`.
5. **Port**: `80`.
6. **VPC**: select `Service-VPC`.
7. **Protocol version**: `HTTP1`.
8. **Health checks**: leave enabled.
9. **Health check protocol**: `HTTP`.
10. **Health check path**: `/`.
11. Click **Next**.
12. **Available instances**: select `Service-Backend`.
13. **Port**: `80`.
14. Click **Include as pending below**.
15. Click **Create target group**.
16. Wait for `Service-Backend` to become `Healthy`.

### Create a VPC Lattice service network
1. Open the **VPC console** -> left sidebar under **VPC Lattice** -> **Service networks** -> **Create service network**.
2. **Name**: `Lab-Service-Network`.
3. **Auth type**: `None`.
4. Leave monitoring and tags at defaults for this lab.
5. Click **Create service network**.

### Associate Consumer-A-VPC with the service network
1. Open **VPC Lattice** -> **Service networks** -> select `Lab-Service-Network`.
2. Open the **VPC associations** tab.
3. Click **Create VPC associations** -> **Add VPC association**.
4. **VPC**: select `Consumer-A-VPC`.
5. **Security groups**: create or select a security group in `Consumer-A-VPC` that allows inbound `TCP` `80` from `10.1.0.0/16`.
6. Click **Save changes**.
7. Wait for the VPC association status to become `Active`.

Only `Consumer-A-VPC` is associated in this step. `Consumer-B-VPC` is deliberately left out so it cannot call the Lattice service.

### Create the VPC Lattice service
1. Open the **VPC console** -> left sidebar under **VPC Lattice** -> **Services** -> **Create service**.
2. **Name**: `service-backend-lattice`.
3. **Service access**: select `None`.
4. Click **Next** to define routing.
5. Click **Add listener**.
6. **Listener name**: `http-80`.
7. **Protocol**: `HTTP`.
8. **Port**: `80`.
9. **Default action**: forward to `Service-Lattice-TG`.
10. Click **Next** to create network associations.
11. **VPC Lattice service networks**: select `Lab-Service-Network`.
12. Click **Next**.
13. Review the settings.
14. Click **Create VPC Lattice service**.
15. After creation, copy the generated service domain name. It should look similar to `<service-id>.<partition-id>.vpc-lattice-svcs.us-east-1.on.aws`.

### Verify (live)
1. Confirm the Python HTTP server is still running on `Service-Backend`:
   ```bash
   sudo python3 -m http.server 80 &
   ```
   If it says the address is already in use, that is fine; the server is already running.
2. Connect to `Consumer-A-Test` via **Session Manager**.
3. Run:
   ```bash
   curl -m 5 http://<vpc-lattice-service-domain-name>
   ```
4. Expected result: an HTML directory listing produced by Python's `http.server`.
5. If the command times out, check these in order:
   - `Service-Lattice-TG` shows `Service-Backend` as `Healthy`.
   - `Lab-Service-Network` has an `Active` VPC association for `Consumer-A-VPC`.
   - The security group attached to the `Consumer-A-VPC` service-network association allows inbound `TCP` `80` from `10.1.0.0/16`.
   - The `Consumer-A-Test` security group outbound rules allow traffic to the VPC Lattice managed prefix list on `TCP` `80`; the default allow-all outbound rule is sufficient.
   - `Service-Backend-SG` allows inbound `TCP` `80` from the VPC Lattice managed prefix list `com.amazonaws.us-east-1.vpc-lattice`.
   - The VPC Lattice service is associated with `Lab-Service-Network`.
   - The VPC Lattice service listener on port `80` forwards to `Service-Lattice-TG`.

### What this proves
This confirms that `Consumer-A-Test` can reach `Service-Backend` through a VPC Lattice service network. Unlike the Transit Gateway step, you did not add route-table entries between `Consumer-A-VPC` and `Service-VPC`. Unlike the PrivateLink step, you did not create a consumer Interface Endpoint for a custom endpoint service. VPC Lattice provides the service-network layer between the client VPC and the service.

**Captured values:**
_Pending — fill in after you complete the steps above._

## Step 4: Advanced Transit Gateway Concepts

This step builds segmented hub-and-spoke routing with one Transit Gateway. `Consumer-A-VPC` and `Consumer-B-VPC` can each reach `Service-VPC`, but they intentionally cannot reach each other over normal unicast routing.

### Create the Transit Gateway
1. Open the **VPC console** -> left sidebar **Transit Gateways** -> **Create Transit Gateway**.
2. **Name tag**: `Lab-TGW`.
3. **Amazon side ASN**: leave default (`64512`).
4. **Auto accept shared attachments**: leave disabled.
5. **Default route table association**: leave enabled.
6. **Default route table propagation**: leave enabled.
7. Click **Create Transit Gateway**.
8. Wait for **State** to become `Available`.

### Attach all 3 VPCs
For each VPC, repeat:
1. Open the **VPC console** -> left sidebar **Transit Gateway Attachments** -> **Create Transit Gateway Attachment**.
2. **Transit Gateway ID**: select `Lab-TGW`.
3. **Attachment type**: `VPC`.
4. **VPC ID**: select the current VPC.
5. **Subnet IDs**: select the matching subnet in `us-east-1a`.
6. **Name tag**: `<VPCName>-Attachment` (for example, `Service-VPC-Attachment`).
7. Click **Create Transit Gateway Attachment**.
8. Wait for the attachment state to become `Available`.

Create these 3 attachments:

| VPC | Attachment Name | Subnet |
|---|---|---|
| `Service-VPC` | `Service-VPC-Attachment` | `Service-Subnet` |
| `Consumer-A-VPC` | `Consumer-A-VPC-Attachment` | `Consumer-A-Subnet` |
| `Consumer-B-VPC` | `Consumer-B-VPC-Attachment` | `Consumer-B-Subnet` |

### Create 3 custom TGW route tables
For each route table, repeat:
1. Open the **VPC console** -> left sidebar **Transit Gateway Route Tables** -> **Create Transit Gateway route table**.
2. **Name tag**: enter the route table name from the table below.
3. **Transit Gateway ID**: select `Lab-TGW`.
4. Click **Create Transit Gateway route table**.

| TGW Route Table | Purpose |
|---|---|
| `RT-ConsumerA` | Associated with `Consumer-A-VPC-Attachment`; only learns `Service-VPC` |
| `RT-ConsumerB` | Associated with `Consumer-B-VPC-Attachment`; only learns `Service-VPC` |
| `RT-Shared` | Associated with `Service-VPC-Attachment`; learns both consumer VPCs |

### Remove the automatic default TGW route table associations
Because `Lab-TGW` was created with default route table association enabled, AWS automatically associated each new VPC attachment with the default TGW route table. A TGW attachment can be associated with only one TGW route table at a time, so remove those automatic associations before creating the custom ones below.

1. Open the **VPC console** -> left sidebar **Transit Gateway Route Tables**.
2. Select the default route table for `Lab-TGW`. It is the one with **Default association route table** set to **Yes**.
3. Open the **Associations** tab.
4. Select `Service-VPC-Attachment` -> **Delete association**.
5. Select `Consumer-A-VPC-Attachment` -> **Delete association**.
6. Select `Consumer-B-VPC-Attachment` -> **Delete association**.
7. Wait until those associations disappear before continuing.

If the console says `Transit Gateway Attachment ... is already associated to a route table`, this is the step that was missed.

### Set TGW route table associations
1. Select `RT-ConsumerA` -> **Associations** tab -> **Create association**.
2. **Choose attachment to associate**: select `Consumer-A-VPC-Attachment`.
3. Click **Create association**.
4. Select `RT-ConsumerB` -> **Associations** tab -> **Create association**.
5. **Choose attachment to associate**: select `Consumer-B-VPC-Attachment`.
6. Click **Create association**.
7. Select `RT-Shared` -> **Associations** tab -> **Create association**.
8. **Choose attachment to associate**: select `Service-VPC-Attachment`.
9. Click **Create association**.

### Set TGW route table propagations
1. Select `RT-ConsumerA` -> **Propagations** tab -> **Create propagation**.
2. **Choose attachment to propagate**: select `Service-VPC-Attachment`.
3. Click **Create propagation**.
4. Select `RT-ConsumerB` -> **Propagations** tab -> **Create propagation**.
5. **Choose attachment to propagate**: select `Service-VPC-Attachment`.
6. Click **Create propagation**.
7. Select `RT-Shared` -> **Propagations** tab -> **Create propagation**.
8. **Choose attachment to propagate**: select `Consumer-A-VPC-Attachment`.
9. Click **Create propagation**.
10. Still on `RT-Shared`, create another propagation for `Consumer-B-VPC-Attachment`.

Do not propagate `Consumer-B-VPC-Attachment` into `RT-ConsumerA`, and do not propagate `Consumer-A-VPC-Attachment` into `RT-ConsumerB`. That omission is the isolation control this step is testing.

### Update each VPC subnet route table
For each VPC route table, repeat:
1. Open the **VPC console** -> left sidebar **Route Tables**.
2. Select the route table associated with the VPC's lab subnet.
3. Open the **Routes** tab -> **Edit routes** -> **Add route**.
4. Add the routes from the table below.
5. Click **Save changes**.

| Route Table | Destination | Target |
|---|---|---|
| `Service-VPC` subnet route table | `10.1.0.0/16` | `Lab-TGW` |
| `Service-VPC` subnet route table | `10.2.0.0/16` | `Lab-TGW` |
| `Consumer-A-VPC` subnet route table | `10.0.0.0/16` | `Lab-TGW` |
| `Consumer-B-VPC` subnet route table | `10.0.0.0/16` | `Lab-TGW` |

Do not add a `10.2.0.0/16` route to `Consumer-A-VPC`, and do not add a `10.1.0.0/16` route to `Consumer-B-VPC`.

### Allow ICMP for the TGW verification
The instances were launched with no inbound rules. Add only the ICMP rules needed for this verification.

1. Open the **EC2 console** -> **Instances** -> select `Service-Backend`.
2. Open the **Security** tab -> click `Service-Backend-SG`.
3. Open **Inbound rules** -> **Edit inbound rules**.
4. Add a rule:
   - **Type**: `All ICMP - IPv4`
   - **Protocol**: `ICMP`
   - **Port range**: `All`
   - **Source**: `10.1.0.0/16`
5. Click **Save rules**.
6. Open `Consumer-B-Test` -> **Security** tab -> click `Consumer-B-Test-SG`.
7. Open **Inbound rules** -> **Edit inbound rules**.
8. Add a rule:
   - **Type**: `All ICMP - IPv4`
   - **Protocol**: `ICMP`
   - **Port range**: `All`
   - **Source**: `10.1.0.0/16`
9. Click **Save rules**.

The second rule intentionally allows the ping at the security group layer. The expected failure later should come from TGW route-table isolation, not from a security group block.

### Verify (live)
1. In the **EC2 console**, copy the private IPv4 addresses for `Service-Backend` and `Consumer-B-Test`.
2. Connect to `Consumer-A-Test` via **Session Manager**.
3. Run:
   ```bash
   ping -c 3 <service-backend-private-ip>
   ```
4. Expected result: `3 packets transmitted, 3 received, 0% packet loss`. This proves `Consumer-A-VPC` can reach `Service-VPC` through `Lab-TGW`.
5. Run:
   ```bash
   ping -c 3 -W 2 <consumer-b-test-private-ip>
   ```
6. Expected result: `3 packets transmitted, 0 received, 100% packet loss`. This proves `Consumer-A-VPC` cannot reach `Consumer-B-VPC` through ordinary unicast routing.

**Captured values:**
_Pending — fill in after you complete the steps above._

## Progress

| # | Step | Status |
|---|------|--------|
| 1 | Using AWS PrivateLink for Services | Not started |
| 2 | Advanced VPC Endpoint Architectures | Not started |
| 3 | VPC Lattice Services and Service Networks | Not started |
| 4 | Advanced Transit Gateway Concepts | Not started |
| 5 | Multicast and Transit Gateways | Not started |
| 6 | Advanced Transit Gateway Architectures | Not started |

**Captured values so far:**
_None yet._

## Gotchas

- If Session Manager shows `Ping status: Offline` and the latest error mentions a timeout calling `https://ssm.us-east-1.amazonaws.com/`, the instance does not yet have a working path to Systems Manager. In this lab that is expected until Step 2 creates the `ssm`, `ssmmessages`, and `ec2messages` Interface Endpoints in the relevant VPCs. Do not debug SSH or IAM first; complete Step 2, then retry.
- If Session Manager is still offline after Step 2 and the error shows a timeout to a public `ssm.us-east-1.amazonaws.com` IP address, the instance is still resolving SSM publicly instead of through the VPC endpoints. In this lab that usually means one of these is wrong in the instance's VPC: **Enable DNS resolution** is off, **Enable DNS hostnames** is off, or **Private DNS** was not enabled on one or more of the `ssm`, `ssmmessages`, or `ec2messages` Interface Endpoints.
- If `aws s3 ls s3://<allowed-bucket>` fails with `because no identity-based policy allows the s3:ListBucket action`, the instance role is missing S3 permissions. Add the inline `LabS3ReadPolicy` to `EC2-SSM-Role`, then retry. The gateway endpoint policy restricts access; it does not grant it.
