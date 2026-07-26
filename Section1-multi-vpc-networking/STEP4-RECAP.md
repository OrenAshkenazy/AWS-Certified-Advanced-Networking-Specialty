# Step 4 Recap - Transit Gateway vs VPC Peering

Step 4 built a segmented hub-and-spoke network with AWS Transit Gateway. Think of the VPCs as three office buildings that share one mailroom: a building can only send a package where the mailroom's sorting rules allow it.

## The Real-World Picture

```text
Consumer-A-VPC office                 Shared mailroom                 Service-VPC office
       |                                      |                               |
Consumer-A attachment  ->  Lab-TGW (`tgw-024c725b05af1072a`)  ->  Service attachment
```

In this lab, the three office buildings are `Consumer-A-VPC`, `Consumer-B-VPC`, and `Service-VPC`. Their TGW attachments are the doors into the shared mailroom, `Lab-TGW`. A TGW route table is the mailroom's sorting sheet: it decides which building receives a package after it arrives.

For example, a packet from `Consumer-A-VPC` enters through `Consumer-A-VPC-Attachment` (`tgw-attach-0c71cbb3c9037f968`). The mailroom consults `RT-ConsumerA` (`tgw-rtb-0907e33f064c9855e`), finds a route only for the Service building (`10.0.0.0/16`), and sends it through `Service-VPC-Attachment` (`tgw-attach-038f6c367592e43e5`). It does not send that packet to Consumer B because that delivery address is not on the sorting sheet.

## What We Built

| Component | Purpose |
|---|---|
| `Lab-TGW` | Regional routing hub for the lab VPCs |
| `Service-VPC-Attachment` (`tgw-attach-038f6c367592e43e5`) | Connects `Service-VPC` to the TGW |
| `Consumer-A-VPC-Attachment` (`tgw-attach-0c71cbb3c9037f968`) | Connects `Consumer-A-VPC` to the TGW |
| `Consumer-B-VPC-Attachment` (`tgw-attach-0bf4d601df5321b5d`) | Connects `Consumer-B-VPC` to the TGW |
| `RT-ConsumerA` (`tgw-rtb-0907e33f064c9855e`) | Sorting sheet for traffic entering from `Consumer-A-VPC` |
| `RT-ConsumerB` (`tgw-rtb-03f196be3c0333463`) | Sorting sheet for traffic entering from `Consumer-B-VPC` |
| `RT-Shared` (`tgw-rtb-085c0dc832635bc47`) | Sorting sheet for traffic entering from `Service-VPC` |

The intended routing model was:

```text
Consumer-A-VPC -> Lab-TGW -> Service-VPC
Consumer-B-VPC -> Lab-TGW -> Service-VPC
Consumer-A-VPC -X-> Consumer-B-VPC
```

The consumers can reach the shared service VPC, but they cannot reach each other over ordinary unicast routing.

## What The Custom TGW Route Tables Did

Each TGW attachment can be associated with exactly one TGW route table. That route table decides where traffic from that attachment can go.

| Attachment | Associated TGW Route Table | Routes learned |
|---|---|---|
| `Consumer-A-VPC-Attachment` | `RT-ConsumerA` | `Service-VPC` only |
| `Consumer-B-VPC-Attachment` | `RT-ConsumerB` | `Service-VPC` only |
| `Service-VPC-Attachment` | `RT-Shared` | `Consumer-A-VPC` and `Consumer-B-VPC` |

That is the core lesson: with TGW, routing policy lives inside the transit gateway, not only in the VPC subnet route tables.

## Why The Default TGW Route Table Got In The Way

When `Lab-TGW` was created with default association and propagation enabled, AWS automatically associated new VPC attachments with the default TGW route table and propagated their routes there.

That default behavior is convenient for a quick flat network, but it is too broad for this lab. It tends to create:

```text
All attachments associated with one route table
All VPC CIDRs propagated into one route table
```

For segmentation, we needed to remove the automatic default associations and create custom associations instead.

## TGW vs VPC Peering

### VPC Peering Model

VPC peering is point-to-point connectivity between two VPCs.

For three VPCs, a full mesh would require:

```text
Service-VPC <-> Consumer-A-VPC
Service-VPC <-> Consumer-B-VPC
Consumer-A-VPC <-> Consumer-B-VPC
```

If you only created two peerings:

```text
Consumer-A-VPC <-> Service-VPC
Consumer-B-VPC <-> Service-VPC
```

then `Consumer-A-VPC` still cannot route through `Service-VPC` to reach `Consumer-B-VPC`. VPC peering is not transitive.

### Transit Gateway Model

Transit Gateway creates a central hub:

```text
Consumer-A-VPC
       |
    Lab-TGW
       |
  Service-VPC
       |
Consumer-B-VPC
```

Each VPC connects once to the TGW. Routing between VPCs is controlled centrally using TGW route tables.

## Tradeoffs

| Dimension | VPC Peering | Transit Gateway |
|---|---|---|
| Connectivity model | Point-to-point | Hub-and-spoke |
| Transitive routing | Not supported | Supported through TGW route tables |
| Scale | Simple for a few VPCs, messy as the mesh grows | Better for many VPCs/accounts |
| Routing control | Distributed across each VPC route table | Centralized in TGW route tables plus VPC route tables |
| Segmentation | Possible, but managed pair by pair | Strong fit using separate TGW route tables |
| Overlapping CIDRs | Not supported | Still not supported for normal VPC routing through TGW |
| Hybrid networking | Not the right tool | Native fit for VPN, Direct Connect, and Connect attachments |
| Operational complexity | Low at small scale, high in large meshes | Higher upfront, lower for large hub-and-spoke networks |
| Cost model | No hourly peering connection charge, data transfer charges can apply | Attachment-hour and data processing charges apply |
| Failure/blast radius | One peering affects one pair | TGW is central, so routing mistakes can affect many attachments |

## When Peering Is Better

Use VPC peering when:

1. You have two or a small number of VPCs.
2. You need simple private IP connectivity.
3. You do not need transitive routing.
4. You do not need centralized inspection or segmentation.
5. You want to avoid TGW attachment-hour charges.

Example:

```text
One app VPC needs direct access to one database VPC.
```

Peering is simpler than deploying a TGW just for that.

## When Transit Gateway Is Better

Use Transit Gateway when:

1. You have many VPCs or many accounts.
2. You need a hub-and-spoke topology.
3. You need centralized routing policy.
4. You need segmentation between groups of VPCs.
5. You need VPN or Direct Connect integration.
6. You want to avoid a growing peering mesh.

Example:

```text
Many application VPCs need access to shared services, but not to each other.
```

That is exactly what Step 4 modeled.

## Mental Model

Use this shortcut:

```text
VPC Peering = build a private road directly between two office buildings
Transit Gateway = send traffic through one shared mailroom, using its delivery rules
VPC Lattice = connect services, not networks
PrivateLink = expose one private service endpoint
```

## What Step 4 Proved

The successful ping from `Consumer-A-Test` to `Service-Backend` proved:

```text
Consumer-A subnet route table -> Lab-TGW -> RT-ConsumerA -> Service-VPC-Attachment -> Service-VPC
```

The failed ping from `Consumer-A-Test` to `Consumer-B-Test` proved:

```text
RT-ConsumerA does not have a route to Consumer-B-VPC
```

That is intentional isolation. Security groups allowed the ping test at the instance layer, so the failure came from routing policy, not from a security group block.

## Sources

- AWS VPC peering basics and limitations: https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html
- AWS Transit Gateway route tables: https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html
- AWS Transit Gateway route table association behavior: https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html
- AWS VPC-to-VPC connectivity options: https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/amazon-vpc-to-amazon-vpc-connectivity-options.html
- AWS Transit Gateway pricing: https://aws.amazon.com/transit-gateway/pricing/
