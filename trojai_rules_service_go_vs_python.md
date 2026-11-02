# TrojAI Architecture Notes: `troj-rules-service` (Go) vs `troj-python-rules-service` (Python)

> **Source:** TrojAI Enterprise Architecture & Helm Configuration (`docs.troj.ai` v2.20.x)  
> **Topic:** Detailed Technical Comparison & Functional Division of Labor between Rules Microservices

---

## 📌 1. Architectural Rationale: Why Two Rules Engines?

In the TrojAI runtime protection pipeline (**TrojAI DEFEND**), incoming prompts and outgoing model completions must be evaluated against multiple security policies with extreme speed and minimal overhead.

Instead of running a monolithic rules engine, TrojAI decouples the workload into two specialized microservices:
1. **`troj-rules-service` (Go):** Sized for **raw throughput and sub-millisecond deterministic checks** (compiled binary, zero-allocation regex, byte parsing, count checks, secret scanning), plus serving as a low-latency gateway to GPU-accelerated model backends.
2. **`troj-python-rules-service` (Python):** Sized for **complex NLP heuristics, statistical modeling, and multi-stage scoring pipelines** (e.g., n-gram language classification, context-word weighted PII scoring, and external moderation API payload templating).

```
                             Incoming Client Traffic (with x-eag-clientid)
                                                 │
                                                 ▼
                             ┌───────────────────────────────────────┐
                             │       TrojAI Firewall Core (Go)       │
                             │       (troj-firewall-v2:3473)         │
                             └───────┬───────────────────────┬───────┘
                                     │                       │
                       (Fast / Deterministic / Regex)        │ (Statistical / NLP / Scoring)
                                     │                       │
                                     ▼                       ▼
┌────────────────────────────────────────────────────────┐ ┌────────────────────────────────────────────────────────┐
│             troj-rules-service (Go)                    │ │          troj-python-rules-service (Python)          │
│             Port: 3483 | ~64–128 MiB RAM               │ │             Port: 3493 | ~1.0–1.5 GiB RAM            │
│ ────────────────────────────────────────────────────── │ │ ──────────────────────────────────────────────────── │
│ • Secret & Credential Detection                        │ │ • Suspicious Language Detection (Trigram/FastText)   │
│ • Character Count & Repetition DOS Bounds              │ │ • PII Detection With Scoring Pipeline (Regex+Luhn+NER│
│ • Custom Blocklists & Exclusions                       │ │ • PII Detection With Context (Prefix/Suffix weights) │
│ • Base64 Encoding & Source Code Detection              │ │ • Azure Content Safety API Integration               │
│ • Suspicious Pattern Monitor (Zalgo, Emojis, Invisible)│ │ • Bring-Your-Own-LLM (BYO-LLM) / BYO-ML Classifier   │
│                                                        │ └────────────────────────────────────────────────────────┘
│ ┌─ Dispatches to External/GPU Inference ─────────────┐ │
│ │ • trojClassificationService (Prompt Injection/Harm)│ │
│ │ • trojGuard (12 Harm Categories Security LLM)      │ │
│ │ • trojTrooper (Governance & Custom Guardrails)     │ │
│ └────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

---

## 📊 2. Side-by-Side Comparison

| Dimension | `troj-rules-service` (Go) | `troj-python-rules-service` (Python) |
| :--- | :--- | :--- |
| **Language & Runtime** | Compiled Go (GoLang binary) | CPython with scientific / NLP packages |
| **Container Image** | `trojai/troj-rules-service:v2.72.0` | `trojai/troj-python-rules-service:v2.37.0` |
| **Service Port** | `3483` (Health: `/health`) | `3493` (Health: `/health`) |
| **Primary Protocol** | HTTP / gRPC internal REST | gRPC (`TROJ_PYTHON_RULES_SERVICE_GRPC_SERVER_ENABLED=true`) |
| **Memory Footprint** | **64Mi request / 128Mi limit** | **1Gi request / 1.5Gi limit** |
| **CPU Allocation** | **100m request / 500m limit** | **200m request / 1000m limit** |
| **Average Latency** | **Sub-millisecond to < 2ms** (local rules) | **10ms to 50ms+** (heuristics & scoring dependent) |
| **Scaling Target** | Baseline: **8 replicas** in production | Baseline: **8 replicas** in production |
| **Downstream Targets** | Dispatches to `trojClassificationService`, `trojGuard`, `trojTrooper` | Dispatches to Azure APIs, external BYO moderation models |

---

## 🔍 3. Detailed Division of Rules

### 3.1 Rules Evaluated by `troj-rules-service` (Go)

1. **Secret & Credential Detection:**
   * Rapidly detects exposed API keys, bearer tokens, private keys, AWS access IDs, and database credentials.
   * Uses pre-compiled regex automata with zero memory allocation per request.
2. **Count & Boundary Rules:**
   * **Suspicious Character Count:** Checks minimum/maximum character boundaries to mitigate Denial-of-Service (DOS) payload flooding.
   * **Suspicious Repetition:** Identifies repeated words and patterns exceeding repetition count and percentage thresholds.
3. **Data Encoding & Leakage:**
   * **Base64 Encoding Detection:** Detects base64 strings longer than a specified character threshold (default: 12) to prevent obfuscated exfiltration.
   * **Source Code Detection:** Detects code syntax across C, C++, C#, Java, JavaScript, PHP, Python, and Rust.
4. **Text Obfuscation & Patterns:**
   * **Custom Blocklist:** Matches forbidden keywords or regular expressions with configured exclusions.
   * **Suspicious Pattern Monitor:** Fast byte/rune scanning for excessive punctuation, capitalization, Unicode manipulations, emojis, invisible characters, RTL overrides, and Zalgo text.
5. **Model Orchestration / Dispatching:**
   * Evaluates no deep learning locally, but acts as the client dispatcher sending requests to:
     * `http://troj-classification-service-service:8001/classify/fw` (Prompt Injection & Harmful Output)
     * `http://troj-classification-service-service:8001/v1/redact` (Neural PII token sequence labeling)
     * `http://troj-guard-service:8000/v1/chat/completions` (TrojGuard 12-harm Security LLM)
     * `trojTrooper` (Governance LLM)

---

### 3.2 Rules Evaluated by `troj-python-rules-service` (Python)

1. **Suspicious Language Detection:**
   * Evaluates text using statistical n-gram / language identification models (e.g., fastText / langdetect).
   * Checks against an allowed list of languages (e.g., `["en"]`) with confidence thresholds (default: 0.8).
   * **Config:** Skips inputs shorter than `TROJ_PYTHON_RULES_SERVICE_MIN_CHARACTER_COUNT_FOR_LANG_DETECT` (default: 50 characters) to prevent false positives on short greetings like *"Hi"* or *"Bonjour"*.
2. **PII Detection With Scoring Pipeline:**
   * A 3-stage validation pipeline:
     1. **Regex candidate extraction:** Finds candidate number sequences.
     2. **Algorithmic checksum validation:** Evaluates mathematical checksums (Luhn algorithm for credit cards, IBAN mod-97, French INSEE, Mexican CLABE, Argentine CBU/DNI, UK NINO, Indian PAN).
     3. **Contextual scoring:** Analyzes adjacent tokens in prefix/suffix windows using custom keyword weights to generate a composite confidence score (`low`, `medium`, `high`).
3. **PII Detection With Context:**
   * Uses context similarity weighting (`context_similarity_factor: 0.35`, `prefix_count: 5`, `suffix_count: 0`) and custom entity recognizer specifications.
4. **External Content Moderation (Azure Content Safety & BYO-LLM):**
   * **Azure Content Safety:** Formats and signs requests to Azure REST APIs, checking blocklists and 4 severity levels.
   * **Bring-Your-Own-LLM (BYO-LLM):** Formats flexible JSON request templates (`[SYSTEM_PROMPT]` and `[PROMPT]`), executes external model requests, extracts JSON verdicts via JSONPath (`$.verdict`), and evaluates safe category overrides (`$.reason_category`).

---

## 💡 4. Real-World Execution Examples

### Example A: Secret Detection (Handled by Go Rules Service)
* **Incoming User Prompt:**
  ```text
  "Please verify my AWS deployment configuration: AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  ```
* **Execution Path:**
  1. The Firewall identifies that the matching Policy contains a `Secret Detection` rule.
  2. The Firewall sends the payload over HTTP to `troj-rules-service:3483`.
  3. The Go binary passes the byte buffer through its high-speed regex matcher.
  4. Match found for high-entropy secret pattern `AWS_SECRET_ACCESS_KEY`.
  5. **Time taken:** **0.35 milliseconds**.
  6. **Verdict:** `BLOCK` (request halted before reaching target LLM).

---

### Example B: Multi-Stage PII Scoring (Handled by Python Rules Service)
* **Incoming User Prompt:**
  ```text
  "Here is my payment info for the booking: my credit card is 4532-0150-1234-5678, expiration 08/28."
  ```
* **Execution Path:**
  1. The Firewall identifies the `PII Detection With Scoring Pipeline` rule.
  2. The Firewall forwards the payload over gRPC to `troj-python-rules-service:3493`.
  3. **Stage 1 (Pattern Match):** Regex finds numeric candidate `4532015012345678`.
  4. **Stage 2 (Hard Validation):** Luhn mod-10 checksum algorithm runs in Python → Passes.
  5. **Stage 3 (Contextual Scoring):** Python inspects prefix tokens: finds `"credit card"`, `"payment info"`. High positive keyword match boosts score to `0.94`.
  6. Confidence level satisfies `"high"`.
  7. **Time taken:** **14 milliseconds**.
  8. **Verdict:** `FLAG` with action `REDACT` → string becomes `"Here is my payment info for the booking: my credit card is <REDACTED_CREDIT_CARD>, expiration 08/28."`

---

### Example C: Suspicious Language Detection (Handled by Python Rules Service)
* **Policy Settings:** Allowed Languages = `["en"]`, Confidence Threshold = `0.80`, Min Length = `50`.
* **Incoming User Prompt:**
  ```text
  "Hola a todos, me gustaría saber cómo puedo restablecer la contraseña de mi cuenta institucional."
  ```
* **Execution Path:**
  1. Character count = 94 (exceeds the 50-character minimum threshold).
  2. Python language classifier evaluates n-gram frequencies.
  3. Result: Spanish (`es`) with `0.98` confidence.
  4. Spanish is not in `["en"]`.
  5. **Time taken:** **8 milliseconds**.
  6. **Verdict:** `BLOCK` with message: *"Language 'es' not permitted."*

---

## ⚙️ 5. Key Helm Configuration Reference

### Go Rules Provider Configuration (`trojai.apps.rulesProviderGo`)
```yaml
trojai:
  apps:
    rulesProviderGo:
      enabled: true
      replicas: 8
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
      env:
        - name: TROJ_RULES_LOGGING_LEVEL
          value: "INFO"
        - name: TROJ_RULES_MODEL_TROJGUARDCLASSIFICATION_TARGET_URL
          value: "http://troj-classification-service-service:8001/classify/fw"
        - name: TROJ_RULES_MODEL_TROJGUARD_TARGET_URL
          value: "http://troj-guard-service:8000/v1/chat/completions"
        - name: TROJ_RULES_MODEL_POLICY_GUARD_TARGET_MODEL
          value: "trooper"
```

### Python Rules Provider Configuration (`trojai.apps.rulesProviderPython`)
```yaml
trojai:
  apps:
    rulesProviderPython:
      enabled: true
      replicas: 8
      resources:
        requests:
          cpu: "200m"
          memory: "1Gi"
        limits:
          cpu: "1000m"
          memory: "1536Mi"
      env:
        - name: TROJ_PYTHON_RULES_SERVICE_LOGGING_LEVEL
          value: "INFO"
        - name: TROJ_PYTHON_RULES_SERVICE_GRPC_SERVER_ENABLED
          value: "true"
        - name: TROJ_PYTHON_RULES_SERVICE_MIN_CHARACTER_COUNT_FOR_LANG_DETECT
          value: "50"
```
