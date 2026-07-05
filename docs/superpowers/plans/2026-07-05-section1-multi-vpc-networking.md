# Section1 Multi-VPC Networking Lab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans
> to implement this plan task-by-task (NOT subagent-driven-development — this
> plan requires a human to physically click through the AWS Console between
> most steps, which a freshly dispatched subagent cannot do; keep it in one
> session with the user).

**Goal:** Build and document a fully hands-on, console-clickops AWS lab
covering PrivateLink, advanced VPC endpoints, Transit Gateway segmentation,
and TGW multicast, as a single shared 3-VPC topology.

**Architecture:** One Transit Gateway hub with `Service-VPC`,
`Consumer-A-VPC`, `Consumer-B-VPC` as spokes. No NAT/IGW/bastion — SSM
Session Manager only. Because provisioning is 100% AWS Console clickops (per
spec), **the "implementer" of each task is the user, not an agent.** Each
task below writes a README section with literal, numbered console
instructions (console name → button label → field name, in order) and an
explicit expected verification output, then hands off to the user to
perform it in their real AWS account and report back the actual IDs/output.
Record those as "Captured values" before marking a task done. Do not mark a
task complete until the user has confirmed the live verification output
matches (or you've reconciled a mismatch).

**Tech Stack:** AWS Console (VPC, EC2, IAM, Transit Gateway, Systems
Manager), Amazon Linux 2023, Python 3 (backend HTTP responder, multicast
sender/listener — both stdlib only, no install step needed).

## Global Constraints

- Single AWS account, single region (use `us-east-1` in all instructions
  below; note in the README this can be swapped for any region with TGW
  multicast support).
- 100% console clickops — no Terraform, no CLI resource creation. Every
  console action must be written as literal steps: console name, button
  label, field name, in that order.
- No NAT Gateway, no Internet Gateway, no bastion host anywhere.
- SSM Session Manager is the only access method to any EC2 instance.
- Single AZ per VPC (keep the lab simple, per approved spec).
- Cost callouts and a final teardown section are mandatory in the README.
- Every verification step must be a **live check against real deployed
  resources** — an actual command run and its actual output reported back,
  never assumed.

---

### Task 0: Base VPCs, subnets, and shared IAM role

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 0: Base VPCs, Subnets, and IAM Role` section before the Progress table's first row content)

**Interfaces:**
- Produces: VPC IDs and subnet IDs for `Service-VPC` (`10.0.0.0/16` /
  `10.0.1.0/24`), `Consumer-A-VPC` (`10.1.0.0/16` / `10.1.1.0/24`),
  `Consumer-B-VPC` (`10.2.0.0/16` / `10.2.1.0/24`); one IAM role
  `EC2-SSM-Role`; three EC2 instances (`Service-Backend`,
  `Consumer-A-Test`, `Consumer-B-Test`), all t3.micro, Amazon Linux 2023, no
  public IP, security group allowing all outbound (default) and no inbound
  rules. Later tasks reference these VPC/subnet/instance names directly.

- [ ] **Step 1: Write the README section with literal clickops for creating the 3 VPCs**

Add this to the README (fill in real IDs into "Captured values" once done):

```markdown
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
```

- [ ] **Step 2: Commit the README section**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 0 clickops instructions: base VPCs, subnets, IAM role, EC2s"
```

- [ ] **Step 3: Hand off to the user to build Step 0 in the AWS Console**

Ask the user to follow the README's Step 0 section in their AWS account and
report back:
- The 3 VPC IDs and 3 subnet IDs
- The IAM role ARN
- The 3 EC2 instance IDs, and confirmation all 3 show state `running` in
  the EC2 console

- [ ] **Step 4: Record captured values and verify**

Once the user reports back, add a `**Captured values:**` block under Step 0
in the README with the real VPC IDs, subnet IDs, role ARN, and instance
IDs. Confirm all 3 instances are `running`. If anything is missing or
errored, resolve it with the user before continuing — don't proceed to
Task 1 with incomplete base infrastructure.

- [ ] **Step 5: Commit captured values**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Record Step 0 captured values (VPC/subnet/role/instance IDs)"
```

---

### Task 1: Using AWS PrivateLink for Services

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 1: Using AWS PrivateLink for Services`, update Progress table row 1 to `Done`)

**Interfaces:**
- Consumes: `Service-Backend` instance ID (Task 0), `Service-Subnet` /
  `Consumer-A-Subnet` (Task 0).
- Produces: an internal NLB `Service-NLB`, a VPC Endpoint Service, and an
  Interface Endpoint in `Consumer-A-VPC` named `Consumer-A-Endpoint`, whose
  private DNS name later tasks don't depend on (this sub-topic is
  self-contained), but the pattern of "Interface Endpoint bypasses TGW" is
  referenced in Task 3's verification.

- [ ] **Step 1: Write the README section for the backend HTTP responder and NLB**

```markdown
## Step 1: Using AWS PrivateLink for Services

### Start the backend HTTP responder
1. In the **EC2 console**, select `Service-Backend` → **Connect** → **Session Manager** tab → **Connect**.
2. In the Session Manager terminal, run:
   ```bash
   sudo python3 -m http.server 80 &
   ```
3. Leave this running in the background for the rest of the lab.

### Create the internal Network Load Balancer
1. Open the **EC2 console** → left sidebar **Load Balancers** → **Create load balancer** → **Network Load Balancer** → **Create**.
2. **Load balancer name**: `Service-NLB`.
3. **Scheme**: **Internal**.
4. **VPC**: `Service-VPC`.
5. **Mappings**: check `us-east-1a`, select `Service-Subnet`.
6. Under **Listeners and routing**: Listener protocol/port `TCP` / `80` → **Default action** → **Create target group** (opens a new tab/flow):
   - **Target type**: Instances.
   - **Target group name**: `Service-TG`.
   - **Protocol/Port**: TCP / 80.
   - **VPC**: `Service-VPC`.
   - Click **Next**, select the `Service-Backend` instance, **Include as pending below**, **Create target group**.
7. Back on the load balancer creation page, select `Service-TG` as the listener's target group.
8. Click **Create load balancer**.

### Create the VPC Endpoint Service
1. Open the **VPC console** → left sidebar **Endpoint Services** → **Create endpoint service**.
2. **Load balancer type**: Network.
3. **Load balancers**: select `Service-NLB`.
4. **Require acceptance for endpoint**: leave **checked** (so we can demonstrate the accept step).
5. Click **Create**.
6. Note the **Service name** shown (starts with `com.amazonaws.vpce.us-east-1....`) — this is a captured value.

### Consume it from Consumer-A-VPC via an Interface Endpoint
1. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
2. **Name tag**: `Consumer-A-Endpoint`.
3. **Service category**: **Other endpoint services**.
4. **Service name**: paste the service name captured above → **Verify service**.
5. **VPC**: `Consumer-A-VPC`.
6. **Subnets**: check `us-east-1a`, select `Consumer-A-Subnet`.
7. **Security group**: create/select one that allows inbound TCP 80 from `10.1.0.0/16` (the Consumer-A-VPC CIDR).
8. Click **Create endpoint**.
9. Back in the **VPC console** → **Endpoint Services** → select your service → **Endpoint connections** tab → select the pending connection request from `Consumer-A-Endpoint` → **Actions** → **Accept endpoint connection**.
10. Wait for the Interface Endpoint's state to become `Available` (VPC console → Endpoints → `Consumer-A-Endpoint`).
11. Copy the endpoint's **DNS names** (first one shown) — this is a captured value.

### Verify (live)
1. In the **EC2 console**, select `Consumer-A-Test` → **Connect** → **Session Manager** → **Connect**.
2. Run:
   ```bash
   curl -m 5 http://<consumer-a-endpoint-dns-name>
   ```
3. **Expected output**: an HTML directory listing produced by Python's
   `http.server` (e.g. a page titled `Directory listing for /`). This
   confirms traffic reached the `Service-Backend` instance through
   PrivateLink — with no TGW attachment involved on either side.
```

- [ ] **Step 2: Commit the README section**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 1 clickops instructions: PrivateLink NLB, endpoint service, interface endpoint"
```

- [ ] **Step 3: Hand off to the user to build and verify Step 1**

Ask the user to follow the Step 1 README section and report back: the
Endpoint Service name, the Interface Endpoint's DNS name, and the actual
`curl` output from `Consumer-A-Test`.

- [ ] **Step 4: Record captured values, confirm verification, update progress**

Add the reported Endpoint Service name and Interface Endpoint DNS name to
a `**Captured values:**` block under Step 1. Confirm the `curl` output
matches the expected HTML directory listing. Update the Progress table row
"Using AWS PrivateLink for Services" to `Done`.

- [ ] **Step 5: Commit**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Record Step 1 captured values and mark PrivateLink sub-topic done"
```

---

### Task 2: Advanced VPC Endpoint Architectures

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 2: Advanced VPC Endpoint Architectures`, update Progress table row 2 to `Done`)

**Interfaces:**
- Consumes: all 3 VPC/subnet IDs (Task 0), all 3 EC2 instances (Task 0).
- Produces: SSM/SSMMessages/EC2Messages Interface Endpoints in all 3 VPCs
  (this is what every later task's "Connect via Session Manager" step
  depends on); a Gateway Endpoint for S3 in `Consumer-A-VPC` with a
  restrictive endpoint policy naming one test bucket.

- [ ] **Step 1: Write the README section for SSM endpoints and the restricted S3 gateway endpoint**

```markdown
## Step 2: Advanced VPC Endpoint Architectures

### Add SSM Interface Endpoints to all 3 VPCs
For each of the 3 VPCs, repeat this 3 times (once each for `ssm`,
`ssmmessages`, `ec2messages`):
1. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
2. **Name tag**: `<VPCName>-<service>-endpoint` (e.g. `Service-VPC-ssm-endpoint`).
3. **Service category**: **AWS services**.
4. **Services** search box: type the service name (`ssm`, `ssmmessages`, or `ec2messages`), select `com.amazonaws.us-east-1.<service>`.
5. **VPC**: the current VPC.
6. **Subnets**: check `us-east-1a`, select the matching subnet.
7. **Enable DNS name**: leave checked.
8. **Security group**: create/select one allowing inbound TCP 443 from the VPC's own CIDR.
9. Click **Create endpoint**.

(That's 9 Interface Endpoints total: 3 services × 3 VPCs.)

### Add a restricted S3 Gateway Endpoint in Consumer-A-VPC
1. First, in the **S3 console**, create two test buckets (**Create bucket**, accept defaults, uncheck nothing security-relevant): one you'll allow (e.g. `<your-name>-privatelink-lab-allowed`) and one you'll deny (e.g. `<your-name>-privatelink-lab-denied`). Bucket names must be globally unique.
2. Open the **VPC console** → left sidebar **Endpoints** → **Create endpoint**.
3. **Name tag**: `Consumer-A-S3-Gateway-Endpoint`.
4. **Service category**: **AWS services**.
5. **Services** search box: type `s3`, select the **Gateway** type entry `com.amazonaws.us-east-1.s3`.
6. **VPC**: `Consumer-A-VPC`.
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

### Verify (live)
1. Connect to `Consumer-A-Test` via **Session Manager** (this should now work — if it doesn't, wait 1-2 minutes for the SSM endpoints to register, then retry from the EC2 console's **Connect** button).
2. Run:
   ```bash
   aws s3 ls s3://<your-name>-privatelink-lab-allowed --region us-east-1
   ```
   **Expected output**: empty listing (bucket exists, no objects) with exit code 0 — no error.
3. Run:
   ```bash
   aws s3 ls s3://<your-name>-privatelink-lab-denied --region us-east-1
   ```
   **Expected output**: an explicit `AccessDenied` error, caused by the endpoint policy (not an IAM policy) blocking this bucket.
```

- [ ] **Step 2: Commit the README section**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 2 clickops instructions: SSM endpoints and restricted S3 gateway endpoint"
```

- [ ] **Step 3: Hand off to the user to build and verify Step 2**

Ask the user to follow the Step 2 README section and report back: the two
S3 bucket names they created, confirmation Session Manager now connects to
all 3 instances, and the actual output of both `aws s3 ls` commands.

- [ ] **Step 4: Record captured values, confirm verification, update progress**

Add the bucket names to `**Captured values:**` under Step 2. Confirm the
allowed bucket listed successfully and the denied bucket returned
`AccessDenied`. If the denied bucket did NOT fail, the endpoint policy is
wrong — debug with the user (check the policy JSON was pasted correctly)
before marking done. Update the Progress table row "Advanced VPC Endpoint
Architectures" to `Done`.

- [ ] **Step 5: Commit**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Record Step 2 captured values and mark VPC endpoint sub-topic done"
```

---

### Task 3: Advanced Transit Gateway Concepts

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 3: Advanced Transit Gateway Concepts`, update Progress table row 3 to `Done`)

**Interfaces:**
- Consumes: all 3 VPC/subnet IDs (Task 0), `Service-Backend` and
  `Consumer-B-Test` instance IDs (Task 0).
- Produces: one Transit Gateway `Lab-TGW`, 3 TGW VPC attachments, 3 TGW
  route tables (`RT-ConsumerA`, `RT-ConsumerB`, `RT-Shared`) with the
  associations/propagations from the spec. Task 4 (multicast) attaches to
  the same TGW and reuses the same attachments.

- [ ] **Step 1: Write the README section for the TGW, attachments, and route tables**

```markdown
## Step 3: Advanced Transit Gateway Concepts

### Create the Transit Gateway
1. Open the **VPC console** → left sidebar **Transit Gateways** → **Create Transit Gateway**.
2. **Name tag**: `Lab-TGW`.
3. **Amazon side ASN**: leave default (`64512`).
4. Under **Auto accept shared attachments**: leave disabled (not needed, single account).
5. Under **Default route table association / propagation**: leave both **enabled** at the transit gateway level for now — we will still create and use dedicated custom route tables below instead of the default one.
6. Click **Create Transit Gateway**. Wait for **State** to become `Available` (can take a few minutes) before continuing.

### Attach all 3 VPCs
For each VPC, repeat:
1. VPC console → left sidebar **Transit Gateway Attachments** → **Create Transit Gateway Attachment**.
2. **Transit Gateway ID**: `Lab-TGW`.
3. **Attachment type**: VPC.
4. **VPC ID**: the current VPC.
5. **Subnet IDs**: select the matching subnet (`us-east-1a`).
6. **Name tag**: `<VPCName>-Attachment` (e.g. `Service-VPC-Attachment`).
7. Click **Create Transit Gateway Attachment**.

### Create 3 custom TGW route tables
For each, repeat:
1. VPC console → left sidebar **Transit Gateway Route Tables** → **Create Transit Gateway route table**.
2. **Name tag**: `RT-ConsumerA`, `RT-ConsumerB`, or `RT-Shared`.
3. **Transit Gateway ID**: `Lab-TGW`.
4. Click **Create Transit Gateway route table**.

### Set associations and propagations
1. Select `RT-ConsumerA` → **Associations** tab → **Create association** → select the `Consumer-A-VPC-Attachment` → **Create association**.
2. Select `RT-ConsumerA` → **Propagations** tab → **Create propagation** → select the `Service-VPC-Attachment` → **Create propagation**. (This gives Consumer-A a route to Service-VPC only — no route to Consumer-B is ever added here, which is the isolation.)
3. Select `RT-ConsumerB` → **Associations** tab → **Create association** → select the `Consumer-B-VPC-Attachment` → **Create association**.
4. Select `RT-ConsumerB` → **Propagations** tab → **Create propagation** → select the `Service-VPC-Attachment` → **Create propagation**.
5. Select `RT-Shared` → **Associations** tab → **Create association** → select the `Service-VPC-Attachment` → **Create association**.
6. Select `RT-Shared` → **Propagations** tab → **Create propagation** → select `Consumer-A-VPC-Attachment` → **Create propagation**; repeat for `Consumer-B-VPC-Attachment`.

### Update each VPC's subnet route table
For each VPC, repeat:
1. VPC console → left sidebar **Route Tables** → select the route table associated with the VPC's subnet.
2. **Routes** tab → **Edit routes** → **Add route**.
3. **Destination**: the CIDR of the VPC(s) this one should reach over TGW —
   - `Service-VPC`'s subnet route table: add `10.1.0.0/16` → target `Lab-TGW`; add `10.2.0.0/16` → target `Lab-TGW`.
   - `Consumer-A-VPC`'s subnet route table: add `10.0.0.0/16` → target `Lab-TGW`.
   - `Consumer-B-VPC`'s subnet route table: add `10.0.0.0/16` → target `Lab-TGW`.
4. Click **Save changes**.

Note: Consumer-A's and Consumer-B's subnet route tables intentionally have
NO route to each other's CIDR (`10.2.0.0/16` and `10.1.0.0/16`
respectively) — this is what makes them unreachable from one another over
ordinary unicast routing.

### Verify (live)
1. Connect to `Consumer-A-Test` via Session Manager.
2. Run:
   ```bash
   ping -c 3 <service-backend-private-ip>
   ```
   **Expected output**: 3 packets transmitted, 3 received, 0% packet loss — proves Consumer-A reaches Service-VPC over TGW.
3. Run:
   ```bash
   ping -c 3 -W 2 <consumer-b-test-private-ip>
   ```
   **Expected output**: 3 packets transmitted, 0 received, 100% packet loss (times out) — proves Consumer-A cannot reach Consumer-B, confirming route table segmentation.
```

- [ ] **Step 2: Commit the README section**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 3 clickops instructions: TGW, attachments, segmented route tables"
```

- [ ] **Step 3: Hand off to the user to build and verify Step 3**

Ask the user to follow the Step 3 README section and report back: the TGW
ID, the 3 attachment IDs, the 3 route table IDs, the private IPs of
`Service-Backend` and `Consumer-B-Test`, and the actual output of both
`ping` commands.

- [ ] **Step 4: Record captured values, confirm verification, update progress**

Add the TGW ID, attachment IDs, route table IDs, and private IPs to
`**Captured values:**` under Step 3. Confirm the first ping succeeded and
the second timed out. If the second ping unexpectedly succeeds, there's a
stray route somewhere — debug with the user before marking done. Update
the Progress table row "Advanced Transit Gateway Concepts" to `Done`.

- [ ] **Step 5: Commit**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Record Step 3 captured values and mark TGW concepts sub-topic done"
```

---

### Task 4: Multicast and Transit Gateways

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 4: Multicast and Transit Gateways`, update Progress table row 4 to `Done`)

**Interfaces:**
- Consumes: `Lab-TGW` ID, `Consumer-A-VPC-Attachment` / `Consumer-B-VPC-Attachment` IDs, `Consumer-A-Test` / `Consumer-B-Test` instance IDs and their ENI IDs (Task 3).
- Produces: a TGW multicast domain with both Consumer test instances
  registered as group members of `224.0.1.1`.

- [ ] **Step 1: Write the README section for the multicast domain and traffic test**

```markdown
## Step 4: Multicast and Transit Gateways

### Create the TGW multicast domain
1. Open the **VPC console** → left sidebar **Transit Gateway Multicast** → **Create transit gateway multicast domain**.
2. **Name tag**: `Lab-Multicast-Domain`.
3. **Transit Gateway ID**: `Lab-TGW`.
4. **Igmpv2 support**: Disable (we'll register members manually below).
5. Click **Create transit gateway multicast domain**. Wait for **State** to become `Available`.

### Associate the Consumer-A and Consumer-B subnets
For each of Consumer-A and Consumer-B, repeat:
1. Select `Lab-Multicast-Domain` → **Associations** tab → **Create association**.
2. **Attachment ID**: the matching `Consumer-A-VPC-Attachment` or `Consumer-B-VPC-Attachment`.
3. **Subnet ID**: the matching subnet.
4. Click **Create association**.

### Register both test instances as multicast group members
For each of `Consumer-A-Test` and `Consumer-B-Test`, repeat:
1. First, in the **EC2 console**, open the instance's detail page → **Networking** tab → note its **Network interface ID** (ENI ID) — this is a captured value.
2. Back in **Transit Gateway Multicast** → `Lab-Multicast-Domain` → **Group members** tab → **Register group member**.
3. **Group IP address**: `224.0.1.1`.
4. **Network Interface ID**: the ENI ID noted above.
5. Click **Register group member**.

### Verify (live)
1. Connect to `Consumer-A-Test` via Session Manager. Run the listener:
   ```bash
   python3 -c "
import socket, struct
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('', 5007))
mreq = struct.pack('4sl', socket.inet_aton('224.0.1.1'), socket.INADDR_ANY)
s.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)
print('listening...')
print(s.recvfrom(1024))
"
   ```
2. In a second Session Manager tab, connect to `Consumer-B-Test`. Run the sender:
   ```bash
   python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, 4)
s.sendto(b'hello from Consumer-B', ('224.0.1.1', 5007))
print('sent')
"
   ```
3. **Expected output**: the `Consumer-A-Test` listener prints a tuple like
   `(b'hello from Consumer-B', ('10.2.1.x', <port>))` — confirming multicast
   delivery succeeded, even though Step 3 proved these two instances cannot
   reach each other over ordinary unicast ping. This is the key teaching
   point: TGW multicast domain membership is a separate mechanism from the
   TGW route tables.
```

- [ ] **Step 2: Commit the README section**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 4 clickops instructions: TGW multicast domain and traffic test"
```

- [ ] **Step 3: Hand off to the user to build and verify Step 4**

Ask the user to follow the Step 4 README section and report back: the
multicast domain ID, both ENI IDs, and the actual listener output.

- [ ] **Step 4: Record captured values, confirm verification, update progress**

Add the multicast domain ID and both ENI IDs to `**Captured values:**`
under Step 4. Confirm the listener actually received the sender's message.
If it timed out, debug with the user (check both subnet associations and
both group member registrations exist) before marking done. Update the
Progress table row "Multicast and Transit Gateways" to `Done`.

- [ ] **Step 5: Commit**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Record Step 4 captured values and mark multicast sub-topic done"
```

---

### Task 5: Advanced Transit Gateway Architectures (written recap) and teardown

**Files:**
- Modify: `Section1-multi-vpc-networking/README.md` (add `## Step 5: Advanced Transit Gateway Architectures` and a final `## Teardown` section, update Progress table row 5 to `Done`)

**Interfaces:**
- Consumes: all captured values from Tasks 0-4 (used to fill in the exact
  resource names/IDs in the teardown steps).
- Produces: no new infrastructure. Final README state with all 5 progress
  rows `Done` and a complete teardown section.

- [ ] **Step 1: Write the Step 5 written recap section**

```markdown
## Step 5: Advanced Transit Gateway Architectures

No new infrastructure for this sub-topic — it's a recap of what Steps 3-4
already built, plus two concepts that are cost-prohibitive to deploy here.

**What you already built is an advanced TGW architecture:** a hub-and-spoke
topology (`Lab-TGW` as hub, 3 VPCs as spokes) using **per-spoke custom route
tables** (`RT-ConsumerA`, `RT-ConsumerB`, `RT-Shared`) instead of the TGW's
single default route table. This is the standard pattern for **tenant/
workload isolation on a shared Transit Gateway** — each spoke only sees the
routes explicitly propagated into its own route table, so Consumer-A and
Consumer-B stay isolated from each other while both reach the shared
Service-VPC. You also saw that a **TGW multicast domain operates
independently of these route tables** — worth remembering for the exam,
since it's a common trick question (segmented unicast routing does not
imply segmented multicast).

**Related patterns not built in this lab** (cost-prohibitive for a
self-paced lab, but exam-relevant):
- **Centralized inspection VPC with Gateway Load Balancer**: instead of
  VPCs reaching each other directly over TGW, all inter-VPC traffic is
  routed through a dedicated "inspection" VPC where a Gateway Load Balancer
  fronts a fleet of firewall/IDS appliances, then continues to its
  destination. Requires GWLB + GWLB endpoints + appliance instances running
  continuously.
- **TGW Connect / appliance mode**: TGW Connect attachments let SD-WAN or
  third-party virtual appliances peer with the TGW over GRE, and "appliance
  mode" on a VPC attachment keeps flows symmetric through a stateful
  appliance in that VPC. Requires a real SD-WAN/appliance product to
  demonstrate meaningfully.
```

- [ ] **Step 2: Write the Teardown section**

Fill in `<...>` placeholders in the actual README with the real captured
IDs/names from Tasks 0-4 before this section is considered done — this
step is written generically here since the IDs aren't known until the
earlier tasks' captured values exist.

```markdown
## Teardown

Tear down in this exact order (reverse dependency order) to avoid
"resource in use" errors. Approximate running cost while all of this is up:
**$0.35-0.45/hr** (Transit Gateway attachments, ~13 Interface/Gateway
endpoints, 1 NLB, 3x t3.micro EC2) — don't leave it running longer than you
need.

1. **Multicast group members**: Transit Gateway Multicast → `Lab-Multicast-Domain` → Group members tab → select each member → **Deregister group member**.
2. **Multicast domain associations**: same domain → Associations tab → select each → **Disassociate**.
3. **Multicast domain**: Transit Gateway Multicast → select `Lab-Multicast-Domain` → **Delete transit gateway multicast domain**.
4. **TGW route table associations/propagations**: for each of `RT-ConsumerA`, `RT-ConsumerB`, `RT-Shared` → Associations tab → disassociate; Propagations tab → delete propagation.
5. **TGW route tables**: Transit Gateway Route Tables → select each custom route table → **Delete transit gateway route table**.
6. **TGW VPC attachments**: Transit Gateway Attachments → select each of the 3 → **Delete transit gateway attachment**. Wait for all 3 to reach `Deleted`.
7. **Transit Gateway**: Transit Gateways → select `Lab-TGW` → **Delete transit gateway**.
8. **VPC Endpoints**: VPC console → Endpoints → select all Interface and Gateway endpoints created in Steps 1-2 (9 SSM endpoints + 1 PrivateLink interface endpoint + 1 S3 gateway endpoint) → **Actions** → **Delete VPC endpoints**.
9. **Endpoint Service**: VPC console → Endpoint Services → select the service created in Step 1 → **Actions** → **Delete VPC endpoint service**.
10. **NLB and target group**: EC2 console → Load Balancers → delete `Service-NLB`; Target Groups → delete `Service-TG`.
11. **EC2 instances**: EC2 console → Instances → select `Service-Backend`, `Consumer-A-Test`, `Consumer-B-Test` → **Instance state** → **Terminate instance**.
12. **IAM role**: IAM console → Roles → select `EC2-SSM-Role` → **Delete**.
13. **Subnets**: VPC console → Subnets → delete `Service-Subnet`, `Consumer-A-Subnet`, `Consumer-B-Subnet`.
14. **VPCs**: VPC console → Your VPCs → delete `Service-VPC`, `Consumer-A-VPC`, `Consumer-B-VPC` (this also cleans up their default route tables/security groups/NACLs).

### Confirm teardown
Run (from your local machine, with AWS CLI configured — this is the one
place a CLI check is appropriate, since it's read-only verification, not
provisioning):
```bash
aws ec2 describe-vpcs --region us-east-1 --filters "Name=tag:Name,Values=Service-VPC,Consumer-A-VPC,Consumer-B-VPC"
```
**Expected output**: `"Vpcs": []` — empty, confirming all 3 lab VPCs are gone.
```

- [ ] **Step 3: Commit Step 5 and Teardown sections**

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Add Step 5 written recap and full teardown section"
```

- [ ] **Step 4: Hand off to the user to fill in and execute teardown**

Ask the user to review Step 5, then — once they've finished studying and
are ready — fill in the real resource IDs into the Teardown section (using
values already captured in Steps 0-4) and execute it in order, reporting
back the final `describe-vpcs` output.

- [ ] **Step 5: Confirm teardown, update progress, final commit**

Confirm the reported `describe-vpcs` output is empty. Update the Progress
table row "Advanced Transit Gateway Architectures" to `Done`. This
completes the plan's task list — remaining work (curriculum checkbox, final
live-verification gate) is handled by the outer `new-course-lab` skill
flow, not this plan.

```bash
git add Section1-multi-vpc-networking/README.md
git commit -m "Confirm teardown complete, mark Section1 lab progress table fully done"
```

---

## Self-Review Notes

- **Spec coverage**: all 5 sub-topics have a task (0 is prerequisite
  infra, 1-5 map 1:1 to the curriculum bullets). Cost callout and teardown
  are both present (Task 5). SSM-only access, no NAT/IGW/bastion — reflected
  throughout Task 0 (no public IP, no key pair) and Task 2 (SSM endpoints).
  Single-region, single-account, single-AZ constraints are all respected.
- **Placeholder scan**: no TBD/TODO. The only bracketed placeholders left
  (`<your-name>-...` bucket names, `<...>` IDs in Teardown) are intentional
  — they're filled in with real values captured from the user during
  execution, not left unresolved in the final README.
- **Type/naming consistency**: resource names are used identically across
  tasks — `Lab-TGW`, `Service-VPC`/`Consumer-A-VPC`/`Consumer-B-VPC`,
  `Service-Backend`/`Consumer-A-Test`/`Consumer-B-Test`,
  `RT-ConsumerA`/`RT-ConsumerB`/`RT-Shared`, `Lab-Multicast-Domain`. Checked
  each cross-task reference (Task 3's ping targets, Task 4's ENI lookups,
  Task 5's teardown list) against the names defined in Task 0-3 — no
  mismatches.
