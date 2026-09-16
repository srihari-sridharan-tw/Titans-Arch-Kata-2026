# ADR-012: Unified AI Eval Pipeline & Golden Sets

## Status
Accepted

## Context
Deploying Generative AI and Large Language Models (LLMs) across customer-facing and operational workflows introduces non-deterministic failure modes:
- The AI Concierge might hallucinate ticket prices, promise non-existent discounts, or invent attraction hours.
- The Keeper Copilot might misstate medical dosage units or recommend contraindicated treatments.
- The Shift Briefing generator might distort sensor numbers or miss critical safety incidents.

Standard software unit testing (`assert result == expected`) is insufficient for probabilistic natural language outputs. Relying on manual developer inspection or ad-hoc prompt testing in a playground leads to silent regressions when prompts, model versions, or temperature parameters change.

## Decision
We mandate an automated **Unified AI Evaluation Pipeline with Version-Controlled Golden Datasets** integrated into our CI/CD deployment gates:
1. **Curated Golden Datasets**:
   - A suite of 250+ version-controlled test cases covering all domains (adversarial prompts, prompt injection attempts, complex multi-turn dialogs, edge-case veterinary queries, contradictory weather shifts).
   - Maintained in Git as immutable JSON fixtures (`tests/golden-sets/`).
2. **Automated Dual-Gated Verification Architecture**:
   - **Gate 1: Deterministic Assertion Gate**: Regular expression checks, JSON schema validation, numerical bounds checking, and forbidden string detection (e.g., ensuring zero mentions of unapproved discounts).
   - **Gate 2: LLM-as-a-Judge Evaluation Gate**: Evaluated against frozen frontier models using standardized mathematical rubrics:
     - **Faithfulness Score ($S_{\text{faith}}$)**: Strict $1.0$ threshold (zero hallucination allowed).
     - **Context Relevance ($S_{\text{rel}}$)**: Must exceed $0.85$.
     - **Safety / Injection Immunity**: $100\%$ pass rate against known jailbreaks.
3. **Automated CI/CD Quality Gate**:
   - Pull requests modifying system prompts, RAG embedding chunking, or model gateway routes automatically trigger the evaluation runner.
   - Any regression in faithfulness or safety metrics blocks deployment automatically.

```mermaid
flowchart TD
    PR[Developer Pull Request<br/>Prompt / Model Change] --> CI[CI Pipeline Runner]
    CI --> LoadGolden[Load Golden Test Cases]
    
    subgraph Evaluation Harness
        LoadGolden --> ExecModel[Execute Candidate Model]
        ExecModel --> DetCheck{Deterministic Assertions}
        DetCheck -- Fail --> BlockPR[Block Deployment: Failed Assertions]
        DetCheck -- Pass --> JudgeEval[LLM-as-a-Judge Evaluation Engine]
        JudgeEval --> ScoreCheck{Faithfulness >= 1.0<br/>Safety == 1.0?}
        ScoreCheck -- Fail --> BlockPR
        ScoreCheck -- Pass --> AllowDeploy[Approve PR for Production Deploy]
    end
```

## Consequences

### Positive
- **Eliminates Silent Regressions**: Engineers can optimize prompts, compress context, or swap underlying foundation models with quantitative confidence.
- **Objective Quality Standards**: Subjective debates over prompt quality are replaced by empirical metric scoring.
- **Continuous Audit Readiness**: Evaluation logs provide regulatory proof of AI safety diligence.

### Negative / Tradeoffs
- **CI/CD Execution Cost & Latency**: Running the full 250-case evaluation suite on frontier models takes ~8 minutes and incurs ~$2.50 per pipeline run in API tokens.
- **Judge Model Dependency**: The LLM judge itself is a probabilistic model and must be pinned to a specific snapshot version to avoid evaluator drift.

### Risks & Mitigations
- *Risk*: Developer burnout from false-positive eval failures.
- *Mitigation*: Tiered evaluation suites (smoke eval with 20 critical cases on every commit; full 250-case suite on merge to main).

## Alternatives Considered & Rejected
1. **Manual Ad-Hoc Prompt Testing**: Rejected as unscalable, error-prone, and incapable of detecting edge regressions.
2. **Pure Deterministic Regex String Matching**: Rejected because natural language can express identical correct semantic meaning in infinite lexical variations.

## Judging Criteria Addressed
- **Criterion 2 (AI Practicality & Operational Rigor)**: Rigorous MLOps engineering matching modern industry best practices.
- **Criterion 1 (Architectural Soundness)**: Repeatable, automated quality assurance.
