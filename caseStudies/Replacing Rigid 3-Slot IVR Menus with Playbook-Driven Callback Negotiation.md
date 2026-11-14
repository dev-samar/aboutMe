# Case Study: Replacing Rigid 3-Slot IVR Menus with Playbook-Driven Callback Negotiation

**Domain:** Enterprise Contact Center & Generative Conversational AI  
**Role:** Conversation Architect / Module Lead  
**Core Impact:** Transitioned scheduled callbacks from a rigid **top-3 deterministic slot menu** to an **LLM Playbook negotiation flow** capable of filtering dozens of real-time API slots across multiple days—without hallucinating unavailable times.

---

## 1. The Business Problem
In traditional voice bots, scheduling a callback is a frustrating, robotic experience. Even when a company's scheduling system has **dozens of open slots across multiple days**, deterministic IVR flows can typically only read out the *first three available times* (e.g., *"Press 1 for 2:00 PM, Press 2 for 2:15 PM, Press 3 for 2:30 PM"*).

If none of those three times fit the customer's schedule—say, they want *"sometime tomorrow afternoon after 3 PM"* or *"Friday morning"*—the rigid flow hits a dead end, forcing the customer to either hang up or wait on hold for a human agent just to book a time.

---

## 2. The Architectural Challenge
Simply plugging an LLM into a scheduling API creates a dangerous risk: **slot hallucination** or **context overload**.
* If you dump 40+ raw API timestamps into a prompt unguarded, the bot becomes slow and verbose.
* If the LLM isn't strictly bounded to the API's live inventory, it might agree to a time the caller asks for even when no agent is actually available.

---

## 3. The Solution: Bounded Playbook Negotiation + Middleware Filtering
I re-architected the callback scheduler using **Dialogflow CX Playbooks** paired with a **Normalized Scheduling DTO**:

* **Broad Multi-Day Inventory Fetch:** Instead of truncating the scheduling API to 3 results, the middleware fetches the full multi-day availability window (tens of slots across *N* days) and normalizes them into a clean, compact structure (`Date`, `DayOfWeek`, `TimeWindow`, `SlotID`).
* **Natural Language Constraint Negotiation:** The Playbook allows the caller to speak naturally (*"Do you have anything Thursday afternoon?"* or *"Nothing before 11 AM tomorrow"*). The Playbook filters the live inventory against the caller's preferences and proposes 2–3 matching options conversationally.
* **Strict Grounding & API Locking:** The Playbook is instructed to **never invent times** outside the provided `SlotID` array. Once the caller agrees on a window, the Playbook maps their natural-language confirmation back to the exact `SlotID` and locks the booking via the scheduling API.

---

## 4. Before vs. After Architecture Flow

```mermaid
flowchart TD
    %% Before
    subgraph Before["BEFORE: Rigid Deterministic Flow (Top-3 Bottleneck)"]
        A1[Scheduling API returns 40+ slots] --> B1[Truncate to First 3 Slots]
        B1 --> C1[Read 3 Fixed Options to Caller]
        C1 --> D1{Caller Accepts One?}
        D1 -->|No| E1[Dead End / Agent Escalation]
        D1 -->|Yes| F1[Book Slot]
    end

    %% After
    subgraph After["AFTER: Playbook-Driven Slot Negotiation (Multi-Day Scale)"]
        A2[Scheduling API returns 40+ slots across N days] --> B2[Middleware Normalizes into Compact Slot DTOs]
        B2 --> C2[DFCX Playbook: Grounded Slot Negotiator]
        C2 <-->|Caller states preferences: 'Thursday after 2 PM'| D2[Dynamic Filtering & Conversational Turn]
        D2 --> E2{Valid SlotID Selected?}
        E2 -->|No Match Found| F2[Propose Closest Verified Alternatives]
        F2 --> D2
        E2 -->|Confirmed| G2[Lock Exact SlotID via Booking API]
    end
