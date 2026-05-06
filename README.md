# Cloud Compliance Automation

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that maps cloud resources to compliance frameworks (SOC 2, PCI, HIPAA, ISO 27001) with continuous evidence collection and cross-framework deduplication.

Cloud Compliance Automation continuously discovers cloud resources, evaluates them against the controls of major compliance frameworks, and produces audit-ready evidence. It is built for engineering, security, and GRC teams who need fast audit readiness without the steeply-priced enterprise contracts of incumbents like Vanta and Drata.

---

## Why Cloud Compliance Automation?

- Incumbents (Vanta, Drata, Secureframe) start around $7,500–$10,000/yr for a single framework and commonly scale to $80,000–$120,000+/yr for multi-framework enterprise contracts, leaving early-stage startups underserved.
- Existing open-source options (Prowler, CISO Assistant, OpenSCAP) cover scanning and framework mapping but lack integrated audit workflow, evidence tracking, and AI-driven automation.
- Approximately 70% of SOC 2 controls map to ISO 27001 and ~65% overlap with HIPAA, yet most tools still require redundant evidence collection per framework.
- AWS Audit Manager is AWS-only and offers no cross-cloud or SaaS coverage; CSPM tools (Wiz, Prisma Cloud, Orca) treat compliance as a secondary capability with limited audit workflow tooling.
- The EU AI Act, DORA, and NIS2 are emerging compliance surfaces with little open-source coverage today.

---

## Key Features

### Continuous Evidence Collection

- Automated cloud resource discovery and compliance checks across AWS, Azure, and GCP
- Pre-built control mappings for SOC 2, ISO 27001, HIPAA, and PCI DSS
- Cross-framework deduplication so evidence collected once satisfies all mapped frameworks
- Continuous control testing rather than scheduled-only scans

### AI-Powered Compliance Workflows

- Natural-language gap analysis: describe a system change and identify affected controls
- AI-generated remediation playbooks with step-by-step infrastructure fix guidance
- AI-assisted evidence hunting and policy drafting
- Predictive audit risk scoring based on historical evidence patterns

### Audit and Reporting

- Compliance posture dashboard with control status, evidence status, and audit readiness score
- Evidence export and auditor report generation
- Auditor collaboration view for in-platform review
- Trust centre / public compliance status page

### Workflow and Integration

- CI/CD integration for policy-as-code compliance checks at build time
- Integrations with Jira, GitHub, Slack, and Okta for remediation workflows
- Employee compliance workflows: policy acceptance tracking and training completion
- Vendor risk management and security questionnaire automation

### Extensibility

- Custom framework builder for internal policies and emerging standards (EU AI Act, DORA, NIS2)
- Self-hosted deployment option for data sovereignty requirements
- API access for programmatic evidence and control management

---

## AI-Native Advantage

AI moves compliance from periodic scans to continuous, contextual evaluation. Instead of static rule checks, an AI agent autonomously gathers and classifies evidence, evaluates evidence quality, and flags controls likely to fail an upcoming audit. Natural-language gap analysis lets engineers describe a feature change and immediately see which controls across all active frameworks are implicated. AI-generated remediation playbooks turn a failed control into specific, infrastructure-aware fix steps rather than a generic recommendation.

---

## Tech Stack & Deployment

The project targets multi-cloud coverage (AWS, Azure, GCP minimum) with both SaaS and self-hosted deployment options. Control mappings are derived from publicly available standards bodies (AICPA, NIST, PCI SSC, HHS) rather than reproducing proprietary mappings. Integration is expected via REST APIs and webhooks, with CI/CD hooks for policy-as-code checks. Existing open-source engines such as Prowler (Apache 2.0) and CISO Assistant (LGPL-3.0) provide reference implementations for scanning and GRC workflows respectively.

---

## Market Context

The global compliance management software market is valued in the several billions of dollars and growing at roughly 13–16% CAGR (research.md). Drata has raised approximately $328M (valued ~$2B); Vanta has raised over $200M; Secureframe and Sprinto have each raised $80M+. Primary buyers are CTOs and CISOs at growth-stage SaaS companies pursuing first SOC 2 certification, GRC teams managing multiple frameworks at enterprises, and DevSecOps engineers wiring compliance into CI/CD pipelines.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
