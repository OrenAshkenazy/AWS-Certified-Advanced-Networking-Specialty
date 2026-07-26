---
name: new-course-lab
description: Use when starting the next lab in a course repo that has, or should have, a docs/curriculum.md outline and the next queued topic needs multiple coordinated components rather than a single notebook. Triggers on phrases like "next lab", "start the next module", "build the next section", or when bootstrapping docs/curriculum.md from a pasted course outline.
metadata:
  short-description: Start the next multi-component course lab
---

# New Course Lab

Turns a course outline into an ordered queue of labs, then drives the next
queued lab through a spec -> plan -> build -> live-verify workflow.

This skill is an orchestrator. It should not jump straight to code or folder
scaffolding before the gates below are satisfied.

<HARD-GATE>
Do not skip straight to scaffolding a folder or writing code. Steps 0-3 must
happen first, in order, every time this skill fires.
</HARD-GATE>

<DRY-RUN-POLICY>
A lab dry run is optional. Do not add a dry-run phase by default.

Only perform a dry run when:
- the user explicitly asks for one, or
- the implementation is high-risk and the agent asks first and gets confirmation

Do not block lab completion on a dry run. The required completion gate is live
verification in Step 8.
</DRY-RUN-POLICY>

## Workflow

### Step 0: Locate or create `docs/curriculum.md`

Look for `docs/curriculum.md` at the repo root.

- If it exists, read it. Expect one H1 course title, H2 numbered modules, and
  H3 bullet sub-topics under each module.
- If it does not exist, ask the user to paste the course outline, then write it
  verbatim to `docs/curriculum.md` before continuing.

Example shape:

```markdown
# AWS Advanced Networking

## 1. Multi Account, Multi Region, and Multi VPC Networking
* Using AWS PrivateLink for Services
* Advanced VPC Endpoint Architectures

## 2. Implementing Hybrid Networking
* Reviewing AWS and VPN Requirements
* Transit Gateway and Direct Connections
```

### Step 1: Pick the next module

Scan the H2 headings from top to bottom. The first H2 heading without a `[x]`
immediately after `## ` is the next module for this lab.

- If every H2 is already checked, report that the course is complete and stop.
- If none of the H2 headings has any checkbox yet, treat the first H2 as
  unchecked.

The selected heading text, minus any leading number, is the module name. Its H3
bullets are the module sub-topics.

### Step 2: Scope sanity check

This workflow is for multi-component labs only: labs that coordinate multiple
resources, agents, services, or infrastructure pieces.

If the sub-topics clearly describe a simple single-notebook or single-API-call
exercise, stop and ask:

> This module looks like a simple, single-component topic rather than a
> multi-component build. Want me to treat it as an ad-hoc lab instead of
> running the full spec/plan/build/verify workflow?

Do not decide this unilaterally.

### Step 3: Detect the repo's lab naming convention

List the repo's top-level directories. If one or more follow a numbered lab
pattern such as `Section11-AgentCore-MultiAgent` or
`Section12-hybrid-networking`, reuse that pattern and increment the number.

If no existing numbering convention is detectable:

- ask the user once what naming convention to use
- persist that answer in `docs/curriculum.md` as an HTML comment near the top
  in the form `<!-- lab_naming_convention: ... -->`
- reuse that recorded convention on later runs instead of asking again

The result of this step is the new lab folder name.

### Step 4: Scaffold the lab folder

Create the lab folder at the repo root.

Copy `templates/lab-readme.md` from this skill into `<lab-folder>/README.md`
and fill in:

- `{{SECTION_NAME}}` and `{{MODULE_TITLE}}` from the module name
- `{{ONE_PARAGRAPH_DESCRIPTION}}` as a one-sentence module summary
- `{{PROGRESS_TABLE_ROWS}}` with one row per sub-topic, all starting at
  `Not started`
- `{{SPEC_FILENAME}}` and `{{PLAN_FILENAME}}` left as literal placeholder text
  for now
- `{{CAPTURED_VALUES_PLACEHOLDER}}` as `_None yet._`
- `{{GOTCHAS_PLACEHOLDER}}` as `_None yet._`

### Step 5: Produce the design/spec

Drive a design pass for the selected module before planning or coding.

Before the first design question, carry this constraint into the conversation:

> Any step in this lab that needs manual console setup, third-party app
> registration, IAM console edits, provider dashboards, or similar clickops
> must be written into the README as literal clickops: named console sections,
> button labels, and field names in order. Do not collapse that work into a
> CLI-only summary or assume the reader knows the console.

Write the resulting spec to a dated file under `docs/specs/` unless the repo
already has an established spec location. After the spec exists, replace the
`{{SPEC_FILENAME}}` placeholder in the lab README with the real filename.

### Step 6: Produce the implementation plan

Create a concrete implementation plan from the spec and write it to a dated file
under `docs/plans/` unless the repo already has an established plan location.

After the plan exists, replace the `{{PLAN_FILENAME}}` placeholder in the lab
README with the real filename.

### Step 7: Execute the plan

Implement the lab from the plan.

As work lands:

- update the lab README progress table from `Not started` to the current state
- append to `## Gotchas` inline, when issues are discovered
- append real captured values such as ARNs, IDs, endpoints, or URLs to
  `## Captured values`, replacing `_None yet._`
- after each completed conceptual model or major lab step, create or update a
  friendly recap file in the lab folder named `STEP<N>-RECAP.md`
  (for example, `STEP4-RECAP.md`). Use `templates/step-recap.md` as the shape.
  This is required when the step teaches a networking model such as
  PrivateLink, VPC Endpoints, VPC Lattice, Transit Gateway, DNS, VPN,
  Direct Connect, Cloud WAN, or inspection architectures.

Recap files are not procedure manuals. They explain what the user just built
and why it matters, using the real resource names/IDs captured during the lab.
Write them for a learner who may lose the thread easily:

- start with a short, concrete real-world analogy before AWS terminology. For
  network routing, prefer the "three office buildings" model: VPCs are
  buildings, attachments are each building's connection to the shared
  mailroom, the central networking service is the mailroom, and route tables
  are its delivery rules. Map every part of the analogy back to the actual AWS
  resource names immediately afterwards.
- follow it with a one-sentence mental model
- show the actual resources created in a small table
- connect the traffic path using the real VPCs, route tables, endpoints,
  attachments, services, ARNs, DNS names, or IDs
- explain the difference between similar concepts that caused confusion
- include tradeoffs with adjacent alternatives when relevant, such as
  Transit Gateway vs VPC peering, PrivateLink vs VPC Lattice, Gateway
  Endpoints vs Interface Endpoints, NAT Gateway vs private endpoints, or
  Route 53 Resolver endpoints vs public DNS
- include a short "what the live verification proved" section
- include "where people get stuck" based on gotchas discovered during the lab
- avoid abstract-only explanations; every concept should tie back to the
  concrete lab topology

### Step 8: Completion gate requires live verification

Before calling the lab complete, verify it live and end-to-end.

Verification must be non-mocked where the workflow supports it: real deployed
calls, a real UI exercised against a real backend, or equivalent live behavior.
Dry runs may be useful during implementation, but they are optional and never
replace or gate live verification. Unit tests alone do not satisfy this gate.

### Step 9: Check off the module

Once live verification passes, mark the selected H2 heading as complete in
`docs/curriculum.md`.

Example:

```markdown
## [x] 2. Implementing Hybrid Networking
```

If the heading had no checkbox before, add `[x]` immediately after `## `.

## Error handling

- Missing `docs/curriculum.md`: ask the user to paste the outline and stop
  until the file exists.
- All modules already checked: report course complete and stop.
- No naming convention detected: ask once, then persist it in
  `docs/curriculum.md`.
- Module appears single-component: flag it and let the user decide.
