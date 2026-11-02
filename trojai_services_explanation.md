# TrojAI — Detailed Services Explanation

> **Source:** [docs.troj.ai](https://docs.troj.ai/) (v2.20.0)

---

## 📌 Product Overview

**TrojAI** is an **AI Security Platform** that provides end-to-end risk management, security, and alignment for machine learning models. It addresses the unique security challenges that arise when organizations deploy ML systems — from training data poisoning, to adversarial evasion attacks, to prompt injection in LLMs.

TrojAI promotes a **Secure Model Development Lifecycle (SMDLC)** where data scientists and security personnel work together across the entire pipeline: development → testing → deployment → production monitoring.

The platform is built around **two core products** and several supporting services:

| Product | Phase | Purpose |
|---------|-------|---------|
| **TrojAI DETECT** | Pre-deployment | Red teaming, pentesting, integrity audits |
| **TrojAI DEFEND** | Runtime | AI firewall, real-time request/response filtering |

---

## 🔴 1. TrojAI DETECT — Pre-Deployment Testing Engine

DETECT is the **pre-deployment testing engine** that subjects ML models to adversarial attacks, integrity checks, and risk audits *before* they reach production. It supports **Generative AI (LLMs)**, **NLP**, and **Tabular** models.

### 1.1 Dataset Registry

The Dataset Registry is a **central catalog** of curated evaluation and training datasets used for pentest campaigns and integrity audits.

- **Tabular datasets**: Full configuration of columns, labels, feature types, data splits
- **NLP datasets**: Tokenizer settings, text fields, label mappings
- **Configuration**: JSON-driven with separate or embedded dataset config files
- Supports custom dataset formats via configuration keys

### 1.2 Model Registry

The Model Registry **stores and manages all ML models** under assessment:

- **LLMs**: Large language models (GPT, Claude, Llama, etc.) accessed via API endpoints
- **NLP classifiers**: Text classification, sentiment analysis, NER models
- **Tabular models**: Gradient boosting, neural networks, decision trees
- **Model Wrappers**: Standardized Python wrappers (tabular/NLP) so the DETECT engine can uniformly query any model

### 1.3 LLM Pentesting (Generative AI)

Automated adversarial testing campaigns designed specifically for generative AI models:

- **Testing Methodology**: Systematic approach to probing LLMs for vulnerabilities — jailbreaks, safety bypasses, harmful output generation, and prompt injection
- **Configurable campaigns**: Select attack libraries, manipulations, output evaluators, and target models
- **Results**: Quantified success rates, vulnerability categories, severity scores

### 1.4 Attack Libraries

Extensive libraries of standard and custom attacks:

#### For LLMs (Generative):
- Prompt injection templates
- Jailbreak prompts
- Role-play and context manipulation attacks
- Multi-turn conversation attacks

#### For NLP:
- **Character-based attacks**: Character swaps, insertions, deletions, homoglyphs
- **Word-based attacks**: Synonym substitution, word insertion/deletion, embedding-based replacement
- **Custom attacks**: Build your own using Constraints + Transformations + Goal Functions + Search Methods

#### For Tabular:
- Standard attacks and corruptions (feature perturbation, adversarial examples)
- Custom attack routines integrated into the TrojAI engine

### 1.5 Manipulations

Optional text transformations applied **on top of** attack prompts to increase coverage:

- Misspelling injection
- Encoding changes (Base64, Unicode)
- Language translation
- Case manipulation
- Designed to test whether safety filters can be bypassed via obfuscation

### 1.6 Output Evaluators

Automated judges that **score every model response** for safety, correctness, and alignment:

- **LLM-as-Judge**: Uses a separate evaluator LLM to assess whether the target model's output is safe/harmful
- **Classifier-based evaluation**: ML classifiers that categorize outputs by safety dimensions
- **Custom evaluators**: Define your own scoring logic

### 1.7 Integrity Checks & Risk Audit

A comprehensive **model risk audit** organized into four categories:

| Category | What It Checks |
|----------|---------------|
| **Robustness** | Adversarial resilience, perturbation sensitivity, noise tolerance |
| **Explainability / Bias** | Feature importance, fairness across subgroups, bias detection |
| **Security / Privacy** | Data leakage, membership inference, model extraction vulnerability |
| **Performance / Methodology** | Accuracy metrics, data drift detection, train/test leakage |

- Both **default** (built-in) and **custom** integrity checks are supported
- Audit results consolidate into a single risk score and report
- Can be run during development **or** against production models

### 1.8 TrojAI SDK & Testing Configuration

- **Python SDK**: Programmatic interface for running evaluation campaigns
- **JSON Configuration**: Modular config for datasets, models, attacks, and integrity checks (main config + separate files)
- **CI/CD Integration**: Embed security testing into automated deployment pipelines
- **Authentication**: API key-based authentication for SDK access

---

## 🛡️ 2. TrojAI DEFEND — Runtime AI Firewall

DEFEND is the **real-time AI firewall** — a scalable Kubernetes deployment written in GoLang. It sits as a proxy between users/applications and your LLMs, intercepting every request and response.

### 2.1 Architecture: Policy → Rule Group → Rule

DEFEND uses a **two-tier model**:

```
Request (with Client ID)
    ↓
┌──────────────────────┐
│   FIREWALL POLICY    │ ← Owns Client ID, contains ordered Rule Groups
│   ┌────────────────┐ │
│   │  Rule Group 1  │ │ ← Reusable bundle (e.g., "Baseline Security")
│   │  • Rule A      │ │
│   │  • Rule B      │ │
│   └────────────────┘ │
│   ┌────────────────┐ │
│   │  Rule Group 2  │ │ ← Another bundle (e.g., "PII Detection")
│   │  • Rule C      │ │
│   └────────────────┘ │
└──────────────────────┘
    ↓
Target LLM → Output Rules → Response
```

### 2.2 Request Evaluation Flow

1. Request arrives with `x-eag-clientid` header
2. Firewall finds the matching **Policy**
3. Runs each **Rule Group's input rules** in order (if any rule blocks, later groups are skipped)
4. Calls the **target model**
5. Runs each **Rule Group's output rules** against the model response
6. Returns: PASS / FLAG / BLOCK / REDACT / ERROR

### 2.3 Firewall Policies

The top-level object:
- Owns a **Client ID** for traffic routing
- Contains an **ordered list of Rule Groups** (run top-to-bottom)
- Settings: default block message, URI base override, multimodal preprocessor, target model
- **Types**: Standard, MCP, OpenAI Compliance
- Can be enabled/disabled

### 2.4 Rule Groups

Reusable bundles of rules:
- **No own Client ID** — only run when a parent Policy runs
- **Shared across Policies**: edit once, applied everywhere
- Per-rule settings and event logging preferences
- Same type as their parent Policy

### 2.5 Firewall Rules — Complete Taxonomy

#### 2.5.1 Adversarial Attacks

| Rule | Purpose | Key Config |
|------|---------|------------|
| **Prompt Injection** | Classifier-based detection of injection attempts | Confidence threshold (default: 0.6) |
| **TrojGuard** | Security LLM checking 12 harm categories in one call | Category toggles, custom system prompt |
| **Governance (Trooper)** | Natural-language policy evaluation model | 13 built-in protections + custom policies, chain-of-thought reasoning |

#### 2.5.2 Content Moderation

| Rule | Purpose | Key Config |
|------|---------|------------|
| **Harmful Output** | Fast classifier for 5 harm categories (violent crimes, non-violent crimes, hate, self-harm, weapons) | No rule-level config needed |
| **External - Azure Content Safety** | Azure API-based moderation | Severity threshold, harm categories, blocklists |
| **External - BYO-ML Classifier** | Your own ML model for classification | Endpoint URL, model URI base |
| **External - BYO-LLM** | Your own moderation LLM | Payload template, system prompt, verdict/category paths |

#### 2.5.3 PII Detection (5 Variants)

| Variant | Method | Highlights |
|---------|--------|------------|
| **PII Detection** | Regex-based | Luhn validation, entity types, exclusions, substring matching |
| **PII Detection With Context** | NER + contextual scoring | Confidence threshold, context prefix/suffix, custom recognizers |
| **PII Detection With Scoring Pipeline** | Regex → format validation → context scoring | Multi-stage pipeline, per-entity keywords, early exit optimization |
| **Neural PII Detection** | Deep learning sequence labeling | 14 entity types, confidence threshold, Block/Redact/Flag modes |
| **PII Pattern Matching** | Custom regex patterns | User-defined regex, Luhn check, exclusions |

**Supported PII entities include**: CREDIT_CARD, US_SSN, US_ITIN, CA_SIN, AR_DNI, AR_CBU, MX_CLABE, IT_FISCAL_CODE, FR_INSEE, UK_NINO, IN_PAN, MX_RFC, IBAN_CODE, US_DRIVER_LICENSE, ADDRESS, API_KEY, BANK_ACCOUNT, CREDENTIALS, EMAIL, IP_ADDRESS, LICENSE_PLATE, LOCATION, PERSON, PHONE_NUMBER, and more.

#### 2.5.4 Data Leakage Prevention

| Rule | Purpose |
|------|---------|
| **Base64 Encoding** | Detects encoded data that may contain sensitive information |
| **Secret Detection** | Identifies API keys, tokens, credentials |
| **Source Code Detection** | Detects code in prompts/responses |
| **System Prompt Leakage** | Detects if the system prompt was echoed in output (syntactic + semantic + Judge LLM) |

#### 2.5.5 Denial of Service Protection

| Rule | Purpose |
|------|---------|
| **Suspicious Character Count** | Blocks inputs with abnormal length (min/max bounds) |
| **Suspicious Repetition** | Detects repeated word patterns (count & percentage thresholds) |

#### 2.5.6 Pattern & Code Rules

| Rule | Purpose |
|------|---------|
| **Suspicious Language** | Blocks unexpected languages (confidence threshold + allowlist) |
| **Misspelling Manipulations** | Detects intentional misspellings for filter bypass |
| **Custom Blocklist** | Block/flag specific words or phrases |
| **Named Entities** | Identifies people, organizations, locations |
| **Suspicious Pattern Monitor** | Detects excessive punctuation, capitalization, whitespace, Unicode, emojis, invisible characters, RTL overrides, control characters, Zalgo text |
| **Malicious & Insecure Code Detector** | Identifies unsafe code patterns in C, C++, C#, Java, JS, PHP, Python, Rust |

### 2.6 Firewall Actions

| Action | Meaning |
|--------|---------|
| **PASS** | No rules triggered — request proceeds cleanly |
| **FLAG** | Rule triggered but set to flag only — logged but allowed through |
| **BLOCK** | Request/response blocked from reaching model/user |
| **REDACT** | Sensitive content replaced (e.g., PII → `<REDACTED>`) — falls under FLAG |
| **ERROR** | Processing exception (rate limits, auth issues) |
| **UNKNOWN** | Uncaught system-level issue |

---

## 🔍 3. Firewall Explorer

A **centralized dashboard** for reviewing all firewall events:

- Filter by action type (PASS / FLAG / BLOCK / ERROR)
- Filter by time range, policy, rule group
- Drill into individual events to see rule verdicts and metadata
- Historical data remains queryable after policy changes
- Supports compliance reporting and threat analysis

---

## 🌐 4. LLM Monitor (Browser Extension)

A **Chromium browser extension** that applies TrojAI firewall protection directly in the browser:

- Protects users interacting with web-based LLMs (ChatGPT, Claude, Gemini, etc.)
- Configurable rulesets mirroring firewall policies
- **Authentication**: OIDC/IDP integration for enterprise SSO
- Settings panel for admins to control extension behavior
- No code changes needed — protection at the browser layer

---

## 📊 5. Metrics & Prometheus Integration

The firewall exports **Prometheus-compatible metrics** for enterprise observability:

- Block rates, flag rates, pass rates by policy and rule
- Latency percentiles (p50, p95, p99)
- Error rates and types
- Top triggered rules
- Integration with Grafana dashboards, PagerDuty, Slack, and other alerting platforms

---

## ☁️ 6. Deployment Options

### 6.1 TrojAI Cloud (SaaS)
- Fully hosted, no infrastructure management
- Instant access to DETECT and DEFEND features
- Managed upgrades and scaling

### 6.2 TrojAI Enterprise (Self-Hosted)
- Deploy on your own Kubernetes cluster
- **Supported platforms**: AWS, GCP, Azure, Custom Helm
- Architecture: Kubernetes-native, scalable microservices
- Security: Isolated tenancy, encryption at rest and in transit
- Includes Helm Values Reference for fine-grained configuration

### 6.3 TrojGuard Security LLM Deployment
- Dedicated deployment via **NVIDIA Triton Inference Server**
- GPU-accelerated inference
- Includes Triton General Setup and AWS Walkthrough guides

---

## 🔌 7. API & SDK

### REST API
- Full platform automation endpoint documentation
- Firewall Proxy API for programmatic LLM proxying
- Header-based routing via `x-eag-clientid`

### Python SDK
- Programmatic evaluation runs
- CI/CD pipeline integration
- Custom workflow orchestration
- Configuration-as-code for reproducible security testing

---

## 🔄 Complete End-to-End Flow Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SMDLC Pipeline                               │
│                                                                     │
│  ┌──────────┐    ┌──────────────────────────────────────┐           │
│  │  DATA &   │───▶│         TrojAI DETECT                │           │
│  │  MODELS   │    │  • Red-team pentesting               │           │
│  └──────────┘    │  • Attack libraries & manipulations   │           │
│                  │  • Output evaluators                  │           │
│                  │  • Integrity & risk audit             │           │
│                  └──────────────┬───────────────────────┘           │
│                                 │ Findings inform                   │
│                                 ▼ firewall rules                    │
│  ┌──────────┐    ┌──────────────────────────────────────┐           │
│  │  USERS /  │───▶│         TrojAI DEFEND                │           │
│  │  APPS     │    │  • Policy → Rule Groups → Rules      │           │
│  └──────────┘    │  • Prompt Injection Detection         │           │
│                  │  • PII Detection & Redaction          │           │
│                  │  • TrojGuard (12 harm categories)     │           │
│                  │  • Governance (custom policies)       │           │
│                  │  • Content Moderation                 │           │
│                  │  • Code & Secret Detection            │           │
│                  └──────────────┬───────────────────────┘           │
│                                 │                                   │
│                                 ▼                                   │
│                  ┌──────────────────────────────────────┐           │
│                  │       MONITORING & RESPONSE           │           │
│                  │  • Firewall Explorer (dashboard)      │           │
│                  │  • Browser Extension (LLM Monitor)    │           │
│                  │  • Prometheus Metrics & Alerting      │           │
│                  │  • PASS / FLAG / BLOCK / REDACT       │           │
│                  └──────────────────────────────────────┘           │
│                                 │                                   │
│                                 ▼                                   │
│                      ✅ Safe, Compliant Output                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

> 📎 **Animated HTML page**: Open [trojai_product_flow_animation.html](file:///d:/Software/Troj/trojai_product_flow_animation.html) in your browser for an interactive step-by-step animation of this entire flow.
