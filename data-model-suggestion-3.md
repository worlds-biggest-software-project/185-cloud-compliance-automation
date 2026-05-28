# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Cloud Compliance Automation · Created: 2026-05-20

## Philosophy

This model uses a relational backbone for core entities (organizations, frameworks, controls, evidence, findings) but stores variable and domain-specific data in JSONB columns. The key insight is that compliance frameworks vary enormously in structure — SOC 2 has 5 Trust Service Categories with ~60 controls, while NIST 800-53 has 20 control families with 1,000+ controls. Similarly, cloud resources from AWS, Azure, and GCP have completely different attribute schemas, and evidence artifacts vary from API JSON snapshots to PDF documents to screenshots.

Rather than modeling every possible variation as a relational column (leading to sparse tables and constant migrations) or going fully document-oriented (losing relational query power), this hybrid approach puts stable, frequently-queried fields in typed columns and puts variable, framework-specific, or provider-specific data in JSONB. PostgreSQL's JSONB operators and GIN indexes make the variable data queryable at near-relational speed.

This pattern is widely used by modern SaaS platforms (Stripe, Shopify, GitHub) that need to support diverse data shapes within a shared schema. For a compliance platform that must handle 30+ frameworks across 3+ cloud providers with jurisdiction-specific requirements, JSONB flexibility prevents schema explosion while keeping core relationships relational.

**Best for:** Rapid MVP development, multi-framework platforms with highly variable control structures, multi-cloud platforms where resource attributes differ by provider, and teams that need extensibility without schema migrations.

**Trade-offs:**
- (+) Fewer tables — variable data lives in JSONB rather than separate tables
- (+) Adding a new framework or cloud provider requires no schema migration
- (+) JSONB containment queries are fast with GIN indexes
- (+) Extensible per-tenant without affecting shared schema
- (+) Natural fit for storing raw API responses and scanner outputs
- (-) JSONB fields lack column-level constraints (validation must be application-level)
- (-) JSONB query syntax is less intuitive than plain SQL for some developers
- (-) No foreign key enforcement within JSONB data
- (-) JSONB storage is larger than equivalent typed columns
- (-) Harder to generate documentation of the complete "schema" since JSONB fields are implicit

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | OSCAL documents stored directly as JSONB for lossless round-tripping |
| ISO/IEC 27001:2022 | Annex A control attributes stored as JSONB with framework-specific fields |
| SOC 2 (AICPA TSC) | Trust Service Category metadata in framework `properties` JSONB |
| PCI DSS 4.0 | Requirement hierarchy and SAQ types in JSONB properties |
| ISO 3166-1/2 | Jurisdiction codes as typed columns; jurisdiction-specific rules in JSONB |
| CIS Controls v8 | Implementation group levels (IG1/IG2/IG3) stored in control `properties` |
| CloudEvents v1.0 | Webhook payloads stored as JSONB events |
| JSON Schema | Used to validate JSONB field structures at the application layer |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    -- JSONB for variable org-level settings
    settings JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "industry": "healthcare",
    --   "primary_jurisdiction": "US",
    --   "jurisdictions": ["US", "EU", "GB"],
    --   "compliance_contact_email": "ciso@example.com",
    --   "scan_schedule_cron": "0 */6 * * *",
    --   "notification_preferences": {
    --     "slack_webhook": "https://hooks.slack.com/...",
    --     "email_alerts": true,
    --     "alert_severities": ["critical", "high"]
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email TEXT NOT NULL,
    display_name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'viewer',
    is_active BOOLEAN NOT NULL DEFAULT true,
    profile JSONB NOT NULL DEFAULT '{}',
    -- Example profile: {
    --   "timezone": "America/New_York",
    --   "notification_channels": ["email", "slack"],
    --   "certifications": ["CISSP", "CISA"],
    --   "departments": ["engineering", "security"]
    -- }
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
```

## Frameworks & Controls

```sql
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    issuing_body TEXT NOT NULL,
    -- JSONB for framework-specific metadata
    properties JSONB NOT NULL DEFAULT '{}',
    -- Example for SOC 2: {
    --   "trust_service_categories": ["security", "availability", "processing_integrity", "confidentiality", "privacy"],
    --   "report_types": ["type_i", "type_ii"],
    --   "observation_period_months": [3, 6, 12],
    --   "oscal_catalog_uri": "https://..."
    -- }
    -- Example for PCI DSS 4.0: {
    --   "saq_types": ["SAQ-A", "SAQ-A-EP", "SAQ-B", "SAQ-C", "SAQ-D"],
    --   "requirement_count": 64,
    --   "sub_requirement_count": 320,
    --   "applies_to": "entities storing/processing/transmitting cardholder data"
    -- }
    -- Example for NIST 800-53: {
    --   "control_families": 20,
    --   "total_controls": 1189,
    --   "baselines": ["low", "moderate", "high"],
    --   "privacy_overlay": true
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    code TEXT NOT NULL,                     -- 'CC6.1', 'A.8.2', '1.1.1'
    title TEXT NOT NULL,
    description TEXT,
    -- Parent control for hierarchical frameworks (NIST 800-53 has 3+ levels)
    parent_control_id UUID REFERENCES controls(id),
    -- Stable, queryable fields
    control_type TEXT NOT NULL DEFAULT 'preventive',
    is_automatable BOOLEAN NOT NULL DEFAULT false,
    -- JSONB for framework-specific control attributes
    properties JSONB NOT NULL DEFAULT '{}',
    -- Example for SOC 2 control: {
    --   "trust_service_category": "security",
    --   "criteria_type": "common",
    --   "points_of_focus": ["Restricts physical access", "Uses encryption"],
    --   "related_coso_principle": "CC6"
    -- }
    -- Example for CIS control: {
    --   "implementation_group": "IG1",
    --   "asset_type": "devices",
    --   "security_function": "protect",
    --   "safeguard_number": "4.1"
    -- }
    -- Example for NIST 800-53 control: {
    --   "family": "AC",
    --   "family_name": "Access Control",
    --   "baseline_low": true,
    --   "baseline_moderate": true,
    --   "baseline_high": true,
    --   "enhancements": ["AC-2(1)", "AC-2(2)", "AC-2(3)"],
    --   "related_controls": ["AC-3", "AC-6", "IA-2"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE INDEX idx_controls_framework ON controls (framework_id);
CREATE INDEX idx_controls_parent ON controls (parent_control_id);
CREATE INDEX idx_controls_type ON controls (control_type);
CREATE INDEX idx_controls_automatable ON controls (is_automatable);
-- GIN index on JSONB for property queries
CREATE INDEX idx_controls_properties ON controls USING gin (properties);

-- Cross-framework mappings
CREATE TABLE control_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_control_id UUID NOT NULL REFERENCES controls(id),
    target_control_id UUID NOT NULL REFERENCES controls(id),
    mapping_strength TEXT NOT NULL DEFAULT 'partial',
    mapping_details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "overlap_percentage": 85,
    --   "shared_evidence_types": ["access_review", "encryption_config"],
    --   "notes": "SOC 2 CC6.1 maps to ISO 27001 A.8 with minor scope differences"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_control_id, target_control_id)
);
```

## Integrations & Cloud Resources

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    -- JSONB for provider-specific configuration
    config JSONB NOT NULL DEFAULT '{}',
    -- Example for AWS: {
    --   "auth_method": "iam_role",
    --   "role_arn": "arn:aws:iam::123456789012:role/ComplianceReader",
    --   "external_id": "abc123",
    --   "regions": ["us-east-1", "us-west-2", "eu-west-1"],
    --   "services_to_scan": ["s3", "ec2", "rds", "iam", "lambda"]
    -- }
    -- Example for Azure: {
    --   "auth_method": "service_principal",
    --   "tenant_id": "...",
    --   "subscription_ids": ["sub-1", "sub-2"],
    --   "resource_groups": ["prod", "staging"]
    -- }
    -- Example for Okta: {
    --   "auth_method": "api_key",
    --   "domain": "company.okta.com",
    --   "scopes": ["okta.users.read", "okta.groups.read"]
    -- }
    credentials_vault_ref TEXT,
    last_sync_at TIMESTAMPTZ,
    sync_interval_minutes INTEGER NOT NULL DEFAULT 60,
    last_sync_result JSONB,
    -- Example: {
    --   "status": "success",
    --   "resources_discovered": 342,
    --   "resources_updated": 15,
    --   "duration_seconds": 45,
    --   "errors": []
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_org ON integrations (organization_id);

CREATE TABLE cloud_resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    integration_id UUID NOT NULL REFERENCES integrations(id),
    provider TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id TEXT NOT NULL,              -- ARN, Azure resource ID, GCP self_link
    resource_name TEXT,
    region TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    -- JSONB for provider-specific resource attributes
    attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for S3 bucket: {
    --   "arn": "arn:aws:s3:::my-bucket",
    --   "versioning": "enabled",
    --   "encryption": "AES256",
    --   "public_access_block": {"block_public_acls": true, "block_public_policy": true},
    --   "lifecycle_rules": [{"id": "archive", "transition_days": 90}],
    --   "tags": {"environment": "production", "team": "backend"}
    -- }
    -- Example for Azure VM: {
    --   "vm_size": "Standard_D2s_v3",
    --   "os": "Ubuntu 22.04",
    --   "disk_encryption": "platform_managed",
    --   "network_security_group": "nsg-prod",
    --   "public_ip": null,
    --   "tags": {"env": "prod"}
    -- }
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider, resource_id)
);

CREATE INDEX idx_resources_org ON cloud_resources (organization_id);
CREATE INDEX idx_resources_type ON cloud_resources (resource_type);
CREATE INDEX idx_resources_provider ON cloud_resources (provider);
-- GIN index for querying resource attributes
CREATE INDEX idx_resources_attributes ON cloud_resources USING gin (attributes);
```

### Example JSONB Query: Find Unencrypted S3 Buckets

```sql
SELECT resource_name, attributes->>'encryption' AS encryption
FROM cloud_resources
WHERE organization_id = 'org-uuid'
  AND provider = 'aws'
  AND resource_type = 's3_bucket'
  AND (
    attributes->>'encryption' IS NULL
    OR attributes->>'encryption' = 'none'
  );
```

### Example JSONB Query: Find Resources with Public Access

```sql
SELECT resource_name, resource_type, provider
FROM cloud_resources
WHERE organization_id = 'org-uuid'
  AND attributes @> '{"public_access_block": {"block_public_acls": false}}'
  AND is_active = true;
```

## Evidence & Findings

```sql
CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    evidence_type TEXT NOT NULL,
    source TEXT,
    storage_uri TEXT,
    content_hash TEXT,
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until TIMESTAMPTZ,
    collected_by UUID REFERENCES users(id),
    integration_id UUID REFERENCES integrations(id),
    -- JSONB for evidence-type-specific metadata
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example for automated scan evidence: {
    --   "scanner": "prowler",
    --   "check_id": "s3_bucket_public_access",
    --   "scan_run_id": "scan-uuid",
    --   "resource_arn": "arn:aws:s3:::my-bucket",
    --   "raw_output": {...}
    -- }
    -- Example for manual upload: {
    --   "file_name": "soc2-audit-report.pdf",
    --   "file_size_bytes": 2048576,
    --   "mime_type": "application/pdf",
    --   "uploaded_by_name": "Jane Smith",
    --   "review_status": "approved"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_org ON evidence (organization_id);
CREATE INDEX idx_evidence_type ON evidence (evidence_type);
CREATE INDEX idx_evidence_collected ON evidence (collected_at);
CREATE INDEX idx_evidence_metadata ON evidence USING gin (metadata);

-- Evidence-to-control links (cross-framework dedup)
CREATE TABLE evidence_controls (
    evidence_id UUID NOT NULL REFERENCES evidence(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    relevance TEXT NOT NULL DEFAULT 'primary',
    ai_confidence NUMERIC(3,2),
    linked_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (evidence_id, control_id)
);

CREATE INDEX idx_ec_control ON evidence_controls (control_id);

-- Findings from control tests
CREATE TABLE findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    cloud_resource_id UUID REFERENCES cloud_resources(id),
    status TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'medium',
    title TEXT NOT NULL,
    description TEXT,
    -- JSONB for scanner-specific finding details
    details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "scanner": "prowler",
    --   "check_id": "iam_password_policy_minimum_length_14",
    --   "check_title": "Ensure IAM password policy requires minimum length of 14",
    --   "resource_arn": "arn:aws:iam::123456789012:root",
    --   "actual_value": 8,
    --   "expected_value": 14,
    --   "remediation_url": "https://docs.aws.amazon.com/...",
    --   "cis_benchmark": "1.8",
    --   "prowler_severity": "medium"
    -- }
    remediation_guidance TEXT,
    ai_remediation JSONB,
    -- Example: {
    --   "summary": "Increase minimum password length to 14 characters",
    --   "steps": [
    --     "Open AWS IAM Console > Account Settings",
    --     "Click 'Change password policy'",
    --     "Set minimum length to 14",
    --     "Enable all complexity requirements"
    --   ],
    --   "terraform_fix": "resource \"aws_iam_account_password_policy\" ...",
    --   "estimated_effort": "5 minutes",
    --   "risk_of_change": "low"
    -- }
    evaluated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_org ON findings (organization_id);
CREATE INDEX idx_findings_control ON findings (control_id);
CREATE INDEX idx_findings_status ON findings (status);
CREATE INDEX idx_findings_severity ON findings (severity);
CREATE INDEX idx_findings_details ON findings USING gin (details);
```

## Organization Compliance State

```sql
CREATE TABLE organization_frameworks (
    organization_id UUID NOT NULL REFERENCES organizations(id),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    status TEXT NOT NULL DEFAULT 'setup',
    certification_date DATE,
    expiry_date DATE,
    config JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "audit_window": {"start": "2026-01-01", "end": "2026-06-30"},
    --   "auditor_firm": "Deloitte",
    --   "auditor_contact": "john@deloitte.com",
    --   "excluded_controls": ["CC8.1"],
    --   "soc2_categories": ["security", "availability"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, framework_id)
);

CREATE TABLE control_implementations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    status TEXT NOT NULL DEFAULT 'not_started',
    implementation_description TEXT,
    responsible_user_id UUID REFERENCES users(id),
    implementation_details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "implementation_type": "automated",
    --   "tools_used": ["aws_config", "opa"],
    --   "review_frequency": "continuous",
    --   "last_manual_review": "2026-03-15",
    --   "compensating_controls": [],
    --   "scoping_notes": "Applies to production environment only"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, control_id)
);

CREATE INDEX idx_ctrl_impl_org ON control_implementations (organization_id);
CREATE INDEX idx_ctrl_impl_status ON control_implementations (status);
```

## Policies, Risks, Vendors, and Audit Log

```sql
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'draft',
    content_uri TEXT,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "category": "access_control",
    --   "applies_to": ["all_employees"],
    --   "review_cycle_months": 12,
    --   "approval_chain": ["ciso", "cto"],
    --   "mapped_controls": ["CC6.1", "A.8.2"],
    --   "regulatory_drivers": ["soc2", "iso27001"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policies_org ON policies (organization_id);

CREATE TABLE risks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    description TEXT,
    likelihood TEXT NOT NULL,
    impact TEXT NOT NULL,
    risk_score INTEGER,
    treatment TEXT NOT NULL DEFAULT 'mitigate',
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'open',
    risk_details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "risk_category": "data_breach",
    --   "affected_assets": ["customer_database", "payment_system"],
    --   "related_controls": ["CC6.1", "CC7.2"],
    --   "treatment_plan": "Implement encryption at rest and in transit",
    --   "residual_risk_score": 4,
    --   "review_history": [
    --     {"date": "2026-01-15", "reviewer": "CISO", "decision": "continue treatment"}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risks_org ON risks (organization_id);
CREATE INDEX idx_risks_status ON risks (status);

CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    risk_tier TEXT NOT NULL DEFAULT 'medium',
    status TEXT NOT NULL DEFAULT 'active',
    vendor_details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "category": "cloud_provider",
    --   "data_processing": true,
    --   "sub_processors": ["Cloudflare", "Datadog"],
    --   "dpa_signed": true,
    --   "dpa_expiry": "2027-01-01",
    --   "soc2_report_date": "2026-03-15",
    --   "security_review_score": 92,
    --   "contacts": [{"name": "John", "email": "john@vendor.com", "role": "security"}]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendors_org ON vendors (organization_id);

-- Audit log with JSONB details
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id UUID,
    actor_type TEXT NOT NULL DEFAULT 'user',
    action TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "before": {"status": "fail"},
    --   "after": {"status": "pass"},
    --   "reason": "Remediation applied: enabled encryption"
    -- }
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log (organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organizations, users |
| Frameworks & Controls | 3 | frameworks, controls, control_mappings |
| Integrations & Resources | 2 | integrations, cloud_resources |
| Evidence & Findings | 3 | evidence, evidence_controls, findings |
| Organization Compliance | 2 | organization_frameworks, control_implementations |
| Policy, Risk, Vendor | 3 | policies, risks, vendors |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **16** | JSONB columns replace 8-10 junction/detail tables from the normalized model |

---

## Key Design Decisions

1. **JSONB `properties` on frameworks and controls.** Each compliance framework has unique metadata (SOC 2's trust categories, PCI's SAQ types, NIST's baselines). Rather than a polymorphic table hierarchy, JSONB properties hold framework-specific fields while stable fields (code, title, description) remain relational.

2. **JSONB `attributes` on cloud_resources.** AWS S3 buckets, Azure VMs, and GCP Cloud SQL instances have completely different attribute schemas. JSONB avoids a separate table per resource type while GIN indexes enable attribute-based queries (e.g., "find all unencrypted resources").

3. **JSONB `details` on findings with AI remediation.** Scanner-specific finding details (Prowler check IDs, CIS benchmark references, actual vs. expected values) vary by scanner. The `ai_remediation` JSONB field stores structured remediation playbooks generated by the AI agent, including step-by-step instructions and Terraform fix snippets.

4. **JSONB `config` on integrations.** AWS authentication uses IAM roles with ARNs; Azure uses service principals with tenant IDs; Okta uses API keys with domains. JSONB config holds provider-specific authentication and scope configuration without a separate config table per provider.

5. **JSON Schema validation at application layer.** While JSONB columns lack database-level constraints, each JSONB field has a corresponding JSON Schema (stored in application code or a schema registry) that validates structure before writes.

6. **Cross-framework deduplication preserved in relational form.** The `evidence_controls` junction table remains relational (not JSONB) because cross-framework dedup queries are performance-critical and benefit from proper indexes and join optimization.

7. **GIN indexes on all JSONB columns.** Every JSONB column that will be queried has a GIN index, enabling fast containment (`@>`) and existence (`?`) queries without full table scans.

8. **Reduced table count vs. normalized model.** This schema uses 16 tables vs. 25+ in the normalized model. The reduction comes from eliminating junction tables (risk_controls, policy_controls) and detail tables (framework_categories) by embedding that data in JSONB.

9. **OSCAL documents stored as JSONB.** OSCAL catalogs, profiles, and SSPs can be stored directly as JSONB in the framework and control tables, enabling lossless round-trip import/export without a separate document store.

10. **Audit log uses JSONB for change tracking.** The `changes` field captures before/after snapshots as JSONB, accommodating any entity type without requiring a fixed column set for every possible change.
