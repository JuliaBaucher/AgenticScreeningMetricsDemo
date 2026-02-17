# Agentic AI Screening Metric Demo  
AI-Powered Candidate Screening Workflow using AWS Step Functions + Lambda + Bedrock

---

## Overview

This project demonstrates a production-style **Agentic AI screening workflow** for high-volume frontline hiring.

It combines:

- Durable orchestration with **AWS Step Functions**
- Modular workers implemented as **AWS Lambda functions**
- Deterministic business rules for scoring
- LLM-powered structured extraction and rationale generation using **Amazon Bedrock (Anthropic Claude Sonnet 4 via Inference Profile)**
- S3-based data persistence
- A lightweight web UI (hostable on GitHub Pages)

The goal is not perfect hiring accuracy, but to demonstrate:

- Agentic orchestration patterns
- Hybrid AI (LLM + deterministic policy)
- Auditability and traceability
- Scalable screening architecture

---

## Purpose

High-volume hiring (e.g., hourly workforce, frontline roles) requires:

- Fast triage of applications  
- Clear scoring logic  
- Transparent decisions  
- Ability to handle incomplete submissions  
- Repeatable and auditable workflow  

This demo shows how to build a structured, production-ready screening pipeline using serverless AWS components.

---

## Context

Traditional screening systems suffer from:

- Manual CV reading  
- Inconsistent evaluation criteria  
- Black-box AI decisions  
- Poor traceability  
- Lack of governance  

This solution introduces:

- Deterministic scoring logic  
- Clear separation between extraction and policy  
- LLM only where it adds value  
- Full execution trace via Step Functions  

---

# Architecture

## High-Level Architecture

```
[ Web UI (HTML) ]
        |
        v
[ Lambda Function URL (API) ]
        |
        v
[ AWS Step Functions ]
        |
        +--> 4A Load JOb Context
        |
        +--> 4B Ingest Application event
        |
        +--> 4C Normalize & Dedupe
        |
        +--> 4D Structured Extraction (Bedrock)
        |
        +--> 4E Fit Scoring (Deterministic + Bedrock rationale)
        |
        +--> 4F Next Best actions
        |
        +--> 4G Update ATS
        |
        +--> 4H Manage communication with candidate
        |
        +--> 4I Schedule interviews
        |
        +--> 4J Output metrics 
        |
        +--> 4K Write Back to S3
```

---

## Design Principles

- Modular worker Lambdas
- Deterministic business scoring
- LLM only for extraction + explanation
- Clear separation of responsibilities
- Durable execution state
- Observable and auditable

---



## Business logic: Extraction, Scoring, and Decision Logic

The workflow begins with a **Structured Extraction** step powered by Claude (via Amazon Bedrock, temperature = 0), which converts the unstructured job description and candidate application into a strict JSON schema containing the following fields:

- `years_experience` (number)  
- `has_required_certification` (true/false)  
- `education_level` (string)  
- `skills` (array of strings)  
- `availability` (string)  
- `confidence` (number, model self-reported extraction reliability)

This structured output is passed to a fully deterministic **Fit Scoring** engine. The scoring rubric is explicitly defined as follows:

- **+40 points** if `years_experience >= 2`  
  - Otherwise: record missing item → `"Minimum 2 years experience"`
- **+30 points** if `has_required_certification == true`  
  - Otherwise: record missing item → `"Required certification"`
- **+20 points** if `availability` is confirmed  
  - Otherwise: record missing item → `"Availability confirmation"`
- **+10 bonus points** if `confidence > 70`

Maximum possible score: **100 points**

The final decision is rule-based:

- **Score ≥ 75 AND no missing required items → "Interview Scheduled"**
- **Score ≥ 40 → "Missing Information / Clarification Required"**
- **Score < 40 → "Rejected"**

All decisions are driven by explicit weighted criteria tied directly to job requirements, ensuring explainability, auditability, and compliance-ready behavior.

# End-to-End Flow

## 1. User Input (Web UI)

User pastes:

- Candidate CV (text)
- Candidate responses to screening questions
- Job description

The UI:

- Calls backend API (Lambda Function URL)
- Starts a Step Function execution
- Polls for completion
- Displays real results:
  - Application Status
  - Score
  - Confidence
  - Explanation
  - Missing Information (if applicable)
  - Execution ARN

---

## 2. Step Function Workflow

### Step 4A Load job context 

It prepares the evaluation configuration for the screening workflow. It receives the `jobDescriptionText`, `jobId`, and `locationId` from the Step Functions input and builds a structured `requirementsSchema` based on keywords detected in the job description (e.g., required certifications, must-have or nice-to-have skills). It also generates rule metadata such as `roleId`, `locationId`, `jdVersion`, and compliance flags (e.g., `eeoc_audit_enabled`). This step does not process candidate CV data; instead, it constructs the job-specific rules and versioning information that will be used later in deterministic scoring and decision-making, ensuring auditability and policy consistency.


### Step 4B Ingest Application event

It standardizes the raw candidate submission into a structured and consistent `application` object for downstream processing. It receives inputs such as `applicationId`, `cvText`, `screeningAnswersText`, `jobDescriptionText`, `jobId`, `locationId`, and `availability`, and organizes them into a normalized schema that the rest of the workflow can reliably consume. This step does not perform scoring, validation, or storage; its sole purpose is to ensure that all candidate data entering the orchestration pipeline follows a predictable structure, enabling clean handoffs to normalization, extraction, and evaluation stages.

### Step 4C — Normalize & Dedupe

It ensures that repeated submissions of the same application (even with minor formatting differences) can be consistently identified.

#### Normalization (Deterministic Text Canonicalization)

The system combines `cvText` and `screeningAnswersText`, then applies the following exact normalization logic:

`normalized = re.sub(r"\s+", " ", (cv + " " + answers).strip().lower())`

This performs:

- `strip()` → removes leading and trailing whitespace  
- `.lower()` → converts all characters to lowercase  
- `re.sub(r"\s+", " ", ...)` → replaces multiple spaces, tabs, and line breaks with a single space  

Example:

`"John DOE\n\n3 Years Experience"`  
→  
`"john doe 3 years experience"`

This ensures formatting differences (extra spaces, capitalization, line breaks) do not affect fingerprint generation.

---

#### Dedupe Key Generation (Deterministic Fingerprint)

After normalization, a SHA-256 hash of the normalized text is generated using:

`dedupe_key = hashlib.sha256(normalized.encode("utf-8")).hexdigest()[:16]`

Key characteristics:

- Uses SHA-256 hashing  
- Encodes text in UTF-8  
- Produces a hexadecimal string  
- Truncates to the first 16 characters for compactness  

Example output:

`a3f91c7b2e4d8f10`

Properties:

- Identical normalized content → identical dedupe key  
- Any textual change → completely different key  
- Fully deterministic  
- No LLM involvement  

---

#### Demo Mode Behavior

In the current demo implementation:

`"isDuplicate": False`

The system always reports the application as not duplicate, but still generates a realistic dedupe key.

In production, this key would be persisted (e.g., in DynamoDB) and checked before processing to prevent reprocessing identical applications.

---


### Step 4D — Structured Extraction (LLM)

It Uses **Amazon Bedrock (Claude Sonnet 4 via inference profile)** to extract structured JSON:

```json
{
  "years_experience": number,
  "has_required_certification": true/false,
  "education_level": "string",
  "skills": ["string"],
  "availability": "string",
  "confidence": number
}
```
Confidence is a model-generated reliability estimate produced during the structured extraction step, reflecting how certain the LLM is about the accuracy and completeness of the extracted fields based on clarity, explicitness, and consistency of the input data (e.g., a CV stating “Forklift certified, 3 years of warehouse experience (2019–2022)” would likely yield high confidence, whereas “Worked in logistics for a while and handled equipment” would likely yield lower confidence due to ambiguity).


Important:

- Temperature = 0
- Prompt enforces strict JSON output
- JSON is sanitized and parsed robustly
- Markdown fences removed automatically
- First JSON object extracted safely

This step transforms unstructured text into structured evaluation fields.

*In production, we would still use an LLM with temperature set close to 0 for stability and repeatability, but not as a decision-maker. The LLM’s role would be limited to semantic normalization of messy, unstructured CV data—extracting structured fields, inferring experience, and mapping skills—while all scoring and final decisions remain fully deterministic and rule-based for auditability and compliance. If inputs were already fully structured and standardized, an LLM would not be necessary; its value lies in handling variability, ambiguity, and semantic interpretation that traditional regex or rule engines cannot reliably manage.*

---

### Step 4E — Fit Scoring (Deterministic + LLM Rationale)

This is hybrid logic:

## Deterministic Score Calculation

Scoring logic:

| Criteria | Points |
|----------|--------|
| ≥ 2 years experience | +40 |
| Required certification present | +30 |
| Availability provided | +20 |
| High confidence (>70) | +10 |

Total possible score: 100

---

## Score Bands

- High ≥ 75
- Medium ≥ 40
- Low < 40

---

## Decision Logic (NextBestAction Step)

Based on score and missing fields:

- High + no missing → Interview Scheduled
- Medium → Missing Information / Clarification Required
- Low → Rejected

---

## Bedrock Rationale

LLM generates:

- Short explanation (max 4 sentences)
- Why score is what it is
- What is missing (if anything)

The LLM does NOT decide outcome.  
It only explains deterministic results.

This ensures:

- Governance
- Predictability
- No hallucinated decisions

---

### Step 4F — Next Best Action

This step applies a deterministic policy layer to translate the calculated `score`, `missingInfoList`, and `confidence` into a final operational decision. Based on explicit thresholds, it returns one of three outcomes: **Interview Scheduled**, **Missing Information / Clarification Required**, or **Rejected**. High scores (≥ 75) trigger an interview invitation; medium scores or missing must-have criteria trigger a clarification request; low scores or low confidence (< 0.6) result in rejection with a structured `rejectionReasonCode` and human-readable explanation. This step ensures that the final decision is rule-based, transparent, and auditable, with no opaque model-driven auto-rejections.

Maps score + missing info into:

- Final Decision
- Status label for UI
- Reason codes (optional)

### Step 4I Schedule Interviews (Simulated)

This step represents the interview scheduling integration layer. In production, this would connect to a scheduling platform or calendar system to create interview slots. In demo mode, it simply returns a simulated scheduling result while preserving the expected response contract. This maintains architectural completeness of the orchestration while keeping the system fully self-contained.

---

### Step 4J Output Metrics 

This step handles execution and funnel metrics. Execution reliability, duration, and throughput are derived directly from Step Functions execution history, while funnel outcomes (Approved / Missing Info / Rejected) are computed from decision artifacts stored in S3; the metrics Lambda aggregates this data in real time and exposes it via a Function URL, which the UI fetches and renders dynamically

---

### Step 4K — Write Back to S3

It persists the complete evaluation summary into the original application package stored in Amazon S3. It reads the existing object, appends a `final` section containing the `executionArn`, decision details, score, confidence, explanation, missing information, ATS status, communication results, scheduling outcome, and metrics, and then updates the application `status` to `COMPLETED`. This ensures full traceability, durable state storage, and audit-ready output for downstream systems or UI consumption.

Stores:

- Raw input
- Extracted structured fields
- Score
- Explanation
- Final decision
- Execution ARN

Ensures full auditability.

---

# LLM Integration Details

## Model

Anthropic Claude Sonnet 4  
Invoked via Bedrock Inference Profile:

```
arn:aws:bedrock:eu-west-3:<ACCOUNT_ID>:inference-profile/eu.anthropic.claude-sonnet-4-20250514-v1:0
```

Why inference profile?

- Required for newer Anthropic models
- Handles throughput routing
- Avoids on-demand invocation errors

---

## IAM Requirements

Lambda role must include:

- bedrock:InvokeModel
- bedrock:InvokeModelWithResponseStream
- AWS Marketplace permissions (first invocation only):
  - aws-marketplace:Subscribe
  - aws-marketplace:ViewSubscriptions
  - aws-marketplace:Unsubscribe

Once subscribed, invocation works account-wide.

---

# Key AWS Resources

- AWS Step Functions (STANDARD workflow)
- Lambda (multiple workers)
- Lambda Function URL (API endpoint)
- Amazon S3 (data persistence)
- Amazon Bedrock (Claude Sonnet 4)
- IAM roles (least privilege recommended)
- CloudWatch Logs (observability)

---

# Observability & Governance

- Every execution has unique ARN
- Full state history available in Step Functions
- Deterministic scoring ensures reproducibility
- LLM used only for:
  - Extraction
  - Explanation
- Final decision logic remains policy-driven

---

# Security Model

- No direct Bedrock access from frontend
- All LLM calls from backend Lambda
- IAM role scoped to required services
- No secrets stored in frontend
- S3 write-back for traceability

---

# UI Behavior

The UI:

- Shows animated step progression (cosmetic)
- Displays real backend results
- Supports:
  - Approved / Interview Scheduled
  - Missing Information
  - Rejected
- Displays:
  - Score
  - Confidence
  - Explanation
  - Execution ID

---

# Why This Is “Agentic”

This workflow demonstrates agentic principles:

- Orchestrated multi-step reasoning
- Task decomposition
- Structured intermediate memory
- Clear execution boundaries
- Hybrid AI + deterministic policy
- Next Best Action decisioning

It is not a single prompt.  
It is a stateful AI workflow.

---

# Scalability

The architecture supports:

- High concurrency (serverless)
- Adding new evaluation criteria
- Swapping models
- Replacing scoring logic
- Adding human-in-the-loop gates
- Integration with ATS systems

---

# Extensibility Ideas

- Add bias detection checks
- Add historical hiring data scoring
- Add human approval gate before scheduling
- Add feedback loop for model improvement
- Store embeddings for semantic search
- Add dashboard analytics

---

# What This Demo Proves

- You can combine Bedrock + deterministic rules safely
- You can build auditable AI hiring workflows
- You can use inference profiles correctly
- You can separate extraction from decision logic
- You can build a production-style orchestration layer

---

# Disclaimer

This is a demonstration system.  
It does not replace human hiring judgment.  
Real-world implementations must include:

- Legal review
- Bias auditing
- Equal opportunity compliance
- Data protection safeguards
- Human oversight

---

# Author

Designed as a demonstration of modern Agentic AI orchestration patterns  
Using AWS Serverless + Bedrock + deterministic governance logic.

---

# FAQ1: Example: Normalization, De-Duplication, and Semantic Extraction (John Doe Case)


| Step | What It Does | Example (John Doe) | Example Output Type |
|------|--------------|--------------------|--------------------|
| Ingest | Normalize structure (standardize schema and field names) | Wraps raw input into a consistent object:<br>`{ "applicationId": "123", "cvText": "JOHN DOE...", "screeningAnswersText": "Available for night shifts.", "jobId": "warehouse-role", "locationId": "paris-01" }` | Clean JSON object |
| Normalize / De-Dupe | Normalize content (canonicalize text + generate fingerprint) | `"JOHN DOE\n\nWorked in Warehouse..."` → `"john doe worked in warehouse..."`<br>Generates dedupe key: `a3f91c7b2e4d8f10` | Clean string + hash |
| LLM Extraction | Understand meaning (semantic parsing into features) | Extracts structured fields:<br>`{ "years_experience": 3, "has_required_certification": true, "skills": ["warehouse operations", "forklift"], "availability": "night shifts", "confidence": 82 }` | Structured evaluation fields |

#FAQ 2: Example: Multii-language support

| Step                        | What It Does                                                                                                                                                                                    | Example (Russian Inputs)                                                                                                                                                       | Example Output Type                                                                                                                                |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Load Job Context            | Reads the job description and builds a requirements schema (keyword-based in demo). With Russian JD, English keyword checks may not match, but job metadata and audit flags are still returned. | JD (RU): “Требуется складской работник… сертификат оператора вилочного погрузчика… ночные смены.” → English keyword checks (`forklift`, `warehouse`, `night`) may not trigger. | Structured job metadata:<br>`{ "jobId": "...", "locationId": "...", "requirementsSchema": { ... }, "jdVersion": "JD_DEMO_V1" }`                    |
| Ingest Application Event    | Standardizes the payload structure into a consistent `application` object. Language is preserved as-is.                                                                                         | CV (RU): “Иван Петров… 4 года опыта на складе…”<br>Answers (RU): “Готов работать в ночные смены.”                                                                              | Clean JSON object:<br>`{ "application": { "cvText": "<RU text>", "screeningAnswersText": "<RU text>", "jobId": "...", "locationId": "..." } }`     |
| Normalize & De-Dupe         | Canonicalizes text (lowercase + whitespace normalization) and generates a deterministic SHA-256 fingerprint to detect duplicates. Works regardless of language.                                 | `"ИВАН ПЕТРОВ\n\n4 ГОДА ОПЫТА"` → `"иван петров 4 года опыта"`<br>Generates dedupe key: `a3f91c7b2e4d8f10`                                                                     | Clean string + hash:<br>`{ "dedupeKey": "a3f91c7b2e4d8f10", "isDuplicate": false }`                                                                |
| Structured Extraction (LLM) | Uses Bedrock / Claude to semantically interpret Russian text and map it into a fixed JSON schema with English field names.                                                                      | From RU CV + answers:<br>“4 года опыта… сертификат погрузчика… готов к ночным сменам.”                                                                                         | Structured evaluation fields:<br>`{ "years_experience": 4, "has_required_certification": true, "availability": "night shifts", "confidence": 88 }` |
| Fit Scoring (Deterministic) | Applies numeric and boolean checks on extracted fields (language-agnostic) and computes score + decision based on thresholds.                                                                   | Extracted: `4 years`, `certification=true`, `availability confirmed`, `confidence=88` → Score = 100 → Interview Scheduled                                                      | Final decision object:<br>`{ "score": 100, "scoreBand": "high", "finalDecision": "Interview Scheduled" }`                                          |


# 30-Minute Oral Presentation Script (with live demo)

For this interview challenge, I designed and built a working prototype of an **agentic screening system** for high-volume hourly hiring. The goal is to show a production-style workflow that can handle **10,000+ applications per week across 50+ locations**, triage candidates quickly, remain **auditable and compliant (EEOC / fair hiring)**, and preserve a strong candidate experience — while acknowledging recruiters cannot manually review every application.

### 1) Problem framing and what “good” looks like
In this context, the system must do four things well:  
First, **speed** — reduce time-to-first-decision so qualified candidates move quickly through the funnel.  
Second, **consistency** — apply the same criteria every time to avoid subjective drift across locations.  
Third, **compliance and auditability** — every decision must be explainable with clear reason codes and an execution trace.  
Fourth, **candidate experience** — if information is missing, we should request clarification instead of rejecting prematurely.

### 2) What the agent has access to (and what I implemented in the demo)
The problem statement mentions access to resumes, job descriptions, historical hiring outcomes, screening answers, scheduling, and an ATS. In this prototype, I implemented the realistic structure of those integrations while keeping the demo lightweight:
- The system ingests **CV text**, **screening answers**, and the **job description**.
- It runs **structured extraction** using an LLM to transform unstructured text into a consistent schema.
- It performs **deterministic fit scoring** and a **policy-based next best action** decision.
- It includes **stub integrations** for ATS update, candidate communications, and scheduling — returning realistic contract-compliant responses without calling external systems.
- Finally, it writes a complete **audit package** to S3 so the output is durable and reviewable.

### 3) System design and high-level architecture
At a high level, the system has three layers:
1) A lightweight **web UI** where a user pastes the CV, answers, and job description, and starts a run.  
2) A serverless orchestration backend based on **AWS Step Functions**, with modular **Lambda workers** per step.  
3) An observability and audit layer using **Step Functions execution history**, **CloudWatch logs/metrics**, and **S3 write-back** for final outcomes.

The key design decision is that this is not a single prompt. It’s a **stateful multi-step workflow** that creates structured intermediate artifacts, applies deterministic policies, and logs everything for governance.

### 4) Walkthrough of the workflow steps (what happens after I click “Start”)
When I submit an application, Step Functions executes a sequence of worker steps:

- **Load Job Context:** prepares job rules and a requirements schema derived from the job description, with versioning and audit flags.  
- **Ingest Application Event:** normalizes the payload into a consistent `application` object for downstream steps.  
- **Normalize & Dedupe:** canonicalizes the text and generates a deterministic dedupe key, which is how you would avoid reprocessing duplicates at scale.  
- **Structured Extraction (LLM):** uses Bedrock / Claude to extract a strict JSON schema like years of experience, certification presence, skills, availability, and a confidence estimate (based on user inputs understanding). Temperature is set close to 0 for stable outputs.  
- **Fit Scoring (Deterministic):** assigning points for experience, required certification, availability, and a confidence factor. Final application status is determined by rule-based thresholds, accompanied by an LLM-generated summary explaining the outcome. This is important: the model does not decide the outcome — scoring is deterministic and auditable.  
- **Next Best Action:** converts the score + missing info into one of three operational outcomes: interview scheduled, missing information request, or rejection. If rejected, it returns a structured **rejection reason code**.  
- **ATS / Comms / Scheduling (Simulated):** these are stub steps that keep the contracts realistic without external dependencies.  
- **Write Back to S3:** stores all outcomes (decision, score, reasons, execution ARN) in the application package for auditability.
- **Output Metrics (execution and funnel)** are computed in real time from Step Functions execution history and S3-stored decision data

### 5) Why this approach is compliant and production-oriented
A major risk in hiring systems is “black box” decisioning. In this prototype:
- The **LLM is constrained to extraction and explanation**, not eligibility decisions.  
- The **decision logic is deterministic** and mapped to explicit requirement checks.  
- Every rejection includes a **reason code** and a human-readable explanation aligned to the rubric.  
- Step Functions provides an immutable execution trace, and S3 stores the final package to support audits and reviews.

### 6) Candidate experience design
Instead of rejecting on missing information, the policy explicitly supports “Missing Information / Clarification Required.” This improves conversion, reduces false negatives, and matches how recruiters work in reality. Even in demo mode, communications and scheduling are modeled so we preserve the end-to-end funnel logic.

### 7) Prompting approach and iteration
I used prompting in two places:
- **Structured extraction prompt:** designed to output strictly valid JSON with a defined schema so it’s machine-consumable and safe.  
- **Rationale generation prompt:** produces a short explanation that references the deterministic scoring outcome and missing items, capped to a few sentences.

During iteration, the key improvements were enforcing strict JSON output, adding robust parsing, and keeping the LLM out of the decision boundary so the system remains predictable.

### 8) Demo (screen share)
Now I’ll show the prototype. I will:
1) Paste a sample job description and candidate application.  
2) Click “Start Demo” to launch an execution.  
3) Show the live status while the workflow runs.  
4) When completed, review the displayed output: execution ARN, structured extraction, score band, decision, and rejection code if applicable.  
5) Optionally open Step Functions to show the execution graph and the intermediate state transitions for auditability.

### 9) Success metrics (what we’d measure in production)
In production, the key success metrics would be:
- **Time to first decision** and end-to-end screening latency  
- **Throughput** (applications processed per hour/day)  
- **Success/failure rate** and error categories  
- **Conversion rates**: invite-to-schedule, schedule completion  
- **Human review rate** (bounded)  
- **Fairness / compliance monitoring**: distribution of outcomes across locations and cohorts, plus audit sampling

In this demo, I included simulated metrics to show how the KPI layer plugs in without building a full analytics pipeline.

### 10) Roadmap: what’s built vs what’s next
What’s built today is a complete end-to-end workflow with durable orchestration, deterministic decisioning, compliance-friendly outputs, and realistic integration stubs.  
Next steps for a production rollout would be:
- Replace stubs with real ATS, comms, and scheduling integrations  
- Add historical hiring outcome signals and calibration of scoring thresholds  
- Add human-in-the-loop gates for edge cases and low-confidence extractions  
- Add automated fairness and bias monitoring dashboards  
- Add scalable ingestion (S3/SQS) for true high-volume asynchronous processing

### Closing
To summarize: this prototype demonstrates a practical agentic screening architecture that is fast, modular, auditable, and designed for high-volume hourly hiring. The key is the separation of concerns: **LLM for semantic extraction**, **deterministic scoring and policy for decisions**, and **Step Functions for traceability and governance**. I’m happy to dive deeper into any component or walk through specific execution traces.

# Presentation Script 
For this interview challenge, I designed and built a working prototype of an agentic screening system for high-volume hourly hiring. The goal is to demonstrate a production-style workflow that can handle 10,000+ applications per week across 50+ locations, identify qualified candidates and move them through the funnel, comply with EEOC and fair hiring regulations, and preserve a strong candidate experience — while acknowledging that recruiters cannot manually review every application.

Problem
The challenge is to design an architecture that handles volume, speed, compliance, and user experience (for both recruiters and candidates).

How it works:
The user enters the job description, candidate CV, and answers to interview questions in the UI, and the system evaluates the application and provides a final decision: approved for interview, missing information, or rejected with a rejection reason. To facilitate the demo, there are predefined examples for each scenario. The system also returns execution and business metrics.

I selected a serverless architecture with a lightweight front end hosted on GitHub and an AWS-based backend with storage on S3, orchestration through Step Functions, and LLM integration via Bedrock.

Load Job Context: prepares job rules and a requirements schema derived from the job description, including versioning and audit flags.
Ingest Application Event: normalizes the payload into a consistent application object for downstream steps.
Normalize & Dedupe: canonicalizes the text and generates a deterministic dedupe key, which is how duplicates would be avoided at scale.
Structured Extraction (LLM): uses Bedrock / Claude to extract a strict JSON schema including years of experience, certification presence, skills, availability, and a confidence estimate (based on the model’s understanding of user inputs). Temperature is set close to 0 for stable outputs.
Fit Scoring (Deterministic): assigns points for experience, required certification, availability, and a confidence factor. The final application status is determined by rule-based thresholds, accompanied by an LLM-generated summary explaining the outcome. This is important: the model does not decide the outcome — scoring is deterministic and auditable.
Next Best Action: converts the score and missing information into one of three operational outcomes: interview scheduled, missing information request, or rejection. If rejected, it returns a structured rejection reason code.
ATS / Comms / Scheduling (Simulated): stub steps that keep contracts realistic without external dependencies.
Write Back to S3: stores all outcomes (decision, score, reasons, execution ARN) in the application package for auditability.
Output Metrics: execution and funnel metrics are computed in real time from Step Functions execution history and S3-stored decision data.

This approach is compliant and production-oriented. It handles high volume thanks to AWS serverless architecture and supports multiple languages through LLM-based extraction. It provides deterministic evaluation of candidates based on scoring, and any rejection is accompanied by a structured reason code to comply with EEOC standards. The architecture anticipates integration with external systems for candidate communication, ATS updates, and interview scheduling.

It is aligned with orchestration principles: execution authority through predefined steps; clear boundaries for LLM usage (the LLM is constrained to specific steps); durable memory and state management using S3 and Step Functions; fully traceable, controlled autonomy with the final decision available for human oversight; and full measurability with execution and business metrics available in the UI and in CloudWatch.

Next steps: integrate external systems (scheduling, candidate communication, ATS), support voice input, implement richer business logic, add SQS for burst handling, introduce a database for configuration and persistence, and incorporate analytics tools.

