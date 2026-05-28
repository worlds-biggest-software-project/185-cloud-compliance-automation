# Data Model Suggestion 4: Graph-Relational

> Project: Cloud Compliance Automation · Created: 2026-05-20

## Philosophy

This model combines a relational backbone for operational CRUD with a property graph layer for relationship-heavy compliance queries. The core insight is that compliance is fundamentally a graph problem: frameworks contain controls, controls map to other controls across frameworks, controls are implemented by cloud resources, evidence satisfies controls, findings affect resources and controls, risks are mitigated by controls, and policies govern controls. These relationships form a dense, interconnected graph where the most valuable queries involve traversal — "which frameworks are affected if this S3 bucket is misconfigured?" or "show me all evidence paths from this finding to every framework it impacts."

The design uses a dedicated graph layer implemented as `graph_nodes` and `graph_edges` tables in PostgreSQL (with recursive CTEs for traversal), keeping the relational tables for day-to-day CRUD operations. This avoids requiring a separate graph database (Neo4j) while delivering graph query capabilities. For organizations that outgrow PostgreSQL's graph performance, the graph layer can be migrated to Neo4j or Amazon Neptune without changing the relational operational tables.

This pattern draws inspiration from Wiz's Security Graph, which contextualizes security findings by linking vulnerabilities to exposure paths, cloud resources, and identities. A compliance graph extends this concept to include framework controls, evidence, and audit relationships — enabling queries like "show me the shortest path from this critical finding to each compliance framework, through the controls and evidence involved."

**Best for:** Platforms where cross-framework impact analysis, blast-radius assessment, and relationship-driven compliance intelligence are key differentiators. Ideal when the AI engine needs to traverse compliance relationships to answer natural-language questions.

**Trade-offs:**
- (+) Powerful traversal queries: "what does this misconfiguration affect across all frameworks?"
- (+) Natural model for cross-framework deduplication — it is literally a graph edge
- (+) AI gap analysis can traverse the graph to find missing evidence paths
- (+) Blast-radius analysis for compliance: "if this resource fails, which controls/frameworks are impacted?"
- (+) Can migrate graph layer to dedicated graph DB (Neo4j, Neptune) if needed
- (-) Graph queries (recursive CTEs) are more complex than simple joins
- (-) Dual storage (relational + graph) means some data is denormalized
- (-) Graph traversal performance degrades with very deep paths without optimization
- (-) Developers need to understand both relational and graph query patterns
- (-) Graph edge maintenance adds write overhead

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | OSCAL relationships (catalog-profile-SSP-assessment) modeled as graph edges |
| ISO/IEC 27001:2022 | Control-to-control relationships in Annex A modeled as graph edges |
| NIST CSF 2.0 | Function-category-subcategory hierarchy as graph paths |
| CIS Controls v8 | Safeguard-to-implementation group relationships as graph edges |
| Wiz Security Graph | Architectural inspiration for resource-vulnerability-exposure graph model |
| NIST SP 800-53 | Control enhancement and related-control links as typed graph edges |
| CloudEvents v1.0 | Graph change events follow CloudEvents spec for notifications |

---

## Graph Layer

```sql
-- Generic graph node table — every compliance entity gets a node
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type TEXT NOT NULL,               -- 'framework', 'control', 'resource', 'evidence', 'finding',
                                           -- 'risk', 'policy', 'vendor', 'user', 'integration'
    entity_id UUID NOT NULL,               -- FK to the corresponding relational table row
    organization_id UUID,                  -- null for global entities (frameworks, controls)
    label TEXT NOT NULL,                   -- human-readable label for visualization
    properties JSONB NOT NULL DEFAULT '{}', -- node metadata for graph queries
    -- Example for a control node: {
    --   "framework_code": "soc2",
    --   "control_code": "CC6.1",
    --   "is_automatable": true,
    --   "control_type": "preventive"
    -- }
    -- Example for a resource node: {
    --   "provider": "aws",
    --   "resource_type": "s3_bucket",
    --   "region": "us-east-1",
    --   "is_active": true
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_org ON graph_nodes (organization_id);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes (entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING gin (properties);

-- Generic graph edge table — typed, directed relationships between nodes
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type TEXT NOT NULL,               -- relationship type (see taxonomy below)
    weight NUMERIC(5,2) DEFAULT 1.0,       -- edge strength/confidence
    properties JSONB NOT NULL DEFAULT '{}', -- edge metadata
    organization_id UUID,                  -- null for global edges
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type)
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_org ON graph_edges (organization_id);
CREATE INDEX idx_graph_edges_props ON graph_edges USING gin (properties);
```

### Edge Type Taxonomy

```sql
-- Document all valid edge types for consistency
CREATE TABLE edge_type_definitions (
    edge_type TEXT PRIMARY KEY,
    source_node_type TEXT NOT NULL,
    target_node_type TEXT NOT NULL,
    description TEXT NOT NULL,
    is_bidirectional BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed edge types
INSERT INTO edge_type_definitions (edge_type, source_node_type, target_node_type, description, is_bidirectional) VALUES
-- Framework & Control relationships
('CONTAINS_CONTROL',    'framework',  'control',    'Framework contains this control',                    false),
('MAPS_TO',             'control',    'control',    'Cross-framework control mapping',                    true),
('PARENT_OF',           'control',    'control',    'Parent-child control hierarchy (NIST enhancements)', false),
('ENHANCES',            'control',    'control',    'Control enhancement relationship (NIST 800-53)',     false),
('RELATED_TO',          'control',    'control',    'Related controls within same framework',             true),
-- Evidence relationships
('SATISFIES',           'evidence',   'control',    'Evidence satisfies this control requirement',        false),
('COLLECTED_FROM',      'evidence',   'resource',   'Evidence was collected from this resource',          false),
('COLLECTED_BY',        'evidence',   'integration','Evidence collected via this integration',            false),
-- Finding relationships
('AFFECTS',             'finding',    'resource',   'Finding affects this cloud resource',                false),
('VIOLATES',            'finding',    'control',    'Finding indicates violation of this control',        false),
('REMEDIATED_BY',       'finding',    'evidence',   'Finding was remediated and evidenced by this',       false),
-- Resource relationships
('EVALUATED_AGAINST',   'resource',   'control',    'Resource is evaluated against this control',         false),
('DEPENDS_ON',          'resource',   'resource',   'Resource depends on another resource',               false),
('HOSTED_IN',           'resource',   'resource',   'Resource is hosted in another (e.g. EC2 in VPC)',   false),
-- Risk relationships
('MITIGATED_BY',        'risk',       'control',    'Risk is mitigated by this control',                  false),
('AFFECTS_RESOURCE',    'risk',       'resource',   'Risk affects this resource',                         false),
-- Policy relationships
('GOVERNS',             'policy',     'control',    'Policy governs this control implementation',         false),
('ACCEPTED_BY',         'policy',     'user',       'Policy accepted by this user',                      false),
-- Vendor relationships
('PROVIDES',            'vendor',     'resource',   'Vendor provides this resource/service',              false),
('ASSESSED_AGAINST',    'vendor',     'control',    'Vendor assessed against this control',               false),
-- Organizational
('RESPONSIBLE_FOR',     'user',       'control',    'User is responsible for this control',               false),
('OWNS',                'user',       'risk',       'User owns this risk',                                false);
```

## Relational Operational Tables

```sql
-- These tables handle day-to-day CRUD; graph nodes/edges mirror their relationships

CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    issuing_body TEXT NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    control_type TEXT NOT NULL DEFAULT 'preventive',
    is_automatable BOOLEAN NOT NULL DEFAULT false,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE INDEX idx_controls_framework ON controls (framework_id);

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    config JSONB NOT NULL DEFAULT '{}',
    credentials_vault_ref TEXT,
    last_sync_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cloud_resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    integration_id UUID NOT NULL REFERENCES integrations(id),
    provider TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    resource_name TEXT,
    region TEXT,
    attributes JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider, resource_id)
);

CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    evidence_type TEXT NOT NULL,
    source TEXT,
    storage_uri TEXT,
    content_hash TEXT,
    metadata JSONB NOT NULL DEFAULT '{}',
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    cloud_resource_id UUID REFERENCES cloud_resources(id),
    status TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'medium',
    title TEXT NOT NULL,
    description TEXT,
    details JSONB NOT NULL DEFAULT '{}',
    evaluated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_org ON findings (organization_id);
CREATE INDEX idx_findings_status ON findings (status);
CREATE INDEX idx_findings_severity ON findings (severity);

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'draft',
    content_uri TEXT,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    risk_tier TEXT NOT NULL DEFAULT 'medium',
    status TEXT NOT NULL DEFAULT 'active',
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id UUID,
    action TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log (organization_id, created_at);
```

## Graph Traversal Queries

### Blast-Radius Analysis: What frameworks are affected by a failing resource?

```sql
-- Given a cloud resource with a critical finding, traverse the graph to find
-- all frameworks affected through control violations
WITH RECURSIVE blast_radius AS (
    -- Start from the finding node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        ge.edge_type,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE gn.entity_id = 'finding-uuid'  -- the critical finding
      AND gn.node_type = 'finding'

    UNION ALL

    -- Traverse outward through edges
    SELECT
        target.id,
        target.node_type,
        target.label,
        ge.edge_type,
        br.depth + 1,
        br.path || target.id
    FROM blast_radius br
    JOIN graph_edges ge ON ge.source_node_id = br.node_id
    JOIN graph_nodes target ON target.id = ge.target_node_id
    WHERE br.depth < 5                    -- limit traversal depth
      AND NOT (target.id = ANY(br.path))  -- prevent cycles
)
SELECT DISTINCT node_type, label, edge_type, depth
FROM blast_radius
WHERE node_type = 'framework'
ORDER BY depth;
```

### Evidence Gap Analysis: Find controls with no evidence path

```sql
-- Find controls that have no SATISFIES edge from any evidence node
-- These are gaps in audit readiness
SELECT
    c.code AS control_code,
    f.code AS framework_code,
    c.title AS control_title
FROM controls c
JOIN frameworks f ON f.id = c.framework_id
JOIN graph_nodes cn ON cn.entity_id = c.id AND cn.node_type = 'control'
LEFT JOIN graph_edges ge ON ge.target_node_id = cn.id AND ge.edge_type = 'SATISFIES'
WHERE ge.id IS NULL
  AND cn.organization_id = 'org-uuid'
ORDER BY f.code, c.code;
```

### Cross-Framework Impact: What other controls are affected if CC6.1 fails?

```sql
-- Traverse MAPS_TO edges from a SOC 2 control to find all mapped controls
-- across all frameworks
WITH RECURSIVE mapped_controls AS (
    SELECT
        gn.id AS node_id,
        gn.properties->>'control_code' AS control_code,
        gn.properties->>'framework_code' AS framework_code,
        0 AS depth
    FROM graph_nodes gn
    WHERE gn.node_type = 'control'
      AND gn.properties->>'control_code' = 'CC6.1'
      AND gn.properties->>'framework_code' = 'soc2'

    UNION ALL

    SELECT
        target.id,
        target.properties->>'control_code',
        target.properties->>'framework_code',
        mc.depth + 1
    FROM mapped_controls mc
    JOIN graph_edges ge ON ge.source_node_id = mc.node_id
    JOIN graph_nodes target ON target.id = ge.target_node_id
    WHERE ge.edge_type = 'MAPS_TO'
      AND mc.depth < 3
)
SELECT DISTINCT framework_code, control_code, depth
FROM mapped_controls
WHERE depth > 0
ORDER BY framework_code, control_code;
```

### Compliance Posture Visualization: Build the full graph for a framework

```sql
-- Extract the subgraph for a specific framework for visualization
-- Returns nodes and edges that a frontend can render as a force-directed graph
SELECT
    gn.id AS node_id,
    gn.node_type,
    gn.label,
    gn.properties,
    ge.id AS edge_id,
    ge.edge_type,
    ge.weight,
    ge.target_node_id
FROM graph_nodes gn
LEFT JOIN graph_edges ge ON ge.source_node_id = gn.id
WHERE gn.organization_id = 'org-uuid'
  AND (
    gn.node_type = 'control'
    AND gn.properties->>'framework_code' = 'soc2'
  )
  OR gn.id IN (
    SELECT ge2.target_node_id FROM graph_edges ge2
    JOIN graph_nodes src ON src.id = ge2.source_node_id
    WHERE src.properties->>'framework_code' = 'soc2'
      AND src.node_type = 'control'
  );
```

## Graph Maintenance Triggers

```sql
-- Automatically create graph nodes when relational entities are inserted
-- (implemented as application-level middleware or database triggers)

-- Example trigger for cloud_resources
CREATE OR REPLACE FUNCTION sync_resource_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_nodes (node_type, entity_id, organization_id, label, properties)
    VALUES (
        'resource',
        NEW.id,
        NEW.organization_id,
        COALESCE(NEW.resource_name, NEW.resource_id),
        jsonb_build_object(
            'provider', NEW.provider,
            'resource_type', NEW.resource_type,
            'region', NEW.region,
            'is_active', NEW.is_active
        )
    )
    ON CONFLICT (node_type, entity_id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_resource_to_graph
    AFTER INSERT OR UPDATE ON cloud_resources
    FOR EACH ROW EXECUTE FUNCTION sync_resource_to_graph();

-- Example trigger for findings — creates node and VIOLATES/AFFECTS edges
CREATE OR REPLACE FUNCTION sync_finding_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    finding_node_id UUID;
    control_node_id UUID;
    resource_node_id UUID;
BEGIN
    -- Create or update the finding node
    INSERT INTO graph_nodes (node_type, entity_id, organization_id, label, properties)
    VALUES (
        'finding',
        NEW.id,
        NEW.organization_id,
        NEW.title,
        jsonb_build_object('status', NEW.status, 'severity', NEW.severity)
    )
    ON CONFLICT (node_type, entity_id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties
    RETURNING id INTO finding_node_id;

    -- Create VIOLATES edge to control
    SELECT id INTO control_node_id FROM graph_nodes
    WHERE node_type = 'control' AND entity_id = NEW.control_id;

    IF control_node_id IS NOT NULL THEN
        INSERT INTO graph_edges (source_node_id, target_node_id, edge_type, organization_id)
        VALUES (finding_node_id, control_node_id, 'VIOLATES', NEW.organization_id)
        ON CONFLICT (source_node_id, target_node_id, edge_type) DO NOTHING;
    END IF;

    -- Create AFFECTS edge to resource (if applicable)
    IF NEW.cloud_resource_id IS NOT NULL THEN
        SELECT id INTO resource_node_id FROM graph_nodes
        WHERE node_type = 'resource' AND entity_id = NEW.cloud_resource_id;

        IF resource_node_id IS NOT NULL THEN
            INSERT INTO graph_edges (source_node_id, target_node_id, edge_type, organization_id)
            VALUES (finding_node_id, resource_node_id, 'AFFECTS', NEW.organization_id)
            ON CONFLICT (source_node_id, target_node_id, edge_type) DO NOTHING;
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_finding_to_graph
    AFTER INSERT OR UPDATE ON findings
    FOR EACH ROW EXECUTE FUNCTION sync_finding_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_nodes, graph_edges, edge_type_definitions |
| Identity | 2 | organizations, users |
| Frameworks & Controls | 2 | frameworks, controls |
| Integrations & Resources | 2 | integrations, cloud_resources |
| Evidence & Findings | 2 | evidence, findings |
| Risk, Policy, Vendor | 3 | risks, policies, vendors |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **15** | Graph layer replaces all junction tables and adds traversal capabilities |

---

## Key Design Decisions

1. **Generic graph layer over domain-specific junction tables.** Instead of `evidence_controls`, `risk_controls`, `policy_controls`, etc., all relationships are graph edges with typed `edge_type` values. This eliminates junction table proliferation and enables new relationship types without schema changes.

2. **Edge type taxonomy as a first-class entity.** The `edge_type_definitions` table documents all valid edge types with their source/target node types. This serves as both documentation and validation — application code can check edge validity before insertion.

3. **Dual storage (relational + graph) with trigger-based sync.** Relational tables handle CRUD operations efficiently; triggers automatically maintain the graph layer. This means operational code (create a finding, collect evidence) does not need to know about the graph — it just writes to relational tables.

4. **Graph properties mirror key relational fields.** The `properties` JSONB on graph nodes contains a subset of the relational data needed for graph queries (e.g., control_code, framework_code, severity). This avoids expensive joins back to relational tables during graph traversal.

5. **Bidirectional edge support.** The `is_bidirectional` flag in edge type definitions indicates when an edge should be traversable in both directions (e.g., MAPS_TO between controls). The application layer creates reciprocal edges or handles bidirectional traversal in queries.

6. **Depth-limited traversal.** All recursive CTE queries include a depth limit (typically 3-5) to prevent runaway traversal in dense graphs. This is a deliberate safety measure for PostgreSQL-based graph queries.

7. **Weight on edges for AI confidence.** The `weight` field on edges can represent AI confidence scores (e.g., how strongly does this evidence satisfy this control?) or mapping strength (exact vs. partial). Graph algorithms can use weight for path ranking.

8. **Cascade delete on graph edges.** `ON DELETE CASCADE` on edge foreign keys means that when a relational entity is deleted and its graph node is removed, all associated edges are automatically cleaned up.

9. **Organization scoping on graph nodes/edges.** The `organization_id` field enables multi-tenant graph queries. Global entities (frameworks, controls) have null organization_id; tenant-specific entities (resources, findings, evidence) are scoped.

10. **Migration path to dedicated graph database.** The graph_nodes/graph_edges schema is designed to be exportable to Neo4j (nodes become labeled nodes, edges become typed relationships, properties map directly). If PostgreSQL graph performance becomes a bottleneck, the graph layer can be migrated without touching operational tables.
