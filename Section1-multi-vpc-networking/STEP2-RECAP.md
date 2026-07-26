# Step 2 Recap - VPC Endpoints and Endpoint Services

Step 2 added private network paths from the lab VPCs to AWS services. The goal was to keep the instances private, with no public IPs, no Internet Gateway, no NAT Gateway, and no SSH, while still letting them reach Systems Manager and S3.

## What We Built

### SSM Interface Endpoints

For each VPC, we created three AWS-managed Interface Endpoints:

| Service | Endpoint Type | Why it is needed |
|---|---|---|
| `com.amazonaws.us-east-1.ssm` | Interface Endpoint | Lets the SSM Agent call the Systems Manager control plane |
| `com.amazonaws.us-east-1.ssmmessages` | Interface Endpoint | Carries Session Manager message traffic |
| `com.amazonaws.us-east-1.ec2messages` | Interface Endpoint | Carries instance-to-SSM message traffic used by the agent |

Because the lab has three VPCs, this means 9 Interface Endpoints total:

| VPC | Endpoints |
|---|---|
| `Service-VPC` | `ssm`, `ssmmessages`, `ec2messages` |
| `Consumer-A-VPC` | `ssm`, `ssmmessages`, `ec2messages` |
| `Consumer-B-VPC` | `ssm`, `ssmmessages`, `ec2messages` |

Each endpoint creates one or more elastic network interfaces inside the selected subnet. Those ENIs receive private IP addresses from the VPC. With Private DNS enabled, names like `ssm.us-east-1.amazonaws.com` resolve to the endpoint ENI private IPs from inside that VPC.

That is why Session Manager started working after these endpoints propagated: the EC2 instances stopped trying to reach public SSM IPs and started reaching private endpoint ENIs inside their own VPCs.

### S3 Gateway Endpoint

In `Consumer-A-VPC`, we created one S3 Gateway Endpoint:

| Service | Endpoint Type | Where it attaches |
|---|---|---|
| `com.amazonaws.us-east-1.s3` | Gateway Endpoint | The route table for `Consumer-A-Subnet` |

Unlike Interface Endpoints, an S3 Gateway Endpoint does not create an ENI and does not use a security group. It adds a route table entry for the AWS S3 prefix list. Traffic from `Consumer-A-Test` to S3 follows that route to the gateway endpoint instead of going through the internet.

We attached an endpoint policy that allowed only the allowed lab bucket:

| Bucket | Expected result |
|---|---|
| `oren-privatelink-lab-allowed` | `aws s3 ls` succeeds |
| `oren-privatelink-lab-denied` | `aws s3 ls` fails with `AccessDenied` |

The endpoint policy restricts what can pass through the endpoint. It does not grant S3 permissions by itself, so we also added `LabS3ReadPolicy` to `EC2-SSM-Role`.

## Endpoint vs Endpoint Service

The names are similar, but they are different things.

### Endpoint

A VPC Endpoint is the consumer-side object that gives a VPC private access to a service.

Examples from this lab:

| Endpoint | Consumer VPC | Service reached |
|---|---|---|
| `Service-VPC-ssm-endpoint` | `Service-VPC` | AWS Systems Manager |
| `Consumer-A-VPC-ssmmessages-endpoint` | `Consumer-A-VPC` | AWS Session Manager messages |
| `Consumer-A-S3-Gateway-Endpoint` | `Consumer-A-VPC` | Amazon S3 |
| `Consumer-A-Endpoint` | `Consumer-A-VPC` | Our custom PrivateLink service from Step 1 |

Think of an endpoint as a private doorway inside a VPC.

### Endpoint Service

An Endpoint Service is the provider-side object behind AWS PrivateLink. It exposes a service through a Network Load Balancer so other VPCs can create Interface Endpoints to it.

In Step 1, we created:

| Provider-side component | Purpose |
|---|---|
| `Service-Backend` | EC2 instance running `python3 -m http.server 80` |
| `Service-TG` | Target group pointing to `Service-Backend` |
| `Service-NLB` | Internal Network Load Balancer in `Service-VPC` |
| VPC Endpoint Service | PrivateLink service backed by `Service-NLB` |

Then `Consumer-A-VPC` created `Consumer-A-Endpoint` to consume that endpoint service.

The Step 1 custom PrivateLink path is:

```text
Consumer-A-Test
  -> Consumer-A-Endpoint interface endpoint
  -> VPC Endpoint Service
  -> Service-NLB
  -> Service-TG
  -> Service-Backend
```

## How Step 2 Connects to Step 1

Step 1 created the custom PrivateLink application path, but the instances could not be accessed yet because they were private. Step 2 created the AWS service endpoint paths that let us use Session Manager.

The two paths are separate:

| Path | What it is for |
|---|---|
| `Consumer-A-Test -> SSM Interface Endpoints -> AWS Systems Manager` | Lets us open Session Manager shells into private EC2 instances |
| `Consumer-A-Test -> Consumer-A-Endpoint -> Endpoint Service -> Service-NLB -> Service-Backend` | Lets the consumer VPC call the custom private service |

Step 2 does not carry the HTTP request to `Service-Backend`. It only makes private instance access and S3 testing possible.

## Security Controls We Used

### Interface Endpoint Security Groups

The SSM Interface Endpoint security groups allow HTTPS from their own VPC CIDR:

| VPC | Source allowed to endpoint SG |
|---|---|
| `Service-VPC` | `10.0.0.0/16` on TCP `443` |
| `Consumer-A-VPC` | `10.1.0.0/16` on TCP `443` |
| `Consumer-B-VPC` | `10.2.0.0/16` on TCP `443` |

This lets EC2 instances in each VPC reach the private SSM endpoint ENIs.

### S3 Gateway Endpoint Policy

The S3 Gateway Endpoint policy allows only the allowed bucket through the endpoint. That is why the allowed bucket works and the denied bucket fails even though the instance role has read permissions for both test buckets.

### IAM Role Policy

`EC2-SSM-Role` has two separate jobs now:

| Policy | Purpose |
|---|---|
| `AmazonSSMManagedInstanceCore` | Lets the instance register with and use Systems Manager |
| `LabS3ReadPolicy` | Lets the instance call `s3:ListBucket` and `s3:GetObject` on the two lab buckets |

IAM grants permission. Endpoint policies and security groups restrict network/service paths.

## Troubleshooting Lessons From This Step

If Session Manager shows an SSM timeout to a public IP, check:

1. The three SSM Interface Endpoints exist in that VPC.
2. The endpoints are `Available`.
3. Private DNS is enabled on all three endpoints.
4. The VPC has DNS resolution and DNS hostnames enabled.
5. The endpoint security group allows TCP `443` from the VPC CIDR.
6. Wait a few minutes for endpoint and DNS propagation.

If `aws s3 ls` fails with `no identity-based policy allows s3:ListBucket`, the S3 Gateway Endpoint is not the issue. Add or fix IAM permissions on `EC2-SSM-Role`.

If the allowed S3 bucket works and the denied bucket returns `AccessDenied`, Step 2 is working as intended.
