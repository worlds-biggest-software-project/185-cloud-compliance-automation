# Standards & API Reference

> Project: Cloud Compliance Automation · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems (ISMS)**
- Official URL: https://www.iso.org/standard/82875.html
- The international standard for information security management. Widely required for enterprise and European sales. Version 2022 introduced 11 new controls and reorganised Annex A. Approximately 70% of its controls map directly to SOC 2, making it a key cross-framework target.

**ISO/IEC 27002:2022 — Information Security Controls**
- Official URL: https://www.iso.org/standard/75652.html
- Implementation guidance for ISO 27001 Annex A controls. Provides the control descriptions and implementation guidance that compliance automation tools use when assessing adherence to ISO 27001. Essential reference for building control libraries.

**ISO/IEC 27017:2015 — Cloud Security Controls**
- Official URL: https://www.iso.org/standard/43757.html
- Provides guidelines for information security controls applicable to cloud services. Directly relevant to any tool that assesses cloud-hosted infrastructure compliance. Extends ISO 27002 with cloud-specific controls.

**ISO/IEC 27018:2019 — Protection of PII in Public Clouds**
- Official URL: https://www.iso.org/standard/76559.html
- Code of practice for protection of personally identifiable information (PII) in cloud environments. Relevant when compliance automation covers GDPR and data privacy obligations for cloud-hosted data processing.

**ISO/IEC 27701:2019 — Privacy Information Management (PIMS)**
- Official URL: https://www.iso.org/standard/71670.html
- Extension to ISO 27001 and 27002 for managing privacy information. Provides the PIMS framework that maps to GDPR requirements, making it a key target for compliance tools covering EU data protection.

**ISO/IEC 42001:2023 — AI Management System**
- Official URL: https://www.iso.org/standard/81230.html
- First international standard for AI management systems. Emerging compliance target as organisations adopt AI systems. Relevant for platforms that also cover EU AI Act compliance obligations.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The OAuth 2.0 standard used for API authentication across all major compliance automation platforms (Vanta, Drata, Secureframe all use OAuth 2.0 client credentials for their APIs). Essential for any compliance tool offering an API.

**RFC 7519 — JSON Web Token (JWT)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is used as the bearer token format in OAuth 2.0 API authentication flows. Compliance automation APIs commonly issue JWTs as access tokens for API consumers.

**RFC 7807 — Problem Details for HTTP APIs**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard error response format for REST APIs. Relevant for any compliance automation API to provide consistent, machine-readable error responses.

**RFC 9110 — HTTP Semantics**
- Official URL: https://datatracker.ietf.org/doc/html/rfc9110
- The base HTTP specification governing REST API design. All compliance automation REST APIs are built on HTTP semantics.

---

### Data Model & API Specifications

**OSCAL — Open Security Controls Assessment Language (NIST)**
- Official URL: https://pages.nist.gov/OSCAL/
- GitHub: https://github.com/usnistgov/OSCAL
- NIST's machine-readable format standard (XML, JSON, YAML) for representing security controls, assessments, and system security plans. Enables automation of compliance documentation and dramatically reduces audit durations from months to minutes. Supports NIST SP 800-53 r5, CSF v2.0, NIST SP 800-171 r3, and NIST SP 800-218. The key interoperability standard for compliance automation tooling — any AI-native compliance tool should consider OSCAL as an import/export format.

**OpenAPI Specification 3.1**
- Official URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting and defining REST APIs. All major compliance automation platforms (Vanta, Drata, Secureframe, Sprinto, Hyperproof) publish OpenAPI-compliant documentation. Any compliance automation API should be described using OpenAPI 3.1.

**GraphQL**
- Official URL: https://spec.graphql.org/
- Sprinto uses a GraphQL-based developer API (vs REST-only competitors), enabling more flexible and efficient querying of compliance data. Relevant for rich control and evidence data models.

**JSON Schema**
- Official URL: https://json-schema.org/
- Used for validating compliance data structures, OSCAL documents, and API request/response payloads. Standard vocabulary for describing the shape of compliance data.

**Cloud Events v1.0**
- Official URL: https://cloudevents.io/
- CNCF standard for describing event data in a common way. Relevant for webhook payloads and event-driven compliance notification architectures.

---

### Security & Authentication Standards

**SOC 2 (AICPA Trust Services Criteria)**
- Official URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2
- The primary compliance framework for US SaaS vendors. Built around five Trust Service Categories: Security, Availability, Processing Integrity, Confidentiality, and Privacy. Type I evaluates design at a point in time; Type II evaluates operational effectiveness over 3–12 months. Approximately 70% of controls overlap with ISO 27001.

**PCI DSS 4.0 (PCI Security Standards Council)**
- Official URL: https://www.pcisecuritystandards.org/document_library/
- Payment card data security standard with 12 core requirements. Version 4.0 is now fully in effect (2024), increasing automation demand. Required for all entities that store, process, or transmit cardholder data.

**HIPAA Security Rule (US Department of Health and Human Services)**
- Official URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- US healthcare data protection regulation covering electronic protected health information (ePHI). Applies to covered entities (hospitals, insurers) and their business associates. Approximately 65% of controls overlap with SOC 2.

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls**
- Official URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The comprehensive US government security control catalogue (1,000+ controls). Basis for FedRAMP compliance requirements. Widely used as an internal control framework by enterprises beyond the federal sector.

**NIST Cybersecurity Framework (CSF) 2.0**
- Official URL: https://www.nist.gov/cyberframework
- Voluntary framework organised around six functions: Identify, Protect, Detect, Respond, Recover, Govern. CSF 2.0 (2024) added the Govern function. Widely adopted for commercial enterprise compliance strategy and maps extensively to SOC 2, ISO 27001, HIPAA, and PCI DSS.

**NIST SP 800-171 Rev. 3 — Protecting CUI**
- Official URL: https://csrc.nist.gov/publications/detail/sp/800-171/rev-3/final
- Requirements for protecting Controlled Unclassified Information (CUI) in non-federal systems. Basis for CMMC (Cybersecurity Maturity Model Certification) compliance required for US Department of Defense contractors.

**FedRAMP — Federal Risk and Authorization Management Program**
- Official URL: https://www.fedramp.gov/
- US government cloud security authorisation programme. Requires NIST 800-53-based control implementation. Growing demand for compliance automation coverage given cloud migration in federal agencies.

**GDPR — General Data Protection Regulation (EU)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679
- EU data protection regulation with global reach. Compliance automation must track Article 30 Records of Processing Activities, DPA management, and evidence of technical/organisational measures.

**NIS2 Directive (EU Network and Information Security)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32022L2555
- EU cybersecurity directive applying to critical infrastructure and essential/important entities. Transposed into national law across EU member states. Overlaps significantly with ISO 27001 and GDPR controls.

**DORA — Digital Operational Resilience Act (EU)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32022R2554
- EU regulation specific to financial services sector; in force from January 17, 2025. Overlaps with NIS2 and ISO 27001 but adds ICT risk management, incident reporting, and third-party risk requirements. Key emerging framework for compliance automation.

**CIS Controls v8 (Center for Internet Security)**
- Official URL: https://www.cisecurity.org/controls/v8
- 18 prioritised security controls widely used as a baseline for cloud security benchmarks. CIS Benchmarks (configuration hardening guides) are used by Prowler, Wiz, Prisma Cloud, and other tools as built-in compliance checks.

**CMMC 2.0 — Cybersecurity Maturity Model Certification**
- Official URL: https://www.acq.osd.mil/cmmc/
- US DoD contractor compliance requirement based on NIST 800-171. Three levels of maturity. Growing compliance automation market for defence supply chain organisations.

**Open Policy Agent (OPA)**
- Official URL: https://www.openpolicyagent.org/
- CNCF open-source policy engine using the Rego language. The most widely adopted open-source policy-as-code standard for compliance enforcement across Kubernetes, Terraform, APIs, and microservices. Generates audit-trail-quality logs suitable for ISO 27001, SOC 2, NIS2, and DORA evidence.

**SCAP — Security Content Automation Protocol (NIST)**
- Official URL: https://csrc.nist.gov/projects/security-content-automation-protocol
- NIST standard protocol for automated vulnerability management and compliance checking. Used by OpenSCAP for OS and container image hardening checks. Relevant for infrastructure-level compliance scanning.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- Official URL: https://modelcontextprotocol.io/
- Open protocol for connecting AI models to external data sources and tools. Directly relevant for an AI-native compliance automation tool: MCP servers can expose compliance data (control status, evidence, findings) to AI agents for natural-language analysis, gap identification, and remediation guidance generation. An MCP server wrapping a compliance automation engine would allow AI agents to query and act on compliance state programmatically.

---

## Similar Products — Developer Documentation & APIs

### Vanta
- **Description:** Leading compliance automation platform for SOC 2, ISO 27001, HIPAA, PCI, and GDPR with 375+ integrations and AI Agent 2.0 for autonomous evidence collection.
- **API Documentation:** https://developer.vanta.com/docs/vanta-api-overview
- **SDKs/Libraries:** No official SDK; API tracker listing at https://apitracker.io/a/vanta
- **Developer Guide:** https://developer.vanta.com/docs/vanta-first-api-request
- **Authentication:** OAuth 2.0 client credentials
- **Standards:** REST/JSON, OpenAPI
- **Rate Limits:** 50 req/min (management), 20 req/min (integrations), 5 req/min (OAuth)

### Drata
- **Description:** Trust management platform with deep compliance automation, risk management, and audit readiness across SOC 2, ISO 27001, HIPAA, PCI DSS, and more.
- **API Documentation:** https://developers.drata.com/api-docs/
- **SDKs/Libraries:** Auto-generated code samples in multiple languages; Python client via dltHub at https://dlthub.com/context/source/drata
- **Developer Guide:** https://developers.drata.com/
- **Authentication:** Bearer token (API key)
- **Standards:** REST/JSON, OpenAPI
- **Rate Limits:** 500 requests/minute per IP
- **Note:** API access requires Advanced plan or above

### Secureframe
- **Description:** Multi-framework compliance platform with 150+ integrations covering SOC 2, ISO 27001, HIPAA, PCI DSS, and GDPR with white-glove onboarding support.
- **API Documentation:** https://api.secureframe.com/docs
- **SDKs/Libraries:** Multiple language samples in developer portal
- **Developer Guide:** https://secureframe.com/features/api
- **Authentication:** API key / OAuth
- **Standards:** REST/JSON, OpenAPI

### Sprinto
- **Description:** Cloud-hosted compliance automation with 200+ integrations and entity-level risk scoring, strong in APAC and emerging markets.
- **API Documentation:** https://developer.sprinto.com/docs/sprinto-developer-api-documentation
- **SDKs/Libraries:** https://docs.sprinto.com/api-references/sprinto-developer-api
- **Developer Guide:** https://docs.sprinto.com/settings/developer-api
- **Authentication:** API key
- **Standards:** GraphQL (unique in category), also REST
- **Note:** GraphQL-based API provides more flexible querying than REST-only competitors

### Hyperproof
- **Description:** Compliance management platform with strong FedRAMP and CMMC support, automated Hypersyncs evidence collection, and executive-oriented reporting.
- **API Documentation:** https://developer.hyperproof.app/
- **SDKs/Libraries:** Flexible SDK for custom integrations
- **Developer Guide:** https://developer.hyperproof.app/
- **Authentication:** OAuth / API key
- **Standards:** REST/JSON, OpenAPI

### Prowler (Open Source)
- **Description:** World's most widely used open-source cloud security platform, automating security and compliance checks across AWS, Azure, GCP, Kubernetes, GitHub, and Microsoft 365 against 130+ frameworks.
- **API Documentation:** https://docs.prowler.com/ (CLI and SaaS API)
- **SDKs/Libraries:** Python package: `pip install prowler`; GitHub: https://github.com/prowler-cloud/prowler
- **Developer Guide:** https://docs.prowler.com/projects/prowler-open-source/en/latest/
- **Authentication:** Cloud provider credentials (AWS, Azure, GCP); API key for Prowler Pro SaaS
- **Standards:** REST/JSON (SaaS API); CLI/Python (open-source)
- **Licence:** Apache 2.0 (open source), Commercial (Prowler Pro)

### CISO Assistant (Open Source)
- **Description:** Open-source GRC platform supporting 130+ frameworks with automatic control mapping, risk management, and REST API for automation. Integrates with Prowler.
- **API Documentation:** https://intuitem.com/ and https://github.com/intuitem/ciso-assistant-community
- **SDKs/Libraries:** REST API; CLI toolbox
- **Developer Guide:** https://github.com/intuitem/ciso-assistant-community/blob/main/README.md
- **Authentication:** API key / session token
- **Standards:** REST/JSON; OSCAL import/export
- **Licence:** LGPL-3.0

### AWS Audit Manager
- **Description:** Native AWS service for automated evidence collection and compliance assessment against pre-built and custom frameworks including CIS, NIST, PCI DSS, HIPAA, and SOC 2.
- **API Documentation:** https://docs.aws.amazon.com/audit-manager/latest/APIReference/
- **SDKs/Libraries:** AWS SDK (Python/Boto3, Java, Go, JavaScript, .NET, Ruby, PHP, C++)
- **Developer Guide:** https://docs.aws.amazon.com/audit-manager/latest/userguide/
- **Authentication:** AWS IAM (roles and policies)
- **Standards:** REST/JSON via AWS SDK; native AWS API conventions

### Open Policy Agent (OPA)
- **Description:** CNCF open-source policy engine using Rego language for policy-as-code compliance enforcement across Kubernetes, Terraform, APIs, and microservices.
- **API Documentation:** https://www.openpolicyagent.org/docs/latest/rest-api/
- **SDKs/Libraries:** Go SDK (native); Python, JavaScript, Java clients available; Conftest for IaC testing
- **Developer Guide:** https://www.openpolicyagent.org/docs/latest/
- **Authentication:** N/A (embedded or self-hosted service)
- **Standards:** REST/JSON; Rego policy language; integrates with OpenTelemetry for audit logging
- **Licence:** Apache 2.0

### Wiz
- **Description:** Agentless CSPM platform with 2,300+ misconfiguration rules and 150+ built-in compliance benchmarks covering CIS, SOC 2, PCI, HIPAA, ISO 27001, and GDPR across multi-cloud environments.
- **API Documentation:** https://docs.wiz.io/wiz-docs/docs/using-the-wiz-api
- **SDKs/Libraries:** GraphQL API; Terraform provider; Kubernetes operator
- **Developer Guide:** https://docs.wiz.io/
- **Authentication:** OAuth 2.0 client credentials
- **Standards:** GraphQL; REST/JSON for webhooks

---

## Notes

**OSCAL as the interoperability foundation**: NIST's OSCAL is the most important emerging standard for compliance automation interoperability. Any AI-native tool should adopt OSCAL as both an input format (importing existing system security plans and control catalogues) and output format (producing machine-readable compliance artefacts). OSCAL is already supported by FedRAMP and is gaining traction in the broader enterprise GRC market.

**Policy-as-code is maturing**: The OPA/Rego ecosystem is now the de-facto standard for infrastructure compliance as code. Compliance automation tools should consider integrating with OPA for CI/CD-level enforcement, complementing their SaaS-layer compliance management.

**EU regulatory frameworks are creating new demand**: DORA (in force January 2025), NIS2, and the EU AI Act (August 2026) are driving demand for compliance automation tools that cover European frameworks alongside US-centric ones. Most current tools have stronger US (SOC 2, HIPAA, PCI) coverage than EU coverage.

**GraphQL vs REST**: Sprinto's adoption of GraphQL for its developer API is an outlier in the category but enables richer querying of compliance data models. For an AI-native tool that needs to expose complex compliance relationships (control-to-evidence-to-framework mappings), GraphQL may be worth considering alongside REST.

**MCP integration opportunity**: Exposing a compliance automation engine via an MCP server would allow AI coding assistants and agents (Claude, GitHub Copilot, etc.) to query compliance status, identify affected controls for proposed code changes, and generate remediation guidance inline in developer workflows — a significant differentiation opportunity not yet exploited by commercial platforms.
