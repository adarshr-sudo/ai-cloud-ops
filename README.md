# AI Cloud Operations Platform

> **Agentic AIOps platform for investigating production incidents, correlating operational evidence, generating citation-grounded root-cause analysis, and proposing safe infrastructure remediation.**

AI Cloud Operations Platform is a production-oriented AIOps system that combines **LLM agents, LangGraph, MCP, Kubernetes, AWS, OpenTelemetry, GitHub, and hybrid retrieval** to investigate cloud infrastructure and application incidents.

The platform is designed around one principle:

> **AI should investigate infrastructure, but infrastructure changes should remain controlled, observable, and human-approved.**

---

## Why This Project?

Modern production incidents require engineers to correlate information across many systems:

* Application logs
* Metrics
* Kubernetes state
* Deployments
* Git commits
* Cloud infrastructure
* Operational runbooks
* Previous incidents
* Cloud-cost data

Today, this investigation is often performed manually.

An engineer might need to:

```text
Check Grafana
    ↓
Search logs
    ↓
Inspect Kubernetes
    ↓
Check recent deployments
    ↓
Inspect GitHub commits
    ↓
Read runbooks
    ↓
Check AWS
    ↓
Form a hypothesis
    ↓
Validate the hypothesis
    ↓
Decide on remediation
```

This project explores how an **agentic system can perform that investigation while maintaining evidence, state, observability, and human control.**

---

# Architecture

```text
                         ┌───────────────────────┐
                         │       Engineer        │
                         │  CLI / Web / Slack    │
                         └───────────┬───────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │   Agent Orchestrator  │
                         │      LangGraph        │
                         └───────────┬───────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
              ┌──────────┐     ┌──────────┐     ┌──────────┐
              │  Logs    │     │ Metrics  │     │   K8s    │
              │  Agent   │     │  Agent   │     │  Agent   │
              └────┬─────┘     └────┬─────┘     └────┬─────┘
                   │                │                │
                   └────────────────┼────────────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
              ┌──────────┐    ┌──────────┐     ┌──────────┐
              │  GitHub  │    │   AWS    │     │   RAG    │
              │  Agent   │    │  Agent   │     │  Agent   │
              └────┬─────┘    └────┬─────┘     └────┬─────┘
                   │               │                │
                   └───────────────┼────────────────┘
                                   │
                                   ▼
                         ┌───────────────────────┐
                         │    RCA / Synthesis    │
                         │        Agent          │
                         └───────────┬───────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │   Evidence + RCA      │
                         │   + Confidence        │
                         └───────────┬───────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │ Human Approval Gate   │
                         └───────────┬───────────┘
                                     │
                       ┌─────────────┼─────────────┐
                       ▼             ▼             ▼
                   Kubernetes     GitHub        AWS
                   Remediation      PR        Remediation
```

---

# Core Capabilities

## 1. Stateful Incident Investigation

Investigations maintain state across multiple steps rather than treating every LLM interaction as an independent request.

Example:

```text
Incident
   ↓
Symptoms
   ↓
Metrics
   ↓
Logs
   ↓
Kubernetes
   ↓
Recent Deployment
   ↓
Git Changes
   ↓
Operational Documentation
   ↓
Hypothesis
   ↓
Validation
   ↓
Root Cause
```

The investigation state contains:

```python
IncidentState
├── incident_id
├── service
├── symptoms
├── logs
├── metrics
├── deployments
├── hypotheses
├── evidence
├── root_cause
└── confidence
```

---

# 2. Multi-Agent Investigation

Different agents specialize in different sources of operational evidence.

### Logs Agent

Investigates:

* Error patterns
* Stack traces
* Request failures
* Time-correlated events

### Metrics Agent

Investigates:

* Latency
* Error rates
* Saturation
* Resource utilization
* Anomalies

### Kubernetes Agent

Investigates:

* Pod failures
* Restarts
* Deployments
* Events
* Resource pressure
* Scheduling problems

### Deployment Agent

Investigates:

* Recent releases
* Rollouts
* Configuration changes
* Deployment timelines

### GitHub Agent

Investigates:

* Recent commits
* Pull requests
* Code changes
* Configuration changes

### AWS Agent

Investigates:

* EC2
* EKS
* RDS
* S3
* Networking
* IAM
* CloudWatch
* AWS Cost Explorer

### RCA Agent

Correlates evidence collected by the other agents and produces a structured hypothesis and root-cause analysis.

---

# 3. MCP-Based Infrastructure Access

Infrastructure systems are exposed to agents through controlled tools rather than allowing the LLM to directly interact with infrastructure.

Conceptually:

```text
LLM
 │
 ├── get_service_logs()
 ├── get_metrics()
 ├── get_pods()
 ├── get_deployment()
 ├── get_github_changes()
 ├── query_runbook()
 └── get_aws_cost()
```

MCP provides a standardized interface between the agent system and operational tools.

This creates an important separation:

```text
Probabilistic reasoning
        │
        ▼
      Agent
        │
        ▼
 Controlled tools
        │
        ▼
 Infrastructure
```

---

# 4. Citation-Grounded RCA

The system should not simply generate:

> "The database is probably overloaded."

Instead, it should provide evidence:

```text
Root Cause
──────────

Database connection pool exhaustion.

Evidence

[1] checkout-api logs
    database connection timeout
    failed to acquire database connection

[2] Metrics
    connection pool utilization: 100%
    p95 latency: 2400ms

[3] Deployment
    v1.4.2 deployed 7 minutes before incident

[4] GitHub
    database connection pool configuration changed
    in commit abc123
```

Every important conclusion should be traceable back to collected evidence.

---

# 5. Human-in-the-Loop Remediation

The agent does not receive unrestricted infrastructure access.

Instead:

```text
Agent detects problem
        ↓
Agent proposes remediation
        ↓
Risk assessment
        ↓
Human approval
        ↓
Controlled tool execution
```

Example:

```text
Recommended action:

Rollback checkout-api from v1.4.2 → v1.4.1

Reason:
Recent deployment correlates with connection pool
exhaustion and increased database timeouts.

Risk:
Medium

Approval:
REQUIRED
```

Only after approval should the remediation tool execute.

---

# 6. Observability

The platform itself is observable.

We will use:

* OpenTelemetry
* LangSmith
* Structured logs
* Trace IDs
* Agent execution traces
* Tool execution traces
* Evaluation metrics

Example:

```text
Incident Investigation
        │
        ├── Agent invocation
        │
        ├── Tool call: get_logs
        │
        ├── Tool call: get_metrics
        │
        ├── Tool call: get_deployment
        │
        ├── LLM reasoning
        │
        └── RCA
```

This allows us to answer:

* Why did the agent choose this tool?
* How long did each tool take?
* Which evidence influenced the RCA?
* How many LLM calls were made?
* How much did the investigation cost?
* Did the agent reach the correct conclusion?

---

# 7. Hybrid Retrieval

Operational knowledge will be retrieved from:

```text
Runbooks
Architecture documentation
Incident reports
Deployment documentation
Engineering documentation
```

The retrieval system will combine:

```text
Keyword / BM25
       +
Vector Search
       +
Metadata filtering
       ↓
Hybrid Retrieval
       ↓
Reranking
       ↓
LLM
```

This is useful because operational queries frequently contain exact identifiers such as:

```text
INC-1823
checkout-api
TS-42566
connectionPool
EKS nodegroup
```

Pure semantic retrieval is not always sufficient.

---

# 8. Cloud Cost Intelligence

The platform will also investigate cloud-cost anomalies.

Example:

```text
AWS Cost
   │
   ▼
Cost anomaly detected
   │
   ▼
Identify affected service
   │
   ▼
Correlate with deployment
   │
   ▼
Inspect infrastructure change
   │
   ▼
Estimate monthly impact
   │
   ▼
Optimization recommendation
```

Example output:

```text
S3 data-transfer cost increased 340%.

Likely contributing change:
checkout-api deployment v1.8.3

Observed:
Cross-region data transfer increased after deployment.

Estimated monthly impact:
$4,300

Recommended action:
Evaluate regional data locality.
```

---

# Technology Stack

| Area                   | Technology                   |
| ---------------------- | ---------------------------- |
| Agent orchestration    | LangGraph                    |
| LLM integration        | LangChain                    |
| Tool protocol          | MCP                          |
| Language               | Python                       |
| Infrastructure tooling | Go + Python                  |
| Containerization       | Docker                       |
| Orchestration          | Kubernetes                   |
| Cloud                  | AWS                          |
| Metrics                | Prometheus / VictoriaMetrics |
| Logs                   | Loki / OpenTelemetry         |
| Tracing                | OpenTelemetry                |
| Agent observability    | LangSmith                    |
| Retrieval              | Vector DB + BM25             |
| Source control         | GitHub                       |
| Infrastructure         | Terraform                    |
| CI/CD                  | GitHub Actions               |

---

# Development Roadmap

## Phase 1 — Agent Foundations

* [x] Repository setup
* [ ] Incident state
* [ ] LangGraph workflow
* [ ] Tool calling
* [ ] Mock observability tools
* [ ] Basic RCA

## Phase 2 — Real Observability

* [ ] Kubernetes
* [ ] Prometheus
* [ ] Loki
* [ ] OpenTelemetry
* [ ] Real log/metric investigation

## Phase 3 — MCP

* [ ] MCP server
* [ ] Kubernetes tools
* [ ] AWS tools
* [ ] GitHub tools
* [ ] Tool authorization

## Phase 4 — Stateful RCA

* [ ] Multi-step investigations
* [ ] Hypothesis generation
* [ ] Evidence collection
* [ ] Hypothesis validation
* [ ] RCA synthesis
* [ ] Confidence estimation

## Phase 5 — Knowledge Retrieval

* [ ] Operational documentation
* [ ] Embeddings
* [ ] Vector search
* [ ] BM25
* [ ] Hybrid retrieval
* [ ] Citation generation

## Phase 6 — Remediation

* [ ] Human approval
* [ ] Kubernetes remediation
* [ ] GitHub PR generation
* [ ] Rollback workflows
* [ ] Risk classification

## Phase 7 — Cloud Cost Intelligence

* [ ] AWS Cost Explorer integration
* [ ] Cost anomaly detection
* [ ] Deployment correlation
* [ ] Cost forecasting
* [ ] Optimization recommendations

## Phase 8 — Evaluation

* [ ] Incident evaluation dataset
* [ ] RCA accuracy
* [ ] Tool-selection accuracy
* [ ] Retrieval quality
* [ ] Citation accuracy
* [ ] Regression tests
* [ ] LLM evaluation

## Phase 9 — Production Architecture

* [ ] Authentication
* [ ] Authorization
* [ ] Secrets management
* [ ] Rate limiting
* [ ] Agent isolation
* [ ] Audit logging
* [ ] Failure handling
* [ ] Cost controls
* [ ] EKS deployment

---

# Example Investigation

Suppose:

```text
checkout-api
```

starts returning 5xx errors.

The engineer asks:

```text
Investigate checkout-api incident INC-1842.
```

The system may execute:

```text
1. Fetch recent metrics
2. Detect latency spike
3. Fetch application logs
4. Detect database timeout
5. Inspect Kubernetes pods
6. Inspect recent deployment
7. Inspect GitHub changes
8. Search operational runbooks
9. Generate hypotheses
10. Validate hypotheses
11. Produce RCA
12. Recommend remediation
13. Request human approval
```

The final result might look like:

```text
INC-1842
────────

Service:
checkout-api

Impact:
18% request failure rate

Root Cause:
Database connection pool exhaustion following deployment v1.4.2.

Confidence:
0.91

Evidence:
✓ Connection pool utilization reached 100%
✓ Database timeout errors increased
✓ Deployment occurred 7 minutes before incident
✓ Deployment modified connection pool configuration
✓ Similar previous incident found in INC-1721

Recommended remediation:
Rollback v1.4.2.

Approval:
REQUIRED
```

---

# Design Principles

### 1. Evidence before conclusions

Agents should gather evidence before producing an RCA.

### 2. Tools over unrestricted access

Agents interact with infrastructure through controlled tools.

### 3. State over stateless conversations

Investigations maintain explicit state.

### 4. Human approval for risky actions

The system should recommend infrastructure changes before executing them.

### 5. Observable agents

Every investigation should be traceable.

### 6. Reproducible investigations

Given the same incident and evidence, the investigation should be testable and evaluable.

### 7. Cost-aware AI

LLM calls and infrastructure operations have measurable costs.

---

# Project Status

🚧 **Early Development**

Currently building the foundational LangGraph investigation workflow.

The project is being developed incrementally from:

```text
Mock tools
    ↓
Local Kubernetes
    ↓
Real observability
    ↓
MCP
    ↓
AWS
    ↓
Multi-agent RCA
    ↓
Safe remediation
    ↓
Production deployment
```

---

# Learning Goals

This project is also a hands-on learning environment for:

* Agentic AI
* LangGraph
* MCP
* RAG
* LLM evaluation
* Kubernetes
* AWS
* OpenTelemetry
* Distributed systems
* Cloud cost optimization
* Infrastructure automation
* GitHub automation
* Production system design

The goal is not merely to build an AI demo, but to understand the **engineering trade-offs required to operate agentic systems against real infrastructure**.

---

## Author

**Adarsh Ravichandran**

Staff-level software engineering project focused on:

**AI × Cloud Infrastructure × Kubernetes × Observability × Distributed Systems**
