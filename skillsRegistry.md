---
registry_name: samar-enterprise-conversational-ai-registry
version: "2.0.0"
architect: Samar Naqvi
domain: Enterprise Conversational AI, IVR & Middleware Orchestration
runtime_platforms:
  - Dialogflow CX (Flows, Playbooks, Generators)
  - Middleware Orchestration Layer (REST / GraphQL / OAuth)
  - Python / Jupyter Solutioning Workbench
---

# Enterprise Conversational AI & Architecture Skills Registry

A centralized routing and contract manifest mapping conversational intents, modular sub-bots (Playbook-to-Playbook / Bot-to-Bot), and architecture discovery workflows to normalized middleware endpoints.

---

## 1. Quick-Lookup Dispatch Table

| Skill ID | Domain / Channel | Primary Trigger / Intent | Orchestration Pattern | Entry Point / Path |
| :--- | :--- | :--- | :--- | :--- |
| `enterprise-payment-skill` | Voice / IVR / Chat | Make a payment, pay bill, update card | `Main Bot → Payment Bot → Isolated IVR → Payment Bot → Main Bot` | `skills/payments/payment-skill.md` |
| `agent-handoff-skill` | Omnichannel (Voice/Chat/SMS) | Speak to an agent, escalate, routing check | Context-aware dynamic routing (EWT, hours, profile) | `skills/routing/agent-handoff-skill.md` |
| `scheduled-callback-skill` | Voice / Playbooks | Schedule a callback, pick time preference | Conversational Playbook + Scheduling API | `skills/scheduling/callback-playbook.md` |
| `bh-prior-auth-skill` | Healthcare Voice / Chat | Check prior authorization, coverage status | Middleware DTO + Member/Provider scoping | `skills/healthcare/prior-auth-skill.md` |
| `onepass-gym-search-skill` | Healthcare Member Portal / Bot | Find a gym, verify facility access near ZIP | Batched GraphQL + Location/Access resolution | `skills/healthcare/onepass-search-skill.md` |
| `api-discovery-workbench` | Architecture / DevOps | Parse OpenAPI/Postman, capture runtime evidence | Python / Jupyter / Pandas automated inventory | `skills/architecture/api-workbench.md` |

---

## 2. Conversational & Sub-Bot Skill Contracts

### 2.1 `enterprise-payment-skill`
* **Purpose:** Secure, PCI-isolated payment collection and token-based processing across conversational channels without exposing sensitive card data to the primary NLU/LLM context.
* **Orchestration Flow:**
  1. Receive payment context (`sessionId`, `accountId`, `amountDue`, `returnContext`) from `Main Bot`.
  2. Transfer caller to **Isolated Payment IVR** for DTMF/PCI card capture.
  3. Receive IVR-generated payment token; invoke normalized payment processing API.
  4. Return transaction status and restore `returnContext` back to `Main Bot`.
* **Input Parameters:**
  * `member_id` (string, required)
  * `balance_amount` (number, required)
  * `originating_flow_id` (string, required)
* **Failure / Fallback Policy:** On gateway timeout or token mismatch, retry once; on second failure, trigger `agent-handoff-skill` with `reason: PAYMENT_GATEWAY_ERROR`.

---

### 2.2 `agent-handoff-skill`
* **Purpose:** Intelligent, multi-factor escalation router evaluating real-time contact center telemetry and caller context before selecting an escalation treatment.
* **Decision Inputs:**
  * `caller_intent` & `sentiment_flag`
  * `estimated_wait_time` (EWT in seconds)
  * `business_hours_status` (boolean)
  * `customer_tier` / profile attributes
* **Supported Routing Outcomes:**
  1. `IMMEDIATE_AGENT_TRANSFER` (within business hours, low EWT, or high-priority tier)
  2. `IMMEDIATE_CALLBACK` (high EWT, preserve queue position)
  3. `SCHEDULED_CALLBACK` (delegates to `scheduled-callback-skill`)
  4. `ASYNC_CHAT_OR_SMS` (deflect voice to messaging channel)
  5. `CALL_LATER_TREATMENT` (after-hours informational deflection)

---

### 2.3 `scheduled-callback-skill`
* **Purpose:** LLM Playbook-driven appointment scheduling replacing rigid deterministic top-3 slot menus with natural language preference negotiation.
* **Capabilities:**
  * Parses natural time constraints (e.g., *"tomorrow afternoon after 2 PM"*, *"Friday morning"*).
  * Queries broader enterprise scheduling availability and filters matching windows.
  * Confirms timezone, callback number, and books slot via REST webhook.
* **Guardrails:** Must confirm callback ANI (phone number) explicitly before committing the booking mutation.

---

### 2.4 `bh-prior-auth-skill` (Behavioral Health Prior Authorization)
* **Purpose:** Abstracts complex healthcare backend schemas into normalized DTOs for checking Behavioral Health prior authorization status.
* **Middleware Normalization:**
  * Resolves `member_context` and `provider_npi` scoping.
  * Shields Dialogflow CX from raw backend status codes by mapping them to standardized conversational states (`APPROVED`, `PENDING_CLINICAL_REVIEW`, `ACTION_REQUIRED`, `NOT_FOUND`).

---

### 2.5 `onepass-gym-search-skill`
* **Purpose:** High-performance conversational facility lookup and member eligibility verification.
* **Performance & Integration Contract:**
  * Resolves ZIP/geolocation coordinates to candidate fitness facilities.
  * Executes **batched GraphQL queries** to validate member-specific access across multiple facilities in a single round-trip (target SLA: `< 450 ms` for up to 25 facility validations).
  * Returns normalized facility cards (name, distance, access tier, eligibility boolean).

---

## 3. Architecture & Discovery Tooling Skills

### 3.1 `api-discovery-workbench`
* **Purpose:** Automated solution discovery, runtime API validation, and evidence-backed High-Level Design (HLD) artifact generation.
* **Tech Stack:** Python, Jupyter Notebook, Pandas, OpenAPI/GraphQL Parsers, Splunk Log Exports.
* **Workflow:**
  1. Ingest OpenAPI specs, GraphQL schemas, or Insomnia/Postman collections.
  2. Execute controlled runtime validation probes against Stage endpoints to verify documented vs. actual behavior.
  3. Export structured API inventories, Source-to-Target mapping sheets, and latency benchmarks to Excel/Markdown.

---

## 4. Governance & Design Principles
1. **Channel Shielding:** Conversational agents must never consume raw enterprise backend payloads directly; all responses pass through normalized middleware DTOs.
2. **Evidence-Driven Contracts:** Every registered endpoint must include verified Stage runtime latency and payload evidence before production promotion.
3. **Graceful Degradation:** Every sub-bot skill must define deterministic fallback triggers (`MAX_NO_INPUT`, `MAX_NO_MATCH`, `API_5XX`) routed cleanly to `agent-handoff-skill`.
