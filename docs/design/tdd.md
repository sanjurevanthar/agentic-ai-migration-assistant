# TDD: Agentic AI Migration Assistant

Source: `Agentic AI Migration Assistant — Production Design.md` (root). Companion docs: [hld.md](hld.md), [lld.md](lld.md), [data-model.md](data-model.md), [api-spec.md](api-spec.md), [nfr.md](nfr.md).

## Summary

A multi-tenant service that takes a Git repository of legacy Java at a pinned commit SHA and produces an approved migration package: microservice boundaries with cited evidence, OpenAPI 3.1 specs, migration plans, a compiling Spring Boot scaffold and a draft PR. Deterministic analysis runs first, LLM reasoning second, and a human approves every externally visible change. Every step is traceable, reviewable and reproducible.

## Goals / Non-Goals

**Goals**
1. Ingest a repo at a pinned SHA (GitHub/GitLab, private repos included).
2. Build a deterministic code model (classes, call graph, Spring stereotypes, injection, table access, git change-coupling).
3. Propose service boundaries with per-module rationale and cited evidence (file, line, metric).
4. Verify proposals against hard rules; revise within a bounded loop (max 3 rounds).
5. Pause for human approval; record edits and rejections as feedback.
6. Generate OpenAPI 3.1 specs, migration plans and a compiling Spring Boot scaffold per approved service.
7. Open draft PRs via MCP only after a second approval.
8. Score every run against ground truth where it exists; record cost, latency and time saved.

**Non-goals (v1):** auto-merging PRs, data migration, rewriting business logic, languages other than Java.

## Background & Context

The tool is for a bank (model-risk policy applies, SR 11-7). It is advisory only: a human decides. Source code must stay in approved regions and secrets are stripped before any LLM call. No code exists yet; this repo currently holds only the production design document.

## Proposed Design

### Architecture Overview

Thin stateless API validates and records a run in Postgres, then enqueues it on Service Bus. Queue-driven workers run a fixed LangGraph state machine. Workers reach out only through the LLM gateway and GitHub (MCP), and OPA authorizes every tool call. See [hld.md](hld.md).

### Components

| Component | Responsibility |
| --- | --- |
| API (FastAPI) | AuthN/Z, validation, run creation, approvals, status; never does heavy work |
| Orchestrator (LangGraph) | Fixed stage graph, checkpoints, human interrupts |
| Ingestor | Read-only clone at SHA, snapshot to Blob |
| Analyzer | tree-sitter + PyDriller + networkx code model; Louvain clustering |
| Summarizer | Per-module LLM summaries (fan-out) |
| Boundary Proposer | ReAct agent over read-only code-model tools + memory retrieval |
| Verifier | Rules engine (hard rules) + LLM critic (soft issues) |
| Spec / Plan Writers | Per-service OpenAPI and migration plan (fan-out) |
| Scaffolder | Templates + LLM fill, sandboxed compile |
| PR Agent | Deterministic; GitHub MCP create-branch, commit, open-draft-PR only |
| LLM gateway (LiteLLM) | Routing, fallback, budgets, key isolation, logging |
| Policy (OPA) | Per-agent tool allowlists; decisions logged |
| Scanners | gitleaks + Presidio before any LLM call |
| Eval harness | Offline suite, LangSmith experiments, CI gate |

Detailed agent table: see [lld.md](lld.md).

### Data Flow

1. Ingest → Analyze (deterministic, cached by SHA).
2. Summarize modules in parallel.
3. Propose boundaries (ReAct + hybrid retrieval).
4. Verify; on hard-rule failure return violations to step 3. After 3 rounds, flag for human review.
5. **Gate 1:** reviewer approves, edits or rejects; edits recorded as feedback.
6. Generate specs and plans per service in parallel; scaffold and compile in sandbox.
7. **Gate 2:** reviewer approves PR (four-eyes on protected repos).
8. Open draft PR; record scores and cost.

### Tech Stack Choices

| Decision | Choice | Alternatives Considered | Why |
| --- | --- | --- | --- |
| Language | Python 3.12, mypy strict | Java + Spring AI | LangGraph ecosystem; tree-sitter, networkx |
| API | FastAPI + Pydantic v2 | Flask | Typed contracts, async, OpenAPI |
| Orchestration | LangGraph | CrewAI, AutoGen | Explicit state machine, checkpoints, interrupts |
| LLM access | LiteLLM proxy gateway | Direct SDK | Central routing, budgets, logging |
| Parsing | tree-sitter-java | JavaParser sidecar | Fast, error-tolerant (see open question) |
| Graph | networkx + Louvain | Neo4j | Sufficient for 5k nodes in memory |
| Git mining | PyDriller | Raw git log | Change-coupling |
| System of record | Azure PostgreSQL Flexible Server | Cosmos DB | Relational queries, consistency |
| Vector store | pgvector (HNSW) | Azure AI Search, Qdrant | One DB to secure/back up |
| Cache/locks | Azure Cache for Redis | In-process | Survives scale-out |
| Queue | Azure Service Bus | Redis Streams, Celery | Durability, DLQ, KEDA |
| Blobs | Azure Blob Storage | Postgres blobs | Immutability policies |
| Runtime | Azure Container Apps + KEDA | AKS | Scale to zero, less ops |
| Identity | Entra ID OIDC + managed identities | Keycloak | SSO, no stored inter-service secrets |
| Secrets | Azure Key Vault | CI variables | Rotation |
| Repo integration | GitHub App + GitHub MCP | PATs | Least-privilege tokens |
| Policy | OPA | Hard-coded checks | Policy as code, testable |
| Scanning | gitleaks + Presidio | Regex | Secrets + PII |
| Tracing/eval | LangSmith + OpenTelemetry → Azure Monitor | Langfuse | LLM + infra traces |
| CI/CD | GitLab CI → ACR → Container Apps | GitHub Actions | Eval gate before deploy |
| IaC | Terraform | Bicep | Reviewable, reproducible |

Architecturally significant decisions (costly to reverse): (1) fixed workflow graph rather than LLM-chosen flow; (2) Postgres + pgvector as the single store; (3) deterministic-first analysis with LLM never re-deriving facts; (4) human-approval-only memory writes.

## Data Model

See [data-model.md](data-model.md).

## API Surface

See [api-spec.md](api-spec.md).

## Non-Functional Requirements

See [nfr.md](nfr.md).

## Rollout Plan

TBD — the source design gives no phasing. Suggested for discussion: single region, zone-redundant v1; eval gate must pass before first production deploy.

## Risks & Open Questions

- Call-graph accuracy on Spring dependency injection with tree-sitter alone — decide vs. JavaParser sidecar after measuring.
- Ground truth beyond the one in-house monolith: which 2–3 open-source Java monoliths with documented decompositions? — TBD
- One reviewer vs. four-eyes at Gate 1 for small repos — TBD
- Which LLM providers/regions are approved for source code under the bank's data policy — TBD
- Capacity numbers (4–6M tokens per cold 500k-LOC run) are planning figures, not measurements — load test.
- Prompt injection via repo content — mitigated by data-block wrapping, OPA, schema validation (see [hld.md](hld.md)).

## Alternatives Considered

See the "Alternative considered" column above. Semantic LLM caching for proposals was explicitly rejected (near-match answers for a different codebase are worse than none); it is allowed only for module summaries.
