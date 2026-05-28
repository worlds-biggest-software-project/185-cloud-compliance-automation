# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Cloud Compliance Automation · Created: 2026-05-20

## Philosophy

This model treats every compliance-relevant action as an immutable domain event recorded in an append-only event store. The event store is the single source of truth. Materialized read models (projections) are built from the event stream to serve dashboards, reports, and API queries. This is the CQRS (Command Query Responsibility Segregation) pattern applied to compliance automation.

The approach is directly motivated by the domain requirement: compliance is fundamentally about proving "what happened, when, and by whom." An event-sourced system provides this by default — the entire compliance history is an unalterable, replayable log. Auditors reviewing a SOC 2 Type II engagement need to see not just the current state of controls, but the complete history of evidence collection, control test results, and remediation actions over the observation period. Event sourcing makes temporal queries ("what was the compliance posture on March 15?") trivial rather than requiring complex audit trail reconstruction.

This pattern is used by financial systems (double-entry bookkeeping is essentially event sourcing), healthcare record systems, and regulatory reporting platforms where immutability and full traceability are non-negotiable. For a compliance automation platform, the audit trail is not a secondary concern bolted onto the side — it IS the product.

**Best for:** Organizations where full temporal traceability is essential, predictive analytics on compliance trends are a priority, and AI-powered pattern detection across historical compliance data is a key differentiator.

**Trade-offs:**
- (+) Complete, immutable audit trail by default — no separate audit log table needed
- (+) Temporal queries are natural: "replay to any point in time"
- (+) Ideal for AI/ML training on compliance patterns (the event stream IS the training data)
- (+) Predictive audit risk scoring is straightforward with historical event patterns
- (+) Event replay enables compliance posture reconstruction for any audit window
- (-) Higher storage requirements (events are never deleted)
- (-) Read model consistency is eventually consistent, not immediate
- (-) More complex to implement than traditional CRUD
- (-) Schema evolution of events requires careful versioning
- (-) Developers less familiar with CQRS patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL Assessment Results | Events map to OSCAL observation and finding structures |
| CloudEvents v1.0 | Event envelope format follows CloudEvents spec |
| SOC 2 Type II | Event stream naturally provides observation period evidence |
| NIST CSF 2.0 | Govern function requirements satisfied by immutable event log |
| ISO 27001 A.8.15 | Logging and monitoring controls satisfied by design |
| OCSF (Open Cybersecurity Schema) | Event categorization aligned with OCSF event classes |
| RFC 7807 | Error events follow Problem Details format |

---

## Event Store (Source of Truth)

```sql
-- The immutable event store — append-only, never updated or deleted
CREATE TABLE compliance_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,               -- aggregate root ID (e.g. organization_id, control_impl_id)
    stream_type TEXT NOT NULL,              -- 'organization', 'control_implementation', 'evidence', 'finding', 'assessment'
    event_type TEXT NOT NULL,               -- e.g. 'ControlTestExecuted', 'EvidenceCollected', 'FindingCreated'
    event_version INTEGER NOT NULL DEFAULT 1,  -- schema version of this event type
    sequence_number BIGINT NOT NULL,        -- monotonically increasing per stream
    payload JSONB NOT NULL,                -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',  -- actor, IP, correlation_id, causation_id
    -- CloudEvents envelope fields
    ce_source TEXT NOT NULL,               -- e.g. '/orgs/{org_id}/integrations/prowler'
    ce_subject TEXT,                       -- e.g. '/controls/CC6.1/findings/abc123'
    organization_id UUID NOT NULL,         -- partition key
    occurred_at TIMESTAMPTZ NOT NULL,      -- when the event actually happened
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when it was stored
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions for the event store
CREATE TABLE compliance_events_2026_01 PARTITION OF compliance_events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions created by automation

CREATE INDEX idx_events_stream ON compliance_events (stream_id, sequence_number);
CREATE INDEX idx_events_org ON compliance_events (organization_id, recorded_at);
CREATE INDEX idx_events_type ON compliance_events (event_type, recorded_at);
CREATE INDEX idx_events_occurred ON compliance_events (occurred_at);

-- Event type registry — documents all valid event types and their schemas
CREATE TABLE event_type_registry (
    event_type TEXT NOT NULL,
    event_version INTEGER NOT NULL,
    json_schema JSONB NOT NULL,            -- JSON Schema validating the payload
    description TEXT,
    is_deprecated BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (event_type, event_version)
);
```

### Example Event Payloads

```sql
-- ControlTestExecuted event
-- payload: {
--   "control_code": "CC6.1",
--   "framework_code": "soc2",
--   "resource_id": "arn:aws:s3:::my-bucket",
--   "result": "fail",
--   "severity": "high",
--   "scanner": "prowler",
--   "check_id": "s3_bucket_public_access",
--   "details": "S3 bucket has public read access enabled"
-- }
-- metadata: {
--   "actor_type": "integration",
--   "actor_id": "integration-uuid",
--   "correlation_id": "scan-run-uuid",
--   "ip_address": null
-- }

-- EvidenceCollected event
-- payload: {
--   "evidence_type": "api_snapshot",
--   "title": "AWS S3 bucket policy for my-bucket",
--   "storage_uri": "s3://evidence-bucket/2026/05/abc123.json",
--   "content_hash": "sha256:abcdef...",
--   "control_codes": ["CC6.1", "A.8.2"],
--   "framework_codes": ["soc2", "iso27001"],
--   "source": "aws_config"
-- }

-- FindingResolved event
-- payload: {
--   "finding_id": "finding-uuid",
--   "resolution": "remediated",
--   "remediation_description": "Disabled public access on S3 bucket",
--   "verified_by_scan": true,
--   "time_to_resolve_hours": 4.5
-- }

-- CompliancePostureChanged event
-- payload: {
--   "framework_code": "soc2",
--   "previous_score": 87.5,
--   "new_score": 92.3,
--   "controls_improved": ["CC6.1", "CC7.2"],
--   "controls_degraded": []
-- }
```

## Materialized Read Models (Projections)

These tables are rebuilt from the event stream. They can be dropped and reconstructed at any time.

```sql
-- Current compliance posture (projection)
CREATE TABLE rm_compliance_posture (
    organization_id UUID NOT NULL,
    framework_code TEXT NOT NULL,
    total_controls INTEGER NOT NULL DEFAULT 0,
    passing_controls INTEGER NOT NULL DEFAULT 0,
    failing_controls INTEGER NOT NULL DEFAULT 0,
    not_applicable_controls INTEGER NOT NULL DEFAULT 0,
    compliance_score NUMERIC(5,2),         -- percentage
    last_scan_at TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, framework_code)
);

-- Current control status (projection)
CREATE TABLE rm_control_status (
    organization_id UUID NOT NULL,
    control_id UUID NOT NULL,
    framework_code TEXT NOT NULL,
    control_code TEXT NOT NULL,
    current_status TEXT NOT NULL,           -- 'pass', 'fail', 'warning', 'not_tested'
    last_test_result TEXT,
    last_test_at TIMESTAMPTZ,
    evidence_count INTEGER NOT NULL DEFAULT 0,
    finding_count INTEGER NOT NULL DEFAULT 0,
    open_finding_count INTEGER NOT NULL DEFAULT 0,
    risk_score NUMERIC(5,2),               -- AI-computed risk of failing audit
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, control_id)
);

CREATE INDEX idx_rm_ctrl_status_framework ON rm_control_status (framework_code);
CREATE INDEX idx_rm_ctrl_status_status ON rm_control_status (current_status);

-- Current findings (projection)
CREATE TABLE rm_findings (
    finding_id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    control_code TEXT NOT NULL,
    framework_code TEXT NOT NULL,
    resource_id TEXT,
    status TEXT NOT NULL,                  -- 'open', 'resolved', 'accepted', 'false_positive'
    severity TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    remediation_guidance TEXT,
    scanner_source TEXT,
    first_detected_at TIMESTAMPTZ NOT NULL,
    resolved_at TIMESTAMPTZ,
    time_to_resolve_hours NUMERIC(10,2),
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_findings_org ON rm_findings (organization_id);
CREATE INDEX idx_rm_findings_status ON rm_findings (status);
CREATE INDEX idx_rm_findings_severity ON rm_findings (severity);

-- Current evidence inventory (projection)
CREATE TABLE rm_evidence (
    evidence_id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    title TEXT NOT NULL,
    evidence_type TEXT NOT NULL,
    source TEXT,
    storage_uri TEXT,
    content_hash TEXT,
    control_codes TEXT[] NOT NULL DEFAULT '{}',
    framework_codes TEXT[] NOT NULL DEFAULT '{}',
    collected_at TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ,
    is_expired BOOLEAN NOT NULL DEFAULT false,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_evidence_org ON rm_evidence (organization_id);
CREATE INDEX idx_rm_evidence_controls ON rm_evidence USING gin (control_codes);

-- Cloud resource inventory (projection)
CREATE TABLE rm_cloud_resources (
    resource_id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    provider TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    provider_resource_id TEXT NOT NULL,
    resource_name TEXT,
    region TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    open_findings_count INTEGER NOT NULL DEFAULT 0,
    last_scanned_at TIMESTAMPTZ,
    first_seen_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_resources_org ON rm_cloud_resources (organization_id);
CREATE INDEX idx_rm_resources_findings ON rm_cloud_resources (open_findings_count) WHERE open_findings_count > 0;

-- Assessment history (projection)
CREATE TABLE rm_assessments (
    assessment_id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    framework_code TEXT NOT NULL,
    assessment_type TEXT NOT NULL,
    status TEXT NOT NULL,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    total_controls INTEGER,
    satisfied_controls INTEGER,
    lead_assessor TEXT,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_assessments_org ON rm_assessments (organization_id);
```

## Reference Data (Mutable, Non-Event-Sourced)

```sql
-- These tables hold relatively static reference data and are managed via standard CRUD.
-- They do NOT participate in event sourcing.

CREATE TABLE ref_frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    issuing_body TEXT NOT NULL,
    oscal_catalog_uri TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES ref_frameworks(id),
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    guidance TEXT,
    control_type TEXT NOT NULL DEFAULT 'preventive',
    is_automatable BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE INDEX idx_ref_controls_framework ON ref_controls (framework_id);

CREATE TABLE ref_control_mappings (
    source_control_id UUID NOT NULL REFERENCES ref_controls(id),
    target_control_id UUID NOT NULL REFERENCES ref_controls(id),
    mapping_strength TEXT NOT NULL DEFAULT 'partial',
    PRIMARY KEY (source_control_id, target_control_id)
);

CREATE TABLE ref_organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES ref_organizations(id),
    email TEXT NOT NULL,
    display_name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'viewer',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
```

## Projection Rebuild Infrastructure

```sql
-- Tracks the position of each projection in the event stream
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID,
    last_sequence_number BIGINT,
    last_processed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'running',  -- 'running', 'paused', 'rebuilding', 'error'
    error_message TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshot store for long-lived aggregates (optional optimization)
CREATE TABLE aggregate_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

## Example Temporal Query: Compliance Posture at a Point in Time

```sql
-- Reconstruct the compliance posture for organization X on March 15, 2026
-- by replaying all ControlTestExecuted events up to that date
WITH latest_results AS (
    SELECT DISTINCT ON (payload->>'control_code', payload->>'framework_code')
        payload->>'control_code' AS control_code,
        payload->>'framework_code' AS framework_code,
        payload->>'result' AS result,
        occurred_at
    FROM compliance_events
    WHERE organization_id = 'org-uuid'
      AND event_type = 'ControlTestExecuted'
      AND occurred_at <= '2026-03-15T23:59:59Z'
    ORDER BY payload->>'control_code', payload->>'framework_code', occurred_at DESC
)
SELECT
    framework_code,
    COUNT(*) AS total_controls,
    COUNT(*) FILTER (WHERE result = 'pass') AS passing,
    COUNT(*) FILTER (WHERE result = 'fail') AS failing,
    ROUND(100.0 * COUNT(*) FILTER (WHERE result = 'pass') / COUNT(*), 1) AS compliance_pct
FROM latest_results
GROUP BY framework_code;
```

## Example Query: Mean Time to Resolve by Severity

```sql
-- AI-powered trend analysis uses this kind of query to predict audit risk
SELECT
    payload->>'severity' AS severity,
    AVG((resolve_event.payload->>'time_to_resolve_hours')::numeric) AS avg_hours,
    PERCENTILE_CONT(0.5) WITHIN GROUP (
        ORDER BY (resolve_event.payload->>'time_to_resolve_hours')::numeric
    ) AS median_hours,
    COUNT(*) AS total_findings
FROM compliance_events create_event
JOIN compliance_events resolve_event
    ON resolve_event.event_type = 'FindingResolved'
    AND resolve_event.payload->>'finding_id' = create_event.stream_id::text
WHERE create_event.event_type = 'FindingCreated'
  AND create_event.organization_id = 'org-uuid'
  AND create_event.occurred_at >= now() - INTERVAL '6 months'
GROUP BY payload->>'severity'
ORDER BY avg_hours DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | compliance_events (partitioned), event_type_registry |
| Read Models (Projections) | 7 | rm_compliance_posture, rm_control_status, rm_findings, rm_evidence, rm_cloud_resources, rm_assessments |
| Reference Data | 5 | ref_frameworks, ref_controls, ref_control_mappings, ref_organizations, ref_users |
| Infrastructure | 2 | projection_checkpoints, aggregate_snapshots |
| **Total** | **16** | Fewer tables than normalized model; complexity is in event processing |

---

## Key Design Decisions

1. **Single event store table, partitioned by month.** All compliance events share one table rather than separate tables per event type. This simplifies event replay and cross-type queries while monthly partitioning keeps queries fast.

2. **CloudEvents envelope for interoperability.** The `ce_source` and `ce_subject` fields follow the CloudEvents v1.0 specification, making events publishable to external systems (webhooks, message brokers) without transformation.

3. **Read models are disposable.** Every `rm_*` table can be dropped and rebuilt from the event stream. This eliminates data migration concerns — schema changes to read models are a rebuild, not a migration.

4. **Reference data is not event-sourced.** Framework definitions and control libraries change infrequently and don't need temporal history. They use traditional CRUD tables to avoid unnecessary complexity.

5. **Sequence numbers per stream.** The `sequence_number` field on events ensures ordering within a single aggregate stream, preventing out-of-order processing during projection rebuilds.

6. **Event versioning via `event_version`.** When event payloads evolve, the version number allows projection code to handle old and new event formats simultaneously without breaking the event stream.

7. **Temporal queries are first-class.** The `occurred_at` vs `recorded_at` distinction supports bi-temporal modeling — "when did it happen" vs "when did we learn about it" — which is critical for compliance evidence windows.

8. **AI training data is the event stream itself.** Predictive audit risk scoring and pattern detection operate directly on the event store, eliminating the need for a separate analytics pipeline.

9. **Correlation and causation IDs in metadata.** The `correlation_id` links events from the same scan run; the `causation_id` links cause-and-effect chains (e.g., FindingCreated causes RemediationTaskCreated).

10. **Projection checkpoints for resilience.** The `projection_checkpoints` table tracks exactly where each read model left off, enabling crash recovery without full replay.
