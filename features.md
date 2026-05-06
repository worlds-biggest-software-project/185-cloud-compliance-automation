# Cloud Compliance Automation — Feature & Functionality Survey

> Candidate #185 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Vanta | GRC / Compliance Automation | Commercial, SaaS | https://www.vanta.com |
| Drata | GRC / Compliance Automation | Commercial, SaaS | https://drata.com |
| Secureframe | GRC / Compliance Automation | Commercial, SaaS | https://secureframe.com |
| Sprinto | GRC / Compliance Automation | Commercial, SaaS | https://sprinto.com |
| Hyperproof | GRC / Compliance Management | Commercial, SaaS | https://hyperproof.io |
| Tugboat Logic (OneTrust) | GRC / Compliance Management | Commercial, SaaS | https://www.onetrust.com |
| Wiz | CSPM / Cloud Security | Commercial, SaaS | https://www.wiz.io |
| Prisma Cloud (Palo Alto) | CSPM / CNAPP | Commercial, SaaS | https://www.paloaltonetworks.com/prisma/cloud |
| Orca Security | CSPM / Cloud Security | Commercial, SaaS | https://orca.security |
| Prowler | Cloud Security Assessment | Open-source, Apache 2.0 | https://github.com/prowler-cloud/prowler |
| CISO Assistant | GRC Platform | Open-source, LGPL-3.0 | https://github.com/intuitem/ciso-assistant-community |
| AWS Audit Manager | Evidence Collection | Commercial (AWS native) | https://aws.amazon.com/audit-manager |

---

## Feature Analysis by Solution

### Vanta

**Core features**
- Continuous automated monitoring with hourly control tests across connected systems
- 375+ native integrations (cloud, identity, endpoint, ticketing, SaaS)
- 1,300+ pre-built automated tests mapped to controls
- Cross-framework mapping across 35+ frameworks (SOC 2, ISO 27001, HIPAA, PCI, GDPR, NIST CSF, DORA)
- Automated evidence collection from connected systems
- Vendor risk management (VRM) with security questionnaire automation
- AI Agent 2.0 that autonomously hunts for evidence, drafts policies, and answers security questionnaires
- Audit management and in-platform collaboration with external auditors
- Employee onboarding compliance (security training, policy acceptance tracking)
- Real-time compliance dashboard and reporting

**Differentiating features**
- "Bring your own control" — AI maps custom controls to Vanta's automated test library
- Largest integration ecosystem in the category (375+)
- Fastest time-to-audit-readiness marketing (2–6 weeks)
- Trust reports (public-facing compliance status pages)

**UX patterns**
- Startup-friendly onboarding with guided setup wizard
- Colour-coded compliance posture dashboards
- Control status timeline for audit evidence windows
- Employee task management interface (training, device enrolment)

**Integration points**
- REST API with OAuth 2.0 client credentials (developer.vanta.com)
- Rate limits: 50 req/min (management), 20 req/min (integration endpoints), 5 req/min (OAuth)
- Supports private integrations for systems not natively covered
- Webhooks for compliance event notifications
- Integrates with Jira, GitHub, Okta, AWS, GCP, Azure, Slack, and 370+ others

**Known gaps**
- Pricing escalates steeply for multi-framework and enterprise tiers
- Limited custom policy language; users must work within Vanta's control library
- Weaker on FedRAMP and CMMC compared to Hyperproof
- Less suited to organisations with heavy on-premises infrastructure
- Evidence review and remediation guidance is less prescriptive than competitors

**Licence / IP notes**
- Proprietary commercial SaaS; no open-source components exposed
- No known patented features in public record

---

### Drata

**Core features**
- Automated evidence collection and control testing across SOC 2, ISO 27001, HIPAA, PCI, GDPR, NIST, and more
- Continuous monitoring with automated test scheduling
- Risk register and risk management workflows
- Access review automation across HR systems and identity providers
- Policy management with version history and acceptance tracking
- Audit management with auditor collaboration portal
- Trust centre (public compliance status pages)
- Open API for programmatic access to evidence and controls
- Workflows engine for automating remediation tasks

**Differentiating features**
- "Trust Management" platform positioning (beyond compliance tooling)
- Highly configurable automation workflows for remediation
- Strong enterprise support with dedicated customer success managers
- Eleven G2 Momentum Leader badges (Winter 2025)

**UX patterns**
- Evidence collection timeline visualisation
- Control health heatmap across frameworks
- Remediation task assignment and tracking
- Guided audit prep checklists

**Integration points**
- REST API at https://public-api.drata.com (Bearer token auth)
- Rate limit: 500 requests/minute per IP
- Auto-generated code samples for multiple languages
- Full audit trail for all API changes
- Developer portal at https://developers.drata.com

**Known gaps**
- API access gated to Advanced plan and above; Foundation tier has no programmatic access
- Initial setup can be complex for enterprise deployments
- Workflow automation requires configuration overhead
- Smaller integration library than Vanta (100+ vs 375+)

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components; no known patents

---

### Secureframe

**Core features**
- Multi-framework compliance across 25+ frameworks (SOC 2, ISO 27001, HIPAA, PCI DSS, GDPR)
- 150+ native integrations with automated evidence collection
- Real-time cloud infrastructure monitoring
- Pre-built policy templates and checklist-driven onboarding
- White-glove onboarding support with compliance experts
- Continuous control monitoring with alert notifications
- Vendor risk management
- Audit management with auditor portal access

**Differentiating features**
- Strong guided onboarding; setup team often manages initial configuration
- Pre-templated, checklist-driven compliance approach reduces time-to-readiness
- Particularly strong for HIPAA and healthcare-adjacent compliance
- SOC 2 + HIPAA combination workflows

**UX patterns**
- Checklist-driven compliance flows
- Step-by-step wizard for each framework
- Control status dashboard with completion percentages
- Auditor collaboration view

**Integration points**
- REST API at https://api.secureframe.com/docs
- Developer portal and custom integration support
- 150+ integrations with cloud providers, SaaS tools, endpoint management

**Known gaps**
- Less automation depth than Vanta or Drata for complex enterprise environments
- Smaller integration library (150+ vs Vanta's 375+)
- Limited AI-native features compared to Vanta's AI Agent 2.0
- Fewer enterprise-grade workflow customisation options

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Sprinto

**Core features**
- Compliance automation for cloud-hosted tech companies
- 200+ native integrations
- Entity-level risk scoring and control health monitoring
- Automated control testing and evidence collection
- Supports GDPR, HIPAA, ISO 27001, SOC 2, PCI DSS, and more
- Real-time monitoring dashboard
- Centralised policy management

**Differentiating features**
- Entity-level risk scoring provides more granular posture visibility
- Strong market position in APAC and emerging markets
- GraphQL-based API (vs REST-only competitors) — more flexible querying
- Competitive pricing for growing companies ($8,000–$15,000/yr range)

**UX patterns**
- Risk-centric dashboard view
- Entity risk timeline
- Compliance calendar for periodic review tasks

**Integration points**
- GraphQL-based developer API at https://developer.sprinto.com
- API documentation at https://docs.sprinto.com
- 200+ integrations covering cloud, identity, HR, and endpoint systems

**Known gaps**
- Smaller brand recognition compared to Vanta and Drata in US market
- Integration depth varies; some connectors are more shallow than competitors
- Less mature AI-native evidence automation

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Hyperproof

**Core features**
- Security compliance management with risk and control organisation
- Automated evidence collection via Hypersyncs (native connectors)
- Cross-framework control mapping to avoid duplicate effort
- User Access Reviews (UAR) across AWS, Azure, Google, Okta
- Custom framework builder for FedRAMP, CMMC, and US government frameworks
- Risk assessment and risk register management
- Reporting for executives and board-level audiences

**Differentiating features**
- Best-in-class for FedRAMP and CMMC compliance
- Custom framework builder supports bespoke internal compliance programmes
- Flexible SDK and REST APIs alongside native Hypersyncs
- Executive-oriented reporting with compliance posture dashboards and risk heatmaps

**UX patterns**
- Risk heatmap visualisation
- Audit readiness score
- Remediation tracking interface
- Board-level reporting views

**Integration points**
- REST API via Hyperproof Developer Portal (https://developer.hyperproof.app)
- Flexible SDK for custom integrations
- Hypersyncs for automated evidence collection
- Integrates with AWS, Azure, GCP, Okta, Jira, ServiceNow, and others

**Known gaps**
- Higher price point ($12,000–$54,000+ per year)
- More manual effort for evidence collection compared to fully automated peers
- Fewer native integrations than Vanta
- Steeper learning curve for non-compliance specialists

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Tugboat Logic (OneTrust)

**Core features**
- Enterprise GRC platform for complex multi-framework compliance
- Comprehensive reporting: compliance posture dashboards, risk heatmaps, remediation tracking, audit readiness scores
- Policy management and risk management workflows
- Multi-framework control mapping
- Part of OneTrust's broader privacy, security, and data governance ecosystem

**Differentiating features**
- Deep integration with OneTrust's privacy and data governance products
- Executive and board-level reporting capabilities
- Well-suited for large enterprises with dedicated GRC teams
- Cross-sells into OneTrust's wider platform (DPIA, consent management, vendor risk)

**UX patterns**
- Dashboard-driven posture reporting
- Risk heatmaps and remediation tracking
- Compliance timeline and audit history views

**Integration points**
- Part of OneTrust platform ecosystem
- Integration with identity, HR, and cloud infrastructure systems
- API access within OneTrust developer ecosystem

**Known gaps**
- Requires more manual evidence collection effort vs Vanta/Drata
- Fewer native automated integrations
- Primarily suited to large enterprise; less appropriate for startups
- OneTrust acquisition has added pricing and contract complexity

**Licence / IP notes**
- Proprietary commercial SaaS (part of OneTrust)
- No open-source components

---

### Wiz

**Core features**
- Agentless CSPM with real-time cloud environment visibility
- 2,300+ cloud misconfiguration rules
- Continuous compliance monitoring across 150+ frameworks
- Infrastructure as Code (IaC) scanning
- Real-time detections and auto-remediation
- Cloud asset inventory and relationship graph (Wiz Security Graph)
- Vulnerability management and container/workload security

**Differentiating features**
- Security Graph — contextual risk assessment linking misconfigurations to exploitability
- Agentless deployment (no agents required in cloud environments)
- Breadth of security capabilities beyond compliance (CSPM, CWPP, CIEM, DSPM)
- 150+ compliance framework benchmarks built-in

**UX patterns**
- Graph-based security risk visualisation
- Risk prioritisation by exploitability context
- Single-pane view across multi-cloud environments

**Integration points**
- REST API and webhook integrations
- Connects to ticketing (Jira, ServiceNow), SIEM, and SOAR platforms
- Multi-cloud: AWS, Azure, GCP, Kubernetes, OCI

**Known gaps**
- Compliance is a secondary capability; limited audit workflow and evidence management
- No built-in GRC features (policy management, employee training, vendor risk)
- Pricing concerns are driving buyer evaluation away (awareness dropped 26.6% to 15.4%)
- Not suited for audit readiness workflows requiring auditor collaboration

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Prisma Cloud (Palo Alto Networks)

**Core features**
- Comprehensive CSPM across multi-cloud environments
- 700+ pre-defined policies from 120+ cloud services
- Continuous compliance posture monitoring and one-click reporting
- Coverage for CIS, GDPR, HIPAA, ISO 27001, NIST 800, PCI DSS, SOC 2
- Infrastructure as Code scanning
- Runtime threat detection and container/workload security (CNAPP)
- Data Security Posture Management (DSPM)

**Differentiating features**
- Broadest CNAPP (Cloud-Native Application Protection Platform) capabilities
- Deep integration with Palo Alto Networks ecosystem (Cortex, firewall, SIEM)
- One-click compliance reporting across frameworks
- Strong for regulated industries (financial services, healthcare, government)

**UX patterns**
- Centralised dashboard for multi-cloud posture
- One-click compliance report generation
- Policy violation drill-down with remediation guidance

**Integration points**
- APIs and integrations with AWS, Azure, GCP, Kubernetes, GitHub, GitLab
- SIEM integrations (Splunk, Sentinel, QRadar)
- Ticketing integrations (Jira, ServiceNow)

**Known gaps**
- Compliance is part of broader CNAPP; limited audit workflow tooling
- Complex licensing and configuration
- Steeper learning curve; requires dedicated security team
- No employee-facing compliance workflows (training, policy acceptance)

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Orca Security

**Core features**
- Agentless CSPM using SideScanning technology
- Full-stack security insights without agent deployment
- Cloud asset discovery and risk visibility
- Compliance monitoring across CIS, SOC 2, PCI, HIPAA, GDPR, and other frameworks
- Vulnerability detection and prioritisation
- Container and workload security

**Differentiating features**
- SideScanning — deep cloud workload scanning without disrupting operations
- Agentless architecture simplifies deployment significantly
- Broad coverage including ephemeral and short-lived cloud resources
- Strong cloud asset context for risk prioritisation

**UX patterns**
- Risk prioritisation by business impact
- Cloud asset inventory and attack surface view
- Compliance posture dashboard

**Integration points**
- Integrates with AWS, Azure, GCP, Kubernetes
- Ticketing and SIEM connectors
- REST API available

**Known gaps**
- Compliance reporting is secondary to security posture management
- Limited GRC and audit workflow capabilities
- No employee compliance features
- Less suited for startups seeking audit readiness

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### Prowler

**Core features**
- Open-source cloud security platform with hundreds of built-in controls
- Compliance checks against CIS, NIST 800, NIST CSF, GDPR, HIPAA, SOC 2, ISO 27001, and others
- Multi-cloud: AWS, Azure, GCP, Kubernetes, GitHub, Microsoft 365
- Export findings in JUnit-XML, JSON, CSV, HTML, and AWS Security Finding formats
- Scan execution in 5–15 minutes per environment
- Automated risk prioritisation by severity, context, and impact
- Integration with CISO Assistant for GRC workflow layer

**Differentiating features**
- Completely free and open-source (Apache 2.0)
- Widest open-source framework coverage (130+ via CISO Assistant integration)
- Active community with 10,000+ GitHub stars
- Extensible: custom checks via Python
- CI/CD pipeline integration for continuous compliance

**UX patterns**
- CLI-first tool; requires engineering effort to operationalise
- HTML/JSON report output
- SaaS offering (Prowler Pro) provides dashboard UI

**Integration points**
- CLI tool with Python SDK
- REST API (Prowler Pro / SaaS)
- Outputs to AWS Security Hub, S3, and third-party SIEMs
- Integrates with CISO Assistant for complete GRC workflows

**Known gaps**
- Requires engineering effort to set up and maintain
- No native audit workflow, policy management, or evidence tracking
- No employee compliance features (training, policy acceptance)
- Evidence is findings-based, not mapped to specific auditor expectations
- SaaS tier (Prowler Pro) adds cost for workflow features

**Licence / IP notes**
- Open-source core: Apache 2.0
- Prowler Pro (SaaS) is commercial

---

### CISO Assistant

**Core features**
- Open-source GRC platform for Risk Management, AppSec, Compliance, Audit, TPRM, Privacy, and Reporting
- 130+ global frameworks with automatic control mapping (ISO 27001, NIST CSF, SOC 2, CIS, PCI DSS, NIS2, DORA, GDPR, HIPAA, CMMC)
- Flexible REST API for automation and data extraction
- CLI toolbox for automation, framework customisation, and control mapping
- Integration with Prowler scanning engine
- Risk register and risk assessment workflows
- Self-hosted or cloud deployment options

**Differentiating features**
- Widest framework coverage of any open-source tool (130+)
- Integrates with Prowler for automated scan-to-compliance workflows
- Self-hosted option for data sovereignty requirements
- LGPL-3.0 licence allows commercial use with modifications

**UX patterns**
- Web-based GRC dashboard
- Framework selection and control mapping wizard
- Risk assessment and treatment plan interface

**Integration points**
- REST API for full automation
- CLI for scripted workflows
- Prowler integration for automated scanning
- JSON/CSV export

**Known gaps**
- Requires self-hosting engineering effort (or SaaS option from intuitem)
- Less polished UI compared to commercial competitors
- Smaller integration library than commercial platforms
- Limited evidence automation (relies on Prowler for scanning)
- Smaller community support compared to Prowler

**Licence / IP notes**
- Open-source: LGPL-3.0
- Commercial SaaS option available from intuitem

---

### AWS Audit Manager

**Core features**
- Native AWS evidence collection and assessment framework
- Pre-built frameworks: CIS, GDPR, HIPAA, PCI DSS, SOC 2, NIST 800-53, ISO 27001
- Automated evidence collection from AWS Config, AWS Security Hub, CloudTrail, and other AWS services
- Assessment reports ready for auditors
- Custom control and framework creation
- Continuous compliance evaluation within AWS

**Differentiating features**
- Seamlessly integrated with AWS ecosystem at no separate licensing cost
- Evidence linked directly to AWS resources for full traceability
- Native integration with AWS Config rules for policy-as-code compliance

**UX patterns**
- AWS Console dashboard
- Assessment progress tracking
- Evidence gallery per control
- Report generation per audit period

**Integration points**
- AWS-native: integrates with Config, Security Hub, CloudTrail, IAM
- Does not natively integrate with non-AWS environments
- API access via AWS SDK (Boto3 for Python, AWS CLI)

**Known gaps**
- AWS-only; no coverage for Azure, GCP, or SaaS applications
- No employee compliance workflows (policy acceptance, training)
- Limited cross-framework deduplication
- Requires AWS expertise to configure and maintain
- No vendor risk management

**Licence / IP notes**
- Commercial (AWS service); per-assessment pricing
- No open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Continuous automated evidence collection from cloud and SaaS systems
- Pre-built control mappings for SOC 2, ISO 27001, HIPAA, PCI DSS
- Compliance posture dashboard with real-time status
- Auditor collaboration portal or report export
- Policy management (creation, versioning, employee acceptance tracking)
- Employee security training completion tracking
- Integration with major cloud providers (AWS, Azure, GCP)
- Integration with identity providers (Okta, Azure AD, Google Workspace)
- Vulnerability and risk management workflows
- Vendor/third-party risk management

### Differentiating Features
- Cross-framework control deduplication (evidence collected once, applied to multiple frameworks)
- AI-powered evidence hunting and policy drafting
- Security Graph / contextual risk visualisation (Wiz)
- Agentless cloud scanning (Wiz, Orca, Prisma Cloud)
- Custom framework builder (Hyperproof)
- GraphQL API for flexible data access (Sprinto)
- Self-hosted deployment option (CISO Assistant)
- Open-source with extensible custom checks (Prowler)
- Trust centre / public compliance status pages (Vanta, Drata)

### Underserved Areas / Opportunities
- Natural-language compliance gap analysis (describe a feature change; system identifies affected controls)
- AI-generated remediation playbooks with step-by-step infrastructure guidance
- Predictive audit risk scoring based on historical evidence patterns
- True multi-cloud continuous compliance without agent deployment and at low cost
- Open-source GRC with enterprise-grade audit workflow tooling
- Unified compliance coverage spanning cloud infrastructure, SaaS, and on-premises systems
- EU AI Act and emerging framework support (DORA, NIS2) in open-source tools
- CI/CD-integrated compliance checks with developer-friendly output
- Affordable entry-level option for early-stage startups (current market floor ~$7,500/yr)
- Automated security questionnaire response using AI (still largely manual)

### AI-Augmentation Candidates
- Evidence collection: AI agents autonomously gather and classify evidence from connected systems
- Control testing: AI evaluates evidence quality and flags controls likely to fail
- Policy generation: AI drafts policies from templates and customises to organisational context
- Security questionnaire automation: AI reads incoming vendor questionnaires and drafts responses
- Cross-framework mapping: AI identifies which evidence satisfies overlapping controls across frameworks
- Gap analysis: AI explains what is missing and what actions resolve each gap
- Audit communication: AI summarises compliance status for non-technical stakeholders
- Remediation guidance: AI generates infrastructure-specific fix instructions for control failures

---

## Legal & IP Summary

All major commercial platforms (Vanta, Drata, Secureframe, Sprinto, Hyperproof, Wiz, Prisma Cloud, Orca Security) are proprietary SaaS products with no open-source components exposed. No patents on specific compliance automation techniques were found in public records, though core automation workflows may be protected by trade secrets. Open-source tools (Prowler, CISO Assistant) are permissively licensed (Apache 2.0 and LGPL-3.0 respectively), allowing commercial use and modification. Any AI-native open-source tool built in this category can freely reference these open-source tools and their approaches. Care should be taken not to reproduce proprietary control mapping content verbatim — instead, control mappings should be derived from the official standards bodies (AICPA, NIST, PCI SSC, HHS) whose documentation is publicly available.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Automated cloud resource discovery and compliance check execution (AWS, Azure, GCP minimum)
- Pre-built control mappings for SOC 2, ISO 27001, HIPAA, and PCI DSS
- Cross-framework deduplication — evidence collected once applies to all mapped frameworks
- AI-powered natural-language gap analysis (describe a system change; identify affected controls)
- Compliance posture dashboard with control status, evidence status, and audit readiness score
- Evidence export and auditor report generation

**Should-have (v1.1)**
- Employee compliance workflows (policy acceptance tracking, training completion)
- Vendor risk management and security questionnaire automation
- CI/CD integration for policy-as-code compliance checks at build time
- Remediation playbook generation with infrastructure-specific fix guidance
- Integration with Jira, GitHub, Slack, and Okta for workflow automation
- Trust centre / public compliance status page

**Nice-to-have (backlog)**
- Predictive audit risk scoring based on historical evidence quality
- Custom framework builder for internal policies or emerging standards (EU AI Act, DORA)
- Self-hosted deployment option for data sovereignty requirements
- EU regulatory framework support (NIS2, DORA, GDPR Article 30 records)
- FedRAMP and CMMC support for US government contractors
- Mobile device management (MDM) evidence collection
