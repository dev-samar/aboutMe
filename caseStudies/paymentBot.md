# Case Study: Bank-Grade Payment Security via Isolated IVR Tokenization

**Domain:** Enterprise Telecom & Conversational AI  
**Role:** Conversation Architect / Module Lead  
**Core Impact:** Completely isolated PCI/credit card data from the AI layer while preserving seamless conversational context across **Main Bot → Payment Bot → Isolated IVR → Main Bot**.

---

## 1. The Business Problem
When customers call an AI voice assistant to pay a bill or set up recurring payments, letting the AI model "hear" or process raw credit card numbers creates a massive security and compliance risk (PCI-DSS). 

At the same time, forcing callers to hang up, call a separate payment line, and repeat their account details ruins the customer experience. The challenge was to build a **reusable payment engine** that keeps sensitive card numbers 100% invisible to the AI while making the experience feel like one smooth conversation.

---

## 2. The Architectural Solution
I designed a modular **Channel-Shielded Payment Architecture** orchestrated through the Contact Center (CCAAS) layer:

* **Zero-Exposure Card Capture:** The AI Payment Bot handles conversational intent, payment type (one-time vs. recurring), and amount confirmation, then hands the call via CCAAS to an **Isolated Payment IVR** for DTMF/card entry.
* **Tokenized Return Loop:** The Isolated IVR verifies the card externally and generates a secure **Payment Token**. Raw card numbers never enter Dialogflow CX logs, NLU context, or LLM transcripts.
* **Session Continuity:** Before transferring to the isolated IVR, the Payment Bot saves session parameters (`Session ID`, verified amount, payment type). When the caller returns, the bot retrieves the token and completes the transaction automatically before handing the caller back to the Main Bot.

---

## 3. End-to-End Architecture Flow

```mermaid
flowchart TD
    %% Ingress Layer
    subgraph Ingress["1. Ingress & Routing Layer (CCAAS)"]
        A[Main Bot] -->|Intent = Payment| B[CCAAS Routing Layer]
        B -->|Transfer| C[Ingress Payment Bot]
    end

    %% Payment Bot Layer
    subgraph PayBot["2. Conversational Payment Bot (Zero Card Exposure)"]
        C --> R{One-Time Payment?}
        R -->|Yes| D[One-Time Flow]
        R -->|No| E[Recurring Payment Flow]
        
        D --> Q{New Session ID?}
        Q -->|No: Reuse Stored ID| O[Save Session Params & Exit]
        Q -->|Yes| F{Amount != Null?}
        
        F -->|No| G[Collect Amount]
        F -->|Yes| O
        G --> P{Amount Confirmed?}
        P -->|Yes| O
        P -->|No: Max 2 Retries| G
        E --> O
        
        O -->|Hand-off for Card Entry| B
    end

    %% Isolated PCI Zone
    subgraph PCI["3. Isolated Credit Card Collection (PCI Scope)"]
        B -->|Route to Secure IVR| I[Isolated Card Capture IVR]
        I --> J[Collect Card Number & CVV]
        J --> K{Card & CVV Confirmed?}
        K -->|No: Max 2 Retries| I
        K -->|Yes| L[External Card Verification]
        L <--> M[External Verification Gateway]
        L --> N[Generate Secure Token]
    end

    %% Tokenized Execution
    subgraph Exec["4. Tokenized Payment & Return"]
        O -.->|Restore Session Context| S[Collect Payment Token]
        N <-->|Token Exchange| S
        S -->|Execute Payment via Token| T[Call Payment API]
        T <--> V[External Payment System]
        T --> W{Payment Success?}
        W -->|Complete| U[Transfer Back to Main Bot]
        U --> B
    end
