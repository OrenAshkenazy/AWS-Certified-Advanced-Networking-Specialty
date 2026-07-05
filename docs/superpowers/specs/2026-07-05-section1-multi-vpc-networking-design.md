# Section1 — Multi Account, Multi Region, and Multi VPC Networking: Lab Design

## Context

This is the first hands-on lab in the course (`docs/curriculum.md` module 1).
No prior lab folders exist in this repo, so this design also establishes the
`SectionN-kebab-topic` naming convention for future labs.

Covers these sub-topics from the curriculum:
- Using AWS PrivateLink for Services
- Advanced VPC Endpoint Architectures
- Advanced Transit Gateway Concepts
- Multicast and Transit Gateways
- Advanced Transit Gateway Architectures

## Scope decisions

- **Account scope**: single AWS account, multi-VPC. Account boundaries are
  simulated with separate VPCs rather than real AWS Organizations accounts.
- **Region scope**: single region (e.g. `us-east-1`). No inter-region TGW
  peering — kept in scope as a written note only, not built.
- **Provisioning**: 100% AWS Console clickops. No Terraform/CLI automation.
  Every step in the README must be literal clickops: named console sections,
  button labels, and field names in order.
- **Access method**: SSM Session Manager only. No SSH keys, no bastion host,
  no NAT Gateway, no Internet Gateway anywhere in the topology.
- **PrivateLink depth**: build a real custom PrivateLink service (NLB +
  Endpoint Service + Interface Endpoint from a separate consumer VPC), not
  just consumption of AWS-managed service endpoints.
- **Multicast depth**: real multicast traffic test (UDP sender/listener)
  between two EC2 instances registered in a TGW multicast domain, not just a
  console membership check.
- **Cost/cleanup**: README includes approximate per-resource hourly cost
  callouts and an explicit end-of-lab teardown section in strict dependency
  order.

## Topology

Three VPCs in one region, one Transit Gateway as hub:

| VPC | CIDR | Purpose |
|---|---|---|
| `Service-VPC` | `10.0.0.0/16` | Hosts the PrivateLink provider: internal NLB + backend EC2 (tiny HTTP responder) |
| `Consumer-A-VPC` | `10.1.0.0/16` | Consumes the PrivateLink service via Interface Endpoint; also has an S3 Gateway Endpoint with a restrictive endpoint policy; test EC2 |
| `Consumer-B-VPC` | `10.2.0.0/16` | Used for TGW segmentation + multicast testing only; test EC2 |

Each VPC has one private subnet (single AZ, kept simple for the lab). No NAT
Gateway, no Internet Gateway anywhere. Every VPC gets Interface Endpoints for
`ssm`, `ssmmessages`, and `ec2messages` so all instance access is via SSM
Session Manager — this is itself the hands-on example for "Advanced VPC
Endpoint Architectures."

**Transit Gateway**: one TGW, all three VPCs attached.

**TGW route tables** (this is the segmentation mechanism):
- `RT-ConsumerA` — associated with the Consumer-A attachment. Only route:
  Service-VPC CIDR. No route to Consumer-B — demonstrates isolation.
- `RT-ConsumerB` — associated with the Consumer-B attachment. Only route:
  Service-VPC CIDR. No route to Consumer-A.
- `RT-Shared` — associated with the Service-VPC attachment. Both Consumer-A
  and Consumer-B propagate into it, so return traffic works.

**TGW multicast domain**: spans the Consumer-A and Consumer-B attachments/
subnets. Both test EC2 instances are registered as group members of the same
multicast group (e.g. `224.0.1.1`). This deliberately crosses the unicast
segmentation boundary above — multicast domain membership is a separate
mechanism from route tables, so multicast traffic between A and B succeeds
even though unicast ping between A and B fails. This is the core teaching
point of the "Multicast and Transit Gateways" sub-topic.

## Build order (maps 1:1 to the 5 sub-topics)

**Step 0 (prerequisite, not a sub-topic on its own)**: create all three VPCs
(`Service-VPC`, `Consumer-A-VPC`, `Consumer-B-VPC`) and their single private
subnets upfront, empty. Every step below adds resources into this base —
this avoids having to backfill endpoints into a VPC created partway through
the lab.

1. **Using AWS PrivateLink for Services**
   - In `Service-VPC`: the backend EC2 (e.g. `python3 -m http.server 80` run
     via SSM), an internal Network Load Balancer with a target group
     pointing at the backend instance.
   - Create a VPC Endpoint Service backed by that NLB.
   - In `Consumer-A-VPC`: create an Interface Endpoint consuming the
     Endpoint Service (accept the connection request on the provider side).
   - **Verify (live)**: from the Consumer-A test EC2 (over SSM), `curl` the
     Interface Endpoint's private DNS name and confirm the backend's
     response. No TGW involved in this path at all.

2. **Advanced VPC Endpoint Architectures**
   - Add Interface Endpoints for `ssm`, `ssmmessages`, `ec2messages` in all
     three VPCs (this is what makes SSM access to every instance possible —
     build this early since later steps depend on SSM access; the VPCs
     already exist from Step 0 so all three can get their endpoints here).
   - Add a Gateway Endpoint for S3 in `Consumer-A-VPC` with an endpoint
     policy restricting access to one named test bucket.
   - **Verify (live)**: `aws s3 ls s3://<allowed-bucket>` succeeds from the
     Consumer-A test EC2; `aws s3 ls s3://<other-bucket>` fails with an
     explicit access-denied error caused by the endpoint policy.

3. **Advanced Transit Gateway Concepts**
   - Create the TGW, attach all three VPCs.
   - Add the test EC2 in `Consumer-B-VPC` (its VPC/subnet already exist from
     Step 0).
   - Create `RT-ConsumerA`, `RT-ConsumerB`, `RT-Shared`; set associations and
     propagations as described in Topology above.
   - **Verify (live)**: from the Consumer-A test EC2, ping the Service-VPC
     backend's private IP over TGW (succeeds) and ping the Consumer-B test
     EC2's private IP (times out — no route, proving isolation).

4. **Multicast and Transit Gateways**
   - Create the TGW multicast domain over the Consumer-A/B attachments;
     register both test EC2 ENIs as group members of the same multicast
     group IP.
   - **Verify (live)**: run a UDP multicast listener on the Consumer-A test
     EC2 and a sender on the Consumer-B test EC2 (or vice versa); confirm
     the listener receives the packets — despite step 3 proving these two
     instances cannot reach each other over ordinary unicast routing.

5. **Advanced Transit Gateway Architectures**
   - No new infrastructure. Written recap section in the README explaining
     that the segmented hub-and-spoke built in steps 3–4 (per-spoke route
     tables for tenant isolation on a shared TGW) *is* an example of an
     advanced TGW architecture.
   - Short conceptual note (no deployment) on two related exam-relevant
     patterns that are cost-prohibitive to build in this lab: a centralized
     inspection VPC using Gateway Load Balancers, and TGW Connect/appliance
     mode for stateful network appliances.

## IAM

One IAM role, reused by all three EC2 instances, with the AWS-managed
`AmazonSSMManagedInstanceCore` policy attached. Created via console clickops
(IAM console → Roles → Create role → AWS service → EC2 → attach policy →
name it), documented as literal clickops steps in the README.

## Cost

Approximate running cost: **$0.35–0.45/hr** (TGW attachments + interface
endpoints + NLB + 3x t3.micro EC2). Called out per-resource as it's built in
the README, with an explicit prompt to tear down promptly after finishing.

## Teardown

Explicit end-of-README section, strict dependency order:
1. Multicast domain group members
2. Multicast domain
3. TGW route table associations/propagations
4. TGW VPC attachments
5. Transit Gateway
6. VPC Endpoints (interface + gateway) in all VPCs
7. NLB + target group
8. EC2 instances
9. IAM role
10. Subnets and VPCs

## Out of scope

- Real multi-account (AWS Organizations / RAM sharing) — noted as a written
  extension, not built.
- Inter-region TGW peering — noted as a written extension, not built.
- Terraform/CLI automation — this lab is deliberately clickops-only so every
  console click a reader needs is captured; automation may be revisited in
  the course's later "Automating Network Deployments" module.
