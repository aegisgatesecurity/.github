# AegisGate Security

**Self-hosted AI security platform — inspect every AI request and response for secrets, PII, prompt injection, and compliance violations.**

[![Website](https://img.shields.io/badge/website-aegisgatesecurity.io-blue)](https://aegisgatesecurity.io)
[![License](https://img.shields.io/badge/license-Apache%202.0-green)](https://www.apache.org/licenses/LICENSE-2.0)
[![CI](https://img.shields.io/badge/CI-all%20repos%20green-brightgreen)](https://github.com/aegisgatesecurity)

---

## What We Do

AegisGate inspects traffic between your applications and AI services (LLMs, MCP servers, A2A agents) — catching sensitive data before it leaves your environment and scanning AI responses before they reach your users.

**Three products:**

| Product | What | Target |
|---------|------|--------|
| [Platform](https://github.com/aegisgatesecurity/aegisgate-platform) | Gateway proxy — inspects all AI traffic at the network level | Enterprises, teams |
| [Rampart](https://github.com/aegisgatesecurity/aegisgate-rampart) | Local proxy — protects AI coding tools (Copilot, Cursor, etc.) | Developers |
| [Lens](https://github.com/aegisgatesecurity/aegisgate-lens) | Browser extension — warns before you send sensitive data to AI chat | Everyone |

**Enterprise edition** (private): Trust Framework (per-agent ECDSA identity + capability contracts), SIEM integration (12 platforms), premium compliance (SOC 2, HIPAA, PCI DSS), and model training pipeline.

## Key Technology

- **3-layer detection pipeline**: Regex scanner → ATLAS/compliance engine → Char CNN-BiLSTM neural network
- **176 regex patterns** across 6 pillars (PII, secrets, harmful content, injection, compliance, OT/ICS)
- **31+ compliance frameworks** (OWASP LLM Top 10, MITRE ATLAS, NIST CSF, ISO 27001, EU AI Act, HIPAA, PCI DSS, SOC 2, and more)
- **Multi-protocol support**: HTTP, MCP, A2A, ACP, ANP
- **Zero dependencies** — single Go binary for Platform and Rampart
- **Privacy-first** — Lens runs 100% on-device, zero data collection

## Quick Links

- 🌐 [Website](https://aegisgatesecurity.io)
- 📖 [Documentation](https://aegisgatesecurity.io/docs/)
- 🔒 [Security Policy](https://github.com/aegisgatesecurity/.github/blob/main/SECURITY.md)
- 💬 [Discussions](https://github.com/aegisgatesecurity/aegisgate-platform/discussions)
- 📧 Contact: security@aegisgatesecurity.io

## Repositories

| Repo | Description |
|------|-------------|
| [aegisgate-platform](https://github.com/aegisgatesecurity/aegisgate-platform) | Core gateway proxy (Go, Apache 2.0) |
| [aegisgate-rampart](https://github.com/aegisgatesecurity/aegisgate-rampart) | Local proxy for AI coding tools (Go, Apache 2.0) |
| [aegisgate-lens](https://github.com/aegisgatesecurity/aegisgate-lens) | Browser extension (JS, Apache 2.0) |
| [aegisgate-enterprise](https://github.com/aegisgatesecurity/aegisgate-enterprise) | Enterprise components (private, commercial license) |
| [aegisgate-site](https://github.com/aegisgatesecurity/aegisgate-site) | Marketing website (Hugo + Netlify) |
| [aegisgate-demo](https://github.com/aegisgatesecurity/aegisgate-demo) | Live demo environment |
| [aegisgate-rampart-jetbrains](https://github.com/aegisgatesecurity/aegisgate-rampart-jetbrains) | JetBrains IDE plugin |
| [aegisgate-rampart-ext](https://github.com/aegisgatesecurity/aegisgate-rampart-ext) | VS Code / Cursor extension |

## License

Open-source products are licensed under **Apache 2.0**. Trained ML model weights are separately licensed under the **AegisGate Model Weight License v1.0** (non-commercial use granted, commercial use requires a license — see [WEIGHTS-LICENSE.md](https://github.com/aegisgatesecurity/aegisgate-platform/blob/main/WEIGHTS-LICENSE.md) in each repo).

Enterprise components are proprietary — see the [enterprise repository](https://github.com/aegisgatesecurity/aegisgate-enterprise) for licensing inquiries.

## Copyright

Copyright © 2026 AegisGate Security, LLC. All rights reserved.