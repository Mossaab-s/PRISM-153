# PRISM — Protocol for Routed Intelligent Specialized Models

[![Zenodo](https://img.shields.io/badge/Zenodo-PRISM%20Paper-blue)](https://doi.org/10.5281/zenodo.18750029)
[![vLLM Issue](https://img.shields.io/badge/vLLM-Issue%20%2335804-purple)](https://github.com/vllm-project/vllm/issues/35804)
[![License](https://img.shields.io/badge/License-MIT-green)]()
[![Status](https://img.shields.io/badge/Status-Research%20%2F%20PoC-orange)]()

> **"Knowledge must be routed before it is resolved."**

---

## What is PRISM?

PRISM is a DNS-inspired semantic routing architecture for specialized Small Language Models (SLMs).

The core problem it solves: LLMs hallucinate, improvise outside their domain, and answer confidently when they shouldn't. Standard routing systems decide *which model* to call. PRISM decides *whether a model is legitimate to respond at all*.

**The fundamental distinction:**
- Existing routers answer: *"Which model is best suited to handle this request?"*
- PRISM answers: *"Is this model legitimate to respond to this specific request?"*

**Refusal is a first-class output — not a fallback, not an error.**

---

## The 153-key Protocol

The 153-key is the core mechanism of PRISM. It is a structured prompt architecture that forces any SLM or LLM to:

1. **Declare** its domain boundaries explicitly before any query is processed
2. **Self-assess** whether an incoming query falls within its declared scope
3. **Formally refuse** with justification if the query is out of scope

It works through **encapsulation, not trust**. The user query never reaches the model directly — it is wrapped inside the 153-key protocol, making it architecturally impossible for the model to skip the legitimacy check.

### 153-key Prompt Structure

```
ABSOLUTE RULE:
ACCEPTANCE CRITERIA — A question is ACCEPTED if AND ONLY IF:
REFUSAL CRITERIA — A question is REFUSED if:
MANDATORY OUTPUT FORMAT — Respond in valid JSON on ONE LINE:

```

### Real Output Example

Query sent to `Mistral-Dev:latest` (specialized in software development):

```json
{
  "requestId": 79,
  "userQuery": "In IT development, what is the most efficient sorting algorithm to implement in Python for a dataset of 10 million integers with low memory constraints?",
  "classifiedDomain": "Informatique_Dev",
  "classificationConfidence": 0.9,
  "routingDecision": "ACCEPTED",
  "routingReason": "Routed to Mistral-Dev:latest (score: 0.900)",
  "routedModel": {
    "modelName": "Mistral-Dev:latest",
    "providerName": "Ollama",
    "isQualified": true,
    "isBanned": false,
    "capability": {
      "primaryDomain": "Informatique_Dev",
      "selfAssessedScore": 0.95,
      "empiricalScore": 0.9,
      "totalRequests": 3,
      "acceptedRequests": 2,
      "refusedRequests": 1
    }
  },
  "executionStatus": "ACCEPTED",
  "executionReason": "Technical terms matched: IT development, sorting algorithm, Python, dataset, memory constraints",
  "modelResponse": "For a dataset of 10 million integers with low memory constraints in Python, the most efficient sorting algorithm is Timsort — Python's default in sorted() and list.sort(). O(n log n) worst-case, combines merge sort and insertion sort. For extreme memory constraints, External Merge Sort is recommended.",
  "performance": {
    "vslmLatencyMs": 1714,
    "modelLatencyMs": 2531,
    "totalLatencyMs": 4245
  }
}
```

**Key observations:**
- The VSLM classifies the domain with 0.90 confidence **before** any model is called
- The selected model self-validates its legitimacy via the 153-key **before** executing
- Every decision is logged, scored, and traceable
- Running entirely on **CPU via Ollama** — no GPU required

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                      USER QUERY                         │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    VSLM CONSOLE                         │
│         Semantic classification of query intent         │
│         Domain identification + confidence score        │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   153-REGISTRY                          │
│         Dynamic directory of qualified SLMs             │
│         Each model: domain, score, ban status           │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   153-KEY PROTOCOL                      │
│         Query encapsulated → model self-validates       │
└──────────┬──────────────────────────────┬───────────────┘
           │                              │
           ▼                              ▼
    ┌─────────────┐               ┌──────────────┐
    │ ACCEPTED ✅ │               │  REFUSED ❌ │
    └──────┬──────┘               └──────┬───────┘
           │                             │
           ▼                             ▼
    ┌─────────────┐            ┌──────────────────────┐
    │  RESPONSE   │            │  Log + Score update  │
    └─────────────┘            │  Re-route or reject  │
                               └──────────────────────┘
```

---

## Production Deployment

PRISM has been validated in production at a textile manufacturing company (Châtellerault, France):

| Component | Details |
|---|---|
| SLMs deployed | 11 specialized models |
| Base model | Phi-3 3.8B (Microsoft) |
| Infrastructure | On-premise CPU only |
| Serving | Ollama |
| Domains covered | Textile methods, quality control, production, finance, IT development |
| Training data | 20 years of internal operating sequences |
| Validation | 100+ manual tests, 100% consistency in qualification/acceptance/refusal |
| Cost vs cloud | ~80% reduction vs cloud API solutions |

---

## Integration with vLLM-SR

PRISM is designed to complement, not compete with, the vLLM Semantic Router (Signal-Decision Architecture).

**Three integration options are under community discussion** in [vLLM Issue #35804](https://github.com/vllm-project/semantic-router/issues/1422):

### Option 1 — Fine Filter (PRISM after vLLM-SR)
```
HTTP Request → API Server → vLLM-SR (routing) → PRISM 153-key (legitimacy) → Scheduler → Response
```
✅ Low integration complexity · ✅ Re-routing if REFUSED · ☁️ Ideal for cloud enterprise

### Option 2 — Coarse Filter (PRISM before vLLM-SR)
```
HTTP Request → API Server → PRISM VSLM (pre-qualification) → vLLM-SR (optimization) → Scheduler → Response
```
✅ Zero GPU waste on illegitimate requests · 🏭 Ideal for sovereign on-premise / GDPR

### Option 3 — Hybrid (PRISM surrounds vLLM-SR)
```
HTTP Request → PRISM [FILTER 1] → vLLM-SR → PRISM 153-key [FILTER 2] → Scheduler → Response
```
✅ Maximum legitimacy guarantee · 🏥 Ideal for critical enterprise (healthcare, legal, finance)

---

## Key Differentiators

| Feature | PRISM | vLLM-SR | RouteLLM |
|---|---|---|---|
| Routing mechanism | Legitimacy-based | Signal-based | Cost/performance |
| Refusal as output | ✅ First-class | ❌ Not addressed | ❌ Not addressed |
| Self-assessment | ✅ Model declares boundaries | ❌ External classification | ❌ External classification |
| GPU required | ❌ CPU only | ✅ GPU recommended | ✅ GPU recommended |
| GDPR / on-premise | ✅ Native constraint | ⚠️ Cloud-centric | ⚠️ Cloud-centric |
| Target | Industrial SMEs | Cloud enterprise | Cloud enterprise |

---

## Roadmap

### ✅ Done
- [x] PRISM architecture designed and validated in production
- [x] 153-key protocol implemented and tested on 11 SLMs
- [x] 153-Registry with qualification, scoring, and ban mechanism
- [x] Feedback loop scoring formula (Equation 1)
- [x] Paper published on Zenodo and HAL
- [x] vLLM-SR integration proposal — Issue #35804

### 🔄 In Progress
- [ ] Open-source release of 153-key prompt architecture
- [ ] VSLM specification and training approach
- [ ] Quantitative evaluation on standard benchmarks (MMLU-Pro)

### 📋 Planned
- [ ] PoC integration with vLLM-SR (Option 1 — Fine Filter)
- [ ] Automated feedback loop
- [ ] Adversarial testing (jailbreak resistance)
- [ ] Multi-SLM cooperation for hybrid domain queries (V2)

---

## Limitations

PRISM is an architectural research contribution, not a production-ready framework yet.

- Validation: 100 manual tests on controlled samples — not statistically rigorous
- VSLM: classification mechanism described but model/performance metrics not yet published
- Veracity: PRISM guarantees legitimacy of the decision to respond — **not** the correctness of the response
- Automation: routing console not yet fully automated

---

## Citation

If you use PRISM in your research, please cite:

```bibtex
@misc{souaissa2026prism,
  title  = {PRISM: A DNS-Inspired Semantic Routing Architecture for Distributed Specialized Language Models},
  author = {Souaissa, Mossaab},
  year   = {2026},
  doi    = {10.5281/zenodo.18750029},
  url    = {https://doi.org/10.5281/zenodo.18750029}
}
```

---

## Links

- 📄 **Paper**: [Zenodo — doi.org/10.5281/zenodo.18750029](https://doi.org/10.5281/zenodo.18750029)
- 🔗 **vLLM Integration Issue**: [GitHub #1422](https://github.com/vllm-project/semantic-router/issues/1422)


---

## Author

**Mossaab Souaissa**  
Independent AI Architecture Researcher

---

*PRISM is an independent research contribution. It is not affiliated with or endorsed by any of the organizations mentioned in this repository.*
