# SmartScape Repo — Claude Context

## Purpose

This is a **public, customer-facing** reference guide for Dynatrace SmartScape on Grail.
Maintained under `gabriel-dynatrace/smartscapes`. All content must follow the
documentation standards in the global CLAUDE.md.

---

## Subject Matter Expert

**Aeric Walls** — Dynatrace internal SME on SmartScape on Grail.
His recorded sessions are the authoritative source for behavior not yet in official docs.
When his statements conflict with anything else (including this file), trust Aeric and update accordingly.

Session referenced: *Western Region Community Session — SmartScape on Grail* (~early 2026)

---

## Validated DQL Examples

All DQL in README.md has been validated against a live tenant via `mcp__dynatrace-mcp__execute-dql`
**unless** the example is explicitly marked with:

> **Note: Not validated against tenant**

Current unvalidated examples (require manual tenant verification):
- `getNodeName(id)` — function name confirmed by SME; exact DQL syntax not tenant-verified
- `getNodeField(id, "field.name")` — same as above
- Multi-hop metric join recipe — pattern is SME-confirmed; field names may vary
- Kubernetes compliance recipe — pattern is SME-confirmed; exact `k8s.object` field paths may vary by operator version

---

## Key Facts — SmartScape on Grail (from SME Session)

### What is and isn't on SmartScape on Grail
- **On Grail:** hosts, processes, services, frontends, containers, OneAgent, all Kubernetes entity types, AWS, Azure, GCP (as integrations mature)
- **NOT on Grail yet:** network devices, custom devices — still in SmartScape Classic
- Network devices will eventually get dedicated typed nodes (e.g., `NETWORK_DEVICE`)
- Custom devices are being phased out in favor of properly typed extension entities
- Extensions are beginning to ship with OpenPipeline configs that include node/edge extraction

### Entity type naming changes
| Old name | SmartScape on Grail name |
|----------|--------------------------|
| `APPLICATION` | `FRONTEND` |
| process group instance | `PROCESS` |
| `dt.entity.service` (dimension) | `dt.smartscape.service` |

### Management zones → Segments
- SmartScape on Grail does **not** use management zones and will not
- **Segments** replace management zones for filtering/scoping
- **IAM policies** replace management zones for access control
- Segment filters use primary Grail fields (e.g., `cloud.provider`, `k8s.namespace.name`)
- Same field names work across entities, metrics, logs, spans, events simultaneously
- No automated migration path from management zones to segments — must be rebuilt manually
- Simple MZ rules (tag-based) are straightforward; dependency-based rules may need redesign

### Helper function changes
| Classic | SmartScape on Grail |
|---------|---------------------|
| `entityName()` | `getNodeName()` — type-agnostic |
| `entityAttribute()` | `getNodeField()` — type-agnostic |

### Deprecations in progress
- `dt.entity.*` fetch tables — deprecated, moving to `dt.smartscape.*`
- `dt.source_entity` dimension — moving to `dt.smartscape.source.id` in anomaly detectors
- Dynatrace already shows deprecation warnings on dashboards/alerts using old format
- Both formats work concurrently during transition — no hard removal date set

### DQL and pipeline behavior
- `dt.smartscape.*` dimensions only auto-populate via **OpenPipeline** — not classic pipeline
- If `dt.smartscape.*` dimensions are missing, check whether data flows through OpenPipeline
- SmartScape querying is **free** — no DPS or license cost

### `dt.traverse.history` array
- Ordered and append-only — each `traverse` pipe adds one entry
- Safe to index by position: `[0]` = first hop, `[1]` = second hop, etc.
- Contains: `id`, `edge_type`, `direction`, plus any `fieldsKeep` values

### Edge types confirmed by SME
`runs_on`, `calls`, `belongs_to`, `contains`, `is_part_of`, `is_attached_to`,
`monitors`, `uses`, `routes_to`, `bounces`

### OpenPipeline topology
- Logs, metrics, traces in OpenPipeline can have **node extraction** and **relationship extraction** processors
- This allows building fully custom SmartScape topology from any data source (SAP, mainframe, custom integrations)
- Extensions will increasingly ship with pre-built pipeline configs that do this automatically

### SmartScape App
- New app in Dynatrace — search "SmartScape", use the pink new app (not SmartScape Classic)
- Pre-built views: Explore, Infrastructure Overview, Problem Graph, Service Dependency Graph, cloud provider views
- Service Dependency Graph works with OneAgent, OTel, or hybrid — renders call chains the same way
- Segments control what appears in all views
- Custom SmartScape views (user-created documents) are planned but not yet available

### Multi-hop traversal
- A script (JavaScript) exists internally on Aeric's team that iterates traversals for upstream/downstream multi-hop analysis (e.g., tracing all callers of a mainframe)
- Reach out to account team/Aeric for this if a customer needs recursive traversal

### Kubernetes compliance use case
- `k8s.object` field on Kubernetes nodes contains the full YAML/JSON config (kubectl describe equivalent)
- Secret values are masked (not exposed) — only metadata like secret name and creation timestamp
- Can be used to query secret age, volume mounts, config maps, etc. across all clusters in Dynatrace

---

## What to Do When Updating This Repo

1. **Never add unvalidated DQL** without marking it `> **Note: Not validated against tenant**`
2. **Validate with:** `mcp__dynatrace-mcp__execute-dql` — empty results are fine, syntax validation only
3. **Check with SME facts above** before documenting behavior — if something conflicts, flag it
4. **Segments, not management zones** — never write management zones as a filtering mechanism for SmartScape on Grail content
5. **Entity naming:** use `FRONTEND` not `APPLICATION`, `PROCESS` not `process group instance`
6. **Always include the disclaimer** at the bottom of every public-facing doc
7. **Push only when explicitly told to push**
