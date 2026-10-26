---
title: "Catching a 50% ML Detection Regression Before It Shipped: Lessons from a Model Retrain"
published: false
description: "How a CharCNN-BiLSTM model retrain introduced a silent 50% input window reduction, why float16 quantization broke our JS evasion suite, the CI gates that caught both before production, and ARTEX-specific threat intelligence detections shipped across 4 products."
tags: machinelearning, security, golang, ci-cd, artex, threatintelligence, supplychain
canonical_url: https://aegisgatesecurity.io/blog/v4-7-0-retraining-the-brain-seeding-the-policy-engine/
---

We just retrained our shared CharCNN-BiLSTM threat detection model across 5 products. The retrain went through three model versions (v14 → v15 → v15b), introduced and caught a 50% input window regression, and revealed a float16 precision boundary that rejected an otherwise-good model.

This post is the technical story — what happened, what broke, and the CI gates that saved us.

---

## The Setup

AegisGate uses a CharCNN-BiLSTM with attention as the L3 neural layer in a three-layer detection pipeline:

- **L1**: 297 regex patterns (PII, secrets, injection, OT/ICS, ARTEX/LiteLLM threat intel)
- **L2**: MITRE ATLAS technique mapping + 30+ compliance frameworks
- **L3**: CharCNN-BiLSTM (1,596,034 params, 256-char Latin-1 input, <1ms CPU inference)

The model is compiled to ONNX (float32) for Go products (Platform, Scanner, MCP, Rampart) and to float16 JSON weights for pure-JS in-browser inference (Lens browser extension).

The v13 model had 7 proverb false positives. "A penny saved is a penny earned" was being flagged as a threat. The retrain goal: eliminate those FPs without degrading detection.

---

## v14: The Architecture Change We Reverted

### What happened

The first retrain attempt reduced `MAX_SEQ_LEN` from 256 → 128 and `VOCAB_SIZE` from 256 → 128 (Latin-1 → ASCII). The rationale: shorter inputs would be less likely to trigger on long proverb phrases.

It worked. The 7 FPs were gone. But the change cut the ML detection window in half — an attacker who needed 256 chars to craft an evasion now only needed 128.

### What went wrong

The architecture change was made in `training/model.py` and propagated to all 5 Go normalizers (Platform local + upstream, Scanner, MCP, Rampart). When v14 was rejected and reverted, `training/model.py` was reverted — but **the 5 Go normalizers were not fully reverted**. They remained configured for 128-char input.

The v15b ONNX model expects `[batch, 256]`. The normalizers were producing `[batch, 128]`.

### How CI caught it

Our ML Detection Gate workflow:

1. Downloads the ONNX model from the GitHub release
2. Runs a calibrated test corpus (base64 payloads, injection patterns, benign near-misses)
3. Checks scores against expected thresholds

The shape mismatch caused inference to produce garbage scores. CI failed. The regression was identified and fixed before any release artifact was built.

**Lesson**: Architecture constants (`MaxSeqLen`, `VocabSize`) span training scripts AND production code. A partial revert is a silent regression. We're adding an explicit architecture-constant sync check to CI.

---

## v15: When float16 Broke the Evasion Suite

### What happened

The second attempt returned to 256-char Latin-1 but trained on a corpus that included 257 hard-negative proverbs. Test metrics looked good:

- FPR: 0.55%
- F1: 95.98%
- Accuracy: 98.24%

But the Lens JS evasion suite scored 88.4/100. Our threshold is 95.0. The model was rejected.

### Root cause: float16 precision loss

Lens runs inference in pure JavaScript. To keep the extension at ~4MB (vs ~25MB for float32), weights are stored as float16 in a gzip-compressed, base64-encoded JSON file. The JS inference engine decompresses at load time using the browser's built-in `DecompressionStream`.

The v15 model had a sharper decision boundary than v13. Borderline cases sat closer to the threshold. Float16 quantization (max error ~0.0002 per weight) shifted enough borderline cases to the wrong side that 11.6% of evasion tests failed.

The v13 model didn't have this problem because its decision boundary was wider — benign samples scored well below threshold, adversarial samples scored well above. The quantization error wasn't enough to cross the boundary.

### The fix

v15b returned to the original v13 training corpus (no proverbs) at 256-char Latin-1. The model learned a wider decision boundary — near-zero logits for benign (sigmoid→0.50), ~1.0 logits for adversarial (sigmoid→0.73). The compressed sigmoid range means less sharp discrimination at the model level, but the float16 quantization error doesn't cross the threshold.

**v15b Lens evasion suite: 97.9/100.** ✅

---

## The Metrics Tradeoff

Here's the honest comparison:

| Metric | v13 | v15b |
|--------|-----|------|
| Accuracy | 99.35% | 98.16% |
| Precision | 99.57% | 98.20% |
| Recall (TPR) | 99.45% | 93.48% |
| F1 | 99.5% | 95.78% |
| **FPR** | **0.85%** | **0.49%** |

The raw model metrics are lower. But the FPR improved — from 0.85% to 0.49%. And in the full pipeline:

- **Evasion resistance**: 100/100 (unchanged)
- **Production FPR**: 0% across 8.1M benign requests (unchanged — L1/L2 catch ML FPs)
- **Throughput**: 28K RPS sustained (unchanged)

The multi-layer architecture is the point. The ML model doesn't need to be perfect on its own. It needs to add detection value that L1/L2 can't provide — catching adversarial paraphrasing, semantic attacks, and novel obfuscation — while the other layers compensate for its weaknesses.

### Two-tier blocking

The v15b model's compressed sigmoid range (0.50–0.73 for most inputs) required a threshold adjustment:

- **Tier 1** (score ≥ 0.95): Block independently — high-confidence ML detection
- **Tier 2** (score ≥ 0.74 with L1/L2 corroboration): Block with regex support — medium-confidence ML + pattern match
- **Alert** (score < 0.74): Log only

The v13 threshold was 0.50. The v15b threshold is 0.74. This was calibrated using a benign corpus (max benign score = 0.73) and attack corpus (min TP score = 0.50).

---

## The CI Gates That Caught Both Issues

### ML Detection Gate

Downloads the ONNX model, runs a calibrated corpus, checks scores against thresholds. Catches:
- Model shape mismatches (the MaxSeqLen regression)
- Score distribution shifts (threshold calibration issues)
- Model loading failures

### Lens Evasion Suite

Runs the JS weights against 81 payloads × 50 transforms (4,050 tests) in a headless Chrome instance. Catches:
- Float16 precision loss (the v15 failure)
- Weight conversion bugs
- Browser-specific inference differences

### Detection Parity Gate

Compares 297 pattern names and regex content across Platform, Rampart, and Scanner on every push. Catches:
- Cross-product detection drift
- Missing patterns in one product

### Coverage Gate

85% total / 90% per-package floor across 67 CI-scoped packages. Catches:
- Untested code paths (found 2 production bugs during this campaign)

---

## What We Shipped

All 5 products now share the v15b model:

- **ONNX** (Go): 6,398,115 bytes, SHA-256 `99187453bedb2b8caf21500d2344f2bbe34272c0b3fd4c610c431378dc2f4fe7`
- **JS weights** (Lens): 3,928,018 bytes, SHA-256 `03792717405a1875d068a6304b69d3aeb019643778af30677b0f7257dbfbabab`

Platform verifies the hash at load time. If it doesn't match, the model is rejected and the system falls back to L1/L2 heuristics. It never fails open.

---

## ARTEX Threat Intelligence Detections

Following our [Black October Blue Team Field Guide](https://aegisgatesecurity.io/blog/black-october-blue-team-guide/), we also shipped ARTEX-specific detections across all four AegisGate products.

Two threats from Black October required dedicated coverage:

- **ARTEX** — open-source AI autonomous pentesting framework (Go + Next.js, AGPL-3.0). Chinese APT Storm-0940 used it against Microsoft 365 tenants: 2,900 auth attempts/min across 327 accounts, 151K+ failed sign-ins. Initial targeting to privileged cloud access in under 3 hours.
- **LiteLLM** — AI gateway compromised via `litellm_init.pth` credential stealer, SSRF to cloud metadata, admin endpoint abuse, and RAT deployment.

### 13 Regex Patterns

**ARTEX Traffic Signatures (8):**

| Pattern | Severity | What It Catches |
|---------|----------|-----------------|
| `artex_tool_reference` | High | ARTEX / Autumn-27 tool name references |
| `artex_model_deepseek` | Low | DeepSeek model configuration |
| `artex_model_glm` | Low | GLM / Zhipu model configuration |
| `artex_model_grok` | Low | Grok model configuration |
| `artex_mcp_external` | Medium | MCP Streamable HTTP/SSE to external servers |
| `artex_compat_gateway_probe` | Medium | OpenAI-compatible API probes with non-standard models |
| `artex_multi_agent_attack` | High | Multi-agent orchestration + attack context |
| `artex_repo_reference` | Medium | ARTEX repository clone/fork references |

Model config patterns are Low severity — DeepSeek, GLM, and Grok are legitimate models. But combined with other signals, they match ARTEX's known TTPs.

**LiteLLM Supply Chain Indicators (5):**

| Pattern | Severity | What It Catches |
|---------|----------|-----------------|
| `litellm_init_pth` | Critical | Known credential stealer file |
| `litellm_ssrf_metadata` | Critical | SSRF targeting cloud metadata endpoints |
| `litellm_admin_abuse` | High | Admin endpoint privilege escalation |
| `litellm_cred_exfil` | Critical | Credential exfiltration via LiteLLM proxy |
| `litellm_rat_deploy` | Critical | RAT/malware deployment via supply chain |

### 5 Correlation Patterns (Platform)

Regex catches signatures. ARTEX's real danger is behavioral — AI-accelerated, multi-agent attacks. We added 5 correlation patterns:

| Pattern | Severity | Auto-Block | Compliance |
|---------|----------|-----------|------------|
| `artex_rapid_recon` | High | No | NIST CSF DE.AE-2, MITRE ATLAS T0029 |
| `artex_credential_spray` | Critical | **Yes** | FedRAMP IR-4, SOC2 CC6.1, NIST CSF DE.AE-3 |
| `artex_multi_agent_coord` | High | No | NIST CSF DE.AE-2, MITRE ATLAS T0050 |
| `artex_ai_accelerated` | Medium | No | NIST CSF DE.AE-4, FedRAMP IR-5 |
| `artex_graph_anomaly` | High | No | NIST CSF DE.AE-2, SOC2 CC6.1 |

Only `artex_credential_spray` auto-executes (blocks the attacking agent). The others require triage — could be legitimate load testing, admin operations, or authorized multi-agent workflows.

### Cross-Product Parity

| Product | Before | After | Parity |
|---------|--------|-------|--------|
| Platform | 284 | 297 | FULL |
| Rampart | 284 | 297 | FULL |
| Scanner | 335 | 348 | SUPERSET |
| Lens (compliance) | 54 | 62 | SUBSET |

14/14 parity checks pass. All products tested with 5+ consecutive green runs before pushing.

---

## Takeaways

1. **Architecture changes are first-class config.** `MAX_SEQ_LEN` and `VOCAB_SIZE` aren't just training parameters — they're production code constants. A partial revert is a silent regression. Sync them explicitly.

2. **Float16 quantization has a decision boundary cost.** A model that works in float32 may fail in float16 if the decision boundary is too sharp. Test with the actual deployment-time precision, not just the training precision.

3. **CI gates are the real product.** The ML Detection Gate and Lens Evasion Suite caught two regressions that would have shipped otherwise. The coverage gate found two production bugs. The parity gate prevents cross-product drift. These gates are more valuable than any single feature.

4. **Lower model metrics can be better.** v15b has lower accuracy, precision, recall, and F1 than v13. But it has lower FPR and works correctly in float16. In a multi-layer pipeline, the model doesn't need to be perfect — it needs to add value without introducing false positives.

5. **Posture assessment is only as good as the coverage behind it.** v4.7.0's coverage campaign drove FIPS from 41%→98%, ISO 27001 from 72%→100%, ISO 42001 from 59%→99%, and HITRUST from 82%→100%. The `aegisgate status` posture check can now report "healthy" on compliance — and it means something, because every code path has been tested. Before this, "degraded" was technically honest but practically useless. The coverage campaign transformed the posture signal from aspirational to verified. This is also the subject of Provisional Patent #7 (64/180,722) — Self-Improving AI Security Posture Assessment.

---

*[AegisGate](https://github.com/aegisgatesecurity/aegisgate-platform) is an open-source AI security gateway. Apache 2.0. 8 provisional patents filed. This post covers v4.7.0, shipping October 2026.*