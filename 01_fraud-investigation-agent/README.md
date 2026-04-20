# UC1 - Real-Time Fraud Investigation Agent


---

## The Use Case

Financial institutions process hundreds of millions of transactions daily. When a transaction is flagged as potentially fraudulent, the window to act is narrow — often seconds to minutes before further damage occurs or funds become irrecoverable.

Banks need to move from *detecting* fraud to *responding* to it — end-to-end, fast, and at scale. That means: retrieving transaction history, assessing risk, deciding on action (freeze, notify, escalate), communicating with the customer, and closing the case with a full audit trail — all within a timeframe that actually protects the customer.

Today this is largely a manual, analyst-driven process:
- A rules engine flags a transaction
- An analyst reviews the alert, looks up history, cross-references patterns
- The analyst decides whether to freeze the card and contacts the customer
- A case is opened and documented manually

This workflow exists across virtually every major financial institution and is the industry baseline.

---

## The Challenges

The manual approach has well-documented limitations:

- **Speed**: Manual investigation of a single alert takes 45 minutes to 3+ hours for a genuine case — far too slow when fraud windows are measured in minutes ([source](https://www.roe-ai.com/blog/the-cognitive-cost-of-manual-aml-investigations))
- **Scale**: Banks typically process 15,000–50,000 fraud/AML alerts per month; manual review costs $25–75 per case ([source](https://www.finantrix.com/glossary/design-a-manual-review-queue-for-aml-alerts))
- **False positive overload**: 85–95% of alerts from rule-based systems are false positives, causing analyst fatigue and inconsistent decisions ([source](https://www.facctum.com/blog/aml-false-positive-report))
- **Rules engines don't adapt**: Static thresholds can be probed by fraudsters; updating rules takes weeks, by which time attack patterns have evolved ([source](https://thecondia.com/ai-fraud-detection-banks/))
- **Inconsistency**: Decisions vary by analyst, shift, and workload — creating compliance risk and uneven customer experience
- **Audit and compliance burden**: Regulatory requirements (PCI-DSS, GDPR, BSA) demand a complete, traceable record of every decision — which is hard to guarantee in a manual process
- **Not scalable**: 70% of financial crime compliance labor costs go to investigators; adding headcount is not a sustainable answer ([source](https://www.niceactimize.com/blog/aml-improving-financial-investigations-essential-strategies-for-success/))

---

## The Solution

For this use case we plan to design and use a multi agentic solution which automates most of the tasks which are currently done manually by fraud/transactions analysts.

Solution' objective is to provide an autonomous fraud investigation system that handles the full response lifecycle, while keeping a human in the loop where required.

Another key requirement is to help Financial Institution comply with regulatory requirements, hence all key actions, either automated or manual, are automatically tracked for later review (third party audits, internal reviews, troubleshooting/improvement).

At high level, the following is the envisioned workflow:

- **A.** Solution is triggered by either an automated rule (e.g. a transaction exceeds a threshold or originates from a high-risk country) or a direct human action (e.g. a customer flags a suspicious charge via the bank app).
- **B–C.** Solution autonomously retrieves and analyses the customer transaction history, cross-references known fraud patterns, and computes a risk score.
- **D.** Solution takes action based on the risk level — automatically for high-risk or low-risk cases, or requesting human approval on medium-risk cases.
- **D-HIGH → F–H.** If high risk, solution freezes the card, notifies the customer, and records the full case with an audit trail.
- **D-LOW → I.** If low risk, the event is logged and no further action is taken.
- **D-MED → E.** If medium risk, human review is required to properly classify the transaction. 

Solution is designed with **human-in-the-loop** pattern for medium-risk cases: the system pauses, notifies a fraud analyst with a case summary, and waits until the analyst review and classify the transaction. Then agentic system resume and follows high-risk path or low-risk path based on fraud analyst decision.

---

## Why AgentCore

AgentCore provides the production-grade infrastructure layer that makes this solution possible at scale. Without it, a team would need to build and maintain: secure agent deployment and scaling, session isolation between concurrent investigations, persistent memory across interactions, authenticated access to downstream APIs, policy enforcement for automated vs human-gated actions, and a full observability stack for audit and operations. AgentCore handles all of this out of the box.

### AgentCore Capabilities Required

| Capability | Why it is needed |
|---|---|
| **Runtime** | Deploys and scales the multi-agent system; provides true session isolation so each fraud case runs in its own secure context; supports async extended runtime (up to 8h) for the human approval pause/resume pattern |
| **Memory** | Short-term memory maintains full investigation context within a session; long-term memory persists customer fraud history and pattern data across sessions, making the agent smarter over time |
| **Gateway** | Converts internal APIs (transaction history, card management, fraud pattern DB) and notification services into MCP-compatible tools the agents can call securely |
| **Policy** | Cedar rules enforce business logic deterministically - which risk levels trigger automatic action, which require analyst approval, and what the maximum autonomous freeze amount is |
| **Identity** | Manages agent authentication to each downstream system (card API, notification services, databases) using existing identity providers (Cognito, Okta, IAM) with least-privilege access |
| **Observability** | Traces every agent decision and action via OpenTelemetry to CloudWatch, producing a complete audit trail required for regulatory compliance (e.g. PCI-DSS, GDPR) |

### Human in the Loop

This solution includes an explicit human approval gate for medium-risk cases. This is not a fallback - it is a deliberate design choice that reflects real-world fraud operations, where fully automated decisions on ambiguous cases carry regulatory and reputational risk.

The flow is:
- The Orchestrator Agent receives the risk score from the Fraud Analysis Agent
- AgentCore Policy evaluates the score against Cedar rules
- If the risk is MEDIUM, the agent pauses execution and triggers a notification to the fraud analyst (email via SES with a case summary and dashboard link)
- The fraud analyst reviews the case on an internal dashboard and approves or rejects the freeze
- The agent resumes from the exact point it paused, with the analyst decision recorded in the audit trail
- If the risk is HIGH or LOW, the AI agent proceeds autonomously, with no further human involvement required.

This pattern demonstrates AgentCore capability to pause a long-running agent workflow, hand off to a human, and resume when human feedback is received, all without polling, timeouts, or additional infrastructure complexity.

---

## Multi-Agent Architecture

This solution uses 5 specialised agents. The Orchestrator delegates to peer/sub-agents rather than executing everything itself - this is the core AgentCore multi-agent pattern being showcased.

| Agent | Type | Responsibility |
|---|---|---|
| Orchestrator Agent | Orchestrator | Receives trigger events, manages investigation lifecycle, enforces policy rules, coordinates all other agents, owns the async pause/resume for human approval |
| Fraud Analysis Agent | Peer agent | Independently analyses the transaction: fetches history, cross-references fraud patterns, computes risk score, returns structured result to Orchestrator |
| Card Freeze Agent | Sub-agent | Executes the card freeze action via Card Management API, confirms freeze status, reports back to Orchestrator |
| Notification Agent | Sub-agent | Handles all outbound communications - customer email, customer push notification, fraud analyst email with dashboard link |
| Case Management Agent | Sub-agent | Writes the full investigation record with audit trail; optionally integrates with external case management systems |

The **Fraud Analysis Agent** is designed as a peer agent because in production it could be independently deployed and reused across other use cases (loan underwriting, account opening). This makes this solution more decoupled and open to extension to other use cases.

---

## Functional Architecture

```mermaid
flowchart TD
    A["A. Trigger Event"] --> B["Event Type?"]
    B -->|Human Request| C["User submits fraud report\nvia App or API"]
    B -->|Automated Rule| D["Transaction monitoring\ndetects anomaly"]
    C --> E["B. Orchestrator Agent\nreceives event and session context"]
    D --> E
    E --> F["C. Fraud Analysis Agent\nfetches transaction history\ncross-references fraud patterns\ncomputes risk score"]
    F --> G["D. Risk Level?"]
    G -->|Low| H["D-LOW. Log event\nNo action taken"]
    G -->|Medium| I["D-MED. PAUSE - await human approval\nAsync wait / extended runtime"]
    G -->|High| K["D-HIGH. AUTO - proceed to card freeze"]
    I --> J["E. Notify Fraud Analyst\nEmail with case summary\nand dashboard link"]
    J --> J2["E. Fraud Analyst reviews case\non internal dashboard"]
    J2 -->|Approved| K
    J2 -->|Rejected| H
    K --> L["F. Card Freeze Agent\nexecutes freeze via\nCard Management API"]
    L --> M["G. Notification Agent\nnotifies customer\nEmail and Push Notification"]
    M --> N["H. Case Management Agent\nwrites case record\nwith full investigation trail"]
    N --> O["External Case Mgmt?"]
    O -->|Integrated| P["Open case in\nexternal system"]
    O -->|Standalone demo| Q["Save case to DynamoDB"]
    H --> R["I. End\nAudit log written"]
    P --> R
    Q --> R
```

### Flow Description

| Step | Actor | Description |
|---|---|---|
| Trigger | EventBridge / API Gateway | Event fired by automation rule or human report |
| Orchestrator Agent | AgentCore Runtime | Receives event, initialises session, coordinates all agents |
| Fraud Analysis Agent | AgentCore Runtime | Fetches tx history, checks fraud patterns, scores risk |
| Human Approval (medium risk) | Fraud Analyst | Agent pauses async; analyst notified via email, reviews on dashboard, approves or rejects |
| Card Freeze Agent | AgentCore Runtime | Calls Card Management API to freeze card |
| Notification Agent | AgentCore Runtime | Sends email + push notification to customer |
| Case Management Agent | AgentCore Runtime | Writes full case record; optionally integrates with external CRM |
| Audit Log | AgentCore Observability | Every step traced and stored for regulatory trail |

---

## Agent Interaction

> Node letters (A, B, C...) match the functional architecture diagram steps.
> Arrow numbers show the order of calls.

```mermaid
flowchart TD
    A["A. Trigger\nEventBridge / API Gateway"]
    B["B. Orchestrator Agent"]
    C["C. Fraud Analysis Agent\npeer"]
    D["D. AgentCore Policy\nCedar Rules"]
    E["E. Fraud Analyst\nHuman in the Loop"]
    F["F. Card Freeze Agent"]
    G["G. Notification Agent"]
    H["H. Case Mgmt Agent"]
    I["I. End\nAudit log written"]

    A -->|"1"| B
    B -->|"2"| C
    C -->|"3 - risk score"| B
    B -->|"4"| D
    D -->|"5a - LOW"| I
    D -->|"5b - MEDIUM"| E
    D -->|"5c - HIGH"| F
    E -->|"6 - approved"| F
    F -->|"7"| G
    B -->|"8"| H
    H --> I
```

---

## Fraud Analysis Agent - Security and Observability

> This diagram shows how AgentCore Gateway, Identity and Observability work
> for the Fraud Analysis Agent. The same pattern applies to all agents.

```mermaid
flowchart LR
    subgraph Agent["C. Fraud Analysis Agent"]
        FA["Fraud Analysis Agent\nreceives task from Orchestrator"]
    end

    subgraph ID["AgentCore Identity"]
        IDN["Authenticates agent\nCognito / Okta / IAM\nLeast-privilege token per API"]
    end

    subgraph GW["AgentCore Gateway - MCP Tools"]
        GW1["Transaction History Tool\nLambda + DynamoDB"]
        GW2["Fraud Pattern Tool\nLambda + S3"]
    end

    subgraph OB["AgentCore Observability"]
        OBS["OpenTelemetry trace\nCloudWatch\nEvery call logged"]
    end

    FA -->|"1 - request auth token"| IDN
    IDN -->|"2 - token issued"| FA
    FA -->|"3 - call tool"| GW1
    FA -->|"4 - call tool"| GW2
    GW1 -->|"5 - result"| FA
    GW2 -->|"6 - result"| FA
    FA -.->|"trace emitted"| OBS
    GW1 -.->|"trace emitted"| OBS
    GW2 -.->|"trace emitted"| OBS
```

---

## AWS Services

| AgentCore Component | AWS Service | Role |
|---|---|---|
| Runtime | AgentCore Runtime | Hosts all 5 agents with session isolation and async pause/resume |
| Memory | AgentCore Memory | Short-term session context + long-term fraud history |
| Gateway - Transaction History | Lambda + DynamoDB | Fetches customer transaction data (synthetic for demo) |
| Gateway - Fraud Patterns | Lambda + S3 | Cross-references known fraud patterns (mock rules for demo) |
| Gateway - Card Management | Lambda | Executes card freeze (mock for demo, real API in production) |
| Gateway - Email | Amazon SES | Notifies customer and fraud analyst |
| Gateway - Push | Amazon SNS | Push notification to customer mobile app |
| Gateway - Case Mgmt | Lambda + DynamoDB | Writes case record (optional: Salesforce / CRM adapter) |
| Policy | AgentCore Policy | Cedar rules enforcing freeze thresholds and approval gates |
| Identity | Amazon Cognito / IAM | Agent authentication to downstream APIs |
| Observability | AgentCore Observability + CloudWatch | OpenTelemetry traces, full audit trail |
| Analytics | Amazon QuickSight | Fraud case dashboard and trend analysis |
