<div align="center">
  <img src="assets/cover.svg" alt="SecTest Atlas — Authorized Security Testing Handbook" width="100%">
</div>

# SecTest Atlas

<p align="center">
  <img alt="31 chapters" src="https://img.shields.io/badge/chapters-31-2563eb.svg">
  <img alt="700+ checks" src="https://img.shields.io/badge/security_checks-700%2B-0f766e.svg">
  <img alt="zero dependencies" src="https://img.shields.io/badge/dependencies-zero-16a34a.svg">
  <img alt="2026 field edition" src="https://img.shields.io/badge/edition-2026-7c3aed.svg">
</p>

<p align="center"><strong>Map the attack path. Verify the defense. Preserve the evidence.</strong></p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="INSTALL.md">Install</a> ·
  <a href="AI_INSTALL.md">AI setup</a> ·
  <a href="README.zh-CN.md">简体中文</a> ·
  <a href="CONTRIBUTING.md">Contribute</a>
</p>

An execution-focused handbook for **authorized security assessments**. SecTest Atlas turns real attack paths into searchable, checkable, and actionable test cases across networks, servers, wireless, applications, cloud-native platforms, and AI systems.

[Open locally](./index.html) · [Chinese field handbook](./docs/handbook.md) · [Security policy](./SECURITY.md)

> [!WARNING]
> For authorized assessment, education, and defensive validation only. Obtain written permission and define scope, test windows, data handling, rollback, and emergency contacts before running any test.

## Why this project

- **31 chapters and 700+ checks** covering the full assessment lifecycle.
- **No build step:** one self-contained HTML file works directly in a browser.
- **Operator-friendly UI:** navigation search, persistent checkboxes, dark mode, mobile drawer, and print/PDF export.
- **Evidence-oriented:** findings are structured around observation, proof, impact, remediation, and retest.
- **Bilingual onboarding:** English-first project documentation with the complete Chinese field handbook included.

![Interactive handbook preview](assets/interface-preview.svg)

## Quick start

```bash
git clone https://github.com/cndoin/sectest-atlas.git
cd sectest-atlas
python -m http.server 8080
```

Open `http://localhost:8080`. There are no package dependencies and no build command. See [INSTALL.md](./INSTALL.md) for Windows/macOS/Linux instructions, or give [AI_INSTALL.md](./AI_INSTALL.md) to your coding agent.

## Coverage

| Area | Included topics |
| --- | --- |
| Frameworks | PTES, NIST SP 800-115, OSSTMM, MITRE ATT&CK, CIS Benchmarks |
| Network & host | L2–L4, DNS, segmentation, Linux/Windows, identity, credentials |
| Application & platform | Web, API, microservices, cloud, containers, Kubernetes, CI/CD |
| Wireless & emerging tech | WPA2/WPA3, Wi-Fi 6E/7, BLE, ZigBee, AI/LLM/agents |
| Resilience | Detection, purple teaming, recovery, ransomware, performance, chaos testing |
| Delivery | Risk scoring, reports, authorization, ROE, evidence, acceptance criteria |

## Repository layout

```text
.
├── index.html              # Interactive handbook and GitHub Pages entry
├── assets/                 # Brand and interface visuals
├── docs/handbook.md        # Complete Chinese Markdown handbook
├── INSTALL.md              # Local installation and serving guide
├── AI_INSTALL.md           # Ready-to-use instructions for coding agents
├── CONTRIBUTING.md         # Contribution rules
└── SECURITY.md             # Disclosure policy and legal boundary
```

## Responsible use

1. Test only assets covered by explicit written authorization.
2. Prefer isolated or pre-production validation before approved production windows.
3. Minimize evidence collection, redact sensitive data, and follow agreed retention rules.
4. Document every finding with observation, evidence, impact, remediation, and retest status.
5. Commands in the handbook are references, never permission to test an external target.

## Maintenance

Security standards and platform behavior change over time. Updates should cite authoritative sources, state applicable versions and verification dates, and distinguish mandatory requirements from best practices and environment-specific guidance.
