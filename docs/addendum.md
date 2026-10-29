# Addendum: Landscape, Build-vs-Reuse, and Positioning

> Companion to the project PRD ([`README.md`](../README.md)). This holds the technical and
> market depth that does not belong in the PRD narrative. Research date: 2026-10-29. Vendor
> claims are self-reported unless a source is given; unverified items are flagged.

## 1. What already exists

### 1.1 Open source

| Project | Does | MCP | Cited RCA | Human approval | Gap |
|---|---|---|---|---|---|
| [kagent](https://github.com/kagent-dev/kagent) (CNCF) | K8s-native agent runtime, CRDs, A2A | Native | No evidence artifact | Per-tool `requireApproval` | No incident model, RCA schema, or eval |
| [k8sgpt](https://github.com/k8sgpt-ai/k8sgpt) (CNCF) | Deterministic K8s analyzers + LLM explanation | Server only | K8s object refs only | Deny-by-default alpha, image-only | No agentic loop, no Prometheus/Loki correlation |
| [HolmesGPT](https://github.com/HolmesGPT/holmesgpt) (Robusta/CNCF) | Agentic alert investigation across K8s/cloud/DB/SaaS | Client, MCP-native toolsets | Tool output + confidence, no structured citations | `approval_required_tools` / `restricted_tools` | Grouping/history in commercial Robusta; cost tuning is on you |
| [Robusta Classic](https://github.com/robusta-dev/robusta) | Alert enrichment/grouping/playbooks | Via Holmes | No | Playbook actions can auto-run | Rules engine, not agents |
| [OpenSRE](https://github.com/Tracer-Cloud/opensre) | Framework + RL/eval env, 60+ tools, opens PRs | MCP | Claims evidence-linked, no schema | Default-deny allowlist | Alpha; self-authored benchmark, low scores |
| [Aurora](https://github.com/Arvo-AI/aurora) | LangGraph multi-agent RCA across clouds/K8s | Server | Claims evidence trail | "Destructive actions human-gated" | Tiny community; self-run benchmark |
| [Keep](https://github.com/keephq/keep) | Alert dedupe/correlation/workflows | Unverified | No | Workflow actions | AI correlation is Enterprise-only |

Also relevant: [AIOpsLab](https://github.com/microsoft/AIOpsLab) (Microsoft) — a neutral
fault-injection + scoring harness, not a product.

### 1.2 Commercial

| Product | Acts? | Evidence | Cost RCA | Notable limit |
|---|---|---|---|---|
| Datadog Bits Investigation | PRs, K8s fixes, guardrail tiers | Citations across logs/metrics/traces/code/deploys | No (separate product) | Datadog telemetry only |
| Dynatrace Intelligence | Multi-cloud remediation | Deterministic causal engine over topology | No native | Enterprise lock-in/complexity |
| PagerDuty Paige | Approved runbook automations | Partial (history + diagnostics) | No | Requires AIOps + AI SKUs |
| incident.io Investigations | PR only | Confidence + source links per finding | No | Slack/incident-centric |
| Rootly AI SRE | Draft PRs/steps | Confidence + evidence | No | Acts only within user perms |
| Resolve.ai | Engineer-approved | Citation grounding | "Future scope" | Self-reported accuracy, no public price |
| Cleric | PR + post-fix verification (14 days) | 7+ signals, links, audit log | No | Small vendor, narrow integrations |
| Traversal | Permission ladder, BYOC | Causal indexer, multi-hop | No | Enterprise-only |
| Splunk AI SRE / Cisco AI Canvas | Human-approved PRs, SOAR | Evidence chain incl. ruled-out | No | AI Canvas alpha |
| AWS DevOps Agent + FinOps Agent | Mitigation plans | Telemetry + code + deploy + CloudTrail | **Yes** (FinOps, preview) | AWS-shaped; reliability and cost are separate agents |
| Azure SRE Agent | Mitigation with oversight | Debug trace of tool calls | Vague | Azure-first |
| Google Gemini Cloud Assist | Read-only investigations | "See what it sees" | **Yes** (24/7 anomaly RCA) | Preview, Premium Support gate |

## 2. Verdict: commodity vs. differentiated

| Capability in the original idea | Status |
|---|---|
| Stateful multi-step investigation | Commodity |
| Multi-source evidence agents | Commodity |
| MCP tool access | Commodity (every vendor ships MCP now) |
| Citation-grounded RCA | Commoditizing fast; bare citations are table stakes |
| HITL remediation + risk class | Commodity; most stop at "PR you merge" |
| Agent tracing / observability | Commodity; *public, reproducible* eval is rare |
| Hybrid retrieval of runbooks/past incidents | Commodity |
| Cloud-cost investigation | **Split** — clouds do cost-anomaly RCA; no third-party joins cost ↔ change ↔ telemetry |

Three genuinely open wedges:

1. **Cross-domain correlation: cost anomaly → change → telemetry → owner, in one
   investigation.** Every third-party AI SRE treats cost as a separate dashboard.
2. **Replayable, signed evidence bundle.** A stable evidence object (query, time range,
   deep link, commit SHA, confidence) that an auditor or postmortem can replay. Nobody
   ships this portably today.
3. **An open, pre-registered incident benchmark.** Every vendor quotes self-graded
   accuracy. A public harness with ground truth is itself a contribution.

Repair remains unsolved industry-wide: the SRE-Marathon paper (Sep 2026) reports the best
of 10 methods at 41.3/100, with agents "almost never" completing repairs while faults are
active ([arXiv](https://arxiv.org/abs/2609.33023)). Treat "autonomous remediation" as a
liability, not a demo.

## 3. Build vs. reuse (MCP)

Reusable read paths, don't build:

| Need | Server | Note |
|---|---|---|
| K8s | `containers/kubernetes-mcp-server` (Red Hat) | Read-only SA; disable exec/delete in prod |
| K8s rollback | `Flux159/mcp-server-kubernetes` | Only mature `kubectl rollout` exposure; use non-destructive mode |
| AWS | AWS MCP Server (GA May 2026) + awslabs EKS/CloudWatch/Billing | `call_aws` ≈ whole account; scope IAM/SCP |
| GitHub | `github/github-mcp-server` | Read-only mode; writes = PR only |
| Prometheus | `prometheus/prometheus-mcp` | Remove `quit`/`reload` |
| Grafana/Loki | `grafana/mcp-grafana` | Viewer token + tool allowlist |
| PagerDuty / Atlassian / Slack | Official servers | Gate writes; treat content as untrusted |

Must build: incident state machine, service topology, hypothesis ranking, correlation/dedup,
SLO/error-budget logic, memory, eval harness, and an **action gateway**.

Security posture: all tool output is untrusted (prompt injection via log lines, K8s events,
Confluence pages, PR titles is the dominant risk); deny-by-default mutating tools; per-action
policy (OPA/Cedar) bound to a hash of tool + arguments; immutable audit of every call;
read-only credentials for investigation, separate short-lived credentials for approved
actions. MCP 2026-07-28 has no standard per-action approval semantics — a gateway + policy
engine + human gate is the de facto pattern.

## 4. Evaluation

Neutral benchmarks worth targeting: [ITBench / ITBench-Lite](https://github.com/itbench-hub/ITBench)
(K8s + FinOps, ground-truth entities, frontier agents <50%), [AIOpsLab](https://github.com/microsoft/AIOpsLab)
(fault injection + trajectory scoring), [OpenRCA](https://github.com/microsoft/OpenRCA),
[RCAEval](https://github.com/phamquiluan/RCAEval).

Smallest credible harness: 15–25 incidents (3 fault classes × 2–3 services, self-injected on
Sock Shop or OTel Astronomy Shop) plus 5 replayed public postmortems.

Four metrics only:
- localization top-1 / top-3 (entity)
- reasoning correctness (human-labeled 0 / 0.5 / 1)
- false-hypothesis rate (extra entities per case)
- cost + latency per case

Three seeds, report mean ± range. PRD targets are top-1 ≥ 60% and top-3 ≥ 80% (SM-1) with
false hypotheses ≤ 1.0/case; the literature puts strong systems at 50–60% top-1, and below
40% the result is a demo, not a finding. Pair LLM-as-judge with deterministic
citation-existence and span-overlap checks, and report judge–human κ on 30–50 hand-labeled
outputs.

Honest claims without a full benchmark: "works on N hand-built scenarios, here is the
per-case table and every failure." Never claim MTTR, production-readiness, or dollar savings.

## 5. Locked v1 decisions (2026-10-29)

| Area | Decision | Why | Rejected / deferred |
|---|---|---|---|
| Billing | Cost Explorer API (`GetCostAndUsage`, `GetCostAndUsageWithResources`) + Cost Anomaly Detection (`GetAnomalies`); `BillingSource` adapter interface; CUR/FOCUS deferred. Account/role/region come from config | $0.01/request, no data pipeline, covers service- and resource-level attribution; config keeps the repo portable | CUR 2.0/FOCUS in v1 (S3 + Athena setup), third-party billing MCP as primary, hard-coded account IDs |
| Observability | **SigNoz** run from the repo's Docker Compose stack; optional in-cluster deploy via Terraform/Helm; queried through its HTTP API / MCP server | One OTel-native backend for metrics, logs, and traces; official MCP server and agent skills | VictoriaMetrics stack (three components, VictoriaTraces preview), Prometheus + Loki, SigNoz Cloud for v1 |
| Kubernetes | Configurable target: `kind` for demos, `eks` for cloud, `existing` via kubeconfig context | Keeps the eval suite runnable locally while EKS remains a real target | Requiring a live EKS cluster to develop or evaluate |
| LLM | **LiteLLM abstraction**; **Anthropic Claude** (Sonnet-class) for planning/synthesis, **DeepSeek** (OpenAI-compatible) for extraction; model IDs in config; `local-only` mode | Two vendors keep cost down without lock-in; Claude for tool use, DeepSeek for cheap bulk work | Single-vendor pin, local-only as the default |
| AWS auth | SigV4 via SSO profile or assumed role; read-only `ce:Get*`, `ce:Describe*` | Short-lived credentials, standard audit path | Long-lived access keys in `.env` |
| Deployment | Local CLI; SigNoz via Docker Compose; **Terraform** provisions AWS infra (VPC, optional EKS, read-only IAM role, CUR/FOCUS-ready S3 bucket) | Single-operator internal tool with reproducible infra | Terraform managing local containers; hosted multi-tenant service |

### Constraints these decisions create

- **Cost Explorer resource-level data is opt-in and limited to the last 14 days.** It requires
  a `SERVICE` filter plus a `ResourceId` group or filter; granularity is daily except EC2,
  which can be hourly. Consequence: snapshot Evidence Items at collection time, mark older
  bundles `snapshot-only`, and let the `BillingSource` adapter absorb CUR/FOCUS later.
- **Every Cost Explorer request costs $0.01.** The Budget Guard is a combined LLM + cloud-API
  ceiling with per-source tracking, so a chatty investigation cannot quietly cost more than
  the anomaly.
- **SigNoz self-hosting brings ClickHouse.** Budget roughly 4–8 GB RAM and a few GB of disk
  for a small demo; the CLI must degrade to `partial` when SigNoz is unreachable.
- **Provider-agnostic LLM does not mean provider-neutral behavior.** Tool-call formats and
  context limits differ; pin model IDs per role and record them in every Evidence Bundle.
- **DeepSeek is an OpenAI-compatible API, not a neutral one.** Redaction runs before any
  payload leaves the machine; evidence that cannot be safely redacted uses `local-only` mode.
- **Terraform scope is deliberate.** It provisions AWS infrastructure and can deploy SigNoz
  via Helm; it does not manage the local Docker Compose stack, where the Docker provider would
  add state and credential complexity for no benefit.
- **Config is the seam for accounts and clusters.** `aiops doctor` must fail loudly on a
  missing account, expired SSO session, or unreachable kubeconfig context before any billable
  call.
- **The original README proposed a different stack.** It named LangGraph, LangChain,
  LangSmith, Prometheus/VictoriaMetrics, and Loki. v1 deliberately keeps orchestration and
  retrieval out of the PRD so architecture can choose them, replaces the metrics/logs stack
  with SigNoz, and replaces LangSmith with local OpenTelemetry traces plus the bundle's
  tool-call log. This is a scoping decision, not an oversight.
