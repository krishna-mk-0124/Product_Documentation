# TrojAI — Product Documentation

> Reviewed and compiled from the official [TrojAI Documentation](https://docs.troj.ai/) (v2.20.0)

This repository contains a comprehensive breakdown of the TrojAI AI Security Platform — including an interactive animated product flow and detailed explanations of all services and microservices.

---

## 📁 Contents

| File | Description |
|------|-------------|
| [`trojai_product_flow_animation.html`](trojai_product_flow_animation.html) | Self-contained, downloadable HTML page with step-by-step CSS/JS animation of the complete TrojAI product flow — from input sources through DETECT and DEFEND to secure output |
| [`trojai_services_explanation.md`](trojai_services_explanation.md) | Detailed written explanation of every TrojAI service, module, and sub-feature — including DETECT, DEFEND, platform microservices, firewall rule taxonomy, and deployment architecture |
| [`trojai_rules_service_go_vs_python.md`](trojai_rules_service_go_vs_python.md) | In-depth technical comparison of `troj-rules-service` (Go) vs `troj-python-rules-service` (Python), including exact rule division, latency/memory profiles, and execution examples |
| [`trojai_1.x_architecture_and_flow.md`](trojai_1.x_architecture_and_flow.md) | Detailed architectural breakdown of TrojAI 1.x (MongoDB document storage, monolithic backend, flat ruleset, Triton integration, and evolution to 2.x) |
| [`trojai_v2_20_defend_end_to_end_flow.md`](trojai_v2_20_defend_end_to_end_flow.md) | Comprehensive step-by-step DEFEND flow in v2.20 with sequence diagram, payload excerpts, and database interactions across all microservices |

---

## 🧭 Product Overview

**TrojAI** is an end-to-end **AI Security Platform** that provides:

- **TrojAI DETECT** — Pre-deployment red teaming, adversarial pentesting, attack libraries, output evaluators, integrity & risk audits for LLMs, NLP, and tabular models
- **TrojAI DEFEND** — Real-time AI Firewall with configurable Policies → Rule Groups → Rules, including TrojGuard (12-harm-category Security LLM), Governance (Trooper model), PII detection, prompt injection detection, content moderation, and code security
- **Monitoring** — Firewall Explorer dashboard, Prometheus metrics, Browser Extension (LLM Monitor), and audit reporting

---

## 🎬 Interactive Flow Animation

Open `trojai_product_flow_animation.html` in any modern browser. Click **▶ Play Flow** to walk through all 15 steps of the end-to-end platform flow — from user input, through pre-deployment testing and runtime firewall protection, to safe compliant output.

---

## 🏗️ Platform Microservices

The enterprise platform deploys as Kubernetes microservices via Helm charts. Key services include:

| Service | Role |
|---------|------|
| `troj-platform-config-service` | Stores all Firewall Policies, Rule Groups, and routing config |
| `troj-platform-data-service` | Ingests and indexes firewall events into Elasticsearch |
| `troj-content-service` | Manages attack libraries, rule templates, and platform content |
| `troj-firewall-v2` | GoLang reverse proxy — the core runtime firewall |
| `troj-rules-service` (Go) | Evaluates deterministic pattern/regex rules at high speed |
| `troj-python-rules-service` | Evaluates contextual and scoring-based rules |
| `troj-classification-service` | GPU-accelerated ML inference for prompt injection and harmful output |
| `troj-guard` | TrojGuard Security LLM (12 harm categories) |
| `troj-trooper` | Governance guardrails model with custom policy support |
| `troj-web-api-v2` | Public REST API gateway |
| `troj-frontend` | React/Next.js web dashboard |

See [`trojai_services_explanation.md`](trojai_services_explanation.md) for the full breakdown.

---

## 📚 Source

- Documentation: https://docs.troj.ai/
- Product Version: v2.20.0
