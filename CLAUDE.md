# Akasha — Claude Context

This file gives Claude Code the context needed to work effectively in this
repository. Read it before making any changes.

---

## What Akasha is

Akasha is a self-hosted second brain. It captures documents, activity,
biometrics, and location into a Neo4j knowledge graph. It surfaces insights
proactively through agents, runs personal automations called Rituals, and
exposes the graph as context to Claude and other LLMs via MCP.

**Not a chatbot. A second brain.**

---

## Monorepo structure

```
akasha/
├── apps/
│   ├── api/                  NestJS REST API, SSE, chat streaming
│   ├── ingestion-worker/     Document pipeline consumer
│   ├── event-worker/         Normalized event → Neo4j consumer
│   ├── agent-worker/         Agents + Ritual execution engine
│   ├── aether/               Event collection + normalization layer
│   ├── dashboard/            React + Next.js workspace UI
│   ├── hub/                  React + Next.js admin panel
│   └── mcp/                  MCP server (@akasha/mcp)
├── libs/
│   ├── ai/                   Vercel AI SDK wrapper, extraction, embedding, chat
│   ├── connectors/           Connector interface, registry, base classes
│   ├── database/
│   │   ├── postgres/         Drizzle ORM, schema, migrations
│   │   ├── neo4j/            Neo4j driver, GraphService
│   │   ├── qdrant/           QdrantService, vector operations
│   │   ├── minio/            MinIO file storage
│   │   └── meilisearch/      Meilisearch full text search
│   ├── graph/                Neo4j schema, Zod node/rel schemas, migrator
│   ├── messaging/            RabbitMQ module, event contracts
│   ├── rituals/              Ritual engine, step executors, template service
│   ├── search/               Hybrid search (Meilisearch + Qdrant + Neo4j)
│   ├── tenant/               TenantContext (AsyncLocalStorage), guards
│   └── ui/                   Shared React component library
└── packages/
    └── connector-sdk/        @akasha/connector-sdk — public npm package
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Backend | NestJS (Nx monorepo) |
| Frontend | React · Next.js · shadcn/ui · Tailwind |
| Knowledge graph | Neo4j |
| Vector search | Qdrant |
| Full text search | Meilisearch |
| Operational data | PostgreSQL (Drizzle ORM) |
| File storage | MinIO |
| Event bus | RabbitMQ |
| AI | Vercel AI SDK (provider from config) |
| Validation | Zod throughout — no exceptions |
| Mobile | Flutter (watch + location) |

---

## Core architectural rules

### 1. Zod validates everything
Every node written to Neo4j, every relationship, every event payload, every
connector config, every AI extraction output — all validated by Zod before
any write happens. No raw type assertions on external data.

```typescript
// always validate before writing to Neo4j
const parsed = NodeSchemaRegistry[label].parse(data);
```

### 2. TenantContext flows through everything
WorkspaceId is never passed as a function parameter through service layers.
It lives in AsyncLocalStorage and is accessed via TenantContext.

```typescript
// correct
const workspaceId = TenantContext.getWorkspaceId();

// never do this
async function doThing(workspaceId: string) { ... }
```

Set TenantContext at the boundary (HTTP guard or RabbitMQ consumer).
Never set it inside a service.

### 3. NodeLabel and RelType are always constants
Never hardcode label or relationship type strings in queries.

```typescript
// correct
import { NodeLabel, RelType } from '@akasha/graph';
MATCH (n:${NodeLabel.Person} ...

// never
MATCH (n:Person ...
```

### 4. RabbitMQ is infrastructure — crash on disconnect
If RabbitMQ goes down the app crashes and Docker restarts it.
Do not add reconnection logic or silent failure handling.

### 5. LLM unavailability → Dead Letter Queue
LLMUnavailableError causes a nack (no requeue) → message goes to DLQ.
Document status → EXTRACTION_PENDING.
DLQ retry worker polls LLM health and requeues when available.
Data is never lost.

### 6. workspaceId on every node and query
Every Neo4j node has a workspaceId property.
Every Neo4j query filters by workspaceId.
Every Qdrant point has workspaceId in payload.
Every Meilisearch document has workspaceId as filterable attribute.
Every PostgreSQL table has workspace_id column.

### 7. Privacy filter on every graph read
```typescript
WHERE (n.private = false OR n.visibleTo CONTAINS $userId)
```
This filter is injected by Neo4jService — never skip it.

### 8. Schema migrations run on startup
GraphMigrator.migrate() runs before the app serves traffic.
Never modify Neo4j schema outside of a migration file.
Migration files are never deleted or modified after being applied.

---

## Node labels

```typescript
// libs/graph/src/schema/constants.ts
Person · Project · Concept · Task · Organization
Document · Chunk · Place
Signal         // normalized event from connectors
WatchSnapshot  // biometric snapshot
SleepSession
DayNode        // hub — everything on a given day connects through it
Insight        // agent-generated finding
Engram         // synthesized memory — highest order node, created by agents
User           // authenticated user — distinct from Person
```

## Relationship types

```typescript
WORKS_ON · MENTIONS · HAS_CHUNK · OCCURRED_AT · RELATES_TO
PART_OF · AUTHORED_BY · OCCURRED_DURING · FOLLOWED_BY
IDENTIFIED_AS   // links (:User) to (:Person)
GENERATED_BY    // links (:Insight) to agent
REFERENCES      // links (:Document) or (:Chunk) to any node
TRIGGERED_BY    // links (:Insight) to (:Signal) or (:WatchSnapshot)
NEXT            // links (:Chunk) to next (:Chunk) in document
```

---

## Event routing (RabbitMQ)

```
Exchange: akasha.topic (topic)

Routing key                      Queue
─────────────────────────────────────────────────────
document.ingestion.requested     q.ingestion
document.ingestion.completed     q.agent
document.ingestion.failed        q.dlx
event.normalized.#               q.events
agent.action.#                   q.agent
ritual.trigger.#                 q.agent
graph.updated.#                  q.sync        (connectors watching for export)
```

Dead letter exchange: `akasha.dlx`
Dead letter queues: `q.dlq.ingestion` · `q.dlq.events`

---

## AI configuration

Provider is set in `.env` — never hardcoded.

```typescript
// libs/ai/src/ai.module.ts
// provider resolved from AI_PROVIDER env var
// ollama | openai | anthropic | azure | groq

// models are per-task
AI_MODEL_EXTRACTION   // entity/rel extraction from chunks and events
AI_MODEL_EMBEDDING    // chunk embedding
AI_MODEL_CHAT         // dashboard chat interface
AI_MODEL_REASONING    // agent reasoning (optional, use stronger model)
```

Default provider is Ollama (local, no API key required).

Use `generateObject` with a Zod schema for all structured AI output.
Use `streamText` for chat. Never use `generateText` for extraction.

---

## Connector architecture

Every connector implements `IConnector` and has a `ConnectorManifest`.

```
Connector types:
  webhook    → platform exposes HTTP endpoint, source pushes
  poller     → connector fetches on a schedule
  device     → data pushed from local device (watch, browser extension)
  stream     → persistent connection
  import     → bulk one-time or periodic import (Obsidian, Notion)
  sync       → bidirectional — import + export + live sync
```

Connectors have two sides:
- **Inbound** — normalize raw data → publish NormalizedEvent to RabbitMQ
- **Outbound** — execute actions (used by Rituals and sync connectors)

All connector config is validated by the connector's own Zod schema before
being stored in PostgreSQL.

---

## Ritual engine

Rituals are personal automations aware of graph context.

```
Trigger types:  event | graph | schedule | manual | pattern
Step types:     ai | graph | connector | human | condition | loop | wait
```

Key rules:
- All step params are Handlebars templates evaluated against WorkflowContext
- Human steps pause execution — status → WAITING
- All Ritual runs are tracked in PostgreSQL with full step traces
- Rituals subscribe to graph.updated.# for graph triggers
- APS policy is checked before any connector outbound action

---

## Search

Never query just one engine. Always use SearchService for user-facing queries.

```typescript
// libs/search/src/search.service.ts
// combines Meilisearch + Qdrant + Neo4j via Reciprocal Rank Fusion
const results = await this.search.search(query, { limit: 20 });
```

Use individual services only for internal operations (indexing, graph writes).

---

## Import / Export connectors

### Obsidian
- Parse `.md` files + frontmatter + wikilinks `[[]]`
- Wikilinks become graph relationships — high-value signal
- Export: generate living markdown files from graph nodes
- Sync: subscribe to `graph.updated.#`, regenerate affected files

### Notion
- Use Notion API (not HTML export — too lossy)
- Databases with relations → graph nodes with relationships
- Export: push graph nodes back as Notion database rows
- Sync: bidirectional via Notion API webhooks + graph.updated.#

---

## Database responsibilities

```
PostgreSQL    operational state — workspaces, users, documents (job state),
              chunks (cross-store index), ingestion_events (audit),
              agent_configs, connector_configs, ritual_runs, system_config

Neo4j         knowledge graph — all nodes and relationships

Qdrant        vector search — chunk embeddings, workspace-scoped by payload

Meilisearch   full text search — documents, signals, people, projects,
              insights — workspace-scoped by filter

MinIO         raw files — prefixed by workspaceId/documentId
```

---

## Multi-tenancy

Instance scale: 1–10 users per self-hosted instance (household / small team).
Not SaaS scale. No billing. No usage metering.

Isolation strategy:
- PostgreSQL: workspace_id on every table
- Neo4j: workspaceId property on every node, filtered on every query
- Qdrant: workspaceId in payload, filtered on every search
- Meilisearch: workspaceId as filterable attribute, filtered on every search
- MinIO: workspaceId as path prefix

Biometric and location data is additionally scoped to userId.

---

## Auth

Auth is not yet implemented. The platform assumes a single authenticated user
for now. Auth will be added before the open source launch (Phase 8).

Do not add auth-related code yet. Do not hardcode user IDs.
Use TenantContext.getWorkspaceId() — the workspace ID will be set by the
auth layer when it's added.

---

## Setup wizard

On first run (no workspace in system_config), all routes redirect to /setup.

```
Step 1: create workspace + owner account
Step 2: configure LLM provider + test connection
Step 3: enable connectors
Step 4: import existing knowledge (Obsidian / Notion — optional)
Step 5: redirect to /today
```

---

## Dashboard views

```
/today          primary view — readiness, focus, projects, open loops,
                recent signals, agent insights
/chat           power tool for graph exploration
/documents      document library, upload, ingestion status
/graph          visual Neo4j graph explorer (Cytoscape.js)
/insights       agent findings, patterns, correlations
/rituals        ritual list, builder, run history
/connectors     enable and configure connectors
/agents         active agents, logs, configuration
/settings       workspace config, LLM config, retention policy
```

---

## Design system

```
Background:   #0C0B09 (void) → #211F18 (overlay)
Ink:          #2A2820 (ghost) → #F5F0E8 (pure)
Accent:       #C4942A (amber base) — primary accent throughout

Fonts:
  Display:    Cormorant Garamond — headings, brand moments
  Body:       Lora — long-form reading, document content
  UI:         DM Sans — chrome, labels, navigation
  Mono:       DM Mono — code, timestamps, metadata

Graph node colors:
  Person        #4A7C6F
  Project       #C4942A
  Concept       #6B5EA8
  Document      #4A6E8A
  Signal        #3D6B4A
  Task          #8A4A3D
  Place         #7A6B3D
  Insight       #C4942A
  Engram        #A0784A
  WatchSnapshot #4A5C8A
```

---

## Naming conventions

```
Apps          kebab-case directories, @akasha/app-name
Libs          kebab-case directories, @akasha/lib-name
NestJS        PascalCase modules, services, controllers
              camelCase methods and properties
Zod schemas   PascalCase + Schema suffix — PersonNodeSchema
Types         PascalCase — PersonNode, NormalizedEvent
Constants     SCREAMING_SNAKE for true constants — SCHEMA_VERSION
              object const for enums — NodeLabel.Person
Events        dot.separated routing keys — document.ingestion.requested
Env vars      SCREAMING_SNAKE — AI_PROVIDER, RABBITMQ_URL
```

---

## Key commands

```bash
# development
nx serve api                    # start API
nx serve dashboard              # start dashboard
docker compose up               # start all infrastructure

# generation
nx g @nx/nest:module [name] --project=[app]
nx g @nx/nest:service [name] --project=[app]
nx g @nx/react:component [name] --project=ui

# testing
nx test [project]
nx affected:test

# linting
nx lint [project]
nx affected:lint

# build
nx build [project]
nx affected:build
```

---

## What not to do

- Never write raw Cypher strings outside of `Neo4jService` or migration files
- Never use `any` in TypeScript without a comment explaining why
- Never skip Zod validation on data entering the system
- Never hardcode workspaceId, userId, or provider strings
- Never add reconnection logic for RabbitMQ
- Never write to Neo4j without checking privacy filter
- Never create a new node label or relationship type without a migration file
- Never call an LLM directly — always go through `@akasha/ai` services
- Never query a single search engine for user-facing search — use SearchService
- Never send a connector outbound action without APS policy check (when APS is implemented)

---

## Current phase

**Phase 0 — Foundation**

Setting up the Nx monorepo, shared libs, docker-compose, and health checks.
Nothing user-facing yet. Focus on getting the skeleton right before building features.

See the roadmap in README.md for the full phase plan.