# TrojAI 1.x Architecture & Product Flow

> **Historical Reference & Architecture Evolution Document**  
> **Source:** TrojAI 1.x Documentation & Enterprise Guides (`/trojai/...`)

---

## 📌 1. Overview of the 1.x Architecture

The **TrojAI 1.x Platform** was designed as an integrated solution for machine learning risk assessment, model stress-testing, and early real-time LLM protection. Unlike the fully decoupled, cloud-native microservices architecture of version 2.x, the 1.x platform used a **semi-monolithic service model** centered around a Python/Node backend, a batch model evaluation job runner, and a standalone Go-based level-7 proxy firewall.

### Core Architectural Pillars in 1.x:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        TrojAI 1.x Platform                             │
│                                                                        │
│   ┌───────────────┐        ┌────────────────┐                          │
│   │  Frontend UI  │◀──────▶│    Backend     │                          │
│   │ (React/Node)  │        │ (Core Python)  │                          │
│   └───────────────┘        └───────┬────────┘                          │
│                                    │                                   │
│            ┌───────────────────────┼───────────────────────┐           │
│            ▼                       ▼                       ▼           │
│   ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐   │
│   │    MongoDB      │    │    PostgreSQL    │    │      Redis      │   │
│   │ (Configs, Logs, │    │ (User, Tenancy,  │    │  (Celery Queue, │   │
│   │  Audit Events)  │    │  Platform Meta)  │    │   Job Broker)   │   │
│   └─────────────────┘    └──────────────────┘    └─────────────────┘   │
│            ▲                                               │           │
│            │ (Direct Event Logging)                        ▼           │
│   ┌────────┴────────┐                            ┌─────────────────┐   │
│   │ TrojAI Firewall │                            │   Job Runner /  │   │
│   │   (Go Binary)   │                            │    Scheduler    │   │
│   └────────┬────────┘                            │ (Red-Team/Audit)│   │
│            │ (Inference Calls)                   └─────────────────┘   │
│            ▼                                                           │
│   ┌─────────────────┐                                                  │
│   │ NVIDIA Triton   │                                                  │
│   │ (Security Suite)│                                                  │
│   └─────────────────┘                                                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ 2. Primary Components of 1.x

### 2.1 The TrojAI Web UI & Backend (Monolith)
* **Technology:** Node.js / React frontend paired with a centralized Python backend.
* **Role:**
  * Handled user authentication, project creation, model upload wrappers, and dataset registration.
  * Served dashboards for tabular, NLP, and LLM test audit results.
  * Managed firewall rule creation by outputting flat configuration JSON files directly to MongoDB.

### 2.2 Model Evaluation Job Runner & Scheduler
* **Technology:** Celery task workers backed by Redis.
* **Role:**
  * Executed asynchronous batch stress-testing (adversarial perturbations, data poisoning checks, explainability audits) for Tabular and NLP models.
  * Stored test evaluation artifacts and metrics into MongoDB.

### 2.3 TrojAI Firewall (v1)
* **Technology:** Single containerized Alpine Linux image running a compiled GoLang binary.
* **Architecture:**
  * Operated as a Level-7 reverse proxy intercepting HTTP requests between client apps and LLM backends.
  * Extracted payloads using custom **Jayway JSONPath expressions** (`handler_config`).
  * Utilized Google's **`re2` C++ / Go library** for fast, deterministic string parsing and regex checks.
  * Relied directly on an external **NVIDIA Triton Inference Server** container for stochastic checks (bias, toxicity, prompt injection).

### 2.4 The 1.x Storage Layer: MongoDB-Centric Model
* **MongoDB (Primary Document Store):**
  * In 1.x, **MongoDB was the central system of record** for configurations and runtime data.
  * Stored configuration documents (`config_uuid`, `object_id`).
  * Stored all incoming prompts, model completions, and firewall events (`PASS`, `FLAG`, `BLOCK`, `REDACT`) directly from the firewall pods.
* **PostgreSQL:** Handled basic relational platform metadata, user accounts, and tenancy boundaries.
* **Redis:** Served strictly as the Celery task queue broker and ephemeral cache.

---

## ⚙️ 3. The 1.x Configuration Model: Flat Ruleset

In version 1.x, there was **no two-tier separation of Policies and Rule Groups**. Configurations were flat JSON documents created on a per-client or per-user basis:

```json
{
  "config_uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "user_uuid": "user-uuid-1234",
  "name": "Production Customer Support LLM Config",
  "client_id": "cust-support-v1",
  "object_id": "64f1a2b3c4d5e6f7a8b9c0d1",
  "ruleset": {
    "input_rules": [
      {
        "name": "prompt_injection",
        "action": "BLOCK",
        "threshold": 0.7
      },
      {
        "name": "pii_detection",
        "action": "REDACT",
        "entities": ["SSN", "CREDIT_CARD"]
      }
    ],
    "output_rules": [
      {
        "name": "harmful_output",
        "action": "BLOCK"
      }
    ]
  },
  "handler_config": {
    "inputFields": [
      {
        "index": 0,
        "label": "user_prompt",
        "expression": "$.messages[?(@.role=='user')].content"
      }
    ],
    "outputFields": [
      {
        "index": 0,
        "delta": "$.choices[0].delta.content",
        "complete": "$.choices[0].message.content"
      }
    ]
  }
}
```

---

## 🔄 4. 1.x Runtime Data Flow

```
[User / Application]
         │
         │ 1. HTTP Request (with x-eag-clientid: cust-support-v1)
         ▼
┌────────────────────────────────────────────────────────┐
│               TrojAI Firewall (Go / L7 Proxy)          │
│                                                        │
│  2. Look up config JSON in MongoDB or local memory     │
│  3. Unpack input payload using Jayway JSONPath         │
│  4. Run deterministic checks via Google re2 (local)    │
│     - Regex blocklists, PII regex, character limits    │
│                                                        │
│  5. Run stochastic checks via NVIDIA Triton:           │
│     ┌──────────────────────────────────────────────┐   │
│     │ POST http://triton:8000/v2/models/classifier │   │
│     │ (Prompt Injection, Toxicity, Sentiment)      │   │
│     └──────────────────────────────────────────────┘   │
│                                                        │
│  6. Action Decision: PASS / BLOCK / REDACT             │
└───────────────┬────────────────────────┬───────────────┘
                │                        │
         (If PASS/REDACT)         (Always)
                │                        │
                ▼                        ▼
       ┌─────────────────┐      ┌────────────────────────┐
       │   Target LLM    │      │    MongoDB Database    │
       │ (OpenAI/Custom) │      │                        │
       └────────┬────────┘      │ Direct write:          │
                │ Response      │ • Request payload      │
                ▼               │ • Response payload     │
┌────────────────────────────────┐ • Rule verdicts       │
│ Firewall Output Verification   │ • Latency & action    │
│ • Run output_rules             │ └────────────────────────┘
│ • Check toxicity & data leaks  │
└───────────────┬────────────────┘
                │ Verified Response
                ▼
        [User / Application]
```

---

## ⚠️ 5. Why 1.x Evolved into 2.x

While 1.x successfully proved the viability of an AI firewall proxy, it encountered fundamental enterprise scaling hurdles:

| Limitation in 1.x | How 2.x Resolved It |
| :--- | :--- |
| **MongoDB Scaling Bottleneck:** High-volume LLM traffic wrote all raw prompts, completions, and metadata directly into MongoDB, leading to index bloating and write contention. | **Replaced MongoDB with Elasticsearch + Kafka + PostgreSQL:** PDS indexes firewall events into Elasticsearch for fast search, while Kafka handles event ingestion streaming. |
| **Flat, Duplicated Configurations:** Reusable rule bundles did not exist. If 50 applications needed prompt injection checks, each configuration duplicated the rule definition. | **Introduced Policies → Rule Groups (Two-Tier Model):** Rule groups are reusable modules. Updating a rule group automatically applies to all linked policies. |
| **Triton GPU Dependency for All Classifications:** All stochastic checks required an NVIDIA Triton cluster, creating high GPU costs and queueing delays. | **Decoupled Rules Engines (Go + Python + PyTorch):** Lightweight rules run in Go; NLP scoring runs in Python; PyTorch handles local GPU prompt injection with BF16 SDPA attention. |
| **No Dedicated Configuration Sync Microservice:** Firewall pods had to periodically poll or reload MongoDB collections directly. | **Added Platform Configuration Service (PCS):** A dedicated gRPC service serving real-time validated policy trees with HashiCorp Vault secrets integration. |
| **No Verdict Caching:** Every repeated utterance in a multi-turn conversation was fully re-evaluated. | **Added Redis Verdict Cache:** Hashes message history and skips re-evaluating unchanged prior conversational turns. |
| **Limited AI Agent & Governance Support:** Rules were restricted to hard-coded classifications. | **Added Trooper Governance Model & MCP Metadata Service:** Allows plain-English custom organization policies and Model Context Protocol agent tool protection. |
