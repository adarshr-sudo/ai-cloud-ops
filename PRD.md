# PRD: AI Cloud Operations Platform

**Status:** final · **Updated:** 2026-10-29 · **Companions:** [Landscape and build-vs-reuse addendum](docs/addendum.md) · [Architecture spine](docs/architecture.md)

## 0. Document Purpose

This PRD defines v1 of the AI Cloud Operations Platform for its builder and sole operator,
and for downstream architecture, epics, and build work. The scope is a **cost-first** wedge:
one investigation that connects an AWS cost anomaly to the change that caused it, the
telemetry that confirms it, and the team that owns it. Functional requirements are numbered
globally (FR-1…FR-25); glossary terms are used verbatim throughout. Research, technology
choices, and build-vs-reuse decisions live in [`docs/addendum.md`](docs/addendum.md) and are
referenced here, not duplicated.

## 1. Vision

Cloud cost incidents are investigated the same way reliability incidents were ten years
ago: manually, across six tabs, by whoever happens to be on call. An engineer sees a cost
spike, opens Cost Explorer, guesses at the service, checks deployments, reads CloudTrail,
looks for a matching telemetry change, and reconstructs a story. The output is a Slack
message nobody can audit and a number nobody can reproduce.

The platform's v1 runs that work as a single, stateful, evidence-first **Investigation**. It
starts from a **Cost Anomaly**, attributes the cost delta, correlates it with **Change
Events**, confirms or refutes the correlation with telemetry, and produces a **Finding** where
every material claim cites an **Evidence Item** that can be replayed. The
output is an **Evidence Bundle** — human-readable report plus machine-readable JSON — that
survives the incident and can be audited later.

Every Investigation is bounded: **$1.00** of combined LLM and cloud-API spend and **3 minutes**
of wall-clock time by default (FR-3). When a ceiling is hit, the platform reports the partial
evidence it has. A partial answer beats an unbounded run.

The bet is not "agents are magic." It is that **cost, change, and telemetry are currently
investigated in three different tools, and joining them is the part nobody ships.** Existing
AI SRE products either treat cost as a separate dashboard or lock the answer inside their own
telemetry. This platform stays read-only, local-first, and portable: AWS in v1, with the
evidence model designed so another cloud can be added without rewriting the investigation.

## 2. Target User

### 2.1 Jobs To Be Done

- **Functional:** when a cloud bill or cost anomaly jumps, find the service, resource, and
  change responsible, with evidence I can show someone.
- **Functional:** prove or disprove a suspected deployment-to-cost correlation without
  manually stitching together CloudTrail, GitHub, and telemetry.
- **Emotional:** stop feeling stupid in a cost review because I can't explain a 340% line
  item.
- **Social:** hand a service owner a defensible package — not "I think it's your deploy."
- **Contextual:** do this at 9am on a Monday, on a laptop, without standing up a platform
  or granting a vendor access to my account.

### 2.2 Non-Users (v1)

- FinOps analysts who want portfolio-level forecasting and chargeback reporting.
- Platform teams wanting multi-tenant, hosted, self-service cost dashboards.
- On-call engineers responding to a live reliability incident (v2, after cost is proven).
- Anyone expecting the tool to change infrastructure on its own.

### 2.3 Key User Journeys

- **UJ-1. Adarsh explains a Monday-morning S3 cost spike before the cost review.**
  - **Persona + context:** sole operator of this platform and a cloud engineer who owns the
    bill; already authenticated to AWS locally with a read-only profile.
  - **Entry state:** an AWS Cost Anomaly notification names S3 in `us-east-1`, up 340%.
  - **Path:** runs `aiops investigate --anomaly <id>`; the platform resolves the affected
    account/resource, decomposes the delta, pulls candidate **Change Events** from the last
    14 days, queries telemetry for the same window, and ranks **Hypotheses**. Adarsh reads
    the top **Finding**, clicks through three **Evidence Items**, and sees the one
    **Hypothesis** it could not rule out.
  - **Climax:** the report names a deployment, a specific configuration change, the matching
    telemetry delta, and an estimated monthly **Cost Impact Estimate** — all cited.
  - **Resolution:** Adarsh exports the **Evidence Bundle** and pastes the summary into the
    cost review. Nothing in AWS was modified.
  - **Edge case:** if the evidence is inconclusive, the report says so, lists what was ruled
    out, and asks for one missing input (for example, a longer CloudTrail window) instead of
    inventing a cause.

- **UJ-2. Adarsh replays a past investigation during a postmortem.**
  - **Persona + context:** same operator, three weeks later, explaining the same spike to
    the team.
  - **Entry state:** the saved **Evidence Bundle** from UJ-1.
  - **Path:** runs `aiops replay <bundle>`; the platform re-executes the recorded queries and
    compares results against the recorded snapshots, flagging drift.
  - **Climax:** the **Finding** and its citations reproduce, and any evidence that can no
    longer be retrieved is marked stale rather than silently dropped.
  - **Resolution:** the postmortem links to a bundle that still supports its conclusion.

- **UJ-3. Adarsh hands an Action Proposal to the owning team.**
  - **Persona + context:** same operator, after a confirmed **Finding**.
  - **Entry state:** a completed **Investigation** with a ranked **Action Proposal**.
  - **Path:** reviews the **Action Proposal** with its risk, **Blast Radius**, and estimated
    savings; selects "export as GitHub issue/PR draft".
  - **Climax:** the platform opens a draft in the target repository containing the proposed
    change, the evidence links, and the rollback note. The **Approval Gate** records who
    approved the export and when.
  - **Resolution:** the owner reviews and merges. The platform never applied the change.

## 3. Glossary

- **Cost Anomaly** — a detected or declared deviation in cloud spend for an account,
  service, resource, or dimension, with a time window and a baseline. It is the v1 entry
  point; the platform consumes anomalies, it does not detect them.
- **Investigation** — one stateful run that starts from a Cost Anomaly and ends in a
  Finding, an inconclusive result, or an aborted run. Identified by `investigation_id`.
- **Change Event** — a recorded change that could plausibly affect cost or behavior:
  deployment, Kubernetes rollout, Git commit or pull request, CloudTrail API call, or
  infrastructure-as-code change. Every Change Event has a timestamp, actor, source, and
  replayable reference.
- **Evidence Item** — one retrieved, immutable observation: the source, the exact query, the
  time window, the retrieval timestamp, a content hash, and a resolvable reference. Evidence
  Items are the only permitted support for a Finding.
- **Evidence Bundle** — the portable artifact of an Investigation: the report, the ranked
  Hypotheses, every Evidence Item, the tool-call log, and the budget spent. Replayable.
- **Hypothesis** — a candidate causal explanation for the Cost Anomaly, with the test that
  would confirm or refute it. Hypotheses are generated before validation and may be
  discarded.
- **Finding** — the Investigation's conclusion: zero or more supported Hypotheses, each with
  a **Confidence**, citations, and an explicit statement of what remains uncertain. A Finding
  may be "inconclusive" and that is a valid outcome.
- **Confidence** — a calibrated label (`low` / `medium` / `high`) with a one-line rationale,
  not a bare probability. The platform must never present Confidence without citations.
- **Cost Impact Estimate** — the estimated monthly dollar impact of the Finding, with a
  range and the assumptions behind it. It is an estimate, not a billing fact.
- **Owner** — the team responsible for the affected service or resource, resolved from
  resource tags, `CODEOWNERS`, or a declared mapping. The Owner receives Action Proposals.
- **Action Proposal** — a proposed change that would reduce cost or prevent recurrence, with
  risk, Blast Radius, estimated savings, and a verification step. It is never applied by the
  platform in v1.
- **Blast Radius** — the set of accounts, services, resources, and users a proposed action
  could affect if it goes wrong.
- **Approval Gate** — the explicit human decision required before an Action Proposal leaves
  the platform (export, PR, or ticket). Recorded with actor, timestamp, and the hash of what
  was approved.
- **Replay** — re-execution of an Evidence Bundle's recorded queries against current sources,
  with drift reported per Evidence Item.
- **Budget Guard** — the per-Investigation ceiling on LLM spend, cloud-API spend, and
  wall-clock time. When reached, the Investigation stops and reports partial evidence.

## 4. Features

### 4.1 Investigation Intake

**Description:** Start an Investigation from a Cost Anomaly identifier or an explicit
account/service/window; validate read-only credentials and record the initial budget.
Realizes UJ-1.

**Functional Requirements:**

#### FR-1: Start an Investigation from a Cost Anomaly or explicit scope

The operator can start an Investigation from an AWS Cost Anomaly identifier **or** an
explicit account, service, and time window. Realizes UJ-1.

**Consequences (testable):**
- Both entry forms create an `investigation_id` and a persisted Investigation record.
- An invalid or expired anomaly identifier fails before any billable tool call, with a
  message naming the missing input.
- Re-running the same anomaly creates a new Investigation rather than mutating the old one.

**Out of Scope:**
- Detecting anomalies from raw billing data.

#### FR-2: Normalize cost scope and baseline

The platform can resolve a Cost Anomaly into an affected account, service, resource set,
dimension, and baseline-vs-observed window, using Cost Explorer or CUR/FOCUS data.

**Consequences (testable):**
- The resolved scope is written into the Investigation record before correlation begins.
- When the anomaly cannot be attributed below account level, the Investigation continues at
  account level and labels the Finding `scope: account`.
- The baseline window is explicit in the report and in every cost-related Evidence Item.
- Resource-level attribution is only claimed when the billing source supports it for the
  requested window; for windows it does not cover, the report falls back to service-level
  attribution and states the downgrade.

#### FR-3: Enforce the Budget Guard

Every Investigation runs under a Budget Guard of **$1.00 of combined LLM and cloud-API
spend, and 3 minutes wall-clock** by default, configurable per run via CLI flags.

**Consequences (testable):**
- When any ceiling is hit, no new LLM or tool call starts; the Investigation ends with a
  partial report marked `budget_exhausted`.
- Actual LLM spend, cloud-API spend, and elapsed time are recorded in the Evidence Bundle.
- Lowering any ceiling via CLI takes effect immediately, including during an Investigation.

### 4.2 Cost Attribution

**Description:** Decompose the spend delta to say *why* the number moved, not just that it
did. Realizes UJ-1.

**Functional Requirements:**

#### FR-4: Attribute the delta to service and resource

The platform can attribute the Cost Anomaly delta to a ranked list of services and
resources, with the attributed amount per entry.

**Consequences (testable):**
- The report shows the top five contributors with dollar amounts that sum to the observed
  delta within a stated tolerance.
- Unattributed remainder is reported explicitly; it is never silently allocated.
- Every attribution entry links to the query that produced it.

#### FR-5: Decompose delta into usage, rate, and mix

The platform can decompose the delta into usage change, unit-rate change, and resource-mix
change, and state which component dominates.

**Consequences (testable):**
- The report labels the dominant component and its share of the delta.
- A delta driven by a rate change cites the pricing or commitment evidence separately from
  the usage evidence.
- When decomposition is not possible for a service, the report says so rather than
  estimating silently.

#### FR-6: Produce a Cost Impact Estimate with a range

The platform produces a monthly **Cost Impact Estimate** with a low/high range, the
assumptions behind it, and the observation window used.

**Consequences (testable):**
- The estimate never appears without its assumptions and window.
- The estimate is labelled an estimate in both the report and the JSON output.
- If the anomaly window is under 24 hours, the report says the extrapolation is weak.

### 4.3 Change Correlation

**Description:** Pull candidate Change Events around the anomaly window and rank by temporal
and causal fit. Realizes UJ-1 and UJ-3.

**Functional Requirements:**

#### FR-7: Collect Change Events from declared sources

The platform can collect Change Events from AWS CloudTrail, GitHub commits and pull
requests, Kubernetes rollout history, and deployment markers for the affected scope.

**Consequences (testable):**
- Each Change Event records timestamp, actor, source, and a replayable reference.
- Sources that are unavailable are listed in the report; their absence never appears as
  "no change occurred."
- Duplicate Change Events across sources are merged with all source references retained.

#### FR-8: Rank Change Events by fit, and record the ones ruled out

The platform ranks Change Events against the anomaly window and the affected scope, and
records which candidates were ruled out and why.

**Consequences (testable):**
- The top three candidates appear with their rank rationale.
- A ruled-out candidate appears in a "ruled out" section with the disqualifying evidence.
- Ranking never presents a Change Event as causal without at least one corroborating
  telemetry or cost Evidence Item.

### 4.4 Telemetry Correlation

**Description:** Confirm or refute each leading candidate with logs, metrics, and traces
from the same window. Realizes UJ-1.

**Functional Requirements:**

#### FR-9: Retrieve telemetry for the affected scope and window

The platform can retrieve metrics, logs, and traces for the affected service and window
from the configured observability sources.

**Consequences (testable):**
- Every telemetry query is recorded as an Evidence Item with query, window, and source.
- Missing telemetry is reported as `no_data`, distinct from `no_change`.
- The platform never substitutes a different service's telemetry without labelling it.

#### FR-10: Correlate telemetry change with the cost window

The platform can compare telemetry before, during, and after the anomaly window and state
whether a change aligns, contradicts, or is inconclusive.

**Consequences (testable):**
- The report states `aligned`, `contradicts`, or `inconclusive` per leading candidate.
- A contradictory telemetry result downgrades or removes the candidate's Finding.
- The comparison windows are explicit and identical across candidates unless stated.

### 4.5 Evidence and Citation

**Description:** Make evidence a first-class, portable artifact — the feature the product is
judged on. Realizes UJ-1 and UJ-2.

**Functional Requirements:**

#### FR-11: Every material claim cites an Evidence Item

Every material claim in a Finding — cause, cost attribution, telemetry change, and change
correlation — carries at least one Evidence Item citation.

**Consequences (testable):**
- A Finding with zero citations fails validation and is emitted as `inconclusive`.
- Removing any single Evidence Item from a bundle makes the dependent claim fail replay
  validation.
- The report renders citations inline and as a numbered list.

#### FR-12: Evidence Items follow a fixed schema

Every Evidence Item records source, query, time window, retrieval time, content hash, and a
resolvable reference.

**Consequences (testable):**
- The JSON output validates against the published schema.
- An Evidence Item missing any required field is rejected at write time.
- Content hashes are stable across repeated retrievals of unchanged data.

#### FR-13: Export and replay an Evidence Bundle

The operator can export an Evidence Bundle and later replay it, with per-item drift status.

**Consequences (testable):**
- `aiops replay <bundle>` reproduces the Finding when sources are unchanged.
- Changed sources are reported per Evidence Item as `drift`, not as a failure of the run.
- A bundle whose source retention has expired is reported as `snapshot-only`, never as a
  successful live replay.
- A bundle can be replayed even when no write credentials are configured.

#### FR-14: Surface contradiction and uncertainty explicitly

The platform reports what it could not establish, including contradictory evidence, missing
sources, and unresolved Hypotheses.

**Consequences (testable):**
- The report contains a non-empty "What we could not establish" section whenever any
  source is unavailable or any Hypothesis is unresolved.
- An `inconclusive` Finding lists the specific next input that would resolve it.
- Confidence is never `high` when a material claim has only one Evidence Item.

### 4.6 Hypothesis and Finding Synthesis

**Description:** Generate candidate explanations, test them, and emit a ranked Finding.
Realizes UJ-1.

**Functional Requirements:**

#### FR-15: Generate ranked Hypotheses before validation

The platform generates a ranked list of Hypotheses, each with a stated test, before
collecting confirming evidence.

**Consequences (testable):**
- At least two Hypotheses are generated unless only one is plausible from the scope; the
  reason for a single Hypothesis is recorded.
- Each Hypothesis names the tool call or query that would refute it.
- The generated Hypotheses are stored in the bundle even when discarded.

#### FR-16: Validate Hypotheses and record refutations

The platform executes each Hypothesis's test and records confirmed, refuted, or unresolved.

**Consequences (testable):**
- Refuted Hypotheses remain in the output with the refuting Evidence Item.
- An unresolved Hypothesis cannot become the top Finding.
- Validation uses at most the remaining Budget Guard allowance.

#### FR-17: Emit a structured report and JSON result

The platform emits a Markdown report for humans and a JSON result for machines from the
same Investigation.

**Consequences (testable):**
- Markdown and JSON agree on the Finding, Confidence, citations, and Cost Impact Estimate.
- The JSON validates against the published schema and includes the tool-call log.
- The report is self-contained: no external dashboard login is required to read it.

### 4.7 Owner Resolution

**Description:** Route the Finding to a team, not to a Slack channel. Realizes UJ-3.

**Functional Requirements:**

#### FR-18: Resolve an Owner with a declared fallback

The platform resolves the Owner from resource tags, `CODEOWNERS`, or a configured mapping,
and states which source was used.

**Consequences (testable):**
- The report names the Owner and the resolution source.
- When no Owner resolves, the report says `unowned` and does not guess.
- A repository with no `CODEOWNERS` falls back to the configured mapping without failing
  the Investigation.

### 4.8 Action Proposals (No Execution)

**Description:** Recommend what to change, with risk and verification, and require an
Approval Gate before anything leaves the platform. Realizes UJ-3.

**Functional Requirements:**

#### FR-19: Propose actions with risk, Blast Radius, and savings

For a supported Finding, the platform produces ranked Action Proposals with risk level,
Blast Radius, estimated savings, and a verification step.

**Consequences (testable):**
- Every Action Proposal names the evidence that justifies it.
- A proposal whose savings cannot be estimated says `unknown` rather than inventing a figure.
- No supported Finding produces a proposal that requires an unreviewed write credential.

#### FR-20: Gate every state-changing output behind Approval

Any output that leaves the platform — a GitHub issue, a pull-request draft, or a ticket —
requires an explicit Approval Gate action.

**Consequences (testable):**
- Without approval, the platform writes nothing outside its local output directory.
- The approval record includes actor, timestamp, and a hash of the exact payload approved.
- A changed payload after approval invalidates the approval and requires a new one.

### 4.9 Agent Observability and Audit

**Description:** Make the Investigation explainable after the fact: every tool call, LLM
call, cost, and decision. Realizes UJ-2.

**Functional Requirements:**

#### FR-21: Trace every LLM and tool call

The platform records every LLM call and tool call with timing, tokens, cost, inputs, and
outputs (redacted where required), and emits OpenTelemetry traces.

**Consequences (testable):**
- The Evidence Bundle contains a tool-call log sufficient to reconstruct the Investigation
  order.
- Total spend in the bundle equals the sum of recorded call costs within 1%.
- A run with tracing disabled still writes the local log; tracing is additive.

#### FR-22: Maintain an append-only audit log

The platform appends every investigation start, tool call, approval, export, and termination
to a local append-only audit log.

**Consequences (testable):**
- Entries cannot be modified or deleted through the CLI.
- Each entry carries the `investigation_id` and a monotonically increasing sequence number.
- The audit log is separate from the Evidence Bundle and survives bundle deletion.

### 4.10 CLI Surface

**Description:** One binary, scriptable, no daemon required. Realizes UJ-1 through UJ-3.

**Functional Requirements:**

#### FR-23: Provide the v1 CLI commands

The operator can run `investigate`, `replay`, `export`, `eval`, `doctor`, and `config` from
the CLI.

**Consequences (testable):**
- `aiops doctor` verifies AWS identity is read-only and reports each configured source.
- Every command exits non-zero on failure and prints a machine-readable error code.
- `aiops investigate --help` lists the Budget Guard flags and supported sources.

### 4.11 Evaluation Harness

**Description:** Prove the wedge works and keep it working; evaluation is a product feature,
not an afterthought. Realizes UJ-1.

**Functional Requirements:**

#### FR-24: Run a pre-registered evaluation suite

The operator can run the evaluation suite against committed scenarios and receive a score
report per scenario and in aggregate.

**Consequences (testable):**
- One command reproduces the published score table from a clean checkout.
- Ground truth is committed before results and is not modified by the run.
- The report includes per-scenario failures, not only aggregate scores.

#### FR-25: Report localization, citation, cost, and false-hypothesis metrics

The evaluation reports top-1 and top-3 localization, citation validity, false-hypothesis
rate, spend, and latency per scenario.

**Consequences (testable):**
- Metrics are reported as mean and range across at least three runs.
- Citation validity is computed by deterministic resolution first, with LLM judging only as
  a secondary check.
- A scenario that errors is counted as a failure, never dropped.

## 5. Cross-Cutting NFRs

- **Read-only by default.** v1 holds no write credentials for AWS, Kubernetes, or GitHub
  beyond opening a draft issue/PR after an Approval Gate.
- **Local-first.** Investigation data, bundles, and the audit log stay on the operator's
  machine; no hosted service is required.
- **Replayability.** Given the same bundle and unchanged sources, results reproduce; drift is
  reported, never hidden.
- **Latency and spend.** Bounded by the Budget Guard (FR-3); a partial result is always
  preferable to an unbounded run.
- **Explainability.** Every material claim resolves to an Evidence Item; every Evidence Item
  resolves to a query the operator could run by hand.
- **Failure isolation.** A failing source degrades the Investigation to `partial`; it does
  not abort the run unless the cost source itself is unavailable.

## 6. Constraints and Guardrails

### 6.1 Safety

- Tool output is untrusted data, never instructions. Log lines, alert text, commit messages,
  and issue titles cannot alter the Investigation's goals.
- Mutating tools are disabled in v1. The only outward write is an approved draft issue/PR.
- Every Action Proposal carries a Blast Radius and a verification step.
- Any prompt-injection signal in tool output is recorded and surfaced, not silently obeyed.

### 6.2 Privacy

- Secrets and credentials never enter LLM context; they are injected at the tool boundary.
- Redaction runs before content is sent to a model or written to a bundle.
- Bundles are local by default and are not uploaded anywhere.

### 6.3 Cost

- The Budget Guard is the enforcement point for LLM and cloud-API spend.
- Cost Explorer requests cost **$0.01 each**, so AWS API calls are counted, attributed, and
  reported per Investigation; the report warns when a run is API-heavy.
- Cost of investigation is a first-class metric, not an operational afterthought.

## 7. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Over-broad read credentials expose the account | Dedicated read-only role, permission boundaries, `doctor` check before first run |
| Prompt injection through attacker-controlled telemetry | Treat tool output as data, label provenance, never execute instructions from it |
| Confidently wrong RCA damages trust | Forced citations, explicit `inconclusive`, false-hypothesis metric, confidence capped by evidence count |
| Cost of investigation exceeds the anomaly | Budget Guard, per-run spend in the bundle, eval tracks $/scenario |
| AWS-only design bakes in lock-in | Evidence schema and source adapters kept provider-neutral; v1 implements AWS only |
| Replay drifts and invalidates past reports | Snapshot hashes plus explicit `drift` status; postmortems can cite bundle version |
| Evaluation becomes self-graded theatre | Pre-registered ground truth, committed before results, neutral metrics |

## 8. Integration and Dependencies

- **Billing (locked):** AWS Cost Explorer for attribution — `GetCostAndUsage` for
  service/usage-type/region dimensions, `GetCostAndUsageWithResources` for resource-level
  data — plus Cost Anomaly Detection (`GetAnomalies`) for intake. Resource-level data is
  opt-in and covers only the **last 14 days**; older windows fall back to service-level
  attribution. A `BillingSource` interface keeps a CUR 2.0 / FOCUS adapter possible without
  schema changes. AWS requests are SigV4-signed with read-only scopes (`ce:Get*`,
  `ce:Describe*`); no static API key is stored. **Account ID, profile or role ARN, and region
  are configuration values** — no account is hard-coded.
- **Observability (locked):** **SigNoz** for metrics, logs, and traces — OpenTelemetry-native,
  with its official MCP server and HTTP API. Queries are recorded verbatim as Evidence Items.
  SigNoz runs from the repository's **Docker Compose** stack locally; Terraform can deploy it
  in-cluster. An `ObservabilitySource` interface keeps other backends possible.
- **Kubernetes (configurable):** the cluster target is `kind` for demos, `eks` for cloud, or
  `existing` via a kubeconfig context supplied in config. Credentials come from the kubeconfig
  or IAM; no static cluster key is stored in the repo.
- **Infrastructure:** **Terraform** provisions the demo AWS environment (VPC, optional EKS,
  read-only IAM role, CUR/FOCUS-ready S3 bucket). Docker Compose is the supported local path
  for SigNoz; Terraform does not manage local containers in v1.
- **Source control:** GitHub for commits, pull requests, `CODEOWNERS`, and draft issue/PR
  output.
- **LLM provider (locked):** **Anthropic Claude** (Sonnet-class) for planning and synthesis,
  and **DeepSeek** (OpenAI-compatible) for extraction and classification, both behind
  LiteLLM. Model IDs live in config, not code. A `local-only` mode routes all calls to a local
  model for sensitive evidence.

Exact servers, transports, and build-vs-reuse decisions are in
[`docs/addendum.md`](docs/addendum.md) §3 and §5.

## 9. Non-Goals

- Not an autonomous remediation system. The platform never applies AWS or Kubernetes changes
  in v1; the Approval Gate exists now so the interface does not change later.
- Not an anomaly detector. v1 consumes AWS Cost Anomaly Detection events or an explicit
  window; it does not train a detector.
- Not a FinOps reporting product. No budgets, chargeback, forecasting, or portfolio
  dashboards.
- Not multi-cloud in v1. GCP and Azure are the expansion path, not the starting scope — and
  the only real differentiator once the wedge is proven.
- Not a general incident-response platform. Reliability-incident intake and RCA, Slack triage,
  and on-call workflows are deferred to v2, and the evidence model must not preclude them.
- Not a hosted multi-tenant service. No web UI, no accounts, no tenancy model, no Slack app.
- Not a model-training project. Fine-tuning is not required to demonstrate the wedge.
- Not a fixed agent topology. The original design proposed a multi-agent orchestrator; v1
  requires one stateful Investigation with source adapters. How many agents implement it is
  an architecture decision, not a PRD requirement.
- Not a knowledge-retrieval product. Runbook and postmortem retrieval (hybrid BM25 plus
  vector search) is deferred to v2; v1 correlates live sources only.
- Not a replacement for human judgment. It produces evidence and proposals; a human decides.

## 10. MVP Scope

v1 is one CLI, one AWS account, one cluster target, and one replayable artifact.

- **Investigation:** anomaly intake, cost attribution and decomposition, Change Event
  correlation, telemetry correlation, citation-gated Findings with an explicit inconclusive
  path, Owner resolution, and Action Proposals gated behind approval.
- **Evidence:** JSON Evidence Bundle plus Markdown report, replay with per-item drift, local
  append-only audit log, OpenTelemetry traces, Budget Guard.
- **Sources:** Cost Explorer plus Cost Anomaly Detection; SigNoz; CloudTrail; GitHub;
  Kubernetes (`kind`, `eks`, or `existing`).
- **LLM:** LiteLLM with Anthropic Claude (planning/synthesis) and DeepSeek (extraction),
  configurable model IDs, `local-only` mode.
- **Infra and evaluation:** Terraform module for the demo AWS environment; Docker Compose for
  SigNoz; pre-registered suite of 15–25 scenarios plus 5 replayed postmortems.

## 11. Success Metrics

### Primary

- **SM-1:** Top-1 localization accuracy on the evaluation suite — target **≥ 60%**, top-3
  **≥ 80%** across 15–25 scenarios. Validates FR-4, FR-8, FR-25.
- **SM-2:** Citation validity — **≥ 95%** of cited Evidence Items resolve to real, in-window
  sources, and **0** material claims without a citation across the suite. Validates FR-11,
  FR-12, FR-25.
- **SM-3:** Combined LLM and cloud-API cost (median **≤ $0.50**, p95 **≤ $1.00**) and p95
  wall-clock **≤ 3 minutes**. Validates FR-3, FR-21, FR-25.

### Secondary

- **SM-4:** False-hypothesis rate — **≤ 1.0** extra Hypothesis per Investigation on the suite.
  Validates FR-15, FR-16.
- **SM-5:** Replay fidelity — **100%** of bundles replay with either identical results or
  per-item `drift`, never a silent change. Validates FR-13.
- **SM-6:** Owner resolution — **≥ 80%** of suite scenarios resolve an Owner from tags or
  `CODEOWNERS`. Validates FR-18.

### Counter-metrics (do not optimize)

- **SM-C1:** Spend per Investigation — do not drive this to zero by cutting evidence
  coverage. Counterbalances SM-3; pair with SM-2.
- **SM-C2:** Number of Action Proposals per Investigation — do not optimize for volume. A
  precise, low-volume set beats a long list; counterbalances any "recommendations" vanity
  metric.
- **SM-C3:** Investigations run per week — do not optimize adoption. Running this on noise
  wastes money and trains the operator to ignore it; counterbalances SM-1 through SM-3.

## 12. Open Questions

1. Is CloudTrail retention long enough (90 days default) to cover the intended replay window?
2. Are there real historical cost incidents to replay, or must all five replayed scenarios be
   reconstructed from public postmortems?
3. Should the evaluation suite be published as an open benchmark in v1, or kept internal?
4. What is the acceptable false-negative rate for "no Finding" — is missing a real cause worse
   than an inconclusive report?
5. Which exact Claude and DeepSeek model IDs ship as the config defaults? The providers are
   fixed; the IDs are not.
6. Does DeepSeek's data policy permit redacted production logs, or must those runs use
   `local-only` mode?

## 13. Assumptions Index

- **[ASSUMPTION: §1, §2, §10]** v1 is AWS-only, one account and one cluster target at a time,
  run by a single operator from a local CLI. GCP/Azure and multi-user auth are later.
- **[ASSUMPTION: §4.1, §10]** v1 consumes Cost Anomaly Detection events rather than building
  a detector.
- **[ASSUMPTION: §4.2]** Cost attribution to service/resource is feasible from the available
  billing dimensions within the stated tolerance.
- **[ASSUMPTION: §4.3]** CloudTrail, GitHub, and Kubernetes rollout history are sufficient to
  reconstruct the Change Events that matter.
- **[ASSUMPTION: §4.8, §6]** GitHub draft issue/PR is an acceptable outward action, and
  read-only IAM plus disabled mutating tools is an adequate safety boundary for a
  single-operator internal tool.
- **[ASSUMPTION: §8]** Cost Explorer's 14-day resource-level window and SigNoz's local
  retention are sufficient for evaluation and replay; CUR/FOCUS is deferred behind the
  `BillingSource` adapter, and `kind` stands in for EKS in the suite.
- **[ASSUMPTION: §8]** Claude Sonnet-class planning and DeepSeek extraction fit the $1
  combined Budget Guard for a single Investigation.
- **[ASSUMPTION: §10]** 15–25 seeded scenarios plus 5 replayed postmortems are enough to make
  a credible internal claim.
- **[ASSUMPTION: §11]** The stated targets are calibrated to a solo builder and a small
  scenario set, not to public-benchmark performance.
