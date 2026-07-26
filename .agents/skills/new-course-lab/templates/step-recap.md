# Step {{STEP_NUMBER}} Recap - {{STEP_TITLE}}

{{ONE_SENTENCE_MENTAL_MODEL}}

## The Real-World Picture

Start with one short analogy that makes the model intuitive before introducing
AWS terms. For a network-routing step, use three office buildings:

```text
Building A                  Shared mailroom                  Building B
{{SOURCE_VPC}} --- {{SOURCE_ATTACHMENT}} --- {{CENTRAL_SERVICE}} --- {{DESTINATION_ATTACHMENT}} --- {{DESTINATION_VPC}}
```

State exactly what the analogy represents in this lab. For example: buildings
are VPCs, each building's mailroom connection is an attachment, the shared
mailroom is a Transit Gateway, and its sorting rules are route tables. Then
name the rule that allows or blocks the delivery.

## What We Built

| Resource | Real name / ID | Role |
|---|---|---|
| {{RESOURCE_KIND}} | `{{RESOURCE_NAME_OR_ID}}` | {{RESOURCE_ROLE}} |

## The AWS Model

{{FRIENDLY_EXPLANATION_FOR_A_DISTRACTED_LEARNER}}

Use concrete names from the lab. Prefer:

```text
{{SOURCE_RESOURCE}}
  -> {{CONTROL_OR_NETWORK_HOP}}
  -> {{DESTINATION_RESOURCE}}
```

Avoid explaining the cloud service only in generic terms. Tie every concept to
the resources the learner created.

## How The Traffic Or Control Path Works

```text
{{REAL_SOURCE}}
  -> {{REAL_STEP_1}}
  -> {{REAL_STEP_2}}
  -> {{REAL_DESTINATION}}
```

Explain what each hop does in one short paragraph.

## Key Terms

| Term | Meaning in this lab |
|---|---|
| {{TERM}} | {{MEANING_USING_LAB_RESOURCE_NAMES}} |

## Tradeoffs

Compare this model with the closest alternative when relevant.

| Decision | Use this model when | Use the alternative when |
|---|---|---|
| {{MODEL}} vs {{ALTERNATIVE}} | {{WHEN_THIS_FITS}} | {{WHEN_ALTERNATIVE_FITS}} |

Common comparisons:

- PrivateLink vs VPC Lattice
- Transit Gateway vs VPC peering
- Gateway Endpoint vs Interface Endpoint
- NAT Gateway vs private endpoints
- Route 53 Resolver endpoints vs public DNS
- AWS Network Firewall vs Gateway Load Balancer appliances
- VPN vs Direct Connect
- Transit Gateway vs Cloud WAN

## What The Live Verification Proved

{{WHAT_SUCCESS_SHOWED}}

If there was an intentional failure test, explain it:

```text
{{FAILED_TEST}} failed because {{INTENTIONAL_CONTROL}}
```

## Where People Get Stuck

| Symptom | Likely cause | Fix |
|---|---|---|
| {{SYMPTOM}} | {{LIKELY_CAUSE}} | {{FIX}} |

## What To Remember

{{ONE_OR_TWO_SENTENCES_THAT_MAKE_THE_MODEL_STICK}}
