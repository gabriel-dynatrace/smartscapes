# SmartScape on Grail — Developer Guide

A practical, easy-to-digest guide for querying topology data using Dynatrace's SmartScape DQL commands.

---

## Table of Contents

1. [What is SmartScape on Grail?](#what-is-smartscape-on-grail)
2. [Core Concepts](#core-concepts)
3. [The Three Commands](#the-three-commands)
   - [smartscapeNodes](#smartscapenodes)
   - [smartscapeEdges](#smartscapeedges)
   - [traverse](#traverse)
4. [Helper Functions](#helper-functions)
5. [Entity Views (`dt.entity.*`)](#entity-views-dtentity)
6. [Node Types Reference](#node-types-reference)
7. [Common Patterns & Recipes](#common-patterns--recipes)

---

## What is SmartScape on Grail?

SmartScape on Grail is Dynatrace's topology engine, accessible via DQL. It models your environment as a **graph**:

```
 ┌────────────┐     calls      ┌────────────────┐
 │  SERVICE   │ ─────────────► │    SERVICE     │
 │ (frontend) │                │   (backend)    │
 └────────────┘                └────────────────┘
       │ runs on                      │ runs on
       ▼                              ▼
 ┌────────────┐                ┌────────────────┐
 │    HOST    │                │      HOST      │
 └────────────┘                └────────────────┘
```

Each box is a **node** (entity). Each arrow is an **edge** (relationship).

You query nodes and edges directly in DQL using three commands:
- `smartscapeNodes` — fetch entities
- `smartscapeEdges` — fetch relationships between entities
- `traverse` — walk the graph from a starting set of nodes

---

## Core Concepts

| Term | Meaning |
|------|---------|
| **Node** | An entity (host, service, application, etc.) |
| **Edge** | A relationship between two entities (e.g. `CALLS`, `RUNS_ON`) |
| **Node type** | The entity type pattern, e.g. `dt.entity.host` |
| **Edge type** | The relationship type pattern, e.g. `SERVICE_CALLS_SERVICE` |
| **SmartScape ID** | The internal graph ID for an entity — different from classic entity IDs |

---

## Nodes vs Edges

Understanding the distinction between nodes and edges is key to working with SmartScape effectively.

### Nodes — What exists

A **node** represents a monitored entity: a host, service, application, process group, Kubernetes pod, etc. Each node has:

- A **type** — the kind of entity it is (e.g. `dt.entity.host`, `dt.entity.service`)
- An **ID** — a unique SmartScape identifier for that entity
- **Attributes** — properties like name, tags, management zones, OS type, etc.

Nodes are the "things" in your environment. When you query `smartscapeNodes`, you're asking: *"Give me all entities of this type."*

```
 ┌──────────────────────────────────────┐
 │  NODE: dt.entity.service             │
 │  id:   "SERVICE-00000ABC"            │
 │  name: "checkout-service"            │
 │  tags: ["production", "payments"]    │
 └──────────────────────────────────────┘
```

Nodes don't inherently tell you anything about how entities relate to each other — that's what edges are for.

---

### Edges — How things connect

An **edge** is a directed relationship between two nodes. It has:

- A **type** — what kind of relationship it is (e.g. `SERVICE_CALLS_SERVICE`, `PROCESS_GROUP_INSTANCE_RUNS_ON_HOST`)
- A **source** — the entity the relationship originates from
- A **target** — the entity the relationship points to

Edges are directional. `SERVICE_CALLS_SERVICE` goes from the calling service → to the called service. If you want to find *callers* of a service, you traverse backwards along that edge type.

```
 source_type: dt.entity.service          target_type: dt.entity.host
 source_id:   "SERVICE-00000ABC"         target_id:   "HOST-00000XYZ"
              │                                        ▲
              └──── edge_type: PROCESS_GROUP_INSTANCE_RUNS_ON_HOST ────┘
```

When you query `smartscapeEdges`, you're asking: *"Give me all relationships of this type across the entire environment."* This returns a flat table of source/target pairs — useful for mapping dependencies or counting connections, but you need to join with node data to get entity names/attributes.

---

### When to use which

| Goal | Use |
|------|-----|
| List entities of a type with their attributes | `smartscapeNodes` |
| Find all relationships of a given type | `smartscapeEdges` |
| Start from specific nodes and walk to connected nodes | `smartscapeNodes` + `traverse` |
| Count how many services call each other | `smartscapeEdges` + `summarize` |
| Get service names + the hosts they run on | `smartscapeNodes` + `traverse` |

The most powerful pattern is combining all three: use `smartscapeNodes` to define your starting set, `traverse` to walk edges to related entities, and `smartscapeEdges` when you need raw relationship data without a fixed starting point.

---

## The Three Commands

---

### `smartscapeNodes`

Loads entity nodes into the DQL pipeline. Think of it as `fetch dt.entity.*` but specifically for graph traversal.

#### Syntax

```dql
smartscapeNodes
  | nodeType: "dt.entity.<type>"
  | nodeType: "dt.entity.<type2>"
```

You can list multiple `nodeType` lines to load entities of several types at once.

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `nodeType` | Yes | Entity type pattern. Glob supported: `dt.entity.serv*` |

#### Fields returned

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | SmartScape entity ID |
| `type` | string | Entity type (e.g. `dt.entity.service`) |
| `name` | string | Display name |

Additional entity attributes are available depending on type.

#### Examples

**Load all hosts:**
```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| fields id, name, type
```

**Load all services and applications:**
```dql
smartscapeNodes
| nodeType: "dt.entity.service"
| nodeType: "dt.entity.application"
| fields id, name, type
```

**Use a glob to match multiple types:**
```dql
smartscapeNodes
| nodeType: "dt.entity.serv*"
| fields id, name
```

---

### `smartscapeEdges`

Loads topology edges — the relationships between entities. Each edge has a source and a target.

#### Syntax

```dql
smartscapeEdges
  | edgeType: "<EDGE_TYPE_PATTERN>"
```

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `edgeType` | Yes | Edge type pattern. Glob supported: `*CALLS*` |

#### Fields returned

| Field | Type | Description |
|-------|------|-------------|
| `edge_type` | string | The relationship type (e.g. `SERVICE_CALLS_SERVICE`) |
| `source_id` | string | SmartScape ID of the source entity |
| `source_type` | string | Entity type of the source |
| `target_id` | string | SmartScape ID of the target entity |
| `target_type` | string | Entity type of the target |

#### Examples

**All edges of any type:**
```dql
smartscapeEdges
| edgeType: "*"
| fields edge_type, source_id, target_id
```

**Only "runs on" relationships:**
```dql
smartscapeEdges
| edgeType: "*RUNS_ON*"
| fields edge_type, source_id, source_type, target_id, target_type
```

**Service-to-service call relationships:**
```dql
smartscapeEdges
| edgeType: "SERVICE_CALLS_SERVICE"
| fields source_id, target_id
```

**Count all relationship types:**
```dql
smartscapeEdges
| edgeType: "*"
| summarize count(), by: {edge_type}
| sort count desc
```

---

### `traverse`

Starts from a set of source nodes and follows edges to reach connected target nodes. This is how you walk the topology graph.

#### Syntax

```dql
smartscapeNodes
| nodeType: "dt.entity.<source_type>"
| traverse
    | edgeType: "<EDGE_TYPE>"
    | direction: "forward" | "backward" | "both"
    | maxDepth: <number>
```

`traverse` is **piped after** `smartscapeNodes` — it uses the nodes in the pipeline as the starting point.

#### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `edgeType` | Yes | — | Edge type(s) to follow. Glob supported. |
| `direction` | No | `"forward"` | `"forward"` follows edges away from source; `"backward"` follows edges toward source; `"both"` follows in either direction |
| `maxDepth` | No | `1` | How many hops to traverse |

#### Special field: `dt.traverse.history`

Every node result from `traverse` includes a `dt.traverse.history` field — an array showing the path of IDs taken to reach that node.

```
dt.traverse.history = ["HOST-AAA", "SERVICE-BBB", "SERVICE-CCC"]
                        └─ start       └─ hop 1        └─ current
```

This is useful for tracing call chains or dependency paths.

#### Examples

**From hosts, find all services running on them:**
```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| traverse
    | edgeType: "*RUNS_ON*"
    | direction: "backward"
| fields id, name, type, dt.traverse.history
```

**From a service, find what it calls (1 hop):**
```dql
smartscapeNodes
| nodeType: "dt.entity.service"
| filter name == "my-frontend-service"
| traverse
    | edgeType: "SERVICE_CALLS_SERVICE"
    | direction: "forward"
    | maxDepth: 1
| fields id, name, dt.traverse.history
```

**Multi-hop: from application → services → hosts (2 hops):**
```dql
smartscapeNodes
| nodeType: "dt.entity.application"
| traverse
    | edgeType: "*"
    | direction: "forward"
    | maxDepth: 2
| fields id, name, type, dt.traverse.history
```

**Find all services that call a specific service (reverse lookup):**
```dql
smartscapeNodes
| nodeType: "dt.entity.service"
| filter name == "payment-service"
| traverse
    | edgeType: "SERVICE_CALLS_SERVICE"
    | direction: "backward"
| fields id, name, dt.traverse.history
```

---

## Helper Functions

These DQL functions convert between classic Dynatrace entity IDs and SmartScape IDs.

---

### `toSmartscapeId()`

Converts a classic entity ID string to a SmartScape-compatible ID.

```dql
| fieldsAdd ss_id = toSmartscapeId("HOST-0000000000000123")
```

Use this when you have a classic entity ID (from an alert, API call, or variable) and need to use it in a SmartScape query.

**Example — start traverse from a known host ID:**
```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| filter id == toSmartscapeId("HOST-0000000000000123")
| traverse
    | edgeType: "*RUNS_ON*"
    | direction: "backward"
| fields id, name
```

---

### `classicEntitySelector()`

Converts a classic entity selector string into a list of SmartScape-compatible IDs. Useful for filtering nodes by tags, management zones, or other classic selector criteria.

```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| filter id in classicEntitySelector("type(HOST),tag(production)")
| traverse
    | edgeType: "*"
    | direction: "forward"
| fields id, name, type
```

**Selector examples:**
| Selector | Meaning |
|----------|---------|
| `type(HOST),tag(production)` | All production hosts |
| `type(SERVICE),mzName(my-zone)` | Services in management zone |
| `type(APPLICATION),entityName(checkout*)` | Apps matching name glob |

---

## Entity Views (`dt.entity.*`)

For simpler attribute lookups (not topology traversal), you can query entity tables directly using `fetch`:

```dql
fetch dt.entity.host
| fields entity.name, entity.detected_name, tags, managementZones
| limit 50
```

These views are indexed and fast for filtering/counting but don't include topology edges.

**Common entity views:**

| View | Entity type |
|------|-------------|
| `dt.entity.host` | Hosts |
| `dt.entity.service` | Services |
| `dt.entity.application` | Web applications |
| `dt.entity.mobile_application` | Mobile applications |
| `dt.entity.process_group` | Process groups |
| `dt.entity.process_group_instance` | Process group instances |
| `dt.entity.kubernetes_cluster` | Kubernetes clusters |
| `dt.entity.kubernetes_node` | Kubernetes nodes |
| `dt.entity.aws_credentials` | AWS credential entities |
| `dt.entity.azure_subscription` | Azure subscriptions |
| `dt.entity.gcp_zone` | GCP zones |

**Example — count hosts per management zone:**
```dql
fetch dt.entity.host
| fieldsAdd mz = managementZones[0]
| summarize count(), by: {mz}
| sort count desc
```

---

## Node Types Reference

### Core Infrastructure

| Node Type | Description |
|-----------|-------------|
| `dt.entity.host` | Physical or virtual hosts |
| `dt.entity.host_group` | Host groups |
| `dt.entity.process_group` | Process groups |
| `dt.entity.process_group_instance` | Individual processes |
| `dt.entity.service` | Services |
| `dt.entity.application` | Web applications |
| `dt.entity.mobile_application` | Mobile applications |
| `dt.entity.custom_application` | Custom applications |
| `dt.entity.synthetic_test` | Synthetic monitors |

### Kubernetes

| Node Type | Description |
|-----------|-------------|
| `dt.entity.kubernetes_cluster` | K8s clusters |
| `dt.entity.kubernetes_node` | K8s nodes |
| `dt.entity.cloud_application` | K8s workloads/deployments |
| `dt.entity.cloud_application_namespace` | K8s namespaces |
| `dt.entity.container_group` | Container groups (pods) |
| `dt.entity.container_group_instance` | Individual containers |

### Cloud

| Node Type | Description |
|-----------|-------------|
| `dt.entity.aws_credentials` | AWS credential/account entities |
| `dt.entity.azure_subscription` | Azure subscriptions |
| `dt.entity.gcp_zone` | GCP zones |

---

## Common Patterns & Recipes

---

### Map all services on a host

```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| filter name == "prod-web-01"
| traverse
    | edgeType: "*RUNS_ON*"
    | direction: "backward"
| filter type == "dt.entity.service"
| fields name, id
```

---

### Find orphaned services (no upstream callers)

```dql
// Get all services that ARE called
smartscapeEdges
| edgeType: "SERVICE_CALLS_SERVICE"
| fields target_id
| dedup target_id

// Then compare against all services
// (Use in a notebook or workflow with two steps)
```

---

### Topology map: application → services → hosts

```dql
smartscapeNodes
| nodeType: "dt.entity.application"
| traverse
    | edgeType: "*"
    | direction: "forward"
    | maxDepth: 3
| summarize count(), by: {type}
```

---

### Count relationship types in your environment

```dql
smartscapeEdges
| edgeType: "*"
| summarize edges = count(), by: {edge_type}
| sort edges desc
```

---

### Filter by management zone then traverse

```dql
smartscapeNodes
| nodeType: "dt.entity.host"
| filter id in classicEntitySelector("type(HOST),mzName(Production)")
| traverse
    | edgeType: "*RUNS_ON*"
    | direction: "backward"
| filter type == "dt.entity.service"
| fields name, id, dt.traverse.history
```

---

### Trace a call chain (multi-hop path)

```dql
smartscapeNodes
| nodeType: "dt.entity.service"
| filter name == "api-gateway"
| traverse
    | edgeType: "SERVICE_CALLS_SERVICE"
    | direction: "forward"
    | maxDepth: 5
| fields name, dt.traverse.history
| sort arraySize(dt.traverse.history) asc
```

The `dt.traverse.history` array length tells you how many hops away each service is.

---

### Find all Kubernetes services in a cluster

```dql
smartscapeNodes
| nodeType: "dt.entity.kubernetes_cluster"
| filter name == "my-cluster"
| traverse
    | edgeType: "*"
    | direction: "forward"
    | maxDepth: 4
| filter type == "dt.entity.service"
| fields name, id, dt.traverse.history
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    SmartScape DQL Cheatsheet                    │
├──────────────────────┬──────────────────────────────────────────┤
│ Load nodes           │ smartscapeNodes | nodeType: "dt.entity.X"│
│ Load edges           │ smartscapeEdges | edgeType: "PATTERN"    │
│ Walk graph           │ ... | traverse | edgeType: "..." |       │
│                      │   direction: "forward"/"backward"/"both" │
│                      │   maxDepth: N                            │
├──────────────────────┼──────────────────────────────────────────┤
│ Convert classic ID   │ toSmartscapeId("HOST-000...")            │
│ Filter by selector   │ classicEntitySelector("type(HOST),...")  │
├──────────────────────┼──────────────────────────────────────────┤
│ Path history         │ dt.traverse.history  → array of IDs      │
│ Simple entity fetch  │ fetch dt.entity.host                     │
└──────────────────────┴──────────────────────────────────────────┘
```

---

## Further Reading

- [SmartScape on Grail — Dynatrace Docs](https://docs.dynatrace.com/docs/platform/grail/smartscape-on-grail)
- [SmartScape DQL Commands Reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands/smartscape-commands)
- [DQL Language Reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)
- [Entity Selector Reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/functions/classicEntitySelector)
