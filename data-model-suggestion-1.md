# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Cloud Compliance Automation · Created: 2026-05-20

## Philosophy

This model follows a fully normalized relational design where every distinct compliance concept — framework, control, evidence, finding, assessment, cloud resource — has its own table with strict foreign key relationships. Junction tables handle the many-to-many relationships that are fundamental to GRC (a single piece of evidence can satisfy controls across multiple frameworks; a single cloud resource can be evaluated against dozens of controls).

The design is heavily influenced by NIST's OSCAL data model, which defines separate layers for control catalogs, profiles (baselines), system security plans, assessment plans, and assessment results. By mirroring OSCAL's conceptual structure in relational form, this model ensures that data can be imported from and exported to OSCAL-format documents without lossy transformation.

This approach prioritizes data integrity and query flexibility. Every relationship is explicit and enforced at the database level. Cross-framework deduplication — the platform's key differentiator — is handled through junction tables that link evidence artifacts to controls across multiple frameworks simultaneously.

**Best for:** Teams that value referential integrity, need complex cross-entity reporting, and operate in heavily regulated environments where every relationship must be auditable.

**Trade-offs:**
- (+) Maximum referential integrity — broken relationships are impossible
- (+) Straightforward SQL queries for compliance reporting
- (+) Natural fit for OSCAL import/export
- (+) Well-understood by most developers and ORMs
- (-) High table count (~45-55 tables) increases schema complexity
- (-) Schema migrations required when adding new entity types or relationships
- (-) Many-to-many junction tables can make queries verbose
- (-) Adding jurisdiction-specific fields requires schema changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | Model structure mirrors OSCAL layers: catalog, profile, SSP, assessment plan, assessment results |
| ISO/IEC 27001:2022 | Control categories align with Annex A clause structure |
| SOC 2 (AICPA TSC) | Trust Service Categories modeled as framework categories |
| NIST CSF 2.0 | Six functions (Identify, Protect, Detect, Respond, Recover, Govern) as category taxonomy |
| ISO 3166-1/2 | Jurisdiction codes for multi-region compliance |
| RFC 6749 (OAuth 2.0) | API authentication model for integrations |
| CloudEvents v1.0 | Event notification schema for webhook payloads |
| CIS Controls v8 | Benchmark control identifiers as external references |

---

## Core Identity & Multi-Tenancy

```sql
-- Organizations (tenants)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    industry TEXT,                          -- e.g. 'healthcare', 'fintech', 'saas'
    jurisdiction_code TEXT,                 -- ISO 3166-1 alpha-2 primary jurisdiction
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_slug ON organizations (slug);

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email TEXT NOT NULL,
    display_name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'viewer',    -- 'owner', 'admin', 'compliance_manager', 'auditor', 'viewer'
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_email ON users (email);

-- API Keys for integrations
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL,                 -- bcrypt hash of the key
    scopes TEXT[] NOT NULL DEFAULT '{}',    -- e.g. '{read:controls, write:evidence}'
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org ON api_keys (organization_id);
```

## Framework & Control Library

```sql
-- Compliance frameworks (SOC 2, ISO 27001, HIPAA, PCI DSS, etc.)
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code TEXT NOT NULL UNIQUE,              -- e.g. 'soc2', 'iso27001', 'hipaa', 'pci_dss_4'
    name TEXT NOT NULL,                     -- e.g. 'SOC 2 Type II'
    version TEXT NOT NULL,                  -- e.g. '2022', '4.0'
    issuing_body TEXT NOT NULL,             -- e.g. 'AICPA', 'ISO/IEC', 'NIST'
    description TEXT,
    oscal_catalog_uri TEXT,                 -- URI to OSCAL catalog document if available
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Framework categories / sections
CREATE TABLE framework_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    parent_id UUID REFERENCES framework_categories(id),  -- hierarchical categories
    code TEXT NOT NULL,                     -- e.g. 'CC1' (SOC 2), 'A.5' (ISO 27001)
    name TEXT NOT NULL,                     -- e.g. 'Control Environment', 'Information Security Policies'
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE INDEX idx_fw_categories_framework ON framework_categories (framework_id);
CREATE INDEX idx_fw_categories_parent ON framework_categories (parent_id);

-- Controls (the atomic compliance requirements)
CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    category_id UUID REFERENCES framework_categories(id),
    code TEXT NOT NULL,                     -- e.g. 'CC6.1', 'A.8.2', 'R1.1'
    title TEXT NOT NULL,
    description TEXT,
    guidance TEXT,                          -- implementation guidance
    control_type TEXT NOT NULL DEFAULT 'preventive',  -- 'preventive', 'detective', 'corrective'
    frequency TEXT,                         -- 'continuous', 'daily', 'weekly', 'monthly', 'annual'
    is_automatable BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE INDEX idx_controls_framework ON controls (framework_id);
CREATE INDEX idx_controls_category ON controls (category_id);
CREATE INDEX idx_controls_automatable ON controls (is_automatable);

-- Cross-framework control mappings (the deduplication engine)
CREATE TABLE control_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_control_id UUID NOT NULL REFERENCES controls(id),
    target_control_id UUID NOT NULL REFERENCES controls(id),
    mapping_strength TEXT NOT NULL DEFAULT 'partial',  -- 'exact', 'partial', 'related'
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_control_id, target_control_id),
    CHECK (source_control_id != target_control_id)
);

CREATE INDEX idx_control_mappings_source ON control_mappings (source_control_id);
CREATE INDEX idx_control_mappings_target ON control_mappings (target_control_id);
```

## Organization-Specific Compliance State

```sql
-- Frameworks activated for an organization
CREATE TABLE organization_frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    status TEXT NOT NULL DEFAULT 'setup',   -- 'setup', 'active', 'audit_prep', 'certified', 'expired'
    certification_date DATE,
    expiry_date DATE,
    audit_window_start DATE,
    audit_window_end DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, framework_id)
);

CREATE INDEX idx_org_frameworks_org ON organization_frameworks (organization_id);

-- Organization's control implementations (OSCAL SSP equivalent)
CREATE TABLE control_implementations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    status TEXT NOT NULL DEFAULT 'not_started',  -- 'not_started', 'in_progress', 'implemented', 'not_applicable'
    implementation_description TEXT,
    responsible_user_id UUID REFERENCES users(id),
    review_frequency TEXT,                  -- 'continuous', 'monthly', 'quarterly', 'annual'
    last_reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, control_id)
);

CREATE INDEX idx_ctrl_impl_org ON control_implementations (organization_id);
CREATE INDEX idx_ctrl_impl_control ON control_implementations (control_id);
CREATE INDEX idx_ctrl_impl_status ON control_implementations (status);
```

## Cloud Resources & Integrations

```sql
-- Integration connections (AWS, Azure, GCP, Okta, GitHub, etc.)
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider TEXT NOT NULL,                 -- 'aws', 'azure', 'gcp', 'okta', 'github', 'jira'
    display_name TEXT NOT NULL,
    auth_method TEXT NOT NULL,              -- 'iam_role', 'service_principal', 'oauth', 'api_key'
    credentials_vault_ref TEXT,             -- reference to secrets manager entry
    status TEXT NOT NULL DEFAULT 'pending', -- 'pending', 'connected', 'error', 'disabled'
    last_sync_at TIMESTAMPTZ,
    sync_interval_minutes INTEGER NOT NULL DEFAULT 60,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_org ON integrations (organization_id);
CREATE INDEX idx_integrations_provider ON integrations (provider);

-- Discovered cloud resources
CREATE TABLE cloud_resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    integration_id UUID NOT NULL REFERENCES integrations(id),
    provider TEXT NOT NULL,                 -- 'aws', 'azure', 'gcp'
    resource_type TEXT NOT NULL,            -- 'ec2_instance', 'rds_instance', 's3_bucket', 'iam_role'
    resource_id TEXT NOT NULL,              -- provider-specific ARN/ID
    resource_name TEXT,
    region TEXT,                            -- e.g. 'us-east-1', 'westeurope'
    tags TEXT[],                            -- resource tags as key=value pairs
    is_active BOOLEAN NOT NULL DEFAULT true,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider, resource_id)
);

CREATE INDEX idx_cloud_resources_org ON cloud_resources (organization_id);
CREATE INDEX idx_cloud_resources_type ON cloud_resources (resource_type);
CREATE INDEX idx_cloud_resources_provider ON cloud_resources (provider);
```

## Evidence & Findings

```sql
-- Evidence artifacts (screenshots, API responses, config snapshots)
CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    description TEXT,
    evidence_type TEXT NOT NULL,            -- 'automated_scan', 'manual_upload', 'api_snapshot', 'screenshot', 'policy_doc'
    source TEXT,                            -- 'prowler', 'aws_config', 'manual', 'opa'
    storage_uri TEXT,                       -- S3/blob URI to the evidence file
    content_hash TEXT,                      -- SHA-256 hash for integrity verification
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until TIMESTAMPTZ,               -- evidence expiry for periodic controls
    collected_by UUID REFERENCES users(id),
    integration_id UUID REFERENCES integrations(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_org ON evidence (organization_id);
CREATE INDEX idx_evidence_type ON evidence (evidence_type);
CREATE INDEX idx_evidence_collected ON evidence (collected_at);

-- Junction: evidence linked to controls (cross-framework dedup happens here)
CREATE TABLE evidence_controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evidence_id UUID NOT NULL REFERENCES evidence(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    relevance TEXT NOT NULL DEFAULT 'primary',  -- 'primary', 'supporting'
    ai_confidence_score NUMERIC(3,2),       -- 0.00-1.00, AI's assessment of evidence relevance
    linked_by UUID REFERENCES users(id),    -- null if AI-linked
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (evidence_id, control_id, organization_id)
);

CREATE INDEX idx_evidence_controls_evidence ON evidence_controls (evidence_id);
CREATE INDEX idx_evidence_controls_control ON evidence_controls (control_id);
CREATE INDEX idx_evidence_controls_org ON evidence_controls (organization_id);

-- Compliance findings (results of control tests)
CREATE TABLE findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    cloud_resource_id UUID REFERENCES cloud_resources(id),
    integration_id UUID REFERENCES integrations(id),
    status TEXT NOT NULL,                   -- 'pass', 'fail', 'warning', 'error', 'not_applicable'
    severity TEXT NOT NULL DEFAULT 'medium', -- 'critical', 'high', 'medium', 'low', 'informational'
    title TEXT NOT NULL,
    description TEXT,
    remediation_guidance TEXT,
    scanner_source TEXT,                    -- 'prowler', 'opa', 'custom', 'manual'
    scanner_check_id TEXT,                  -- e.g. 'prowler:aws_iam_1'
    evaluated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_org ON findings (organization_id);
CREATE INDEX idx_findings_control ON findings (control_id);
CREATE INDEX idx_findings_status ON findings (status);
CREATE INDEX idx_findings_severity ON findings (severity);
CREATE INDEX idx_findings_resource ON findings (cloud_resource_id);
CREATE INDEX idx_findings_evaluated ON findings (evaluated_at);
```

## Assessments & Audits

```sql
-- Assessment runs (OSCAL assessment plan equivalent)
CREATE TABLE assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    name TEXT NOT NULL,
    assessment_type TEXT NOT NULL,          -- 'automated_scan', 'manual_review', 'audit_prep', 'formal_audit'
    status TEXT NOT NULL DEFAULT 'planned', -- 'planned', 'in_progress', 'completed', 'cancelled'
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    lead_assessor_id UUID REFERENCES users(id),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessments_org ON assessments (organization_id);
CREATE INDEX idx_assessments_framework ON assessments (framework_id);

-- Assessment results (OSCAL assessment results equivalent)
CREATE TABLE assessment_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id UUID NOT NULL REFERENCES assessments(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    result TEXT NOT NULL,                   -- 'satisfied', 'not_satisfied', 'other'
    finding_ids UUID[],                     -- references to supporting findings
    evidence_ids UUID[],                    -- references to supporting evidence
    assessor_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessment_results_assessment ON assessment_results (assessment_id);
CREATE INDEX idx_assessment_results_control ON assessment_results (control_id);
```

## Policy & Employee Compliance

```sql
-- Policies
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'draft',   -- 'draft', 'published', 'archived'
    content_uri TEXT,                       -- storage URI for the policy document
    content_hash TEXT,
    approved_by UUID REFERENCES users(id),
    published_at TIMESTAMPTZ,
    review_due_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policies_org ON policies (organization_id);

-- Junction: policies linked to controls
CREATE TABLE policy_controls (
    policy_id UUID NOT NULL REFERENCES policies(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    PRIMARY KEY (policy_id, control_id)
);

-- Employee policy acceptance tracking
CREATE TABLE policy_acceptances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id),
    user_id UUID NOT NULL REFERENCES users(id),
    accepted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address INET,
    UNIQUE (policy_id, user_id)
);

CREATE INDEX idx_policy_acceptances_user ON policy_acceptances (user_id);
```

## Risk & Remediation

```sql
-- Risk register
CREATE TABLE risks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    description TEXT,
    likelihood TEXT NOT NULL,              -- 'rare', 'unlikely', 'possible', 'likely', 'almost_certain'
    impact TEXT NOT NULL,                  -- 'insignificant', 'minor', 'moderate', 'major', 'severe'
    risk_score INTEGER,                    -- computed: likelihood x impact
    treatment TEXT NOT NULL DEFAULT 'mitigate',  -- 'accept', 'mitigate', 'transfer', 'avoid'
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'open',   -- 'open', 'in_treatment', 'accepted', 'closed'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risks_org ON risks (organization_id);
CREATE INDEX idx_risks_status ON risks (status);

-- Junction: risks to controls
CREATE TABLE risk_controls (
    risk_id UUID NOT NULL REFERENCES risks(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    PRIMARY KEY (risk_id, control_id)
);

-- Remediation tasks
CREATE TABLE remediation_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    finding_id UUID REFERENCES findings(id),
    risk_id UUID REFERENCES risks(id),
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT NOT NULL DEFAULT 'medium',  -- 'critical', 'high', 'medium', 'low'
    status TEXT NOT NULL DEFAULT 'open',       -- 'open', 'in_progress', 'completed', 'wont_fix'
    assigned_to UUID REFERENCES users(id),
    due_date DATE,
    external_ticket_url TEXT,                  -- Jira/GitHub issue link
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remediation_org ON remediation_tasks (organization_id);
CREATE INDEX idx_remediation_finding ON remediation_tasks (finding_id);
CREATE INDEX idx_remediation_status ON remediation_tasks (status);
CREATE INDEX idx_remediation_assigned ON remediation_tasks (assigned_to);
```

## Vendor Risk Management

```sql
-- Third-party vendors
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    category TEXT,                          -- 'cloud_provider', 'saas', 'contractor', 'data_processor'
    risk_tier TEXT NOT NULL DEFAULT 'medium', -- 'critical', 'high', 'medium', 'low'
    status TEXT NOT NULL DEFAULT 'active',
    review_due_at DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendors_org ON vendors (organization_id);

-- Vendor assessments / questionnaires
CREATE TABLE vendor_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id UUID NOT NULL REFERENCES vendors(id),
    assessment_type TEXT NOT NULL,          -- 'security_questionnaire', 'soc2_review', 'pen_test_review'
    status TEXT NOT NULL DEFAULT 'pending', -- 'pending', 'in_progress', 'completed', 'expired'
    score NUMERIC(5,2),
    completed_at TIMESTAMPTZ,
    next_review_at DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_assessments_vendor ON vendor_assessments (vendor_id);
```

## Audit Trail

```sql
-- Immutable audit log for all compliance-relevant actions
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    actor_id UUID REFERENCES users(id),
    actor_type TEXT NOT NULL DEFAULT 'user',  -- 'user', 'system', 'integration', 'ai_agent'
    action TEXT NOT NULL,                     -- 'evidence.created', 'finding.resolved', 'control.status_changed'
    entity_type TEXT NOT NULL,                -- 'evidence', 'finding', 'control_implementation', etc.
    entity_id UUID NOT NULL,
    changes_before TEXT,                      -- JSON snapshot of changed fields (before)
    changes_after TEXT,                       -- JSON snapshot of changed fields (after)
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log (organization_id);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_action ON audit_log (action);
CREATE INDEX idx_audit_log_created ON audit_log (created_at);

-- Partition audit_log by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, api_keys |
| Framework & Controls | 4 | frameworks, framework_categories, controls, control_mappings |
| Organization Compliance | 2 | organization_frameworks, control_implementations |
| Cloud Resources | 2 | integrations, cloud_resources |
| Evidence & Findings | 3 | evidence, evidence_controls, findings |
| Assessments | 2 | assessments, assessment_results |
| Policy & Employee | 3 | policies, policy_controls, policy_acceptances |
| Risk & Remediation | 3 | risks, risk_controls, remediation_tasks |
| Vendor Risk | 2 | vendors, vendor_assessments |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **25** | Core schema; additional tables for notifications, AI jobs, etc. |

---

## Key Design Decisions

1. **Cross-framework deduplication via `evidence_controls` junction table.** A single evidence artifact can be linked to controls from SOC 2, ISO 27001, HIPAA, and PCI DSS simultaneously, eliminating redundant evidence collection.

2. **OSCAL alignment at the structural level.** The frameworks/controls/assessments/assessment_results hierarchy mirrors OSCAL's catalog-profile-SSP-assessment layers, ensuring lossless OSCAL import/export.

3. **Control mappings as a first-class entity.** The `control_mappings` table enables the platform to know that SOC 2 CC6.1 maps to ISO 27001 A.8.1, powering the cross-framework view.

4. **Hierarchical framework categories.** The self-referencing `parent_id` on `framework_categories` supports arbitrarily deep nesting (NIST 800-53 has 3+ levels of nesting).

5. **Temporal evidence validity.** The `valid_from` / `valid_until` fields on evidence support audit windows — evidence must be collected during the observation period to be valid for a Type II audit.

6. **Separation of findings from evidence.** Findings are the result of evaluating a control (pass/fail); evidence is the artifact that supports the finding. This matches how auditors think.

7. **AI confidence scoring on evidence linkage.** The `ai_confidence_score` on `evidence_controls` supports the AI-powered evidence hunting feature, allowing human reviewers to prioritize low-confidence links.

8. **Partitioned audit log.** The audit_log table is designed for monthly partitioning to handle the high write volume of continuous compliance monitoring without degrading query performance.

9. **Cloud resource tracking with `first_seen_at` / `last_seen_at`.** Supports detection of ephemeral resources that appear and disappear between scans, which is critical for container and serverless environments.

10. **Vendor risk as a separate domain.** Vendor assessments are modeled independently from internal compliance, matching how GRC teams manage third-party risk as a distinct workflow.
