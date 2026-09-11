# AegisGate Security Policy

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security issue in any AegisGate product, please report it responsibly.

### How to Report

**Do NOT open a public GitHub issue for security vulnerabilities.**

Instead, please email: **security@aegisgatesecurity.io**

Include the following:
1. Product name (Platform, Rampart, Lens, or Enterprise)
2. Affected version
3. Description of the vulnerability
4. Steps to reproduce
5. Potential impact
6. Suggested fix (if any)

### Response Timeline

| Milestone | Target |
|-----------|--------|
| Acknowledgment | Within 48 hours |
| Initial assessment | Within 5 business days |
| Fix or mitigation | Within 30 days (severity-dependent) |
| Public disclosure | After fix is released, coordinated with reporter |

### Scope

**In scope:**
- AegisGate Platform (gateway proxy)
- AegisGate Rampart (local proxy)
- AegisGate Lens (browser extension)
- AegisGate Enterprise components
- Marketing website (aegisgatesecurity.io)

**Out of scope:**
- Vulnerabilities in third-party dependencies (report to upstream maintainers)
- Social engineering attacks
- Physical security
- DoS or DDoS attacks
- Automated scanner reports without manual verification

### Safe Harbor

We will not pursue legal action against security researchers who:
- Act in good faith
- Do not access or modify data belonging to others
- Do not degrade or disrupt our services
- Report vulnerabilities through the process above
- Give us reasonable time to fix before public disclosure

### Recognition

With your permission, we will acknowledge your contribution in our release notes and security advisories.

## Security Measures

### Development Practices
- All code changes require DCO (Developer Certificate of Origin) sign-off
- CI pipelines include: static analysis (gosec, CodeQL), secret scanning (gitleaks, Trivy), dependency scanning (Dependabot, govulncheck), and SBOM generation (CycloneDX + SPDX)
- All releases are GPG-signed
- All release artifacts include SBOM

### Product Security
- **Platform/Rampart**: TLS-encrypted proxy, zero outbound telemetry by default, all detection on-device
- **Lens**: 100% on-device detection, zero prompt data leaves the browser, strict CSP (`script-src 'self'`), no external dependencies
- **Enterprise**: Trade secret protected, confidentiality headers on all source files, proprietary license

### Cryptography
Products use standard cryptographic primitives (TLS 1.2+, ECDSA P-256 for Trust Framework, AES-256 for data at rest). No custom cryptography is used.

## Contact

- Security: security@aegisgatesecurity.io
- General: https://github.com/aegisgatesecurity/aegisgate-platform/discussions